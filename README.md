# Подготовка к собеседованию: Senior .NET Developer

Конспект по темам для технического собеседования. Каждая тема — отдельный файл в `topics/`.

<sub>Статусы: ⬜ не начато · 🟡 в работе · ✅ готово</sub>

## Структура

### Язык и платформа

| # | Тема | Файл | Статус |
|---|------|------|--------|
| 1 | C# язык и его возможности | [language-platform/01-csharp-language.md](./topics/language-platform/01-csharp-language.md) | 🟡 |
| 2 | CLR, GC и управление памятью | [language-platform/02-clr-gc-memory.md](./topics/language-platform/02-clr-gc-memory.md) | 🟡 |
| 3 | Многопоточность и асинхронность | [language-platform/03-async-multithreading.md](./topics/language-platform/03-async-multithreading.md) | 🟡 |
| 4 | Коллекции и структуры данных | [language-platform/04-collections-data-structures.md](./topics/language-platform/04-collections-data-structures.md) | 🟡 |

### Проектирование и архитектура

| # | Тема | Файл | Статус |
|---|------|------|--------|
| 5 | ООП, SOLID и принципы проектирования | [architecture/05-oop-solid-principles.md](./topics/architecture/05-oop-solid-principles.md) | 🟡 |
| 6 | Паттерны проектирования | [architecture/06-design-patterns.md](./topics/architecture/06-design-patterns.md) | 🟡 |
| 7 | Архитектура приложений | [architecture/07-architecture.md](./topics/architecture/07-architecture.md) | 🟡 |
| 8 | DDD (Domain-Driven Design) | [architecture/08-ddd.md](./topics/architecture/08-ddd.md) | 🟡 |
| 9 | CQRS и Event Sourcing | [architecture/09-cqrs-event-sourcing.md](./topics/architecture/09-cqrs-event-sourcing.md) | 🟡 |

### Web, API и данные

| # | Тема | Файл | Статус |
|---|------|------|--------|
| 10 | ASP.NET Core и Web API | [web-data/10-aspnet-core.md](./topics/web-data/10-aspnet-core.md) | 🟡 |
| 11 | Конвейер обработки запросов и HTTP в Highload | [web-data/11-request-pipeline-http-highload.md](./topics/web-data/11-request-pipeline-http-highload.md) | 🟡 |
| 12 | gRPC и контракты API (REST/gRPC/GraphQL) | [web-data/12-grpc-api-contracts.md](./topics/web-data/12-grpc-api-contracts.md) | 🟡 |
| 13 | Entity Framework Core и доступ к данным | [web-data/13-ef-core-data-access.md](./topics/web-data/13-ef-core-data-access.md) | 🟡 |
| 14 | Базы данных и SQL | [web-data/14-databases-sql.md](./topics/web-data/14-databases-sql.md) | 🟡 |

### Распределённые системы

| # | Тема | Файл | Статус |
|---|------|------|--------|
| 15 | Микросервисы и распределённые системы | [distributed/15-microservices.md](./topics/distributed/15-microservices.md) | 🟡 |
| 16 | Брокеры сообщений и интеграция | [distributed/16-messaging-integration.md](./topics/distributed/16-messaging-integration.md) | 🟡 |
| 17 | Кэширование (in-memory, Redis) | [distributed/17-caching.md](./topics/distributed/17-caching.md) | 🟡 |

### Качество и эксплуатация

| # | Тема | Файл | Статус |
|---|------|------|--------|
| 18 | Тестирование (unit, integration, моки) | [quality-ops/18-testing.md](./topics/quality-ops/18-testing.md) | 🟡 |
| 19 | Безопасность (auth, OWASP) | [quality-ops/19-security.md](./topics/quality-ops/19-security.md) | 🟡 |
| 20 | Производительность и профилирование | [quality-ops/20-performance.md](./topics/quality-ops/20-performance.md) | 🟡 |
| 21 | DevOps, CI/CD, Docker, Kubernetes | [quality-ops/21-devops-cicd.md](./topics/quality-ops/21-devops-cicd.md) | 🟡 |
| 22 | Наблюдаемость (логи, метрики, трейсинг) | [quality-ops/22-observability.md](./topics/quality-ops/22-observability.md) | 🟡 |

### Собеседование в целом

| # | Тема | Файл | Статус |
|---|------|------|--------|
| 23 | System Design (проектирование систем) | [interview/23-system-design.md](./topics/interview/23-system-design.md) | 🟡 |
| 24 | Алгоритмы и задачи на собеседовании | [interview/24-algorithms-coding.md](./topics/interview/24-algorithms-coding.md) | 🟡 |
| 25 | Soft skills и поведенческие вопросы | [interview/25-soft-skills-behavioral.md](./topics/interview/25-soft-skills-behavioral.md) | 🟡 |

### Нововведения C# по версиям

Интересные возможности языка от версии к версии — для чтения и повторения перед собеседованием.

| Версия | .NET | Файл | Ключевое |
|--------|------|------|----------|
| C# 8 | Core 3.0 | [csharp-08.md](./topics/language-platform/csharp-versions/csharp-08.md) | NRT, async streams, switch expressions, ranges, default interface methods |
| C# 9 | .NET 5 | [csharp-09.md](./topics/language-platform/csharp-versions/csharp-09.md) | records, `init`, target-typed `new()`, top-level statements |
| C# 10 | .NET 6 | [csharp-10.md](./topics/language-platform/csharp-versions/csharp-10.md) | `record struct`, global usings, file-scoped namespaces |
| C# 11 | .NET 7 | [csharp-11.md](./topics/language-platform/csharp-versions/csharp-11.md) | raw strings, `required`, list patterns, generic math |
| C# 12 | .NET 8 | [csharp-12.md](./topics/language-platform/csharp-versions/csharp-12.md) | primary constructors, collection expressions `[...]`, alias типов |
| C# 13 | .NET 9 | [csharp-13.md](./topics/language-platform/csharp-versions/csharp-13.md) | `params` коллекции, `Lock`, partial-свойства, `\e` |

## Как пользоваться

1. Идём по темам сверху вниз, отмечаем статус в таблице.
2. В каждом файле: ключевые вопросы → краткий ответ → код/пример → подводные камни.
3. Записываем сюда реальные вопросы с собеседований, чтобы не забыть.

## Вопросы с реальных собеседований

> Сюда выписываем вопросы, которые задавали, и ссылку на тему, к которой они относятся.

| Вопрос | Перейти к разбору |
|--------|------|
| Где применяются `Span<T>`/`Memory<T>` при работе с HTTP, зачем они нужны | [→ тема 11](./topics/web-data/11-request-pipeline-http-highload.md#q-span-memory-http) |
| Подробно опиши, как работает HTTP-конвейер обработки запроса в .NET Core | [→ тема 11](./topics/web-data/11-request-pipeline-http-highload.md#q-http-pipeline) |
| Как обеспечить безопасность при работе по HTTP | [→ тема 19](./topics/quality-ops/19-security.md#q-http-security) |
| System Design: спроектируй сервис создания коротких ссылок | [→ тема 23](./topics/interview/23-system-design.md#q-url-shortener) |

---

📑 **[Индекс всех вопросов по темам](./QUESTIONS.md)** — сводный список, чтобы быстро найти нужную тему или похожий вопрос.
