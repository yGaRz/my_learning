# 13. Entity Framework Core и доступ к данным

> Статус: 🟡 в работе · Уровень: Senior

## Что проверяют
Понимание работы ORM, отслеживания изменений, генерации SQL, типичных проблем производительности (N+1) и транзакций.

## Ключевые вопросы (чеклист)
- [x] Change Tracking, DbContext как Unit of Work, время жизни
- [x] `AsNoTracking()`
- [x] Eager/lazy/explicit loading
- [x] Проблема N+1
- [x] `IQueryable` vs `IEnumerable`, client vs server eval
- [x] Отложенное выполнение, материализация
- [x] Проекции
- [x] Транзакции, SaveChanges, потокобезопасность
- [x] Optimistic concurrency
- [x] Миграции
- [x] Связи и конфигурация
- [x] Compiled queries, ExecuteUpdate/Delete
- [x] Split queries
- [x] Connection pooling, DbContextFactory
- [x] Когда raw SQL / Dapper
- [x] Repository поверх EF

## Разбор вопросов

### Вопрос: Change Tracking, DbContext, время жизни
**Краткий ответ:** `DbContext` отслеживает загруженные сущности (snapshot), при `SaveChanges` генерирует INSERT/UPDATE/DELETE для изменённых. Это **Unit of Work**. Контекст **не потокобезопасен**, должен быть короткоживущим — в вебе scoped (один на запрос).

**Подводные камни:** долгоживущий контекст накапливает отслеживаемые сущности (память + замедление) и устаревшие данные.

---

### Вопрос: `AsNoTracking()`
**Краткий ответ:** Отключает отслеживание для read-only запросов → меньше памяти и быстрее (нет snapshot). Используйте для запросов, результат которых не будете менять/сохранять. `AsNoTrackingWithIdentityResolution` — если нужны общие ссылки на одинаковые сущности.

---

### Вопрос: Eager / lazy / explicit loading
**Краткий ответ:**
- **Eager** — `Include`/`ThenInclude`, грузит связанные данные сразу одним/несколькими запросами.
- **Lazy** — данные грузятся при первом обращении к навигационному свойству (нужны proxies); удобно, но провоцирует N+1.
- **Explicit** — `context.Entry(e).Collection/Reference(...).Load()` вручную.

---

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

---

### Вопрос: `IQueryable` vs `IEnumerable`, client vs server eval
**Краткий ответ:** см. [тему 01](./01-csharp-language.md). Пока работаем с `IQueryable`, фильтры/проекции транслируются в SQL (server eval). После `AsEnumerable()`/`ToList()` — выполнение в памяти (client eval). EF Core по умолчанию **бросает исключение**, если не может транслировать выражение в SQL (раньше молча тянул всё в память — была частая беда).

---

### Вопрос: Отложенное выполнение, материализация
**Краткий ответ:** Запрос не выполняется, пока не перечислен (`ToList`, `First`, `Count`, `foreach`). Множественный перебор = повторные запросы. Материализуйте один раз, если нужно несколько проходов.

---

### Вопрос: Проекции
**Краткий ответ:** `Select` в DTO/анонимный тип тянет из БД только нужные колонки (а не всю сущность), снижает трафик и память, часто убирает необходимость `Include`.

```csharp
var dto = db.Orders.Select(o => new { o.Id, Customer = o.Customer.Name }).ToList();
```

---

### Вопрос: Транзакции, SaveChanges, потокобезопасность
**Краткий ответ:** `SaveChanges` выполняется в неявной транзакции (все изменения атомарно). Явные транзакции — `BeginTransaction`/`TransactionScope` (несколько SaveChanges/контекстов). `DbContext` не потокобезопасен — нельзя параллельно использовать один экземпляр (ошибка «second operation started»).

---

### Вопрос: Optimistic concurrency
**Краткий ответ:** Колонка версии (`[Timestamp]`/`rowversion` или concurrency token); при UPDATE EF добавляет условие на версию. Если строку изменили параллельно — 0 строк затронуто → `DbUpdateConcurrencyException`, обрабатываем (retry/merge/уведомление).

---

### Вопрос: Миграции
**Краткий ответ:** `Add-Migration`/`dotnet ef migrations add` фиксируют изменения модели как код; `Update-Database`/`dotnet ef database update` применяют. В проде — генерация SQL-скрипта/идемпотентных скриптов, применение в пайплайне. Осторожно с разрушающими изменениями и большими таблицами.

---

### Вопрос: Связи и конфигурация
**Краткий ответ:** one-to-many, many-to-many (с C# 5+ без явной join-сущности), one-to-one, owned types (VO). Конфигурация — Fluent API (`OnModelCreating`/`IEntityTypeConfiguration`) предпочтительнее атрибутов для сложных случаев и чистоты домена.

---

### Вопрос: Compiled queries, ExecuteUpdate/ExecuteDelete
**Краткий ответ:**
- **Compiled queries** (`EF.CompileAsyncQuery`) кэшируют план построения запроса — для очень горячих запросов.
- **`ExecuteUpdate`/`ExecuteDelete`** (EF Core 7+) — bulk-операции **без загрузки сущностей** в память, прямой UPDATE/DELETE в БД.

```csharp
await db.Orders.Where(o => o.Old).ExecuteDeleteAsync(ct);
```

---

### Вопрос: Split queries
**Краткий ответ:** При нескольких `Include` коллекций один JOIN-запрос даёт **картезианский взрыв** (дублирование строк). `AsSplitQuery()` разбивает на отдельные запросы — меньше данных, но больше round-trips. Выбор по ситуации.

---

### Вопрос: Connection pooling, DbContextFactory
**Краткий ответ:** ADO.NET пулит соединения автоматически. `AddDbContextPool` переиспользует экземпляры контекста (снижает аллокации). `IDbContextFactory<T>` — для создания контекстов вне scope (фоновые задачи, Blazor, параллельные операции).

---

### Вопрос: Когда raw SQL / Dapper
**Краткий ответ:** EF Core хорош для CRUD/доменной модели. Для сложных тяжёлых запросов/максимальной производительности — `FromSqlRaw`/`SqlQuery` или микро-ORM **Dapper** (быстрее, ближе к SQL, без трекинга). Частый подход: EF для записи, Dapper для тяжёлого чтения.

---

### Вопрос: Repository поверх EF
**Краткий ответ:** см. [тему 06](./06-design-patterns.md). `DbContext` уже Unit of Work, `DbSet` уже репозиторий. Доп. Repository оправдан для изоляции домена/тестируемости/смены провайдера, но часто избыточен и прячет возможности EF (Include, проекции, split queries).

---

## Полезные ссылки
- EF Core docs: https://learn.microsoft.com/ef/core/
- Performance: https://learn.microsoft.com/ef/core/performance/
- Dapper: https://github.com/DapperLib/Dapper