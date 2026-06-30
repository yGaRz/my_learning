# 03. Многопоточность и асинхронность

> Статус: 🟡 в работе · Уровень: Senior

## Что проверяют
Понимание разницы потоков и async/await, синхронизации, типичных дедлоков и гонок, паттернов параллельной обработки.

## Ключевые вопросы (чеклист)
- [x] `Thread` vs `ThreadPool` vs `Task` vs `async/await`
- [x] Как работает `async/await` под капотом (стейт-машина, continuation)
- [x] `SynchronizationContext`, `ConfigureAwait(false)` — зачем и когда
- [x] Дедлок при `.Result`/`.Wait()` на UI/ASP.NET (классика)
- [x] `Task` vs `ValueTask` — когда использовать ValueTask
- [x] CPU-bound vs IO-bound, `Task.Run` — когда оправдан
- [x] Отмена: `CancellationToken`, `CancellationTokenSource`, таймауты
- [x] Примитивы синхронизации: `lock`/`Monitor`, `SemaphoreSlim`, `Mutex`, `ReaderWriterLockSlim`
- [x] `Interlocked`, атомарность, `volatile`, memory barriers
- [x] Гонки данных (race conditions), как обнаружить и избежать
- [x] `async void` — почему опасен, где допустим
- [x] Обработка исключений в async, `AggregateException`, `Task.WhenAll`
- [x] `Parallel.For/ForEach`, PLINQ, степень параллелизма
- [x] `IAsyncEnumerable`, `await foreach`, async-стримы
- [x] `Channel<T>`, producer/consumer, Dataflow (TPL)
- [x] Потокобезопасные коллекции, `ConcurrentDictionary` нюансы
- [x] Async от начала до конца (async all the way)

## Разбор вопросов

### Вопрос: `Thread` vs `ThreadPool` vs `Task` vs `async/await`
**Краткий ответ:**
- **`Thread`** — низкоуровневый ОС-поток (~1 МБ стека). Создание дорого. Нужен редко (долгоживущий выделенный поток, специфичный приоритет).
- **`ThreadPool`** — пул переиспользуемых потоков; экономит создание. Сюда планируются короткие задачи.
- **`Task`** — абстракция над «единицей работы», обычно исполняется на пуле; даёт композицию, продолжения, отмену, обработку ошибок.
- **`async/await`** — синтаксис для неблокирующего ожидания `Task`/`ValueTask`. Главное: **async ≠ многопоточность**. IO-bound async вообще не занимает поток во время ожидания.

**Подводные камни:** `Task.Run` для IO-bound — антипаттерн (занимает поток пула вместо освобождения).

---

### Вопрос: Как работает `async/await` под капотом
**Краткий ответ:**
- Компилятор превращает async-метод в **стейт-машину** (struct, реализующий `IAsyncStateMachine`). Каждый `await` — точка, где метод может «приостановиться».
- При `await` на незавершённой задаче: регистрируется **continuation** (продолжение), управление возвращается вызывающему, поток освобождается. Когда задача завершится — continuation планируется на исполнение (на captured context или пуле).
- Если задача уже завершена — выполнение продолжается синхронно без переключений.

**Подводные камни:**
- Локальные переменные «поднимаются» в поле стейт-машины (в куче, если задача асинхронна) → `ref struct` (`Span`) нельзя держать через `await`.
- Лишние async-обёртки добавляют аллокации; для горячего пути — `ValueTask` и синхронные fast-path.

---

### Вопрос: `SynchronizationContext` и `ConfigureAwait(false)`
**Краткий ответ:**
- `SynchronizationContext` определяет, **куда** вернётся continuation после `await`. В UI (WinForms/WPF) — обратно в UI-поток; в классическом ASP.NET — в контекст запроса. В **ASP.NET Core контекста синхронизации нет**.
- `ConfigureAwait(false)` говорит: «не возвращай меня в captured context, продолжи на пуле». Это:
  - предотвращает дедлоки (см. ниже);
  - снижает накладные расходы переключения.
- В библиотечном коде — **всегда** `ConfigureAwait(false)`. В приложении ASP.NET Core это не обязательно (контекста нет), в UI — осторожно (после await может понадобиться UI-поток).

---

### Вопрос: Дедлок при `.Result`/`.Wait()` (классика)
**Краткий ответ:**
- В среде с `SynchronizationContext` (UI, классический ASP.NET): поток блокируется на `.Result`, ожидая задачу. Задаче для завершения нужно вернуть continuation в **тот же** (уже заблокированный) поток → взаимная блокировка.
- Правила: **async all the way**, не смешивать блокирующие вызовы с async; если совсем нужно — `ConfigureAwait(false)` в библиотеке разрывает цикл.

**Пример (как НЕ делать):**
```csharp
// UI/ASP.NET classic -> дедлок
public ActionResult Get() => Json(GetDataAsync().Result);
// Правильно:
public async Task<ActionResult> Get() => Json(await GetDataAsync());
```

---

### Вопрос: `Task` vs `ValueTask`
**Краткий ответ:**
- `Task` — ссылочный тип, аллокация на каждый вызов; универсален, можно ждать многократно, `WhenAll` и т.п.
- `ValueTask`/`ValueTask<T>` — struct, **избегает аллокации**, когда результат часто доступен синхронно (кэш-хит, буфер уже заполнен). Хорош на горячем пути высоконагруженных API.
- Ограничения `ValueTask`: нельзя ждать дважды, нельзя `.Result` до завершения, нельзя хранить/параллельно ждать. Если нужно — `.AsTask()`.

**Подводные камни:** использовать `ValueTask` без необходимости усложняет код; применяйте, когда измеренная аллокация Task — узкое место.

---

### Вопрос: CPU-bound vs IO-bound, когда `Task.Run`
**Краткий ответ:**
- **IO-bound** (запрос к БД/сети/диску): используйте нативный async API (`await ...Async()`); поток не занимается во время ожидания. `Task.Run` тут вреден.
- **CPU-bound** (тяжёлые вычисления): оффлоад на пул через `Task.Run`, чтобы не блокировать текущий (особенно UI) поток. В ASP.NET Core оффлоадить CPU-работу обычно не нужно (она и так на пуле), `Task.Run` лишь снижает доступность потоков.

---

### Вопрос: Отмена — `CancellationToken`
**Краткий ответ:**
- Кооперативная отмена: `CancellationTokenSource` создаёт токен, передаётся вниз по цепочке; методы периодически проверяют `token.ThrowIfCancellationRequested()` или передают токен в async-API.
- Таймауты: `new CancellationTokenSource(TimeSpan.FromSeconds(5))`; объединение токенов — `CreateLinkedTokenSource`.

**Пример:**
```csharp
public async Task<Data> GetAsync(CancellationToken ct)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
    cts.CancelAfter(TimeSpan.FromSeconds(3));
    return await _client.GetAsync(url, cts.Token);
}
```

**Подводные камни:** проглатывание `OperationCanceledException`; не прокинутый дальше токен делает отмену бесполезной.

---

### Вопрос: Примитивы синхронизации
**Краткий ответ:**
- **`lock` (Monitor)** — взаимное исключение в пределах процесса; нельзя `await` внутри `lock`. С .NET 9 есть тип `Lock`.
- **`SemaphoreSlim`** — ограничение числа одновременных входов; имеет `WaitAsync` (можно в async). Замена lock для async-кода.
- **`Mutex`** — межпроцессная синхронизация (named mutex), дороже.
- **`ReaderWriterLockSlim`** — много читателей / один писатель.
- **`Interlocked`** — атомарные операции без блокировок.

**Пример (async-safe lock):**
```csharp
private readonly SemaphoreSlim _gate = new(1, 1);
public async Task DoAsync()
{
    await _gate.WaitAsync();
    try { /* критическая секция с await */ }
    finally { _gate.Release(); }
}
```

**Подводные камни:** `lock` на `this`/`typeof`/строке-литерале — опасно; используйте приватный `object`/`Lock`.

---

### Вопрос: `Interlocked`, `volatile`, memory barriers
**Краткий ответ:**
- **`Interlocked`** — атомарные `Increment`/`Add`/`Exchange`/`CompareExchange`; основа lock-free алгоритмов.
- **`volatile`** — запрещает кэширование переменной и переупорядочивание чтений/записей относительно неё (acquire/release семантика). Не делает составные операции атомарными.
- **Memory barrier** — гарантия порядка операций памяти между потоками (модель памяти .NET).

**Подводные камни:** `volatile` не заменяет блокировку для `count++` (это read-modify-write, не атомарно) — нужен `Interlocked.Increment`.

---

### Вопрос: Race conditions — обнаружение и предотвращение
**Краткий ответ:**
- Гонка — результат зависит от недетерминированного порядка доступа к общему состоянию.
- Предотвращение: иммутабельность, локализация состояния (нет шаринга), синхронизация (lock/Interlocked), потокобезопасные коллекции.
- Обнаружение: нагрузочные/стресс-тесты, code review, анализаторы, логирование, иногда `Debug` сборки с проверками.

**Подводные камни:** «работает на моей машине» — гонки проявляются под нагрузкой/на других ядрах.

---

### Вопрос: `async void` — почему опасен
**Краткий ответ:**
- `async void` нельзя дождаться (`await`), исключения из него **не ловятся** вызывающим и обычно роняют процесс (попадают в `SynchronizationContext`).
- Допустим только для обработчиков событий (по сигнатуре) — и там лучше обернуть тело в try/catch.
- Везде иначе — `async Task`.

---

### Вопрос: Исключения в async, `AggregateException`, `WhenAll`
**Краткий ответ:**
- `await` пробрасывает первое исключение задачи напрямую (разворачивает `AggregateException`).
- `Task.WhenAll` ждёт все задачи; при ошибках бросает первое исключение, но в `Task.Exception` содержится `AggregateException` со всеми. Чтобы собрать все ошибки — проверяйте `task.Exception` или `Task.WhenAll(...).ContinueWith`.
- `.Result`/`.Wait()` оборачивают исключение в `AggregateException`.

**Пример:**
```csharp
var results = await Task.WhenAll(ids.Select(GetAsync)); // параллельно, ждём всех
```

---

### Вопрос: `Parallel.For/ForEach`, PLINQ, степень параллелизма
**Краткий ответ:**
- `Parallel.For/ForEach` — распараллеливание **CPU-bound** циклов по пулу; `MaxDegreeOfParallelism` ограничивает потоки.
- `Parallel.ForEachAsync` (.NET 6+) — для async-операций с ограничением параллелизма (удобно для пакетных IO с лимитом).
- PLINQ (`.AsParallel()`) — параллельный LINQ для CPU-bound запросов.

**Подводные камни:** для IO предпочтительнее `Parallel.ForEachAsync`/`SemaphoreSlim`, а не `Parallel.ForEach` с блокирующим IO (исчерпание пула).

---

### Вопрос: `IAsyncEnumerable`, `await foreach`
**Краткий ответ:**
- Асинхронные потоки данных: значения приходят по мере готовности (стриминг из БД/сети) без материализации всего набора.
- `await foreach (var item in source.WithCancellation(ct))`. Производитель — `async IAsyncEnumerable<T>` с `yield return` и `[EnumeratorCancellation]`.

**Пример:**
```csharp
async IAsyncEnumerable<int> Generate([EnumeratorCancellation] CancellationToken ct = default)
{
    for (int i = 0; i < 10; i++) { await Task.Delay(100, ct); yield return i; }
}
```

---

### Вопрос: `Channel<T>`, producer/consumer, TPL Dataflow
**Краткий ответ:**
- **`System.Threading.Channels`** — высокопроизводительная async очередь для паттерна producer/consumer с поддержкой backpressure (bounded channel) и нескольких писателей/читателей.
- **TPL Dataflow** — блоки-конвейеры (`TransformBlock`, `ActionBlock`) для сложных пайплайнов обработки.

**Пример:**
```csharp
var channel = Channel.CreateBounded<Work>(100);   // backpressure
// producer:
await channel.Writer.WriteAsync(work, ct);
// consumer:
await foreach (var w in channel.Reader.ReadAllAsync(ct)) Process(w);
```

---

### Вопрос: Потокобезопасные коллекции, нюансы `ConcurrentDictionary`
**Краткий ответ:**
- `ConcurrentDictionary`, `ConcurrentQueue`, `ConcurrentBag`, `BlockingCollection` — для конкурентного доступа без внешнего lock.
- Нюансы `ConcurrentDictionary`:
  - `GetOrAdd`/`AddOrUpdate` с фабрикой **могут вызвать фабрику несколько раз** (она не под локом) — фабрика должна быть идемпотентной/дешёвой; для дорогого создания используйте `Lazy<T>` как значение.
  - `Count`/перечисление — снапшот, относительно дорого.

---

### Вопрос: Async от начала до конца
**Краткий ответ:** Не блокируйте async-цепочку синхронными `.Result`/`.Wait()`/`.GetAwaiter().GetResult()`. Прокидывайте `CancellationToken`. Используйте нативные async-API на всех уровнях. Это предотвращает дедлоки и thread pool starvation.

---

## Полезные ссылки
- Async in depth: https://learn.microsoft.com/dotnet/standard/async-in-depth
- ConfigureAwait FAQ (Stephen Toub): https://devblogs.microsoft.com/dotnet/configureawait-faq/
- Channels: https://learn.microsoft.com/dotnet/core/extensions/channels