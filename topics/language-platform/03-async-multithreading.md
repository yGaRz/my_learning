<!-- nav-top -->
[← К оглавлению](../../README.md)

# 03. Многопоточность и асинхронность

<a id="top"></a>

> Статус: ✅ готово · Уровень: Senior

## Что проверяют
Понимание разницы потоков и async/await, синхронизации, типичных дедлоков и гонок, паттернов параллельной обработки.

## Ключевые вопросы (чеклист)
- [x] [`Thread` vs `ThreadPool` vs `Task` vs `async/await`](#q1)
- [x] [Внутреннее устройство `async/await` (state machine, continuation)](#q2)
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

**Пояснения:**

**Откуда берётся аллокация у `Task`.** Async-метод, возвращающий `Task`/`Task<T>`, при первом «настоящем» await (когда нельзя завершиться синхронно) аллоцирует объект задачи в куче. Плюс стейт-машина async-метода при асинхронном пути тоже уходит в кучу (см. [q2](#q2)). Для редких вызовов это нормально; для метода, который дергают миллионы раз в секунду (кэш, pipe, middleware) — заметный вклад в GC.

**Что внутри `ValueTask`.** Struct-обёртка над одним из трёх вариантов:
1. **Синхронный результат** (`T` или `void`) — никакого `Task`, только значение в struct.
2. **`IValueTaskSource` / `IValueTaskSource<T>`** — пулинговый или кастомный источник без аллокации `Task` (используют BCL: `Socket`, `Stream`, `PipeReader` и т.д.).
3. **`Task` / `Task<T>`** — fallback, если операция асинхронна и пулинг недоступен; аллокация как у обычного `Task`.

То есть `ValueTask` **не магически убирает async** — он убирает аллокацию в **частом синхронном** (или пулинговом) сценарии.

**Когда `Task` — выбор по умолчанию:**
- Публичный API библиотеки, если не уверены в паттерне вызова.
- Нужно хранить результат, передавать дальше, ждать из нескольких мест.
- Композиция: `Task.WhenAll`, `Task.WhenAny`, `ContinueWith`, очереди задач.
- Метод **обычно** завершается асинхронно (сетевой вызов без кэша) — выигрыш от `ValueTask` минимален.

**Когда `ValueTask` оправдан:**
- **Fast-path:** результат уже есть в 80–90%+ вызовов (кэш, буфер прочитан целиком, no-op, валидация отсеяла запрос до I/O).
- **Горячий путь:** профилировщик показал аллокации `Task` / `AsyncTaskMethodBuilder` как узкое место.
- **Внутренний** код с контролируемым потребителем: «вызвал → один раз `await` → выбросил».

**Модель в голове (главное):**

`ValueTask` — **не** «заготовка под await» и **не** отдельный механизм async. Это **обёртка-контейнер** с двумя (тремя) вариантами содержимого:

| Что внутри `ValueTask<T>` | Когда | Аллокация `Task`? | Стейт-машина у *этого* метода? |
|---------------------------|-------|-------------------|--------------------------------|
| Готовое значение `T` | кэш-hit, fast-path | **Нет** | **Нет** (метод без `async`) |
| `Task<T>` | кэш-miss, редкий I/O | **Да** (у `FetchAndCacheAsync`) | **Да** (у внутреннего async-метода) |
| `IValueTaskSource<T>` | пулинг в BCL | обычно нет | зависит от источника |

Смысл паттерна: снаружи **одинаковый** вызов `await GetUserAsync(...)`, а внутри в 90% случаев всё проходит **синхронно без `Task` и без лишней стейт-машины**. Редкий async-путь делегируется отдельному методу, который возвращает обычный `Task<T>`.

---

**Разбор по шагам — полный пример:**

```csharp
// ── Вызов (контроллер, сервис — где угодно) ──
var user = await GetUserAsync(id, ct);   // ← await ЗДЕСЬ, у потребителя
```

```csharp
// ── Обёртка: без async, без своей стейт-машины ──
public ValueTask<User?> GetUserAsync(int id, CancellationToken ct)
{
    if (_cache.TryGetValue(id, out User? user))
        return new ValueTask<User?>(user);   // ветка A: синхронный fast-path

    return new ValueTask<User?>(FetchAndCacheAsync(id, ct));  // ветка B: оборачиваем Task
}

// ── Настоящий async только на редком пути ──
private async Task<User?> FetchAndCacheAsync(int id, CancellationToken ct)
{
    var user = await _repo.GetByIdAsync(id, ct);  // I/O → стейт-машина, await, освобождение потока
    if (user is not null)
        _cache[id] = user;
    return user;
}
```

**Ветка A — кэш-hit (типичный случай, ~90% вызовов):**

```
1. Controller: await GetUserAsync(42, ct)
2. GetUserAsync: _cache.TryGetValue → нашли user
3. return new ValueTask<User?>(user)     // struct на стеке, внутри просто User
4. await на ValueTask:
   - IsCompleted == true
   - результат берётся сразу, continuation не нужен
   - поток НЕ освобождался (не было I/O)
5. user в Controller — готово

Аллокации: 0 Task, 0 стейт-машина у GetUserAsync
```

**Ветка B — кэш-miss (редкий случай):**

```
1. Controller: await GetUserAsync(99, ct)
2. GetUserAsync: кэш пуст → вызываем FetchAndCacheAsync(99, ct)
   → возвращается Task<User?> (уже создан компилятором async-метода)
3. return new ValueTask<User?>(task)       // ValueTask хранит ссылку на этот Task
4. await на ValueTask:
   - видит внутри незавершённый Task
   - ведёт себя как await task: continuation, поток освобождается
5. когда БД ответила — continuation, user в Controller

Аллокации: Task + стейт-машина у FetchAndCacheAsync (но НЕ у GetUserAsync)
```

**Сравнение — если написать «проще», но хуже для hot path:**

```csharp
// Работает, но на КАЖДЫЙ вызов (включая кэш-hit) компилятор создаёт
// стейт-машину для GetUserAsync — зря, если hit в 90% случаев
public async ValueTask<User?> GetUserAsync(int id, CancellationToken ct)
{
    if (_cache.TryGetValue(id, out User? user))
        return user;                              // hit, но стейт-машина уже есть

    return await FetchAndCacheAsync(id, ct);      // miss
}
```

**Где какой `await`:**

| Место | Есть `await`? | Зачем |
|-------|---------------|-------|
| `Controller: await GetUserAsync(...)` | **Да** | потребитель ждёт результат |
| `GetUserAsync` | **Нет** | только выбирает ветку и возвращает `ValueTask` |
| `FetchAndCacheAsync` | **Да** (`await _repo...`) | реальное I/O, стейт-машина, освобождение потока |

**Итог одной фразой:** `ValueTask` даёт **единый async-контракт** (`await` снаружи), но внутри в большинстве случаев отдаёт результат **сразу**, не создавая `Task` и не поднимая стейт-машину в обёртке; на редком пути внутрь кладётся уже готовый `Task` от настоящего async-метода.

--- 

**Почему ограничения `ValueTask` — не придирка.** После первого `await`/`AsTask()` внутреннее состояние может быть **потреблено** (особенно у `IValueTaskSource` из пула — объект возвращают в пул). Повторный await → undefined behavior / исключение. То же с `.Result` до завершения: у `Task` состояние стабильно, у `ValueTask` — нет.

**Правило потребления:** один владелец, один await. Нужно раздать наружу или дождаться дважды → сразу `.AsTask()` (это аллоцирует `Task`, но безопасно).

```csharp
// Плохо — ValueTask в поле / коллекции
List<ValueTask<int>> jobs = ...;
await Task.WhenAll(jobs.Select(async v => await v));  // двойное потребление

// Хорошо — конвертация при «размножении»
Task<int> t = GetValueAsync().AsTask();
await t;
await t;  // ok
```

**`ValueTask` vs «просто не делать async».** Если метод **всегда** синхронен — не объявляйте `async ValueTask<T>`, возвращайте `T` или `ValueTask.FromResult(x)`. Лишний `async` добавляет стейт-машину даже при `ValueTask`.

**Подводные камни:**
- Использовать `ValueTask` без измерений — усложняет код ради микрооптимизации.
- Возвращать `ValueTask` из API, где вызывающий неизвестен (может сохранить и await дважды).
- Думать, что `ValueTask` ускоряет I/O — ускоряет/разгружает GC только когда **часто** срабатывает синхронная ветка или пулинг.
- `ConfigureAwait` работает с обоими; ограничения потребления одинаковы.

**Практическое правило:** пишите `Task`/`Task<T>`. Переходите на `ValueTask`, когда профиль показал проблему **и** у вас есть явный fast-path или `IValueTaskSource`.

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
- **Кооперативная** отмена: код сам периодически проверяет токен или передаёт его в async-API. Принудительно «убить поток» токен не может.
- `CancellationTokenSource` (CTS) — **источник** отмены; `CancellationToken` — лёгкая **копия-ссылка**, которую прокидывают вниз по цепочке.
- Таймаут: `cts.CancelAfter(TimeSpan.FromSeconds(3))` или `new CancellationTokenSource(TimeSpan.FromSeconds(3))`.
- Объединение: `CreateLinkedTokenSource(внешний, локальный)` — сработает любой из токенов (клиент отключился **или** наш таймаут).

**Как это устроено:**

```
Controller                    Service                      Repository / HttpClient
    │                             │                                │
    │  ct = RequestAborted        │                                │
    │  + локальный таймаут 3 сек  │                                │
    ├────────────────────────────►│  тот же ct (linked)            │
    │                             ├───────────────────────────────►│
    │                             │                                │ await с ct
    │                             │                                │ → I/O прерван
    │◄── OperationCanceledException (если не успели)              │
    │   → вернуть 408/504 клиенту                                 │
```

**Роли типов:**

| Тип | Кто создаёт | Зачем |
|-----|-------------|-------|
| `CancellationTokenSource` | тот, кто **решает** отменить (таймер, кнопка Stop, middleware) | владеет таймером, вызывает `Cancel()` |
| `CancellationToken` | из `cts.Token` | только **читать** и передавать дальше |
| `HttpContext.RequestAborted` | Kestrel | отмена, если клиент закрыл соединение |

**Пример: таймаут 3 секунды в контроллере + ответ клиенту**

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> GetUser(int id, CancellationToken cancellationToken)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
    cts.CancelAfter(TimeSpan.FromSeconds(3));   // наш лимит на операцию

    try
    {
        var user = await _userService.GetUserAsync(id, cts.Token);
        return user is null ? NotFound() : Ok(user);
    }
    catch (OperationCanceledException) when (cancellationToken.IsCancellationRequested)
    {
        // клиент сам отключился — не пишем в лог как ошибку бизнес-логики
        return StatusCode(499); // или просто пробросить — запрос уже мёртв
    }
    catch (OperationCanceledException) when (cts.IsCancellationRequested)
    {
        // сработал именно наш таймаут 3 сек
        return StatusCode(StatusCodes.Status408RequestTimeout, "Превышено время ожидания (3 сек)");
    }
}
```

**Сервис — только прокидывает токен (не создаёт свой CTS без нужды):**

```csharp
public async Task<User?> GetUserAsync(int id, CancellationToken ct)
{
    // HttpClient, EF, Dapper — все принимают CancellationToken
    return await _repo.GetByIdAsync(id, ct);
}
```

**Репозиторий:**

```csharp
public async Task<User?> GetByIdAsync(int id, CancellationToken ct)
{
    return await _db.Users
        .AsNoTracking()
        .FirstOrDefaultAsync(u => u.Id == id, ct);   // ct уходит в провайдер БД
}
```

Что происходит при таймауте:
1. `CancelAfter(3 сек)` вызывает `cts.Cancel()`.
2. Незавершённый `await` (HTTP, SQL) получает отмену и бросает `OperationCanceledException`.
3. Исключение всплывает в контроллер → ловим и возвращаем **408 Request Timeout** (или 504, если это gateway-прокси сценарий).

**Паттерн «локальный таймаут + внешняя отмена» в одном методе (без контроллера):**

```csharp
public async Task<Data> GetWithTimeoutAsync(CancellationToken ct)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
    cts.CancelAfter(TimeSpan.FromSeconds(3));

    return await _client.GetFromJsonAsync<Data>(url, cts.Token);
    // отменится и при разрыве клиента (ct), и через 3 сек (cts)
}
```

**Долгий CPU-цикл — токен нужно проверять вручную** (I/O-API делают это сами):

```csharp
public async Task ProcessBatchAsync(IEnumerable<Item> items, CancellationToken ct)
{
    foreach (var item in items)
    {
        ct.ThrowIfCancellationRequested();              // синхронная проверка
        await ProcessOneAsync(item, ct);                // I/O с токеном
    }
}
```

**`OperationCanceledException` vs таймаут:**
- При отмене по токену почти всегда летит `OperationCanceledException` (в .NET 6+ иногда `TaskCanceledException` — наследник).
- Отдельный `TimeoutException` появится только если API сам так задуман (например, `HttpClient.Timeout` без вашего токена). В async-коде с `CancellationToken` привычнее ловить `OperationCanceledException` и различать причину через `when (cts.IsCancellationRequested)`.

**Подводные камни:**
- **Не прокинутый токен** — `CancelAfter` сработает, но БД/HTTP продолжат работу до конца; таймаут «на бумаге».
- **Глотать `OperationCanceledException`** без `when` — скрывает и таймаут, и отмену клиента.
- **Не диспозить CTS** — `using var cts = ...` обязателен (таймер держит ресурсы).
- **`HttpClient.Timeout` + свой `CancelAfter`** — два независимых механизма; лучше один: либо `CancelAfter` на linked token, либо `client.Timeout`, не оба сразу без понимания.
- **После отмены не использовать `cts.Token` повторно** — создайте новый CTS на следующую попытку.
- В ASP.NET Core `CancellationToken` в action **уже** содержит `RequestAborted` — не нужен отдельный `[FromServices]`, достаточно параметра метода.

[↑ Наверх](#top)

---

<a id="q8"></a>
### Вопрос: Примитивы синхронизации
**Краткий ответ:**
- **`lock` (Monitor)** — взаимное исключение **внутри одного процесса**; нельзя `await` внутри `lock`. С .NET 9 — тип `Lock`.
- **`SemaphoreSlim`** — ограничение числа одновременных входов; есть `WaitAsync` (для async). Замена `lock`, когда в критической секции нужен `await`.
- **`Mutex`** — блокировка **между процессами** (named mutex), дороже `lock`.
- **`ReaderWriterLockSlim`** — много читателей / один писатель (внутри процесса).
- **`Interlocked`** — атомарные операции над одной переменной **без** полноценной блокировки.

**Уровни блокировок — от лёгкого к тяжёлому:**

| Уровень | Где действует | Примеры | Скорость | Когда |
|---------|---------------|---------|----------|-------|
| **Поток в процессе** | один процесс, shared memory | `Interlocked`, `volatile` | наносекунды–микросекунды | счётчик, флаг, lock-free |
| **Приложение (in-process)** | один процесс, любые потоки | `lock`, `Lock`, `SemaphoreSlim`, `ReaderWriterLockSlim` | микросекунды | защита общего состояния в памяти |
| **ОС / между процессами** | несколько процессов на одной машине | `Mutex` (named), `EventWaitHandle`, file lock | медленнее (syscall) | single instance app, координация воркеров |
| **Распределённая / система** | несколько инстансов, сеть | Redis lock, блокировки БД, advisory locks | миллисекунды + сеть | кластер API, единственный job на ферме |

```
Один сервер ASP.NET                    Несколько инстансов
┌─────────────────────────┐            ┌────────┐  ┌────────┐
│  lock / SemaphoreSlim   │            │ API #1 │  │ API #2 │
│  (достаточно)           │            └───┬────┘  └───┬────┘
└─────────────────────────┘                └──────┬──────┘
                                                  │
                                          Redis / БД lock
                                          (нужен distributed lock)
```

**Правило выбора:** начинайте с самого лёгкого уровня, который решает задачу. `lock` в памяти **не** защитит второй инстанс приложения — для этого нужен Mutex (процессы) или Redis/БД (кластер).

---

#### 1. `lock` (Monitor) — in-process, один владелец

Взаимное исключение: в критической секции только **один поток**.

```csharp
private readonly object _sync = new();   // приватный объект, НЕ this/typeof/string
private int _balance;

public void Withdraw(int amount)
{
    lock (_sync)
    {
        if (_balance < amount) throw new InvalidOperationException();
        _balance -= amount;                // атомарно относительно других потоков
    }
}
```

- Область: **один процесс**.
- Нельзя `await` внутри `lock` → deadlock/захват на время I/O.
- Под капотом: `Monitor.Enter` / `Exit` (владелец может рекурсивно войти повторно).

---

#### 2. `Lock` (.NET 9+) — то же, но явный тип

```csharp
private readonly Lock _lock = new();

public void Update(Dictionary<int, string> cache, int key, string value)
{
    using (_lock.EnterScope())           // scope-based, как lock
    {
        cache[key] = value;
    }
}
```

Предпочтительнее `lock(object)` в новом коде: быстрее, понятнее намерение, нельзя случайно залочить чужой объект.

---

#### 3. `SemaphoreSlim` — in-process, N одновременных входов + async

`lock` = «только 1». `SemaphoreSlim(3, 3)` = «до 3 одновременно».

**Ограничение параллельных вызовов внешнего API:**
```csharp
private readonly SemaphoreSlim _throttle = new(10, 10);   // макс. 10 одновременно

public async Task<string> CallExternalAsync(CancellationToken ct)
{
    await _throttle.WaitAsync(ct);
    try
    {
        return await _http.GetStringAsync(url, ct);        // await допустим!
    }
    finally
    {
        _throttle.Release();                             // всегда в finally
    }
}
```

**Замена `lock` в async-коде** (`new(1, 1)` = как lock, но с `WaitAsync`):
```csharp
private readonly SemaphoreSlim _gate = new(1, 1);

public async Task DoAsync()
{
    await _gate.WaitAsync();
    try
    {
        _state = await LoadAndMergeAsync();                // внутри есть await
    }
    finally { _gate.Release(); }
}
```

---

#### 4. `Mutex` — между процессами (уровень ОС)

`lock` не видит другой процесс. **Named Mutex** — общее имя в ОС, видят все процессы на машине.

**Только один экземпляр приложения:**
```csharp
const string mutexName = @"Global\MyCompany.MyApp.SingleInstance";
using var mutex = new Mutex(initiallyOwned: true, name: mutexName, out bool createdNew);

if (!createdNew)
{
    Console.WriteLine("Приложение уже запущено");
    return;
}

// работаем...
```

**Координация двух процессов (очередь на файл):**
```csharp
using var mutex = Mutex.OpenExisting(@"Global\MyApp.FileQueue");
mutex.WaitOne();                    // блокирующий syscall — не для hot path
try
{
    ProcessQueueFile();
}
finally
{
    mutex.ReleaseMutex();
}
```

- Дороже `lock` (переход в ядро ОС).
- Может быть **межмашинным** только если ОС это поддерживает (обычно — одна машина / домен).
- Для async — `WaitOne` блокирует поток; в async-сценариях чаще `SemaphoreSlim` внутри процесса или distributed lock.

---

#### 5. `ReaderWriterLockSlim` — in-process, много читателей / один писатель

Когда чтений много, записей мало — читатели не блокируют друг друга.

```csharp
private readonly ReaderWriterLockSlim _rw = new();
private readonly Dictionary<int, User> _cache = new();

public User? GetUser(int id)
{
    _rw.EnterReadLock();
    try
    {
        return _cache.TryGetValue(id, out var u) ? u : null;
    }
    finally { _rw.ExitReadLock(); }
}

public void PutUser(User user)
{
    _rw.EnterWriteLock();
    try
    {
        _cache[user.Id] = user;
    }
    finally { _rw.ExitWriteLock(); }
}
```

- Upgradeable read lock — если при чтении понадобилась запись.
- Нет `EnterReadLockAsync` — для async-критических секций снова `SemaphoreSlim` или другой подход.

---

#### 6. `Interlocked` — атомарность без блокировки (in-process)

Одна переменная, операция за одну CPU-инструкцию (CAS и т.п.). Не заменяет `lock` для сложных структур.

```csharp
private int _requestCount;
private long _totalBytes;

public void OnRequest(int bytes)
{
    Interlocked.Increment(ref _requestCount);
    Interlocked.Add(ref _totalBytes, bytes);
}

// lock-free обновление «если ещё не занято»
private int _initialized;
public void EnsureInit()
{
    if (Interlocked.CompareExchange(ref _initialized, 1, 0) == 0)
        DoHeavyInit();   // выполнит только один поток
}
```

Подробнее про `volatile` и memory barriers — [q9](#q9).

---

#### Блокировки на уровне системы (за пределами памяти процесса)

**БД — pessimistic lock (транзакция):**
```sql
-- PostgreSQL / SQL Server: строка заблокирована до COMMIT
SELECT * FROM orders WHERE id = @id FOR UPDATE;
```
Используется, когда несколько инстансов API пишут в одну БД и нужна гарантия на уровне данных. См. [тему 14 — блокировки БД](../web-data/14-databases-sql.md).

**БД — advisory lock (PostgreSQL):**
```sql
SELECT pg_advisory_lock(42);   -- логический замок по числовому id
-- критическая работа
SELECT pg_advisory_unlock(42);
```
Координация job'ов без блокировки таблиц.

**Redis — распределённый lock (несколько инстансов API):**
```csharp
// SET key value NX EX 30 — «замок» с TTL, если ключа не было
// освобождение — DELETE по token (fencing token — для строгой корректности)
```
См. [тему 17 — Redlock](../distributed/17-caching.md). Осторожно: при GC-паузах и сетевых задержках нужны fencing tokens для критичных сценариев.

**Файловая блокировка (ОС):**
```csharp
using var fs = new FileStream(path, FileMode.Open, FileAccess.Read,
    FileShare.None);   // FileShare.None = эксклюзив на файл
```
Два процесса не пишут в один файл одновременно.

---

**Сводка: что выбрать**

| Задача | Примитив |
|--------|----------|
| Защитить поле/словарь в одном процессе | `lock` / `Lock` |
| Ограничить 10 параллельных HTTP-вызовов | `SemaphoreSlim(10)` |
| Критическая секция **с** `await` | `SemaphoreSlim(1)` + `WaitAsync` |
| Кэш: много чтений, редкие записи | `ReaderWriterLockSlim` |
| Счётчик / флаг без lock | `Interlocked` |
| Один экземпляр .exe на машине | Named `Mutex` |
| Один job среди 5 инстансов API | Redis lock / advisory lock / `FOR UPDATE` |
| Конкурентная запись в одну строку БД | Транзакция + `FOR UPDATE` / optimistic concurrency |

**Подводные камни:**
- `lock` на `this` / `typeof(T)` / строке-литерале — другой код может залочить тот же объект → дедлок.
- `await` внутри `lock` — запрещено компилятором (CS1996); используйте `SemaphoreSlim`.
- `Mutex` без `ReleaseMutex` в `finally` — зависший замок для всех процессов.
- `ReaderWriterLockSlim` без `Dispose` — утечка; писатели голодут при постоянных читателях.
- Distributed lock с TTL короче работы — два воркера выполнят job одновременно; TTL должен покрывать worst case + heartbeat.
- `lock` в кластере из 3 pod'ов — **не работает** как distributed lock; каждый pod имеет свою память.

[↑ Наверх](#top)

---

<a id="q9"></a>
### Вопрос: `Interlocked`, `volatile`, memory barriers
**Краткий ответ:**
- **`Interlocked`** — атомарные операции над **одной** переменной (CPU-инструкции CAS и т.п.); основа lock-free. Не заменяет `lock` для нескольких полей / словарей.
- **`volatile`** — **видимость** и порядок чтений/записей **одного поля** между потоками; **не** делает `count++` атомарным.
- **Memory barrier** — «черта», до/после которой CPU/компилятор не переупорядочивают операции с памятью; `volatile`, `lock`, `Interlocked` дают барьеры в разной степени.

**Три инструмента — одна картина:**

| | Видимость между потоками | Атомарность `count++` | Несколько полей / словарь |
|---|--------------------------|----------------------|---------------------------|
| `volatile` | **Да** (одно поле) | **Нет** | **Нет** |
| `Interlocked` | **Да** | **Да** (одно поле) | **Нет** |
| `lock` | **Да** | **Да** | **Да** |

---

#### `Interlocked` — атомарность без `lock`

**Проблема:** `count++` — это три шага (read → add → write). Два потока оба могут прочитать `5` и записать `6` — потеряли инкремент.

**Решение:** `Interlocked` выполняет операцию **атомарно на уровне CPU** (без входа в монитор `lock`).

**Основные методы:**

```csharp
Interlocked.Increment(ref count);           // атомарный ++
Interlocked.Decrement(ref count);           // атомарный --
Interlocked.Add(ref total, bytes);          // атомарный +=
Interlocked.Exchange(ref status, 1);        // записать, вернуть старое
Interlocked.CompareExchange(ref val, newVal, expected);  // CAS
```

**CAS (Compare-And-Swap)** — «если `val == expected`, записать `newVal`; вернуть старое значение»:

```csharp
// Lazy init — создаст ровно один экземпляр
private ExpensiveService? _service;

public ExpensiveService GetService()
{
    if (_service is not null) return _service;
    var created = new ExpensiveService();
    return Interlocked.CompareExchange(ref _service, created, null) ?? created;
}
```

**Когда `Interlocked`, когда `lock`:**

| Задача | Инструмент |
|--------|------------|
| Счётчик запросов / метрик | `Interlocked` |
| Глобальный лимит (один счётчик на всё API) | `Interlocked` |
| Лимит **per-user** (словарь + окно времени) | `ConcurrentDictionary` + **`lock` на entry** |
| Несколько полей меняются вместе (`Count` + `WindowStart`) | **`lock`** |
| `Dictionary`, список, сложный инвариант | **`lock`** / `Concurrent*` |

Пример: per-user rate limit — **не** один `Interlocked`, потому что нужны два поля и сброс окна:

```csharp
private readonly ConcurrentDictionary<string, UserLimit> _limits = new();

public bool TryAllow(string userId, int maxPerSecond)
{
    var entry = _limits.GetOrAdd(userId, _ => new UserLimit());
    lock (entry)   // Interlocked на Count не защитит сброс WindowStart
    {
        if ((DateTime.UtcNow - entry.WindowStart).TotalSeconds >= 1)
        { entry.WindowStart = DateTime.UtcNow; entry.Count = 0; }
        if (entry.Count >= maxPerSecond) return false;
        entry.Count++;
        return true;
    }
}
```

Область действия — **один процесс** (как `lock`). Между инстансами API — Redis / gateway.

---

#### `volatile` — видимость, не атомарность

**Две проблемы без синхронизации:**
1. **Кэш регистра / ядра** — поток читает устаревшее значение.
2. **Переупорядочивание** — CPU/компилятор меняет порядок операций; другой поток видит «не тот» порядок событий.

`volatile` на поле: «каждое чтение/запись — из/в память, не переупорядочивать относительно этого поля произвольно».

**Классический пример — флаг остановки:**

```csharp
private volatile bool _running = true;

public void Stop() => _running = false;

public void Worker()
{
    while (_running)    // каждая проверка — чтение из памяти
        DoWork();
}
```

**Два разных случая (не путать):**

| Ситуация | Это баг? |
|----------|----------|
| Поток 2 поставил `false`, поток 1 ещё в `DoWork()` — выйдет после следующей проверки `while` | **Нет** — нормальная задержка на одну итерацию |
| Поток 2 поставил `false`, поток 1 **никогда** не видит `false` (JIT закэшировал `true` в регистре → `while(true)`) | **Да** — без `volatile` в Release |

```
БЕЗ volatile (плохо):                С volatile (норм):
t1: поток1 читает true → регистр     t1: while → true из памяти
t2: поток2 пишет false               t2: DoWork()
t3: поток1 смотрит в регистр → ∞     t3: поток2 false
                                     t4: while → false → выход
```

**`volatile` НЕ делает `count++` безопасным:**

```csharp
private volatile int _count;
_count++;   // ❌ read-modify-write, гонка остаётся

Interlocked.Increment(ref _count);   // ✅
```

**Почему не ставить `volatile` на всё:**
- **Дорого** — нельзя держать в регистре, каждое обращение к памяти; циклы замедляются.
- **Бессмысленно** для локальных переменных (стек, один поток).
- **Не спасает** словари, несколько связанных полей, составные операции.
- **Ложная безопасность** — кажется «теперь потокобезопасно», но инварианты ломаются.

**Пример переупорядочивания между полями (volatile на одном не поможет):**

```csharp
_data = "hello";      // обычное поле
_ready = true;        // volatile

// Поток B может увидеть _ready == true, но _data ещё null
// Нужен lock на обе операции или volatile/барьеры на оба поля
```

**Когда `volatile` уместен:** флаг `_running` / `_shutdown`, простой флаг «инициализировано» (один пишет, другие читают). В современном коде альтернатива: `Volatile.Read` / `Volatile.Write`.

---

#### Memory barriers — порядок операций между потоками

**Memory barrier (барьер памяти)** — гарантия: операции с памятью **до** барьера не «перепрыгнут» **после** и наоборот (в пределах модели .NET).

Упрощённо:
- **Acquire** (чтение) — всё **после** не уйдёт **раньше** этого чтения.
- **Release** (запись) — всё **до** не уйдёт **позже** этой записи.

**Кто даёт барьеры:**

| Механизм | Барьер |
|----------|--------|
| `volatile` read | acquire |
| `volatile` write | release |
| `lock` enter / exit | полный (и видимость всех полей в критической секции) |
| `Interlocked.*` | полный для затронутого поля |
| `Volatile.Read` / `Volatile.Write` | явный acquire / release |
| `Thread.MemoryBarrier()` | полный барьер (низкоуровневый, редко нужен вручную) |

**Связка «данные + флаг» с барьером:**

```
Поток A                          Поток B
────────                         ────────
data = 42;
Volatile.Write(ref _ready, true) → if (Volatile.Read(ref _ready))
                                     use(data);   // увидит 42
```

`lock` делает то же сильнее: внутри критической секции все записи видны другим после выхода из `lock`.

---

**Шпаргалка: что выбрать**

| Задача | Инструмент |
|--------|------------|
| Флаг «стоп» для фонового потока | `volatile bool` / `Volatile.Read` |
| Счётчик RPS, метрики | `Interlocked` |
| Per-user лимит (словарь + окно) | `ConcurrentDictionary` + `lock(entry)` |
| «Записал данные, потом флаг ready» | `lock` или `Volatile` на обоих этапах |
| Любой сложный инвариант | `lock` |

**Подводные камни:**
- `volatile` на `Dictionary` — потокобезопасна только **ссылка**, не содержимое.
- `Interlocked` между процессами / pod'ами — **не работает**; только shared memory одного процесса.
- `volatile` не прерывает долгий `DoWork()` — для отмены используйте `CancellationToken`.
- Думать, что `volatile` = thread-safe объект — **нет**, только видимость одного поля.

[↑ Наверх](#top)

---

<a id="q10"></a>
### Вопрос: Race conditions — обнаружение и предотвращение
**Краткий ответ:**
- **Гонка (race condition)** — результат зависит от **недетерминированного порядка** доступа потоков к общему изменяемому состоянию; баг может проявляться редко.
- **Предотвращение:** не шарить состояние → иммутабельность → синхронизация (`lock`/`Interlocked`) → потокобезопасные коллекции.
- **Обнаружение:** стресс-тесты под нагрузкой, code review типовых паттернов, анализаторы, «flaky» тесты, профилирование lock contention.

**Что такое гонка — три условия (все сразу):**

1. **Shared** — несколько потоков видят одни и те же данные.
2. **Mutable** — данные **меняются**.
3. **Unsynchronized** — нет `lock` / `Interlocked` / потокобезопасной абстракции.

Убери любое одно — гонки нет.

---

#### Типовые виды гонок

**1. Lost update (потерянное обновление)**

```csharp
// Два потока списывают с одного баланса
_balance -= amount;   // не атомарно: read → subtract → write

// Поток A читает 1000, поток B читает 1000
// A пишет 900, B пишет 800 — одно списание потеряно
```

**2. Check-then-act (проверил — потом сделал)**

```csharp
if (!_cache.ContainsKey(key))       // поток A: нет ключа
    _cache[key] = Load(key);        // поток B: тоже нет → оба грузят
```

Между `ContainsKey` и `Add` другой поток успевает влезть.

**3. Read-modify-write без атомарности**

```csharp
_count++;                          // см. [q9](#q9) — нужен Interlocked
list.Add(item);                    // List<T> не потокобезопасен
```

**4. Torn read / несогласованное состояние**

```csharp
// Поток A пишет: X=1, Y=2
// Поток B читает: X=1, Y=0 (старое) — инвариант X==Y нарушен
```

Два поля без общей синхронизации.

**5. Async-гонки (часто на собеседовании)**

```csharp
private int _total;

public async Task ProcessAsync()
{
    var local = _total;              // прочитали
    await Task.Delay(10);            // поток освободился — другой изменил _total
    _total = local + 1;              // перезаписали устаревшее
}
```

`await` **разрывает** атомарность: между read и write другой поток успевает вмешаться.

```csharp
// Захват HttpContext в fire-and-forget — другой запрос уже на этом потоке
_ = Task.Run(async () => {
    await Task.Delay(100);
    _httpContext.User...             // гонка / чужой запрос
});
```

---

#### Предотвращение — от лучшего к худшему

**1. Не шарить mutable-состояние (лучший способ)**

```csharp
// Каждый запрос — свой scope в DI, свой объект
public class OrderService { ... }   // Scoped — нет гонки между запросами

// Локальные переменные — только этот поток
public void Handle() {
    var items = new List<Item>();   // не shared → безопасно
}
```

В ASP.NET Core большинство багов — **статические поля** и **singleton с mutable state**.

**2. Иммутабельность**

```csharp
// Вместо менять объект — создать новый
public record Order(int Id, decimal Total);

public Order AddItem(Order order, Item item) =>
    order with { Total = order.Total + item.Price };   // старый не тронут
```

Нет shared mutation → нет гонки на этом объекте.

**3. Синхронизация** (см. [q8](#q8), [q9](#q9))

| Ситуация | Решение |
|----------|---------|
| Один счётчик | `Interlocked` |
| Несколько полей / инвариант | `lock` |
| Async-критическая секция | `SemaphoreSlim` + `WaitAsync` |
| Per-user состояние в словаре | `ConcurrentDictionary` + `lock(entry)` |

**4. Потокобезопасные коллекции** (см. [q16](#q16))

```csharp
// ✅
var dict = new ConcurrentDictionary<string, User>();
dict.TryAdd(key, user);

// ❌
var dict = new Dictionary<string, User>();   // гонка при Add/Contains
```

`ConcurrentDictionary` защищает **структуру** словаря, но **не** составные операции (`if (!Contains) Add` — всё ещё гонка).

**5. Один писатель / каналы**

```csharp
// Producer/consumer через Channel — один consumer меняет состояние
await foreach (var msg in channel.Reader.ReadAllAsync(ct))
    Process(msg);
```

---

#### Как обнаружить гонку

**Почему «работает на моей машине»:**
- на dev — 1–2 ядра, мало параллелизма;
- гонка = вероятностный баг: «выигрывает» тот поток, который успел раньше;
- в Debug JIT оптимизирует иначе, чем в Release;
- тест прошёл 100 раз — 101-й под нагрузкой упал.

**Методы обнаружения:**

| Метод | Что даёт |
|-------|----------|
| **Стресс / нагрузочные тесты** | k6, NBomber, `Parallel.For` в тесте — много потоков, длинный прогон |
| **Запуск тестов 100–1000 раз** | `dotnet test --repeat N` — ловит flaky-тесты |
| **Code review паттернов** | `static` mutable, `Check-then-act`, `List<T>` в singleton, `await` между read/write |
| **Анализаторы** | Roslyn: не шарить mutable static; CA1849 (sync over async); custom analyzers на `lock` |
| **Профилирование** | `dotnet-counters` → lock contention; dotTrace — кто держит lock |
| **Логирование с thread id** | видно «два потока одновременно трогали X» |
| **Больше ядер** | CI на многопоточной машине, `-p:ParallelizeTestCollections` |

**Минимальный стресс-тест в коде:**

```csharp
[Fact]
public async Task Balance_StressTest_NoLostUpdates()
{
    var account = new Account();
    var tasks = Enumerable.Range(0, 1000)
        .Select(_ => Task.Run(() => account.Deposit(1)));
    await Task.WhenAll(tasks);
    Assert.Equal(1000, account.Balance);   // без синхронизации часто < 1000
}
```

**Признаки гонки в проде:**
- редкие «неправильные» суммы / дубли записей;
- тесты flaky (то green, то red);
- баг не воспроизводится под отладчиком (Heisenbug);
- после увеличения нагрузки / инстансов — чаще.

---

#### Чеклист code review (типичные места гонок)

- [ ] `static` поля с изменяемым состоянием (`static int`, `static List`, кэш без lock).
- [ ] Singleton-сервис с полями, которые меняются между запросами.
- [ ] `Dictionary` / `List` / `HashSet` из нескольких потоков.
- [ ] `if (!dict.ContainsKey(k)) dict[k] = ...` — check-then-act.
- [ ] Read → `await` → write общего поля.
- [ ] `HttpContext`, `DbContext` в `Task.Run` / fire-and-forget.
- [ ] Double-checked locking без `volatile` / `Lazy<T>`.

---

#### Стратегия выбора (кратко)

```
Можно не шарить?           → локальные переменные, scoped DI, immutable
Только один счётчик?       → Interlocked
Сложный инвариант?         → lock / SemaphoreSlim
Словарь/очередь?           → Concurrent* + осторожно с составными операциями
Между сервисами / pod'ами? → не lock в памяти — БД / Redis / очередь
```

**Подводные камни:**
- `ConcurrentDictionary.GetOrAdd` с фабрикой может вызвать фабрику **несколько раз** — не полагаться на «вызовется ровно один».
- `lock` **внутри** процесса не видит другой pod — гонка между инстансами остаётся.
- Иммутабельность record не спасает, если **ссылка** на mutable-объект шарится: `record Holder(List<int> Items)` — список внутри общий.
- «Добавил lock везде» без профиля — дедлоки и contention; сначала убрать shared state, потом синхронизировать остаток.
- Гонка ≠ deadlock: гонка — **неверный результат**; дедлок — **вечное ожидание** (см. [q4](#q4), [q8](#q8)).

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
- `Parallel.For/ForEach` — распараллеливание **CPU-bound** циклов по ThreadPool; `MaxDegreeOfParallelism` ограничивает число одновременных задач.
- `Parallel.ForEachAsync` (.NET 6+) — пакетный **async I/O** с лимитом параллелизма (не блокирует потоки на ожидании).
- PLINQ (`.AsParallel()`) — параллельный LINQ для **CPU-bound** запросов над коллекциями.

**Когда что использовать:**

| Задача | Инструмент |
|--------|------------|
| Тяжёлые вычисления в цикле (CPU) | `Parallel.For` / `Parallel.ForEach` / PLINQ |
| Много async HTTP/БД с лимитом «не больше N одновременно» | `Parallel.ForEachAsync` / `SemaphoreSlim` |
| Один async-вызов | обычный `await` |
| Независимые async без лимита | `Task.WhenAll` (см. [q18](#q18)) |
| Блокирующий sync I/O в цикле | **не** `Parallel.ForEach` — исчерпание пула |

---

#### `Parallel.For` / `Parallel.ForEach` — CPU-bound

Разбивает диапазон/коллекцию на **партиции**, каждая партиция выполняется на ThreadPool. Имеет смысл, когда **итерация тяжёлая по CPU** и их много.

```csharp
// Обработка миллиона изображений — resize, фильтры
var options = new ParallelOptions
{
    MaxDegreeOfParallelism = Environment.ProcessorCount,  // обычно = числу ядер
    CancellationToken = ct
};

Parallel.ForEach(files, options, file =>
{
    ResizeImage(file);   // CPU-bound, без await
});
```

**`MaxDegreeOfParallelism`:**
- `-1` (по умолчанию) — TPL сам решает (обычно ~число ядер).
- `1` — по сути последовательно.
- `4` — не больше 4 итераций одновременно (защита от перегрузки CPU/диска).

```csharp
Parallel.For(0, items.Length, new ParallelOptions { MaxDegreeOfParallelism = 4 }, i =>
{
    Process(items[i]);
});
```

**Когда НЕ использовать:**
- тело цикла — `await` (нет перегрузки с async-lambda в `Parallel.ForEach`);
- тело — sync I/O (`File.ReadAllText`, sync SQL) → потоки пула **блокируются**;
- мало итераций или работа дешёвая → оверхед TPL больше выигрыша;
- нужен строгий порядок side-effects без синхронизации.

**Потокобезопасность:** каждая итерация — отдельный поток → общий `List<T>` без lock **гонка**. Используйте `ConcurrentBag`, локальные списки + merge, или thread-local:

```csharp
var bag = new ConcurrentBag<Result>();
Parallel.ForEach(items, item => bag.Add(Compute(item)));
```

---

#### `Parallel.ForEachAsync` — async I/O с лимитом (.NET 6+)

Для **IO-bound**: не занимает поток на всё время ожидания, но ограничивает «сколько запросов в полёте».

```csharp
await Parallel.ForEachAsync(
    urls,
    new ParallelOptions
    {
        MaxDegreeOfParallelism = 10,   // макс. 10 одновременных await
        CancellationToken = ct
    },
    async (url, token) =>
    {
        var data = await _http.GetAsync(url, token);   // поток освобождается
        await SaveAsync(data, token);
    });
```

Аналог через `SemaphoreSlim` (см. [q8](#q8)):

```csharp
using var throttle = new SemaphoreSlim(10);
var tasks = urls.Select(async url =>
{
    await throttle.WaitAsync(ct);
    try { await ProcessUrlAsync(url, ct); }
    finally { throttle.Release(); }
});
await Task.WhenAll(tasks);
```

Оба подхода — «не больше N параллельных I/O». `ForEachAsync` — встроенный, компактный; `SemaphoreSlim` — гибче вне цикла.

**Не путать с [q18](#q18):** `Task.WhenAll` без лимита запустит **все** сразу; `ForEachAsync` / `SemaphoreSlim` — с потолком.

---

#### PLINQ — параллельный LINQ

`.AsParallel()` включает параллельное выполнение LINQ-операторов над коллекцией в памяти. Только **CPU-bound**.

```csharp
var primes = Enumerable.Range(2, 1_000_000)
    .AsParallel()
    .WithDegreeOfParallelism(4)
    .Where(IsPrime)
    .ToList();
```

**Полезные настройки:**

```csharp
source.AsParallel()
    .WithDegreeOfParallelism(8)     // лимит потоков
    .AsOrdered()                    // сохранить порядок (медленнее)
    .WithCancellation(ct)           // отмена
    .Select(x => HeavyTransform(x))
    .ToList();
```

| Оператор | Параллелится? |
|----------|---------------|
| `Where`, `Select`, `Aggregate` (осторожно) | да |
| `OrderBy` | да, но дорого |
| `Take(5)` на неупорядоченном | может дать неожиданный порядок |
| I/O внутри `Select` | **антипаттерн** — блокирует потоки |

**`AsSequential()`** — вернуться к обычному LINQ внутри цепочки.

---

#### Степень параллелизма — как выбрать

| Ресурс | Рекомендация |
|--------|--------------|
| CPU (вычисления) | `Environment.ProcessorCount` или чуть меньше |
| I/O (HTTP, БД) | отдельный лимит: 10–50, смотреть лимиты API/пула соединений |
| Диск | часто 2–4 (диск — узкое место, больше потоков ≠ быстрее) |
| ASP.NET Core | не раздувать без нужды — конкурируете с обработкой запросов за ThreadPool |

Правило: **параллелизм CPU ≈ ядра; параллелизм I/O — отдельная настройка под внешнюю систему.**

---

#### Сравнение одной задачи

```csharp
// ❌ CPU в async — лишняя стейт-машина, не ускоряет CPU
await Parallel.ForEachAsync(cpus, async (item, _) => HeavyCpu(item));

// ✅ CPU
Parallel.ForEach(cpus, item => HeavyCpu(item));

// ❌ sync I/O в Parallel.ForEach — thread pool starvation
Parallel.ForEach(urls, url => _client.GetStringAsync(url).Result);

// ✅ async I/O с лимитом
await Parallel.ForEachAsync(urls, opts, async (url, ct) => await _client.GetStringAsync(url, ct));
```

**Подводные камни:**
- `Parallel.ForEach` + блокирующий I/O — главная причина **thread pool starvation** в «параллельных» скриптах.
- `Aggregate` / `Sum` в PLINQ с неассоциативной операцией — неверный результат.
- `Parallel.ForEach` с shared mutable state без синхронизации — гонки (см. [q10](#q10)).
- Слишком высокий `MaxDegreeOfParallelism` на I/O — DDoS своей БД / исчерпание connection pool.
- В UI-приложении тяжёлый `Parallel.*` без offload может подвисать интерфейс — для UI иногда `Task.Run` + прогресс.

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
- **`System.Threading.Channels`** — высокопроизводительная **async-очередь** (producer → consumer) внутри процесса; не broadcast «всем подписчикам».
- **Bounded channel** — backpressure: producer ждёт, если очередь полна.
- **TPL Dataflow** — блоки-конвейеры (`TransformBlock`, `ActionBlock`) для многоступенчатых пайплайнов.

**Главное про модель — не путать с pub/sub:**

| | `Channel<T>` | Pub/Sub (события, message bus) |
|---|-------------|-------------------------------|
| Кто получает сообщение | **Один** consumer (при нескольких — *competing consumers*: кто первый `Read`, тот и забрал) | **Все** подписчики получают копию |
| Порядок | FIFO (очередь) | зависит от шины |
| Область | один процесс / один инстанс | может быть между сервисами |
| Backpressure | встроен (bounded) | зависит от брокера |

`Channel` — это **очередь задач/сообщений**, а не «рассылка всем слушателям». Один `Work` обработает **один** consumer.

```
Producer A ──┐
Producer B ──┼──► [ Channel: W1 W2 W3 ] ──► Consumer 1 (взял W1)
             │                              Consumer 2 (взял W2)  ← competing, не broadcast
             └── WriteAsync                     Consumer 3 (взял W3)
```

---

#### `Channel<T>` — подробнее

**Зачем:** связать producer и consumer **асинхронно** без блокировки потоков и без ручного `lock` + `ConcurrentQueue` + `ManualResetEvent`.

**Создание:**

```csharp
// Ограниченная очередь — backpressure
var channel = Channel.CreateBounded<Work>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait   // producer ждёт, пока есть место
});

// Без лимита — producer никогда не ждёт (риск OutOfMemory при медленном consumer)
var unbounded = Channel.CreateUnbounded<Work>();
```

**Producer (один или несколько потоков):**

```csharp
await channel.Writer.WriteAsync(work, ct);   // async, не блокирует поток на lock

// когда producers закончили:
channel.Writer.Complete();   // сигнал consumer: новых данных не будет
```

**Consumer (обычно один, реже несколько competing):**

```csharp
await foreach (var work in channel.Reader.ReadAllAsync(ct))
{
    await ProcessAsync(work, ct);
}
// ReadAllAsync завершится после Complete() и опустошения очереди
```

**Низкоуровневое чтение:**

```csharp
while (await channel.Reader.WaitToReadAsync(ct))
{
    while (channel.Reader.TryRead(out var item))
        await HandleAsync(item, ct);
}
```

**Backpressure (зачем bounded):**

```
Producer быстрый, Consumer медленный:

Unbounded: очередь растёт → память ↑↑↑
Bounded(100): после 100 элементов WriteAsync ждёт → producer тормозит сам
```

Это и есть **консистентность нагрузки** в рамках инстанса: система не захлёбывается, а **тормозит производителя**.

**Несколько writers / readers:**

| Конфигурация | Поведение |
|--------------|-----------|
| N writers → 1 reader | типичный fan-in: много источников, один обработчик |
| 1 writer → 1 reader | классический pipeline |
| N writers → M readers | **competing consumers**: каждый элемент — **одному** reader; порядок между readers не гарантирован |
| Нужен broadcast всем | `Channel` **не подходит** → события, `IObservable`, отдельный fan-out |

Оптимизация при одном читателе:

```csharp
Channel.CreateBounded<Work>(new BoundedChannelOptions(100) { SingleReader = true, SingleWriter = false });
```

**Где используют в .NET:**
- фоновая обработка в `BackgroundService` (HTTP принял → в channel → воркер обрабатывает);
- ограничение параллелизма I/O (producer пишет быстро, consumer с лимитом);
- замена `BlockingCollection` в async-коде;
- внутренности Kestrel / Pipelines (концептуально похожий паттерн).

**Channel vs `ConcurrentQueue`:**

| | `Channel<T>` | `ConcurrentQueue<T>` |
|---|-------------|----------------------|
| Async wait при пустой/полной очереди | `ReadAsync` / `WriteAsync` — **да** | только `TryDequeue` / spin-wait |
| Backpressure | bounded из коробки | нет |
| Complete / завершение потока | `Writer.Complete()` | нет |

---

#### Полный пример — ASP.NET + BackgroundService

```csharp
// Singleton: очередь на весь инстанс
public class WorkChannel
{
    public Channel<WorkItem> Channel { get; } =
        Channel.CreateBounded<WorkItem>(100);
}

// Controller — producer
[HttpPost]
public async Task<IActionResult> Enqueue(WorkItem item, CancellationToken ct)
{
    await _workChannel.Channel.Writer.WriteAsync(item, ct);
    return Accepted();
}

// BackgroundService — consumer
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    await foreach (var item in _workChannel.Channel.Reader.ReadAllAsync(stoppingToken))
        await ProcessAsync(item, stoppingToken);
}
```

Консистентность: каждый `WorkItem` обработается **ровно одним** воркером (если один consumer). Это не «все подписчики получили», а «задача попала в очередь и будет обработана один раз».

---

#### TPL Dataflow — конвейер из блоков

Когда одной очереди мало — нужен **пайплайн**: получить → преобразовать → сохранить, с разной степенью параллелизма на каждом этапе.

```csharp
var transform = new TransformBlock<int, string>(
    n => ExpensiveTransform(n),
    new ExecutionDataflowBlockOptions { MaxDegreeOfParallelism = 4 });

var action = new ActionBlock<string>(
    async s => await SaveAsync(s),
    new ExecutionDataflowBlockOptions { MaxDegreeOfParallelism = 10 });

transform.LinkTo(action, new DataflowLinkOptions { PropagateCompletion = true });

foreach (var n in numbers)
    await transform.SendAsync(n);

transform.Complete();
await action.Completion;
```

| Блок | Роль |
|------|------|
| `BufferBlock<T>` | буфер / очередь между этапами |
| `TransformBlock<TIn,TOut>` | преобразование (как map) |
| `ActionBlock<T>` | финальный side-effect (save, log) |
| `BroadcastBlock<T>` | **вот тут** fan-out — **один** вход, **много** выходов (всем подписчикам) |

Dataflow тяжелее `Channel`, но удобен для многоступенчатых pipeline с лимитами на каждом блоке.

**Когда что:**
- одна очередь producer/consumer → **`Channel<T>`**;
- несколько этапов обработки с разным parallelism → **TPL Dataflow**;
- broadcast всем подписчикам → **не Channel**, а `BroadcastBlock` / события / message bus.

**Подводные камни:**
- Забыть `Writer.Complete()` — consumer ждёт вечно.
- Несколько consumers без понимания — каждый элемент только одному; для «всем разослать» нужен другой паттерн.
- Unbounded channel при быстром producer — риск OOM.
- `Channel` не переживает рестарт и не шарится между pod'ами — для кластера нужна внешняя очередь (RabbitMQ, Kafka, Azure Queue).
- Исключение в consumer — обработать в `try/catch`, иначе цикл `ReadAllAsync` прервётся.

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
