# Индекс вопросов по темам

> Сводный список всех вопросов и подтем из папки [`topics/`](./topics). Нужен, чтобы при добавлении нового вопроса с собеседования быстро найти подходящую тему и проверить, нет ли уже похожего — не перечитывая все файлы.
>
> Клик по пункту открывает разбор (где расставлены якоря — сразу нужный раздел). ⭐ — реальные вопросы, которые задавали на собеседованиях.

[← К оглавлению (README)](./README.md)

---

### 01. C# язык и его возможности

<sub>[topics/language-platform/01-csharp-language.md](./topics/language-platform/01-csharp-language.md)</sub>

- [Value type vs reference type — где живут, как передаются, boxing/unboxing](./topics/language-platform/01-csharp-language.md#q1)
- [`struct` vs `class` vs `record` vs `record struct` — когда что](./topics/language-platform/01-csharp-language.md#q2)
- [`string` — иммутабельность, интернирование, `StringBuilder`, `Span<char>`](./topics/language-platform/01-csharp-language.md#q3)
- [`ref`, `out`, `in`, `ref readonly`, `ref struct`](./topics/language-platform/01-csharp-language.md#q4)
- [Замыкания (closures) — захват переменных, баги в циклах](./topics/language-platform/01-csharp-language.md#q5)
- [Делегаты, `Func`/`Action`/`Predicate`, события](./topics/language-platform/01-csharp-language.md#q6)
- [`IEnumerable` vs `IQueryable`, отложенное выполнение, `yield`](./topics/language-platform/01-csharp-language.md#q7)
- [`IDisposable`, `using`, `IAsyncDisposable`, финализаторы](./topics/language-platform/01-csharp-language.md#q8)
- [Generics — ковариантность/контравариантность, ограничения](./topics/language-platform/01-csharp-language.md#q9)
- [Nullable reference types, `?`, `!`, `??`, `??=`](./topics/language-platform/01-csharp-language.md#q10)
- [Pattern matching и switch expressions](./topics/language-platform/01-csharp-language.md#q11)
- [`record` — value equality, `with`, деконструкция](./topics/language-platform/01-csharp-language.md#q12)
- [Extension methods — как резолвятся](./topics/language-platform/01-csharp-language.md#q13)
- [`dynamic`, `var`, `object` — различия](./topics/language-platform/01-csharp-language.md#q14)
- [Перегрузка операторов и приведения](./topics/language-platform/01-csharp-language.md#q16)
- [Атрибуты, рефлексия, Source Generators](./topics/language-platform/01-csharp-language.md#q17)

### 02. CLR, GC и управление памятью

<sub>[topics/language-platform/02-clr-gc-memory.md](./topics/language-platform/02-clr-gc-memory.md)</sub>

- [Стек vs управляемая куча — что где хранится](./topics/language-platform/02-clr-gc-memory.md#q1)
- [Поколения GC, LOH, фрагментация](./topics/language-platform/02-clr-gc-memory.md#q2)
- [Workstation vs Server GC, Concurrent/Background GC](./topics/language-platform/02-clr-gc-memory.md#q3)
- [Как работает mark-and-sweep + compaction](./topics/language-platform/02-clr-gc-memory.md#q4)
- [Корни GC (roots)](./topics/language-platform/02-clr-gc-memory.md#q5)
- [Финализация, очередь финализации, `GC.SuppressFinalize`](./topics/language-platform/02-clr-gc-memory.md#q6)
- [`IDisposable` и Dispose pattern vs финализатор](./topics/language-platform/02-clr-gc-memory.md#q7)
- [Boxing/unboxing — где аллокации, как избежать](./topics/language-platform/02-clr-gc-memory.md#q8)
- [`Span<T>`, `Memory<T>`, `stackalloc`, `ArrayPool<T>`](./topics/language-platform/02-clr-gc-memory.md#q9)
- [JIT, AOT, Tiered Compilation, ReadyToRun](./topics/language-platform/02-clr-gc-memory.md#q10)
- [Утечки памяти в .NET — причины](./topics/language-platform/02-clr-gc-memory.md#q11)
- [Инструменты диагностики](./topics/language-platform/02-clr-gc-memory.md#q12)
- [GC pauses, latency modes, `GCSettings`](./topics/language-platform/02-clr-gc-memory.md#q13)
- [Pinned objects, `fixed`, `GCHandle`](./topics/language-platform/02-clr-gc-memory.md#q14)
- [Метаданные сборок, AssemblyLoadContext, выгрузка](./topics/language-platform/02-clr-gc-memory.md#q15)

### 03. Многопоточность и асинхронность

<sub>[topics/language-platform/03-async-multithreading.md](./topics/language-platform/03-async-multithreading.md)</sub>

- [`Thread` vs `ThreadPool` vs `Task` vs `async/await`](./topics/language-platform/03-async-multithreading.md#q1)
- [Как работает `async/await` под капотом](./topics/language-platform/03-async-multithreading.md#q2)
- [`SynchronizationContext` и `ConfigureAwait(false)`](./topics/language-platform/03-async-multithreading.md#q3)
- [Дедлок при `.Result`/`.Wait()` (классика)](./topics/language-platform/03-async-multithreading.md#q4)
- [`Task` vs `ValueTask`](./topics/language-platform/03-async-multithreading.md#q5)
- [CPU-bound vs IO-bound, когда `Task.Run`](./topics/language-platform/03-async-multithreading.md#q6)
- [Отмена — `CancellationToken`](./topics/language-platform/03-async-multithreading.md#q7)
- [Примитивы синхронизации](./topics/language-platform/03-async-multithreading.md#q8)
- [`Interlocked`, `volatile`, memory barriers](./topics/language-platform/03-async-multithreading.md#q9)
- [Race conditions — обнаружение и предотвращение](./topics/language-platform/03-async-multithreading.md#q10)
- [`async void` — почему опасен](./topics/language-platform/03-async-multithreading.md#q11)
- [Исключения в async, `AggregateException`, `WhenAll`](./topics/language-platform/03-async-multithreading.md#q12)
- [`Parallel.For/ForEach`, PLINQ, степень параллелизма](./topics/language-platform/03-async-multithreading.md#q13)
- [`IAsyncEnumerable`, `await foreach`](./topics/language-platform/03-async-multithreading.md#q14)
- [`Channel<T>`, producer/consumer, TPL Dataflow](./topics/language-platform/03-async-multithreading.md#q15)
- [Потокобезопасные коллекции, нюансы `ConcurrentDictionary`](./topics/language-platform/03-async-multithreading.md#q16)
- [Async от начала до конца](./topics/language-platform/03-async-multithreading.md#q17)

### 04. Коллекции и структуры данных

<sub>[topics/language-platform/04-collections-data-structures.md](./topics/language-platform/04-collections-data-structures.md)</sub>

- [`List<T>` — устройство и рост capacity](./topics/language-platform/04-collections-data-structures.md#q1)
- [`Dictionary<K,V>` — хеш-таблица, коллизии](./topics/language-platform/04-collections-data-structures.md#q2)
- [`HashSet`, `SortedDictionary`, `SortedSet`](./topics/language-platform/04-collections-data-structures.md#q3)
- [`Array` vs `List<T>`, многомерные и jagged](./topics/language-platform/04-collections-data-structures.md#q4)
- [`LinkedList<T>` — когда нужен](./topics/language-platform/04-collections-data-structures.md#q5)
- [`Queue`, `Stack`, `PriorityQueue`](./topics/language-platform/04-collections-data-structures.md#q6)
- [Потокобезопасные коллекции](./topics/language-platform/04-collections-data-structures.md#q7)
- [Иерархия интерфейсов коллекций](./topics/language-platform/04-collections-data-structures.md#q8)
- [Immutable collections](./topics/language-platform/04-collections-data-structures.md#q9)
- [`Span<T>` поверх коллекций](./topics/language-platform/04-collections-data-structures.md#q10)
- [Корректные `Equals`/`GetHashCode` для ключей](./topics/language-platform/04-collections-data-structures.md#q11)
- [`IEnumerator`, `yield`, ленивость](./topics/language-platform/04-collections-data-structures.md#q12)
- [LINQ и материализация, множественный перебор](./topics/language-platform/04-collections-data-structures.md#q13)

### 05. ООП, SOLID и принципы проектирования

<sub>[topics/architecture/05-oop-solid-principles.md](./topics/architecture/05-oop-solid-principles.md)</sub>

- [4 столпа ООП](./topics/architecture/05-oop-solid-principles.md#q1)
- [Наследование vs композиция](./topics/architecture/05-oop-solid-principles.md#q2)
- [S — Single Responsibility](./topics/architecture/05-oop-solid-principles.md#q3)
- [O — Open/Closed](./topics/architecture/05-oop-solid-principles.md#q4)
- [L — Liskov Substitution](./topics/architecture/05-oop-solid-principles.md#q5)
- [I — Interface Segregation](./topics/architecture/05-oop-solid-principles.md#q6)
- [D — Dependency Inversion vs DI vs IoC](./topics/architecture/05-oop-solid-principles.md#q7)
- [DRY, KISS, YAGNI](./topics/architecture/05-oop-solid-principles.md#q8)
- [Закон Деметры (Law of Demeter)](./topics/architecture/05-oop-solid-principles.md#q9)
- [Tell, Don't Ask](./topics/architecture/05-oop-solid-principles.md#q10)
- [Coupling и Cohesion](./topics/architecture/05-oop-solid-principles.md#q11)
- [Абстрактные классы vs интерфейсы](./topics/architecture/05-oop-solid-principles.md#q12)
- [Виртуальные методы, `sealed`, `new`](./topics/architecture/05-oop-solid-principles.md#q13)
- [Полиморфизм статический vs динамический](./topics/architecture/05-oop-solid-principles.md#q14)

### 06. Паттерны проектирования

<sub>[topics/architecture/06-design-patterns.md](./topics/architecture/06-design-patterns.md)</sub>

- [Singleton](./topics/architecture/06-design-patterns.md)
- [Factory Method / Abstract Factory](./topics/architecture/06-design-patterns.md)
- [Builder](./topics/architecture/06-design-patterns.md)
- [Prototype](./topics/architecture/06-design-patterns.md)
- [Adapter](./topics/architecture/06-design-patterns.md)
- [Decorator](./topics/architecture/06-design-patterns.md)
- [Facade](./topics/architecture/06-design-patterns.md)
- [Proxy](./topics/architecture/06-design-patterns.md)
- [Composite](./topics/architecture/06-design-patterns.md)
- [Strategy](./topics/architecture/06-design-patterns.md)
- [Observer](./topics/architecture/06-design-patterns.md)
- [Command](./topics/architecture/06-design-patterns.md)
- [Mediator](./topics/architecture/06-design-patterns.md)
- [Chain of Responsibility](./topics/architecture/06-design-patterns.md)
- [State / Template Method](./topics/architecture/06-design-patterns.md)
- [Repository + Unit of Work](./topics/architecture/06-design-patterns.md)
- [Dependency Injection](./topics/architecture/06-design-patterns.md)
- [Options pattern](./topics/architecture/06-design-patterns.md)
- [Result pattern](./topics/architecture/06-design-patterns.md)
- [Specification pattern](./topics/architecture/06-design-patterns.md)

### 07. Архитектура приложений

<sub>[topics/architecture/07-architecture.md](./topics/architecture/07-architecture.md)</sub>

- [Layered (N-tier) архитектура](./topics/architecture/07-architecture.md#q1)
- [Clean / Onion / Hexagonal](./topics/architecture/07-architecture.md#q2)
- [Dependency Rule](./topics/architecture/07-architecture.md#q3)
- [Монолит vs модульный монолит vs микросервисы](./topics/architecture/07-architecture.md#q4)
- [Anemic vs rich domain model](./topics/architecture/07-architecture.md#q5)
- [Vertical Slice Architecture](./topics/architecture/07-architecture.md#q6)
- [DTO vs Entity vs Domain vs ViewModel, маппинг](./topics/architecture/07-architecture.md#q7)
- [Cross-cutting concerns](./topics/architecture/07-architecture.md#q8)
- [12-factor app](./topics/architecture/07-architecture.md#q9)
- [Strangler Fig](./topics/architecture/07-architecture.md#q10)
- [ADR (Architecture Decision Records)](./topics/architecture/07-architecture.md#q11)
- [Conway's Law](./topics/architecture/07-architecture.md#q12)

### 08. DDD (Domain-Driven Design)

<sub>[topics/architecture/08-ddd.md](./topics/architecture/08-ddd.md)</sub>

- [Ubiquitous Language](./topics/architecture/08-ddd.md)
- [Bounded Context](./topics/architecture/08-ddd.md)
- [Context Map](./topics/architecture/08-ddd.md)
- [Subdomains](./topics/architecture/08-ddd.md)
- [Anti-Corruption Layer](./topics/architecture/08-ddd.md)
- [Entity vs Value Object](./topics/architecture/08-ddd.md)
- [Aggregate и Aggregate Root](./topics/architecture/08-ddd.md)
- [Правила агрегатов](./topics/architecture/08-ddd.md)
- [Domain Events](./topics/architecture/08-ddd.md)
- [Repository](./topics/architecture/08-ddd.md)
- [Domain Service vs Application Service](./topics/architecture/08-ddd.md)
- [Factory, Invariants, Specification](./topics/architecture/08-ddd.md)
- [DDD + EF Core](./topics/architecture/08-ddd.md)
- [Anemic vs Rich](./topics/architecture/08-ddd.md)
- [Когда DDD избыточен](./topics/architecture/08-ddd.md)
- [Eventual consistency между агрегатами](./topics/architecture/08-ddd.md)

### 09. CQRS и Event Sourcing

<sub>[topics/architecture/09-cqrs-event-sourcing.md](./topics/architecture/09-cqrs-event-sourcing.md)</sub>

- [Что такое CQRS](./topics/architecture/09-cqrs-event-sourcing.md)
- [CQRS != две базы данных](./topics/architecture/09-cqrs-event-sourcing.md)
- [Реализация через MediatR](./topics/architecture/09-cqrs-event-sourcing.md)
- [Раздельные read-модели](./topics/architecture/09-cqrs-event-sourcing.md)
- [Синхронизация read/write, eventual consistency](./topics/architecture/09-cqrs-event-sourcing.md)
- [Когда CQRS избыточен](./topics/architecture/09-cqrs-event-sourcing.md)
- [Валидация команд](./topics/architecture/09-cqrs-event-sourcing.md)
- [Что такое Event Sourcing](./topics/architecture/09-cqrs-event-sourcing.md)
- [Event Store, replay, snapshots](./topics/architecture/09-cqrs-event-sourcing.md)
- [Проекции](./topics/architecture/09-cqrs-event-sourcing.md)
- [Версионирование событий](./topics/architecture/09-cqrs-event-sourcing.md)
- [CQRS + ES вместе](./topics/architecture/09-cqrs-event-sourcing.md)
- [Идемпотентность / дедупликация](./topics/architecture/09-cqrs-event-sourcing.md)
- [Минусы ES и инструменты](./topics/architecture/09-cqrs-event-sourcing.md)

### 10. ASP.NET Core и Web API

<sub>[topics/web-data/10-aspnet-core.md](./topics/web-data/10-aspnet-core.md)</sub>

- [Generic Host / WebApplication](./topics/web-data/10-aspnet-core.md#q1)
- [Middleware pipeline](./topics/web-data/10-aspnet-core.md#q2)
- [DI и lifetimes, captive dependency](./topics/web-data/10-aspnet-core.md#q3)
- [Жизненный цикл scope](./topics/web-data/10-aspnet-core.md#q4)
- [Configuration и Options pattern](./topics/web-data/10-aspnet-core.md#q5)
- [Controllers vs Minimal APIs](./topics/web-data/10-aspnet-core.md#q6)
- [Model binding и валидация](./topics/web-data/10-aspnet-core.md#q7)
- [Фильтры vs middleware](./topics/web-data/10-aspnet-core.md#q8)
- [Routing](./topics/web-data/10-aspnet-core.md#q9)
- [REST, статус-коды, версионирование](./topics/web-data/10-aspnet-core.md#q10)
- [ProblemDetails](./topics/web-data/10-aspnet-core.md#q11)
- [Глобальная обработка ошибок](./topics/web-data/10-aspnet-core.md#q12)
- [HttpClientFactory, Polly](./topics/web-data/10-aspnet-core.md#q13)
- [CORS, content negotiation](./topics/web-data/10-aspnet-core.md#q14)
- [Health checks, BackgroundService](./topics/web-data/10-aspnet-core.md#q15)
- [OpenAPI/Swagger](./topics/web-data/10-aspnet-core.md#q16)

### 11. Конвейер обработки запросов и HTTP в Highload

<sub>[topics/web-data/11-request-pipeline-http-highload.md](./topics/web-data/11-request-pipeline-http-highload.md)</sub>

- [Конвейер обработки запроса](./topics/web-data/11-request-pipeline-http-highload.md)
- [HTTP и протоколы](./topics/web-data/11-request-pipeline-http-highload.md)
- [JSON и сериализация под нагрузкой](./topics/web-data/11-request-pipeline-http-highload.md)
- [Highload-практики](./topics/web-data/11-request-pipeline-http-highload.md)
- [⭐ Вопрос: Подробно опиши, как работает HTTP-конвейер обработки запроса в .NET Core](./topics/web-data/11-request-pipeline-http-highload.md)
- [⭐ Вопрос: Где применяются `Span<T>`/`Memory<T>` при работе с HTTP и зачем](./topics/web-data/11-request-pipeline-http-highload.md)
- [Порядок middleware — почему критичен](./topics/web-data/11-request-pipeline-http-highload.md)
- [Kestrel — модель потоков, async IO](./topics/web-data/11-request-pipeline-http-highload.md)
- [Reverse proxy перед Kestrel — зачем](./topics/web-data/11-request-pipeline-http-highload.md)
- [`HttpContext`, потоки тела](./topics/web-data/11-request-pipeline-http-highload.md)
- [Backpressure, лимиты, таймауты](./topics/web-data/11-request-pipeline-http-highload.md)
- [HTTP/1.1 vs HTTP/2 vs HTTP/3](./topics/web-data/11-request-pipeline-http-highload.md)
- [Keep-alive, пулинг, socket exhaustion, `HttpClientFactory`](./topics/web-data/11-request-pipeline-http-highload.md)
- [Chunked transfer, streaming, `IAsyncEnumerable`](./topics/web-data/11-request-pipeline-http-highload.md)
- [`System.Text.Json` vs `Newtonsoft.Json`](./topics/web-data/11-request-pipeline-http-highload.md)
- [Source-generated JSON — почему быстрее](./topics/web-data/11-request-pipeline-http-highload.md)
- [Аллокации при (де)сериализации, `Utf8JsonReader`/`Writer`](./topics/web-data/11-request-pipeline-http-highload.md)
- [Стриминговая сериализация больших ответов](./topics/web-data/11-request-pipeline-http-highload.md)
- [Сжатие ответов](./topics/web-data/11-request-pipeline-http-highload.md)
- [Снижение размера payload](./topics/web-data/11-request-pipeline-http-highload.md)
- [Минимизация аллокаций на горячем пути](./topics/web-data/11-request-pipeline-http-highload.md)
- [Кэширование на уровне HTTP](./topics/web-data/11-request-pipeline-http-highload.md)
- [Нагрузочное тестирование и поиск bottleneck](./topics/web-data/11-request-pipeline-http-highload.md)
- [Thread pool starvation](./topics/web-data/11-request-pipeline-http-highload.md)
- [Rate limiting](./topics/web-data/11-request-pipeline-http-highload.md)

### 12. gRPC и контракты API (REST/gRPC/GraphQL)

<sub>[topics/web-data/12-grpc-api-contracts.md](./topics/web-data/12-grpc-api-contracts.md)</sub>

- [Основа: HTTP/2 + Protocol Buffers](./topics/web-data/12-grpc-api-contracts.md)
- [4 типа вызовов](./topics/web-data/12-grpc-api-contracts.md)
- [Почему gRPC быстрее REST/JSON](./topics/web-data/12-grpc-api-contracts.md)
- [Версионирование protobuf](./topics/web-data/12-grpc-api-contracts.md)
- [Deadlines, отмена, interceptors, метаданные](./topics/web-data/12-grpc-api-contracts.md)
- [gRPC-Web](./topics/web-data/12-grpc-api-contracts.md)
- [gRPC vs REST — когда что](./topics/web-data/12-grpc-api-contracts.md)
- [Уровни зрелости (Richardson) и HATEOAS](./topics/web-data/12-grpc-api-contracts.md)
- [Идемпотентность и безопасность методов](./topics/web-data/12-grpc-api-contracts.md)
- [Версионирование REST](./topics/web-data/12-grpc-api-contracts.md)
- [Пагинация/фильтрация](./topics/web-data/12-grpc-api-contracts.md)
- [Когда уместен](./topics/web-data/12-grpc-api-contracts.md)
- [N+1 и DataLoader](./topics/web-data/12-grpc-api-contracts.md)
- [HotChocolate](./topics/web-data/12-grpc-api-contracts.md)
- [Contract-first vs code-first](./topics/web-data/12-grpc-api-contracts.md)
- [Совместимость и contract testing](./topics/web-data/12-grpc-api-contracts.md)

### 13. Entity Framework Core и доступ к данным

<sub>[topics/web-data/13-ef-core-data-access.md](./topics/web-data/13-ef-core-data-access.md)</sub>

- [Change Tracking, DbContext, время жизни](./topics/web-data/13-ef-core-data-access.md#q1)
- [`AsNoTracking()`](./topics/web-data/13-ef-core-data-access.md#q2)
- [Eager / lazy / explicit loading](./topics/web-data/13-ef-core-data-access.md#q3)
- [Проблема N+1](./topics/web-data/13-ef-core-data-access.md#q4)
- [`IQueryable` vs `IEnumerable`, client vs server eval](./topics/web-data/13-ef-core-data-access.md#q5)
- [Отложенное выполнение, материализация](./topics/web-data/13-ef-core-data-access.md#q6)
- [Проекции](./topics/web-data/13-ef-core-data-access.md#q7)
- [Транзакции, SaveChanges, потокобезопасность](./topics/web-data/13-ef-core-data-access.md#q8)
- [Optimistic concurrency](./topics/web-data/13-ef-core-data-access.md#q9)
- [Миграции](./topics/web-data/13-ef-core-data-access.md#q10)
- [Связи и конфигурация](./topics/web-data/13-ef-core-data-access.md#q11)
- [Compiled queries, ExecuteUpdate/ExecuteDelete](./topics/web-data/13-ef-core-data-access.md#q12)
- [Split queries](./topics/web-data/13-ef-core-data-access.md#q13)
- [Connection pooling, DbContextFactory](./topics/web-data/13-ef-core-data-access.md#q14)
- [Когда raw SQL / Dapper](./topics/web-data/13-ef-core-data-access.md#q15)
- [Repository поверх EF](./topics/web-data/13-ef-core-data-access.md#q16)

### 14. Базы данных и SQL

<sub>[topics/web-data/14-databases-sql.md](./topics/web-data/14-databases-sql.md)</sub>

- [Индексы](./topics/web-data/14-databases-sql.md)
- [Когда индекс не используется, селективность](./topics/web-data/14-databases-sql.md)
- [План выполнения, seek vs scan](./topics/web-data/14-databases-sql.md)
- [JOIN, подзапросы, EXISTS vs IN](./topics/web-data/14-databases-sql.md)
- [Оконные функции, CTE](./topics/web-data/14-databases-sql.md)
- [Нормализация/денормализация](./topics/web-data/14-databases-sql.md)
- [ACID](./topics/web-data/14-databases-sql.md)
- [Уровни изоляции и аномалии](./topics/web-data/14-databases-sql.md)
- [Блокировки, deadlock](./topics/web-data/14-databases-sql.md)
- [Optimistic vs pessimistic locking](./topics/web-data/14-databases-sql.md)
- [Репликация](./topics/web-data/14-databases-sql.md)
- [Шардирование и партиционирование](./topics/web-data/14-databases-sql.md)
- [CAP](./topics/web-data/14-databases-sql.md)
- [Типы и выбор](./topics/web-data/14-databases-sql.md)
- [BASE vs ACID](./topics/web-data/14-databases-sql.md)

### 15. Микросервисы и распределённые системы

<sub>[topics/distributed/15-microservices.md](./topics/distributed/15-microservices.md)</sub>

- [Микросервисы vs монолит, когда НЕ дробить](./topics/distributed/15-microservices.md#q1)
- [Границы сервисов](./topics/distributed/15-microservices.md#q2)
- [Sync vs async взаимодействие](./topics/distributed/15-microservices.md#q3)
- [CAP, PACELC](./topics/distributed/15-microservices.md#q4)
- [Strong vs eventual consistency](./topics/distributed/15-microservices.md#q5)
- [Почему 2PC (two-phase commit) плохо](./topics/distributed/15-microservices.md#q6)
- [Saga](./topics/distributed/15-microservices.md#q7)
- [Outbox pattern](./topics/distributed/15-microservices.md#q8)
- [Inbox / идемпотентность потребителя](./topics/distributed/15-microservices.md#q9)
- [Resilience (retry, circuit breaker, bulkhead, timeout)](./topics/distributed/15-microservices.md#q10)
- [Идемпотентность](./topics/distributed/15-microservices.md#q11)
- [API Gateway, BFF](./topics/distributed/15-microservices.md#q12)
- [Service Discovery](./topics/distributed/15-microservices.md#q13)
- [Distributed tracing](./topics/distributed/15-microservices.md#q14)
- [Версионирование сервисов и контрактов](./topics/distributed/15-microservices.md#q15)
- [Data ownership, database-per-service](./topics/distributed/15-microservices.md#q16)
- [Service mesh](./topics/distributed/15-microservices.md#q17)

### 16. Брокеры сообщений и интеграция

<sub>[topics/distributed/16-messaging-integration.md](./topics/distributed/16-messaging-integration.md)</sub>

- [Зачем брокер сообщений](./topics/distributed/16-messaging-integration.md#q1)
- [Queue vs publish/subscribe](./topics/distributed/16-messaging-integration.md#q2)
- [RabbitMQ](./topics/distributed/16-messaging-integration.md#q3)
- [Kafka](./topics/distributed/16-messaging-integration.md#q4)
- [RabbitMQ vs Kafka](./topics/distributed/16-messaging-integration.md#q5)
- [Гарантии доставки](./topics/distributed/16-messaging-integration.md#q6)
- [Идемпотентность/дедупликация](./topics/distributed/16-messaging-integration.md#q7)
- [Порядок сообщений](./topics/distributed/16-messaging-integration.md#q8)
- [Ack/nack, prefetch](./topics/distributed/16-messaging-integration.md#q9)
- [DLQ, retry с backoff](./topics/distributed/16-messaging-integration.md#q10)
- [Poison messages](./topics/distributed/16-messaging-integration.md#q11)
- [Outbox](./topics/distributed/16-messaging-integration.md#q12)
- [Backpressure](./topics/distributed/16-messaging-integration.md#q13)
- [Сериализация, Schema Registry](./topics/distributed/16-messaging-integration.md#q14)
- [MassTransit / NServiceBus](./topics/distributed/16-messaging-integration.md#q15)
- [Event-driven, события vs команды vs сообщения](./topics/distributed/16-messaging-integration.md#q16)
- [Облачные аналоги](./topics/distributed/16-messaging-integration.md#q17)

### 17. Кэширование (in-memory, Redis)

<sub>[topics/distributed/17-caching.md](./topics/distributed/17-caching.md)</sub>

- [Зачем кэш, что кэшировать](./topics/distributed/17-caching.md#q1)
- [In-memory vs distributed (Redis)](./topics/distributed/17-caching.md#q2)
- [Уровни кэширования](./topics/distributed/17-caching.md#q3)
- [HTTP-кэширование](./topics/distributed/17-caching.md#q4)
- [Стратегии кэширования](./topics/distributed/17-caching.md#q5)
- [Инвалидация](./topics/distributed/17-caching.md#q6)
- [TTL, sliding vs absolute](./topics/distributed/17-caching.md#q7)
- [Eviction policies](./topics/distributed/17-caching.md#q8)
- [Cache stampede (thundering herd)](./topics/distributed/17-caching.md#q9)
- [Stale data, согласованность](./topics/distributed/17-caching.md#q10)
- [Redis — структуры, persistence, кластеризация](./topics/distributed/17-caching.md#q11)
- [Распределённые блокировки (Redlock)](./topics/distributed/17-caching.md#q12)
- [Сериализация, размер значений](./topics/distributed/17-caching.md#q13)
- [Hot keys](./topics/distributed/17-caching.md#q14)
- [HybridCache (.NET 9)](./topics/distributed/17-caching.md#q15)

### 18. Тестирование (unit, integration, моки)

<sub>[topics/quality-ops/18-testing.md](./topics/quality-ops/18-testing.md)</sub>

- [Пирамида тестирования](./topics/quality-ops/18-testing.md#q1)
- [Unit test, AAA](./topics/quality-ops/18-testing.md#q2)
- [xUnit / NUnit / MSTest](./topics/quality-ops/18-testing.md#q3)
- [Моки/стабы/фейки/спаи](./topics/quality-ops/18-testing.md#q4)
- [Mock vs Stub, state vs interaction testing](./topics/quality-ops/18-testing.md#q5)
- [Что НЕ мокать](./topics/quality-ops/18-testing.md#q6)
- [Integration tests (WebApplicationFactory)](./topics/quality-ops/18-testing.md#q7)
- [Тесты с БД (Testcontainers)](./topics/quality-ops/18-testing.md#q8)
- [TDD](./topics/quality-ops/18-testing.md#q9)
- [Тестируемый дизайн](./topics/quality-ops/18-testing.md#q10)
- [Покрытие кода](./topics/quality-ops/18-testing.md#q11)
- [FluentAssertions](./topics/quality-ops/18-testing.md#q12)
- [Параметризованные тесты](./topics/quality-ops/18-testing.md#q13)
- [Async-тесты](./topics/quality-ops/18-testing.md#q14)
- [Snapshot/approval testing](./topics/quality-ops/18-testing.md#q15)
- [Flaky tests](./topics/quality-ops/18-testing.md#q16)
- [Нагрузочное тестирование](./topics/quality-ops/18-testing.md#q17)
- [Mutation testing](./topics/quality-ops/18-testing.md#q18)

### 19. Безопасность (auth, OWASP)

<sub>[topics/quality-ops/19-security.md](./topics/quality-ops/19-security.md)</sub>

- [⭐ Вопрос: Как обеспечить безопасность при работе по HTTP](./topics/quality-ops/19-security.md)
- [Аутентификация vs авторизация](./topics/quality-ops/19-security.md#q1)
- [Cookie vs token auth](./topics/quality-ops/19-security.md#q2)
- [JWT — структура и валидация](./topics/quality-ops/19-security.md#q3)
- [Access vs refresh token](./topics/quality-ops/19-security.md#q4)
- [OAuth 2.0 flows](./topics/quality-ops/19-security.md#q5)
- [OpenID Connect](./topics/quality-ops/19-security.md#q6)
- [IdentityServer / Keycloak / Entra ID](./topics/quality-ops/19-security.md#q7)
- [Authorization в ASP.NET Core](./topics/quality-ops/19-security.md#q8)
- [Где хранить токены](./topics/quality-ops/19-security.md#q9)
- [SQL injection, XSS, CSRF](./topics/quality-ops/19-security.md#q10)
- [Broken access control (IDOR)](./topics/quality-ops/19-security.md#q11)
- [Шифрование at-rest/in-transit](./topics/quality-ops/19-security.md#q12)
- [Хеширование паролей](./topics/quality-ops/19-security.md#q13)
- [Secrets management](./topics/quality-ops/19-security.md#q14)
- [Rate limiting](./topics/quality-ops/19-security.md#q15)
- [Mass assignment / over-posting](./topics/quality-ops/19-security.md#q16)
- [Безопасные заголовки](./topics/quality-ops/19-security.md#q17)
- [Симметричное vs асимметричное, хеш vs шифрование vs кодирование](./topics/quality-ops/19-security.md#q18)
- [HTTPS/TLS handshake](./topics/quality-ops/19-security.md#q19)
- [Supply-chain security](./topics/quality-ops/19-security.md#q20)

### 20. Производительность и профилирование

<sub>[topics/quality-ops/20-performance.md](./topics/quality-ops/20-performance.md)</sub>

- [«Сначала измеряй», преждевременная оптимизация](./topics/quality-ops/20-performance.md#q1)
- [BenchmarkDotNet](./topics/quality-ops/20-performance.md#q2)
- [Аллокации и давление на GC](./topics/quality-ops/20-performance.md#q3)
- [Span/Memory/stackalloc/ArrayPool/pooling](./topics/quality-ops/20-performance.md#q4)
- [StringBuilder, string.Create](./topics/quality-ops/20-performance.md#q5)
- [struct/readonly struct](./topics/quality-ops/20-performance.md#q6)
- [Боксинг](./topics/quality-ops/20-performance.md#q7)
- [LINQ overhead](./topics/quality-ops/20-performance.md#q8)
- [Async overhead, ValueTask](./topics/quality-ops/20-performance.md#q9)
- [Thread pool starvation](./topics/quality-ops/20-performance.md#q10)
- [Bottleneck-анализ](./topics/quality-ops/20-performance.md#q11)
- [Инструменты профилирования](./topics/quality-ops/20-performance.md#q12)
- [Кэширование как ускорение](./topics/quality-ops/20-performance.md#q13)
- [N+1 и оптимизация запросов](./topics/quality-ops/20-performance.md#q14)
- [Latency vs throughput, перцентили](./topics/quality-ops/20-performance.md#q15)
- [Native AOT, ReadyToRun, Tiered PGO](./topics/quality-ops/20-performance.md#q16)

### 21. DevOps, CI/CD, Docker, Kubernetes

<sub>[topics/quality-ops/21-devops-cicd.md](./topics/quality-ops/21-devops-cicd.md)</sub>

- [CI и CD](./topics/quality-ops/21-devops-cicd.md)
- [Пайплайн](./topics/quality-ops/21-devops-cicd.md)
- [Стратегии деплоя](./topics/quality-ops/21-devops-cicd.md)
- [Версионирование, semver](./topics/quality-ops/21-devops-cicd.md)
- [Trunk-based vs GitFlow](./topics/quality-ops/21-devops-cicd.md)
- [Образ vs контейнер, слои](./topics/quality-ops/21-devops-cicd.md)
- [Multi-stage build для .NET](./topics/quality-ops/21-devops-cicd.md)
- [Размер образа](./topics/quality-ops/21-devops-cicd.md)
- [Конфигурация в контейнере](./topics/quality-ops/21-devops-cicd.md)
- [Healthcheck, graceful shutdown](./topics/quality-ops/21-devops-cicd.md)
- [Pod, Deployment, Service, Ingress](./topics/quality-ops/21-devops-cicd.md)
- [ConfigMap, Secret](./topics/quality-ops/21-devops-cicd.md)
- [Probes](./topics/quality-ops/21-devops-cicd.md)
- [Requests/limits, HPA](./topics/quality-ops/21-devops-cicd.md)
- [Rolling update, rollback](./topics/quality-ops/21-devops-cicd.md)
- [Helm](./topics/quality-ops/21-devops-cicd.md)
- [.NET в k8s (GC под лимиты)](./topics/quality-ops/21-devops-cicd.md)

### 22. Наблюдаемость (логи, метрики, трейсинг)

<sub>[topics/quality-ops/22-observability.md](./topics/quality-ops/22-observability.md)</sub>

- [Три столпа наблюдаемости](./topics/quality-ops/22-observability.md#q1)
- [Monitoring vs Observability](./topics/quality-ops/22-observability.md#q2)
- [Структурное логирование](./topics/quality-ops/22-observability.md#q3)
- [ILogger, scopes, message templates](./topics/quality-ops/22-observability.md#q4)
- [Correlation / trace id](./topics/quality-ops/22-observability.md#q5)
- [Distributed tracing (OpenTelemetry)](./topics/quality-ops/22-observability.md#q6)
- [Метрики — типы, RED/USE](./topics/quality-ops/22-observability.md#q7)
- [Prometheus / Grafana](./topics/quality-ops/22-observability.md#q8)
- [APM](./topics/quality-ops/22-observability.md#q9)
- [Health checks](./topics/quality-ops/22-observability.md#q10)
- [Алертинг, SLI / SLO / SLA](./topics/quality-ops/22-observability.md#q11)
- [Перцентили](./topics/quality-ops/22-observability.md#q12)
- [Sampling](./topics/quality-ops/22-observability.md#q13)
- [Логи в контейнерах](./topics/quality-ops/22-observability.md#q14)

### 23. System Design (проектирование систем)

<sub>[topics/interview/23-system-design.md](./topics/interview/23-system-design.md)</sub>

- [⭐ Задача: Спроектировать сервис создания коротких ссылок (URL shortener)](./topics/interview/23-system-design.md)
- [Задача: <название>](./topics/interview/23-system-design.md)

### 24. Алгоритмы и задачи на собеседовании

<sub>[topics/interview/24-algorithms-coding.md](./topics/interview/24-algorithms-coding.md)</sub>

- [Big-O](./topics/interview/24-algorithms-coding.md)
- [Two pointers, sliding window, префиксные суммы](./topics/interview/24-algorithms-coding.md)
- [Хеш-таблицы](./topics/interview/24-algorithms-coding.md)
- [Стек/очередь](./topics/interview/24-algorithms-coding.md)
- [Связные списки](./topics/interview/24-algorithms-coding.md)
- [Деревья](./topics/interview/24-algorithms-coding.md)
- [Графы](./topics/interview/24-algorithms-coding.md)
- [Сортировки](./topics/interview/24-algorithms-coding.md)
- [Бинарный поиск](./topics/interview/24-algorithms-coding.md)
- [Рекурсия/мемоизация](./topics/interview/24-algorithms-coding.md)
- [Динамическое программирование](./topics/interview/24-algorithms-coding.md)
- [Жадные алгоритмы](./topics/interview/24-algorithms-coding.md)
- [Битовые операции](./topics/interview/24-algorithms-coding.md)
- [Задача: Two Sum](./topics/interview/24-algorithms-coding.md)
- [Задача: <название>](./topics/interview/24-algorithms-coding.md)

### 25. Soft skills и поведенческие вопросы

<sub>[topics/interview/25-soft-skills-behavioral.md](./topics/interview/25-soft-skills-behavioral.md)</sub>

- [Сложная техническая проблема](./topics/interview/25-soft-skills-behavioral.md)
- [Конфликт в команде](./topics/interview/25-soft-skills-behavioral.md)
- [Несогласие с решением](./topics/interview/25-soft-skills-behavioral.md)
- [Ошибка в проде](./topics/interview/25-soft-skills-behavioral.md)
- [Архитектурное решение](./topics/interview/25-soft-skills-behavioral.md)
- [Менторство](./topics/interview/25-soft-skills-behavioral.md)
- [Жёсткие сроки / неопределённость](./topics/interview/25-soft-skills-behavioral.md)
- [Объяснение не-техническим людям](./topics/interview/25-soft-skills-behavioral.md)
- [Code review](./topics/interview/25-soft-skills-behavioral.md)
- [Приоритизация и техдолг](./topics/interview/25-soft-skills-behavioral.md)
- [Провал / достижение](./topics/interview/25-soft-skills-behavioral.md)
- [Почему уходишь / почему к нам](./topics/interview/25-soft-skills-behavioral.md)
- [Тема: <название>](./topics/interview/25-soft-skills-behavioral.md)

### C# 8 (2019, .NET Core 3.0 / .NET Standard 2.1)

<sub>[topics/language-platform/csharp-versions/csharp-08.md](./topics/language-platform/csharp-versions/csharp-08.md)</sub>

- [1. Nullable reference types (NRT)](./topics/language-platform/csharp-versions/csharp-08.md)
- [2. Async streams — `IAsyncEnumerable<T>` + `await foreach`](./topics/language-platform/csharp-versions/csharp-08.md)
- [3. Switch expressions](./topics/language-platform/csharp-versions/csharp-08.md)
- [4. Pattern matching: property, tuple, positional patterns](./topics/language-platform/csharp-versions/csharp-08.md)
- [5. Ranges и индексы — `^` и `..`](./topics/language-platform/csharp-versions/csharp-08.md)
- [6. `using`-declaration (без фигурных скобок)](./topics/language-platform/csharp-versions/csharp-08.md)
- [7. Default interface methods](./topics/language-platform/csharp-versions/csharp-08.md)
- [Прочее](./topics/language-platform/csharp-versions/csharp-08.md)

### C# 9 (2020, .NET 5)

<sub>[topics/language-platform/csharp-versions/csharp-09.md](./topics/language-platform/csharp-versions/csharp-09.md)</sub>

- [1. Records (record class)](./topics/language-platform/csharp-versions/csharp-09.md)
- [2. `init`-сеттеры](./topics/language-platform/csharp-versions/csharp-09.md)
- [3. Target-typed `new()`](./topics/language-platform/csharp-versions/csharp-09.md)
- [4. Top-level statements](./topics/language-platform/csharp-versions/csharp-09.md)
- [5. Улучшенный pattern matching](./topics/language-platform/csharp-versions/csharp-09.md)
- [Прочее](./topics/language-platform/csharp-versions/csharp-09.md)

### C# 10 (2021, .NET 6)

<sub>[topics/language-platform/csharp-versions/csharp-10.md](./topics/language-platform/csharp-versions/csharp-10.md)</sub>

- [1. `record struct` и `readonly record struct`](./topics/language-platform/csharp-versions/csharp-10.md)
- [2. Global usings](./topics/language-platform/csharp-versions/csharp-10.md)
- [3. File-scoped namespaces](./topics/language-platform/csharp-versions/csharp-10.md)
- [4. Constant interpolated strings](./topics/language-platform/csharp-versions/csharp-10.md)
- [5. Extended property patterns](./topics/language-platform/csharp-versions/csharp-10.md)
- [6. Record struct + улучшения record](./topics/language-platform/csharp-versions/csharp-10.md)
- [Прочее](./topics/language-platform/csharp-versions/csharp-10.md)

### C# 11 (2022, .NET 7)

<sub>[topics/language-platform/csharp-versions/csharp-11.md](./topics/language-platform/csharp-versions/csharp-11.md)</sub>

- [1. Raw string literals (`"""`)](./topics/language-platform/csharp-versions/csharp-11.md)
- [2. `required`-члены](./topics/language-platform/csharp-versions/csharp-11.md)
- [3. List patterns](./topics/language-platform/csharp-versions/csharp-11.md)
- [4. Generic math — `INumber<T>` и static abstract в интерфейсах](./topics/language-platform/csharp-versions/csharp-11.md)
- [5. `field` keyword? — нет, это позже; здесь — auto-default structs](./topics/language-platform/csharp-versions/csharp-11.md)
- [Прочее](./topics/language-platform/csharp-versions/csharp-11.md)

### C# 12 (2023, .NET 8)

<sub>[topics/language-platform/csharp-versions/csharp-12.md](./topics/language-platform/csharp-versions/csharp-12.md)</sub>

- [1. Primary constructors для классов и структур](./topics/language-platform/csharp-versions/csharp-12.md)
- [2. Collection expressions `[...]`](./topics/language-platform/csharp-versions/csharp-12.md)
- [3. Default-значения для лямбда-параметров и `params` в лямбдах](./topics/language-platform/csharp-versions/csharp-12.md)
- [4. Alias любых типов через `using`](./topics/language-platform/csharp-versions/csharp-12.md)
- [5. `ref readonly` параметры](./topics/language-platform/csharp-versions/csharp-12.md)
- [Прочее](./topics/language-platform/csharp-versions/csharp-12.md)

### C# 13 (2024, .NET 9)

<sub>[topics/language-platform/csharp-versions/csharp-13.md](./topics/language-platform/csharp-versions/csharp-13.md)</sub>

- [1. `params` для коллекций (`Span`, `IEnumerable`, и др.)](./topics/language-platform/csharp-versions/csharp-13.md)
- [2. Новый тип `System.Threading.Lock`](./topics/language-platform/csharp-versions/csharp-13.md)
- [3. Partial-свойства и индексаторы](./topics/language-platform/csharp-versions/csharp-13.md)
- [4. Новый escape `\e` (ESC, символ 0x1B)](./topics/language-platform/csharp-versions/csharp-13.md)
- [5. Неявный индекс `^` в инициализаторах объектов](./topics/language-platform/csharp-versions/csharp-13.md)
- [6. `ref` и `unsafe` в итераторах и async](./topics/language-platform/csharp-versions/csharp-13.md)
- [Прочее](./topics/language-platform/csharp-versions/csharp-13.md)