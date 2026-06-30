# 10. ASP.NET Core и Web API

> Статус: 🟡 в работе · Уровень: Senior

## Что проверяют
Понимание хостинга, DI, middleware, жизненного цикла запроса, построения REST API и конфигурации.

## Ключевые вопросы (чеклист)
- [x] Generic Host / WebApplication
- [x] Middleware pipeline
- [x] DI и lifetimes, captive dependency
- [x] Жизненный цикл scope
- [x] Configuration и Options pattern
- [x] Controllers vs Minimal APIs
- [x] Model binding и валидация
- [x] Фильтры vs middleware
- [x] Routing
- [x] REST, статус-коды, версионирование
- [x] ProblemDetails
- [x] Глобальная обработка ошибок
- [x] HttpClientFactory, Polly
- [x] CORS, content negotiation
- [x] Health checks, BackgroundService
- [x] OpenAPI/Swagger

## Разбор вопросов

### Вопрос: Generic Host / WebApplication
**Краткий ответ:** Хост — контейнер приложения: DI, конфигурация, логирование, lifetime, hosted services. Minimal hosting model (`WebApplication.CreateBuilder(args)`) объединяет настройку сервисов и pipeline в `Program.cs`.

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddScoped<IOrderService, OrderService>();
var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.Run();
```

---

### Вопрос: Middleware pipeline
**Краткий ответ:** Конвейер делегатов `RequestDelegate`, через которые проходит каждый запрос (паттерн Chain of Responsibility). Подробный разбор — в [теме 11](./11-request-pipeline-http-highload.md). `Use` — звено с вызовом `next`, `Run` — терминальное, `Map` — ветвление по пути. **Порядок регистрации критичен.**

---

### Вопрос: DI и lifetimes, captive dependency
**Краткий ответ:**
- **Transient** — новый экземпляр на каждый запрос к контейнеру.
- **Scoped** — один на scope (в вебе — на HTTP-запрос).
- **Singleton** — один на всё приложение.
- **Captive dependency** — когда долгоживущий сервис (singleton) держит коротко­живущий (scoped/transient), продлевая его жизнь некорректно. Например, singleton с зависимостью на `DbContext` (scoped) → баг. Контейнер выбрасывает ошибку при валидации scope.

---

### Вопрос: Жизненный цикл scope
**Краткий ответ:** На каждый HTTP-запрос создаётся DI-scope; scoped-сервисы (включая `DbContext`) живут в его пределах и диспоузятся в конце запроса. Вне запроса (в `BackgroundService`) scope нужно создавать вручную (`IServiceScopeFactory.CreateScope()`).

---

### Вопрос: Configuration и Options pattern
**Краткий ответ:**
- Конфигурация собирается из источников по приоритету: appsettings.json → appsettings.{Env}.json → user-secrets → env vars → command line.
- **Options pattern**: связывание секции конфига с POCO:
  - `IOptions<T>` — singleton, читается один раз;
  - `IOptionsSnapshot<T>` — scoped, перечитывается на запрос;
  - `IOptionsMonitor<T>` — singleton с уведомлением об изменениях (hot reload).

```csharp
builder.Services.Configure<SmtpOptions>(builder.Configuration.GetSection("Smtp"));
```

---

### Вопрос: Controllers vs Minimal APIs
**Краткий ответ:**
- **Controllers** — атрибутный роутинг, фильтры, model binding, удобно для крупных API с конвенциями.
- **Minimal APIs** — лёгкие endpoints, меньше церемоний, отличная производительность; хороши для маленьких/быстрых сервисов. С развитием получили валидацию, фильтры, группировки.

---

### Вопрос: Model binding и валидация
**Краткий ответ:** Привязка данных запроса (route/query/body/form/header) к параметрам/моделям. `[ApiController]` включает автоматическую валидацию (400 при невалидной модели). Валидация: DataAnnotations или FluentValidation.

---

### Вопрос: Фильтры vs middleware
**Краткий ответ:**
- **Middleware** работает на уровне всего pipeline, не знает про MVC (controllers/actions).
- **Фильтры** (Authorization/Resource/Action/Exception/Result) работают внутри MVC, имеют доступ к контексту действия (модели, результату). Выбор: общесистемное (логирование, заголовки) → middleware; специфичное для действий (валидация модели, кэш результата) → фильтры.

---

### Вопрос: Routing
**Краткий ответ:** Сопоставление URL → endpoint. Attribute routing (`[HttpGet("orders/{id:int}")]`), constraints (`:int`, `:guid`), endpoint routing (общий механизм для MVC/Minimal/gRPC). `UseRouting`/`UseEndpoints` (в minimal hosting неявно).

---

### Вопрос: REST, статус-коды, версионирование
**Краткий ответ:**
- Методы: GET (чтение, идемпотентен), POST (создание), PUT (полная замена, идемпотентен), PATCH (частичное), DELETE (идемпотентен).
- Коды: 200/201/204, 400/401/403/404/409/422, 500/503.
- Версионирование: URL (`/v1/`), header, media type. `Asp.Versioning` библиотека.

---

### Вопрос: ProblemDetails
**Краткий ответ:** Стандартный формат ошибок (RFC 7807): `type`, `title`, `status`, `detail`, `traceId`. `AddProblemDetails()` + единый exception handler дают согласованные ответы об ошибках.

---

### Вопрос: Глобальная обработка ошибок
**Краткий ответ:** `app.UseExceptionHandler` / `IExceptionHandler` (.NET 8) для централизованной обработки: маппинг исключений на статус-коды и ProblemDetails, логирование с traceId. Лучше, чем try/catch в каждом контроллере.

---

### Вопрос: HttpClientFactory, Polly
**Краткий ответ:** `IHttpClientFactory` управляет пулом `HttpMessageHandler`, решает проблему socket exhaustion и протухания DNS (см. тему 11). Typed/named clients. Resilience через `Microsoft.Extensions.Http.Resilience`/Polly: retry, circuit breaker, timeout, bulkhead.

```csharp
builder.Services.AddHttpClient<IGitHubClient, GitHubClient>()
    .AddStandardResilienceHandler();
```

---

### Вопрос: CORS, content negotiation
**Краткий ответ:**
- **CORS** — политика доступа к API из браузера с других origin; настраивается middleware/политиками.
- **Content negotiation** — выбор формата ответа по `Accept` (JSON по умолчанию; форматтеры).

---

### Вопрос: Health checks, BackgroundService
**Краткий ответ:**
- **Health checks** — `/health` (liveness/readiness) для оркестратора (k8s), проверки БД/зависимостей.
- **BackgroundService / IHostedService** — фоновые задачи (обработка очереди, периодические джобы). Внутри — свой DI-scope для scoped-сервисов. Учитывать graceful shutdown (`stoppingToken`).

---

### Вопрос: OpenAPI/Swagger
**Краткий ответ:** Автогенерация спецификации API. В .NET 9 встроенная поддержка OpenAPI (`AddOpenApi`), ранее — Swashbuckle/NSwag. Документация + клиентогенерация + контракт.

---

## Полезные ссылки
- ASP.NET Core fundamentals: https://learn.microsoft.com/aspnet/core/fundamentals/
- DI: https://learn.microsoft.com/aspnet/core/fundamentals/dependency-injection
- Options: https://learn.microsoft.com/aspnet/core/fundamentals/configuration/options