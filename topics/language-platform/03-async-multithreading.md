<!-- nav-top -->
[← К оглавлению](../../README.md)

# 03. Многопоточность и асинхронность

<a id="top"></a>

> Статус: 🟡 в работе · Уровень: Senior

## Что проверяют
Понимание разницы потоков и async/await, синхронизации, типичных дедлоков и гонок, паттернов параллельной обработки.

## Ключевые вопросы (чеклист)
- [x] [`Thread` vs `ThreadPool` vs `Task` vs `async/await`](#q1)
- [x] [Как работает `async/await` под капотом (стейт-машина, continuation)](#q2)
- [x] [`SynchronizationContext`, `ConfigureAwait(false)` — зачем и когда](#q3)
- [x] [Дедлок при `.Result`/`.Wait()` на UI/ASP.NET (классика)](#q4)
- [x] [`Task` vs `ValueTask` — когда использовать ValueTask](#q5)
- [x] [CPU-bound vs IO-bound, `Task.Run` — когда оправдан](#q6)
- [x] [Отмена: `CancellationToken`, `CancellationTokenSource`, таймауты](#q7)
- [x] [Примитивы синхронизации: `lock`/`Monitor`, `SemaphoreSlim`, `Mutex`, `ReaderWriterLockSlim`](#q8)
- [x] [`Interlocked`, атомарность, `volatile`, memory barriers](#q9)
- [x] [Гонки данных (race conditions), как обнаружить и избежать](#q10)
- [x] [`async void` — почему опасен, где допустим](#q11)
- [x] [Обработка исключений в async, `AggregateException`, `Task.WhenAll`](#q12)
- [x] [`Parallel.For/ForEach`, PLINQ, степень параллелизма](#q13)
- [x] [`IAsyncEnumerable`, `await foreach`, async-стримы](#q14)
- [x] [`Channel<T>`, producer/consumer, Dataflow (TPL)](#q15)
- [x] [Потокобезопасные коллекции, `ConcurrentDictionary` нюансы](#q16)
- [x] [Async от начала до конца (async all the way)](#q17)
- [x] [Последовательный vs параллельный `await`: запуск независимых задач одновременно](#q18)

## Разбор вопросов

<a id="q1"></a>
### Вопрос: `Thread` vs `ThreadPool` vs `Task` vs `async/await`
**Краткий ответ:**
- **`Thread`** — низкоуровневый ОС-поток (~1 МБ стека). Создание дорого. Нужен редко (долгоживущий выделенный поток, специфичный приоритет).
- **`ThreadPool`** — пул переиспользуемых потоков; экономит создание. Сюда планируются короткие задачи.
- **`Task`** — абстракция над «единицей работы», обычно исполняется на пуле; даёт композицию, продолжения, отмену, обработку ошибок.
- **`async/await`** — синтаксис для неблокирующего ожидания `Task`/`ValueTask`. Главное: **async ≠ многопоточность**. IO-bound async вообще не занимает поток во время ожидания.

**Подводные камни:** `Task.Run` для IO-bound — антипаттерн (занимает поток пула вместо освобождения).

[↑ Наверх](#top)

---

<a id="q2"></a>
### Вопрос: Как работает `async/await` под капотом
**Краткий ответ:**
- Компилятор превращает async-метод в **стейт-машину** (struct, реализующий `IAsyncStateMachine`). Каждый `await` — точка, где метод может «приостановиться».
- При `await` на незавершённой задаче: регистрируется **continuation** (продолжение), управление возвращается вызывающему, поток освобождается. Когда задача завершится — continuation планируется на исполнение (на captured context или пуле).
- Если задача уже завершена — выполнение продолжается синхронно без переключений.

**Подводные камни:**
- Локальные переменные «поднимаются» в поле стейт-машины (в куче, если задача асинхронна) → `ref struct` (`Span`) нельзя держать через `await`.
- Лишние async-обёртки добавляют аллокации; для горячего пути — `ValueTask` и синхронные fast-path.

[↑ Наверх](#top)

---

<a id="q3"></a>
### Вопрос: `SynchronizationContext` и `ConfigureAwait(false)`
**Краткий ответ:**
- `SynchronizationContext` определяет, **куда** вернётся continuation после `await`. В UI (WinForms/WPF) — обратно в UI-поток; в классическом ASP.NET — в контекст запроса. В **ASP.NET Core контекста синхронизации нет**.
- `ConfigureAwait(false)` говорит: «не возвращай меня в captured context, продолжи на пуле». Это:
  - предотвращает дедлоки (см. ниже);
  - снижает накладные расходы переключения.
- В библиотечном коде — **всегда** `ConfigureAwait(false)`. В приложении ASP.NET Core это не обязательно (контекста нет), в UI — осторожно (после await может понадобиться UI-поток).

[↑ Наверх](#top)

---

<a id="q4"></a>
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

[↑ Наверх](#top)

---

<a id="q5"></a>
### Вопрос: `Task` vs `ValueTask`
**Краткий ответ:**
- `Task` — ссылочный тип, аллокация на каждый вызов; универсален, можно ждать многократно, `WhenAll` и т.п.
- `ValueTask`/`ValueTask<T>` — struct, **избегает аллокации**, когда результат часто доступен синхронно (кэш-хит, буфер уже заполнен). Хорош на горячем пути высоконагруженных API.
- Ограничения `ValueTask`: нельзя ждать дважды, нельзя `.Result` до завершения, нельзя хранить/параллельно ждать. Если нужно — `.AsTask()`.

**Подводные камни:** использовать `ValueTask` без необходимости усложняет код; применяйте, когда измеренная аллокация Task — узкое место.

[↑ Наверх](#top)

---

<a id="q6"></a>
### Вопрос: CPU-bound vs IO-bound, когда `Task.Run`
**Краткий ответ:**
- **IO-bound** (запрос к БД/сети/диску): используйте нативный async API (`await ...Async()`); поток не занимается во время ожидания. `Task.Run` тут вреден.
- **CPU-bound** (тяжёлые вычисления): оффлоад на пул через `Task.Run`, чтобы не блокировать текущий (особенно UI) поток. В ASP.NET Core оффлоадить CPU-работу обычно не нужно (она и так на пуле), `Task.Run` лишь снижает доступность потоков.

[↑ Наверх](#top)

---

<a id="q7"></a>
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

[↑ Наверх](#top)

---

<a id="q8"></a>
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

[↑ Наверх](#top)

---

<a id="q9"></a>
### Вопрос: `Interlocked`, `volatile`, memory barriers
**Краткий ответ:**
- **`Interlocked`** — атомарные `Increment`/`Add`/`Exchange`/`CompareExchange`; основа lock-free алгоритмов.
- **`volatile`** — запрещает кэширование переменной и переупорядочивание чтений/записей относительно неё (acquire/release семантика). Не делает составные операции атомарными.
- **Memory barrier** — гарантия порядка операций памяти между потоками (модель памяти .NET).

**Подводные камни:** `volatile` не заменяет блокировку для `count++` (это read-modify-write, не атомарно) — нужен `Interlocked.Increment`.

[↑ Наверх](#top)

---

<a id="q10"></a>
### Вопрос: Race conditions — обнаружение и предотвращение
**Краткий ответ:**
- Гонка — результат зависит от недетерминированного порядка доступа к общему состоянию.
- Предотвращение: иммутабельность, локализация состояния (нет шаринга), синхронизация (lock/Interlocked), потокобезопасные коллекции.
- Обнаружение: нагрузочные/стресс-тесты, code review, анализаторы, логирование, иногда `Debug` сборки с проверками.

**Подводные камни:** «работает на моей машине» — гонки проявляются под нагрузкой/на других ядрах.

[↑ Наверх](#top)

---

<a id="q11"></a>
### Вопрос: `async void` — почему опасен
**Краткий ответ:**
- `async void` нельзя дождаться (`await`), исключения из него **не ловятся** вызывающим и обычно роняют процесс (попадают в `SynchronizationContext`).
- Допустим только для обработчиков событий (по сигнатуре) — и там лучше обернуть тело в try/catch.
- Везде иначе — `async Task`.

[↑ Наверх](#top)

---

<a id="q12"></a>
### Вопрос: Исключения в async, `AggregateException`, `WhenAll`
**Краткий ответ:**
- `await` пробрасывает первое исключение задачи напрямую (разворачивает `AggregateException`).
- `Task.WhenAll` ждёт все задачи; при ошибках бросает первое исключение, но в `Task.Exception` содержится `AggregateException` со всеми. Чтобы собрать все ошибки — проверяйте `task.Exception` или `Task.WhenAll(...).ContinueWith`.
- `.Result`/`.Wait()` оборачивают исключение в `AggregateException`.

**Пример:**
```csharp
var results = await Task.WhenAll(ids.Select(GetAsync)); // параллельно, ждём всех
```

[↑ Наверх](#top)

---

<a id="q13"></a>
### Вопрос: `Parallel.For/ForEach`, PLINQ, степень параллелизма
**Краткий ответ:**
- `Parallel.For/ForEach` — распараллеливание **CPU-bound** циклов по пулу; `MaxDegreeOfParallelism` ограничивает потоки.
- `Parallel.ForEachAsync` (.NET 6+) — для async-операций с ограничением параллелизма (удобно для пакетных IO с лимитом).
- PLINQ (`.AsParallel()`) — параллельный LINQ для CPU-bound запросов.

**Подводные камни:** для IO предпочтительнее `Parallel.ForEachAsync`/`SemaphoreSlim`, а не `Parallel.ForEach` с блокирующим IO (исчерпание пула).

[↑ Наверх](#top)

---

<a id="q14"></a>
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

[↑ Наверх](#top)

---

<a id="q15"></a>
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

[↑ Наверх](#top)

---

<a id="q16"></a>
### Вопрос: Потокобезопасные коллекции, нюансы `ConcurrentDictionary`
**Краткий ответ:**
- `ConcurrentDictionary`, `ConcurrentQueue`, `ConcurrentBag`, `BlockingCollection` — для конкурентного доступа без внешнего lock.
- Нюансы `ConcurrentDictionary`:
  - `GetOrAdd`/`AddOrUpdate` с фабрикой **могут вызвать фабрику несколько раз** (она не под локом) — фабрика должна быть идемпотентной/дешёвой; для дорогого создания используйте `Lazy<T>` как значение.
  - `Count`/перечисление — снапшот, относительно дорого.

[↑ Наверх](#top)

---

<a id="q17"></a>
### Вопрос: Async от начала до конца
**Краткий ответ:** Не блокируйте async-цепочку синхронными `.Result`/`.Wait()`/`.GetAwaiter().GetResult()`. Прокидывайте `CancellationToken`. Используйте нативные async-API на всех уровнях. Это предотвращает дедлоки и thread pool starvation.

[↑ Наверх](#top)

---

<a id="q18"></a>
### Вопрос: Последовательный vs параллельный `await`
**Краткий ответ:**
- `await` **не означает** «иди дальше, не дожидаясь результата». Он означает «**останови этот метод** до готовности результата, **не блокируя поток**». Код внутри метода идёт строго сверху вниз; строка после `await` не выполнится, пока задача не завершится.
- Польза `await` не в ускорении *этого* метода, а в том, что **поток освобождается** и обслуживает другую работу, пока мы ждём I/O.
- Чтобы «идти дальше, не дожидаясь» / запустить несколько операций **одновременно** — **не ставьте `await` сразу**: придержите `Task`, а `await` сделайте позже.

**Последовательно (медленно) — время = сумма:**
```csharp
int a = await GetAsync(url1);   // ждём весь первый
int b = await GetAsync(url2);   // ПОТОМ весь второй
int c = await GetAsync(url3);   // ПОТОМ весь третий
return a + b + c;               // ≈ t1 + t2 + t3
```

**Параллельно (быстро) — время = максимум:**
```csharp
Task<int> t1 = GetAsync(url1);  // все три "в полёте" одновременно
Task<int> t2 = GetAsync(url2);
Task<int> t3 = GetAsync(url3);

int[] r = await Task.WhenAll(t1, t2, t3);   // ждём всех
return r[0] + r[1] + r[2];                  // ≈ max(t1, t2, t3)
```

**Подводные камни:**
- Параллелить так можно **только для независимых** операций. Если `url2` строится из результата `url1` — порядок обязателен, остаётся последовательный код.
- `.Result` после `await Task.WhenAll(...)` безопасен (задачи уже завершены), но вариант с массивом из `WhenAll` чище.
- Здесь **не нужен** `Task.Run`: `GetAsync` — I/O, он не занимает поток. Параллелизм даёт сам запуск нескольких Task без немедленного `await`, а не лишние потоки.
- `WhenAll` при ошибке бросит первое исключение; остальные — в `Task.Exception` (см. [q12](#q12)).

[↑ Наверх](#top)

---

## Полезные ссылки
- Async in depth: https://learn.microsoft.com/dotnet/standard/async-in-depth
- ConfigureAwait FAQ (Stephen Toub): https://devblogs.microsoft.com/dotnet/configureawait-faq/
- Channels: https://learn.microsoft.com/dotnet/core/extensions/channels

[↑ Наверх](#top)

---

<!-- nav-bottom -->
[← К оглавлению](../../README.md)
