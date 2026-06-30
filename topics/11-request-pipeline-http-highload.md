# 11. Конвейер обработки запросов и HTTP в Highload

> Статус: 🟡 в работе · Уровень: Senior

## Что проверяют
Глубокое понимание пути запроса от сокета до хендлера, серверов Kestrel, накладных расходов сериализации JSON и оптимизации под высокую нагрузку.

## Ключевые вопросы (чеклист)

### Конвейер обработки запроса
- [x] Полный путь запроса: Kestrel → сервер → middleware pipeline → routing → endpoint
- [x] Как устроен middleware pipeline (делегаты `RequestDelegate`, паттерн Chain of Responsibility)
- [x] Порядок middleware и почему он критичен (exception → https → static → routing → auth → endpoints)
- [x] Kestrel: модель потоков, async IO, connection multiplexing
- [x] Reverse proxy (Nginx/YARP/IIS) перед Kestrel, зачем
- [x] `HttpContext`, `Request`/`Response`, потоки тела (body streams)
- [x] Backpressure, request limits, `MaxConcurrentConnections`, timeouts

### HTTP и протоколы
- [x] HTTP/1.1 vs HTTP/2 vs HTTP/3 (QUIC), мультиплексирование, head-of-line blocking
- [x] Keep-alive, connection pooling в `HttpClient`, проблема сокетов (socket exhaustion)
- [x] `HttpClientFactory` и почему нельзя создавать `HttpClient` в `using`
- [x] Chunked transfer, streaming-ответы, `IAsyncEnumerable` в API

### JSON и сериализация под нагрузкой
- [x] `System.Text.Json` vs `Newtonsoft.Json` — производительность и аллокации
- [x] Source-generated JSON serializer (`JsonSerializerContext`) — почему быстрее
- [x] Аллокации при (де)сериализации, `Utf8JsonReader`/`Writer`, `Span`/`Memory`
- [x] Стриминговая сериализация больших ответов без буферизации в память
- [x] Сжатие ответов (gzip/brotli), trade-off CPU vs bandwidth
- [x] Снижение размера payload: проекции, пагинация, частичные ответы

### Highload-практики
- [x] Минимизация аллокаций на горячем пути (object pooling, `ArrayPool`, `RecyclableMemoryStream`)
- [x] Кэширование (output caching, response caching, ETag) — см. тему 17
- [x] Профилирование нагрузки: bottlenecks, нагрузочное тестирование (k6, NBomber)
- [x] Thread pool starvation при синхронном коде в async-конвейере
- [x] Rate limiting (встроенный middleware в .NET 7+)

## Разбор вопросов

> ⭐ — вопросы, которые реально задавали на собеседовании.

<a id="q-http-pipeline"></a>
### ⭐ Вопрос: Подробно опиши, как работает HTTP-конвейер обработки запроса в .NET Core
**Краткий ответ:**

Полный путь запроса:

1. **Сетевой слой / сервер (Kestrel).** Kestrel слушает сокет, принимает TCP-соединение, разбирает HTTP (парсит стартовую строку, заголовки, тело). Работает поверх `libuv`/сокетов с асинхронным IO и `System.IO.Pipelines` (минимум аллокаций при чтении байтов из сокета).
2. **Формирование `HttpContext`.** Сервер создаёт `HttpContext` (Request, Response, Connection, User, Features, RequestServices — DI-scope запроса).
3. **Middleware pipeline.** `HttpContext` проходит по цепочке middleware — это вложенные делегаты `RequestDelegate` (`Task Invoke(HttpContext)`). Каждое звено решает: обработать и/или вызвать `next(context)` дальше, и что сделать на «обратном пути» (после `await next`).
4. **Routing middleware** сопоставляет URL+метод с endpoint'ом (выбирает, какой обработчик выполнить), но не выполняет его сразу.
5. **Auth middleware** (после routing) — аутентификация/авторизация уже зная выбранный endpoint и его метаданные (`[Authorize]`).
6. **Endpoint middleware** (терминальное) — выполняет выбранный endpoint: для MVC — фильтры → model binding → action → result; для Minimal API — делегат.
7. **Формирование ответа** идёт «назад» по pipeline: сериализация результата, установка статуса/заголовков, запись тела в `Response.Body`, отправка через Kestrel.

Ключевая идея — **Chain of Responsibility**: pipeline строится при старте из зарегистрированных `Use...`, оборачивая каждое следующее звено.

**Пример (как middleware оборачивают друг друга):**
```csharp
var app = builder.Build();

app.UseExceptionHandler("/error"); // 1. ловит исключения из всего, что ниже
app.UseHttpsRedirection();         // 2.
app.UseStaticFiles();              // 3. может завершить запрос (terminal) для статики
app.UseRouting();                  // 4. выбирает endpoint
app.UseAuthentication();           // 5. кто ты
app.UseAuthorization();            // 6. можно ли тебе (нужны метаданные endpoint из routing)
app.MapControllers();              // 7. выполнение endpoint

// Самописное middleware = звено цепочки:
app.Use(async (context, next) =>
{
    var sw = Stopwatch.StartNew();
    await next(context);            // вызвать следующее звено
    logger.LogInformation("{Path} {Status} {Ms}ms",
        context.Request.Path, context.Response.StatusCode, sw.ElapsedMilliseconds);
});
```

**Подводные камни:**
- **Порядок критичен**: `UseAuthorization` обязан идти после `UseRouting` и `UseAuthentication`, иначе авторизация не увидит endpoint/пользователя.
- После начала записи тела (`Response` started) уже **нельзя менять заголовки/статус** — отсюда баги «headers already sent».
- Терминальное middleware (например static files) может «закоротить» pipeline — нижние звенья не выполнятся.
- Тяжёлая синхронная работа в middleware блокирует поток пула (см. thread pool starvation ниже).

---

<a id="q-span-memory-http"></a>
### ⭐ Вопрос: Где применяются `Span<T>`/`Memory<T>` при работе с HTTP и зачем
**Краткий ответ:**

Главная цель — **обрабатывать байты запроса/ответа без лишних аллокаций и копирований**, что критично под нагрузкой (меньше мусора → меньше пауз GC → выше throughput).

Где используются:
- **Чтение тела/сокета.** Kestrel читает из сокета через `System.IO.Pipelines` (`PipeReader`), отдавая данные как `ReadOnlySequence<byte>`/`ReadOnlySpan<byte>` — без копирования в промежуточные массивы.
- **Парсинг JSON.** `System.Text.Json` работает поверх `ReadOnlySpan<byte>`/`Utf8JsonReader` напрямую по UTF-8 байтам тела — без декодирования всей строки в `string` и без промежуточных объектов.
- **Парсинг заголовков/строк/чисел.** `ReadOnlySpan<char>`/`Utf8Parser` позволяют разобрать значения (числа, даты, токены) без `Substring` (который аллоцирует новые строки).
- **Запись ответа.** `IBufferWriter<byte>`/`Utf8JsonWriter` пишут результат прямо в буфер ответа.
- **`Memory<T>`** используется там, где буфер нужно протащить через `async` (методы `Stream.ReadAsync(Memory<byte>)`), потому что `Span` нельзя держать через `await` (он `ref struct`, см. [тему 01](./01-csharp-language.md)).

**Почему именно так быстрее:** обычный путь «байты → string → объект» создаёт промежуточные строки и объекты (аллокации, нагрузка на GC). `Span`/`Memory` дают «окно» в уже существующий буфер — ноль копий и аллокаций на горячем пути.

**Пример:**
```csharp
// Разбор числа из части тела без аллокации строки
ReadOnlySpan<byte> body = GetBodyBytes();          // окно в буфер Kestrel
if (Utf8Parser.TryParse(body, out int value, out _)) { /* ... */ }

// Async-чтение в арендованный буфер (тут именно Memory, не Span)
byte[] rented = ArrayPool<byte>.Shared.Rent(8192);
try
{
    int read = await stream.ReadAsync(rented.AsMemory(0, 8192), ct);
    Process(rented.AsSpan(0, read));               // дальше синхронно как Span
}
finally { ArrayPool<byte>.Shared.Return(rented); }
```

**Подводные камни:**
- `Span<T>` нельзя сохранять в поле, использовать через `await`, в лямбдах/итераторах — только синхронные участки на стеке. Для async — `Memory<T>`, а `Span` брать из `.Span` уже после await.
- Span ссылается на чужой буфер — нельзя использовать после того, как буфер освобождён/возвращён в пул (use-after-free на уровне логики).
- Ручная работа со Span сложнее и опаснее — применять на реально горячих путях, измерив выгоду; для обычного кода `System.Text.Json` и так использует Span внутри.

---

### Вопрос: Порядок middleware — почему критичен
**Краткий ответ:** Pipeline исполняется в порядке регистрации (и в обратном — на пути ответа). Типовой корректный порядок: ExceptionHandler → HSTS/HttpsRedirection → StaticFiles → Routing → CORS → Authentication → Authorization → (RateLimiter) → Endpoints. Нарушение ломает функциональность: авторизация без routing не знает endpoint; CORS после endpoint не добавит заголовки; exception handler не первым — не поймает ошибки верхних звеньев.

---

### Вопрос: Kestrel — модель потоков, async IO
**Краткий ответ:** Kestrel — кроссплатформенный высокопроизводительный веб-сервер по умолчанию. Использует асинхронный IO и `System.IO.Pipelines` для эффективного чтения/записи сокета с минимумом аллокаций. Не «поток на запрос» — небольшое число потоков обслуживает множество соединений через async. Поддерживает HTTP/1.1, HTTP/2, HTTP/3.

**Подводные камни:** блокирующий (sync) код в обработчике занимает поток пула и снижает пропускную способность Kestrel.

---

### Вопрос: Reverse proxy перед Kestrel — зачем
**Краткий ответ:** Nginx/YARP/IIS/Envoy перед Kestrel дают: TLS-терминацию, балансировку, защиту от медленных клиентов (slowloris), буферизацию, статику, rate limiting на краю, единую точку логирования. Kestrel может работать и напрямую (edge), но за прокси — частая прод-конфигурация. Важно прокидывать `X-Forwarded-*` (ForwardedHeaders middleware).

---

### Вопрос: `HttpContext`, потоки тела
**Краткий ответ:** `HttpContext` — состояние запроса: `Request` (метод, путь, заголовки, `Body` как `Stream`/`BodyReader` как `PipeReader`), `Response` (`Body`/`BodyWriter`), `User`, `RequestServices` (DI-scope), `Features`. Тело — поток: читать/писать стримингово, не буферизуя всё в память для больших данных.

**Подводные камни:** `HttpContext` не потокобезопасен и живёт только в рамках запроса — нельзя захватывать в фоновую задачу (используйте копии нужных данных).

---

### Вопрос: Backpressure, лимиты, таймауты
**Краткий ответ:** Kestrel имеет лимиты: `MaxConcurrentConnections`, `MaxRequestBodySize`, `KeepAliveTimeout`, `RequestHeadersTimeout`, минимальная скорость данных (защита от slow clients). Pipelines обеспечивают backpressure (производитель притормаживается, если потребитель не успевает). Это защищает от перегрузки/DoS.

---

### Вопрос: HTTP/1.1 vs HTTP/2 vs HTTP/3
**Краткий ответ:**
- **HTTP/1.1** — одно «активное» сообщение на соединение; конкурентность через много TCP-соединений; head-of-line blocking на уровне соединения.
- **HTTP/2** — мультиплексирование множества потоков (streams) в одном TCP-соединении, бинарные фреймы, сжатие заголовков (HPACK). Но HOL-blocking остаётся на уровне TCP (потеря пакета тормозит все streams).
- **HTTP/3** — поверх **QUIC (UDP)**; устраняет TCP HOL-blocking (потоки независимы), быстрее установка соединения (0-RTT). Хорош для нестабильных сетей/мобильных.

---

### Вопрос: Keep-alive, пулинг, socket exhaustion, `HttpClientFactory`
**Краткий ответ:**
- **Socket exhaustion**: создание нового `HttpClient` в `using` на каждый запрос оставляет TCP-сокеты в состоянии `TIME_WAIT` — порты исчерпываются под нагрузкой.
- **Нельзя** оборачивать `HttpClient` в `using` по этой причине; и статический singleton `HttpClient` не подхватывает изменения DNS.
- **`IHttpClientFactory`** решает обе проблемы: управляет пулом переиспользуемых `HttpMessageHandler` с ротацией (handler lifetime, по умолчанию ~2 мин) → переиспользование соединений + обновление DNS. Плюс интеграция с Polly/resilience.

```csharp
builder.Services.AddHttpClient<IApiClient, ApiClient>(c => c.BaseAddress = new(url))
    .AddStandardResilienceHandler();
```

---

### Вопрос: Chunked transfer, streaming, `IAsyncEnumerable`
**Краткий ответ:** Для больших/потоковых ответов не буферизуйте всё в память — стримьте. `IAsyncEnumerable<T>` из action сериализуется по мере поступления (chunked transfer encoding), снижая память и time-to-first-byte. Подходит для экспорта, больших списков.

---

### Вопрос: `System.Text.Json` vs `Newtonsoft.Json`
**Краткий ответ:**
- **`System.Text.Json` (STJ)** — встроенный, работает по UTF-8 байтам через Span/`Utf8JsonReader`, существенно быстрее и экономнее по памяти. По умолчанию в ASP.NET Core.
- **`Newtonsoft.Json`** — богаче по фичам/гибкости (исторически), но медленнее и больше аллокаций.
- На горячем пути выбирают STJ; Newtonsoft — когда нужны специфичные фичи/совместимость.

---

### Вопрос: Source-generated JSON — почему быстрее
**Краткий ответ:** `JsonSerializerContext` (source generator) генерирует код (де)сериализации **на этапе компиляции** вместо рантайм-рефлексии → быстрее старт, меньше аллокаций, совместимость с Native AOT/trimming.

```csharp
[JsonSerializable(typeof(WeatherForecast))]
internal partial class AppJsonContext : JsonSerializerContext { }
// builder.Services.ConfigureHttpJsonOptions(o =>
//     o.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default));
```

---

### Вопрос: Аллокации при (де)сериализации, `Utf8JsonReader`/`Writer`
**Краткий ответ:** Сериализация прямо в/из UTF-8 байтов (минуя промежуточные `string`) экономит память. `Utf8JsonReader` (ref struct поверх Span) и `Utf8JsonWriter` (поверх `IBufferWriter<byte>`) — низкоуровневый высокопроизводительный путь для кастомных конвертеров/горячих сценариев.

---

### Вопрос: Стриминговая сериализация больших ответов
**Краткий ответ:** Пишите JSON прямо в `Response.BodyWriter`/поток, не собирая весь объект в строку/`byte[]`. Для больших коллекций — `IAsyncEnumerable`. Это снижает пиковую память и LOH-аллокации.

---

### Вопрос: Сжатие ответов
**Краткий ответ:** gzip/brotli уменьшают объём по сети ценой CPU. Brotli — лучше сжатие, дороже CPU. Часто сжатие делают на reverse proxy/CDN. Не сжимать уже сжатое (изображения) и осторожно со сжатием в HTTPS (исторические атаки типа BREACH для чувствительных данных).

---

### Вопрос: Снижение размера payload
**Краткий ответ:** Проекции (отдавать только нужные поля), пагинация, сжатие, частичные ответы (sparse fieldsets), правильные типы. Меньше payload → меньше сериализации, сети и аллокаций.

---

### Вопрос: Минимизация аллокаций на горячем пути
**Краткий ответ:** `ArrayPool<T>`/`MemoryPool<T>`, `RecyclableMemoryStream` (вместо `MemoryStream`), object pooling (`ObjectPool<T>`), `Span`/`Memory`, `ValueTask`, избегание боксинга и лишних LINQ-аллокаций. Цель — реже запускать GC (см. темы 02, 20).

---

### Вопрос: Кэширование на уровне HTTP
**Краткий ответ:** Output caching (.NET 7+), response caching, ETag/`If-None-Match` (304), `Cache-Control`. Снимает нагрузку с приложения/БД. Подробно — [тема 17](./17-caching.md).

---

### Вопрос: Нагрузочное тестирование и поиск bottleneck
**Краткий ответ:** k6, NBomber, wrk, bombardier — генерируют нагрузку и измеряют latency (p50/p95/p99) и throughput. Bottleneck ищут профилированием (dotnet-trace/counters, см. тему 20): CPU, аллокации, lock contention, IO к БД, thread pool. «Сначала измерь».

---

### Вопрос: Thread pool starvation
**Краткий ответ:** Если в async-конвейере есть блокирующий код (`.Result`/`.Wait()`/sync IO), потоки пула заняты ожиданием, новых не хватает, latency растёт лавинообразно. Лечение: async all the way (см. тему 03), не блокировать, выносить тяжёлый CPU аккуратно.

---

### Вопрос: Rate limiting
**Краткий ответ:** Встроенный middleware (.NET 7+): алгоритмы Fixed Window, Sliding Window, Token Bucket, Concurrency. Защищает от перегрузки/abuse, сглаживает пики.

```csharp
builder.Services.AddRateLimiter(o => o.AddFixedWindowLimiter("api", opt =>
{
    opt.PermitLimit = 100; opt.Window = TimeSpan.FromSeconds(1);
}));
app.UseRateLimiter();
```

---

## Полезные ссылки
- Middleware: https://learn.microsoft.com/aspnet/core/fundamentals/middleware/
- Kestrel: https://learn.microsoft.com/aspnet/core/fundamentals/servers/kestrel
- System.Text.Json + source gen: https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/source-generation
- Pipelines: https://learn.microsoft.com/dotnet/standard/io/pipelines