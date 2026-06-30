<!-- nav-top -->
[← К оглавлению](../../README.md)

# 13. Entity Framework Core и доступ к данным

<a id="top"></a>

> Статус: 🟡 в работе · Уровень: Senior

## Что проверяют
Понимание работы ORM, отслеживания изменений, генерации SQL, типичных проблем производительности (N+1) и транзакций.

## Ключевые вопросы (чеклист)
- [x] [Change Tracking, DbContext как Unit of Work, время жизни](#q1)
- [x] [`AsNoTracking()`](#q2)
- [x] [Eager/lazy/explicit loading](#q3)
- [x] [Проблема N+1](#q4)
- [x] [`IQueryable` vs `IEnumerable`, client vs server eval](#q5)
- [x] [Отложенное выполнение, материализация](#q6)
- [x] [Проекции](#q7)
- [x] [Транзакции, SaveChanges, потокобезопасность](#q8)
- [x] [Optimistic concurrency](#q9)
- [x] [Миграции](#q10)
- [x] [Связи и конфигурация](#q11)
- [x] [Compiled queries, ExecuteUpdate/Delete](#q12)
- [x] [Split queries](#q13)
- [x] [Connection pooling, DbContextFactory](#q14)
- [x] [Когда raw SQL / Dapper](#q15)
- [x] [Repository поверх EF](#q16)

## Разбор вопросов

<a id="q1"></a>
### Вопрос: Change Tracking, DbContext, время жизни
**Краткий ответ:** `DbContext` отслеживает загруженные сущности (snapshot), при `SaveChanges` генерирует INSERT/UPDATE/DELETE для изменённых. Это **Unit of Work**. Контекст **не потокобезопасен**, должен быть короткоживущим — в вебе scoped (один на запрос).

**Подводные камни:** долгоживущий контекст накапливает отслеживаемые сущности (память + замедление) и устаревшие данные.

[↑ Наверх](#top)

---

<a id="q2"></a>
### Вопрос: `AsNoTracking()`
**Краткий ответ:** Отключает отслеживание для read-only запросов → меньше памяти и быстрее (нет snapshot). Используйте для запросов, результат которых не будете менять/сохранять. `AsNoTrackingWithIdentityResolution` — если нужны общие ссылки на одинаковые сущности.

[↑ Наверх](#top)

---

<a id="q3"></a>
### Вопрос: Eager / lazy / explicit loading
**Краткий ответ:**
- **Eager** — `Include`/`ThenInclude`, грузит связанные данные сразу одним/несколькими запросами.
- **Lazy** — данные грузятся при первом обращении к навигационному свойству (нужны proxies); удобно, но провоцирует N+1.
- **Explicit** — `context.Entry(e).Collection/Reference(...).Load()` вручную.

[↑ Наверх](#top)

---

<a id="q4"></a>
### Вопрос: Проблема N+1
**Краткий ответ:** 1 запрос за списком + по 1 запросу на каждую связанную сущность = N+1 обращений к БД. Частая причина — lazy loading в цикле. Решение: `Include` (eager) или проекция с нужными полями.

```csharp
// N+1 (lazy):
foreach (var o in db.Orders.ToList())
    Console.WriteLine(o.Customer.Name); // отдельный запрос на каждого

// Fix:
var orders = db.Orders.Include(o => o.Customer).ToList();
```

**Обнаружение:** логирование SQL, профайлер БД, EF Core warnings.

[↑ Наверх](#top)

---

<a id="q5"></a>
### Вопрос: `IQueryable` vs `IEnumerable`, client vs server eval
**Краткий ответ:** см. [тему 01](./01-csharp-language.md). Пока работаем с `IQueryable`, фильтры/проекции транслируются в SQL (server eval). После `AsEnumerable()`/`ToList()` — выполнение в памяти (client eval). EF Core по умолчанию **бросает исключение**, если не может транслировать выражение в SQL (раньше молча тянул всё в память — была частая беда).

[↑ Наверх](#top)

---

<a id="q6"></a>
### Вопрос: Отложенное выполнение, материализация
**Краткий ответ:** Запрос не выполняется, пока не перечислен (`ToList`, `First`, `Count`, `foreach`). Множественный перебор = повторные запросы. Материализуйте один раз, если нужно несколько проходов.

[↑ Наверх](#top)

---

<a id="q7"></a>
### Вопрос: Проекции
**Краткий ответ:** `Select` в DTO/анонимный тип тянет из БД только нужные колонки (а не всю сущность), снижает трафик и память, часто убирает необходимость `Include`.

```csharp
var dto = db.Orders.Select(o => new { o.Id, Customer = o.Customer.Name }).ToList();
```

[↑ Наверх](#top)

---

<a id="q8"></a>
### Вопрос: Транзакции, SaveChanges, потокобезопасность
**Краткий ответ:** `SaveChanges` выполняется в неявной транзакции (все изменения атомарно). Явные транзакции — `BeginTransaction`/`TransactionScope` (несколько SaveChanges/контекстов). `DbContext` не потокобезопасен — нельзя параллельно использовать один экземпляр (ошибка «second operation started»).

[↑ Наверх](#top)

---

<a id="q9"></a>
### Вопрос: Optimistic concurrency
**Краткий ответ:** Колонка версии (`[Timestamp]`/`rowversion` или concurrency token); при UPDATE EF добавляет условие на версию. Если строку изменили параллельно — 0 строк затронуто → `DbUpdateConcurrencyException`, обрабатываем (retry/merge/уведомление).

[↑ Наверх](#top)

---

<a id="q10"></a>
### Вопрос: Миграции
**Краткий ответ:** `Add-Migration`/`dotnet ef migrations add` фиксируют изменения модели как код; `Update-Database`/`dotnet ef database update` применяют. В проде — генерация SQL-скрипта/идемпотентных скриптов, применение в пайплайне. Осторожно с разрушающими изменениями и большими таблицами.

[↑ Наверх](#top)

---

<a id="q11"></a>
### Вопрос: Связи и конфигурация
**Краткий ответ:** one-to-many, many-to-many (с C# 5+ без явной join-сущности), one-to-one, owned types (VO). Конфигурация — Fluent API (`OnModelCreating`/`IEntityTypeConfiguration`) предпочтительнее атрибутов для сложных случаев и чистоты домена.

[↑ Наверх](#top)

---

<a id="q12"></a>
### Вопрос: Compiled queries, ExecuteUpdate/ExecuteDelete
**Краткий ответ:**
- **Compiled queries** (`EF.CompileAsyncQuery`) кэшируют план построения запроса — для очень горячих запросов.
- **`ExecuteUpdate`/`ExecuteDelete`** (EF Core 7+) — bulk-операции **без загрузки сущностей** в память, прямой UPDATE/DELETE в БД.

```csharp
await db.Orders.Where(o => o.Old).ExecuteDeleteAsync(ct);
```

[↑ Наверх](#top)

---

<a id="q13"></a>
### Вопрос: Split queries
**Краткий ответ:** При нескольких `Include` коллекций один JOIN-запрос даёт **картезианский взрыв** (дублирование строк). `AsSplitQuery()` разбивает на отдельные запросы — меньше данных, но больше round-trips. Выбор по ситуации.

[↑ Наверх](#top)

---

<a id="q14"></a>
### Вопрос: Connection pooling, DbContextFactory
**Краткий ответ:** ADO.NET пулит соединения автоматически. `AddDbContextPool` переиспользует экземпляры контекста (снижает аллокации). `IDbContextFactory<T>` — для создания контекстов вне scope (фоновые задачи, Blazor, параллельные операции).

[↑ Наверх](#top)

---

<a id="q15"></a>
### Вопрос: Когда raw SQL / Dapper
**Краткий ответ:** EF Core хорош для CRUD/доменной модели. Для сложных тяжёлых запросов/максимальной производительности — `FromSqlRaw`/`SqlQuery` или микро-ORM **Dapper** (быстрее, ближе к SQL, без трекинга). Частый подход: EF для записи, Dapper для тяжёлого чтения.

[↑ Наверх](#top)

---

<a id="q16"></a>
### Вопрос: Repository поверх EF
**Краткий ответ:** см. [тему 06](./06-design-patterns.md). `DbContext` уже Unit of Work, `DbSet` уже репозиторий. Доп. Repository оправдан для изоляции домена/тестируемости/смены провайдера, но часто избыточен и прячет возможности EF (Include, проекции, split queries).

[↑ Наверх](#top)

---

## Полезные ссылки
- EF Core docs: https://learn.microsoft.com/ef/core/
- Performance: https://learn.microsoft.com/ef/core/performance/
- Dapper: https://github.com/DapperLib/Dapper

[↑ Наверх](#top)

---

<!-- nav-bottom -->
[← К оглавлению](../../README.md)
