# Подготовка к собеседованию: Senior .NET Developer

Материал для подготовки к техническому собеседованию, разбитый по темам.
Каждая тема вынесена в отдельный файл в папке [`topics/`](./topics) и может дополняться.

> Статусы: ⬜ не начато · 🟡 в работе · ✅ готово

## Структура

### Язык и платформа
| # | Тема | Файл | Статус |
|---|------|------|--------|
| 1 | C# язык и его возможности | [topics/01-csharp-language.md](./topics/01-csharp-language.md) | 🟡 |
| 2 | CLR, GC и управление памятью | [topics/02-clr-gc-memory.md](./topics/02-clr-gc-memory.md) | 🟡 |
| 3 | Многопоточность и асинхронность | [topics/03-async-multithreading.md](./topics/03-async-multithreading.md) | 🟡 |
| 4 | Коллекции и структуры данных | [topics/04-collections-data-structures.md](./topics/04-collections-data-structures.md) | 🟡 |

### Проектирование и архитектура
| # | Тема | Файл | Статус |
|---|------|------|--------|
| 5 | ООП, SOLID и принципы проектирования | [topics/05-oop-solid-principles.md](./topics/05-oop-solid-principles.md) | 🟡 |
| 6 | Паттерны проектирования | [topics/06-design-patterns.md](./topics/06-design-patterns.md) | 🟡 |
| 7 | Архитектура приложений | [topics/07-architecture.md](./topics/07-architecture.md) | 🟡 |
| 8 | DDD (Domain-Driven Design) | [topics/08-ddd.md](./topics/08-ddd.md) | 🟡 |
| 9 | CQRS и Event Sourcing | [topics/09-cqrs-event-sourcing.md](./topics/09-cqrs-event-sourcing.md) | 🟡 |

### Web, API и данные
| # | Тема | Файл | Статус |
|---|------|------|--------|
| 10 | ASP.NET Core и Web API | [topics/10-aspnet-core.md](./topics/10-aspnet-core.md) | 🟡 |
| 11 | Конвейер обработки запросов и HTTP в Highload | [topics/11-request-pipeline-http-highload.md](./topics/11-request-pipeline-http-highload.md) | 🟡 |
| 12 | gRPC и контракты API (REST/gRPC/GraphQL) | [topics/12-grpc-api-contracts.md](./topics/12-grpc-api-contracts.md) | 🟡 |
| 13 | Entity Framework Core и доступ к данным | [topics/13-ef-core-data-access.md](./topics/13-ef-core-data-access.md) | 🟡 |
| 14 | Базы данных и SQL | [topics/14-databases-sql.md](./topics/14-databases-sql.md) | 🟡 |

### Распределённые системы
| # | Тема | Файл | Статус |
|---|------|------|--------|
| 15 | Микросервисы и распределённые системы | [topics/15-microservices.md](./topics/15-microservices.md) | 🟡 |
| 16 | Брокеры сообщений и интеграция | [topics/16-messaging-integration.md](./topics/16-messaging-integration.md) | 🟡 |
| 17 | Кэширование (in-memory, Redis) | [topics/17-caching.md](./topics/17-caching.md) | 🟡 |

### Качество и эксплуатация
| # | Тема | Файл | Статус |
|---|------|------|--------|
| 18 | Тестирование (unit, integration, моки) | [topics/18-testing.md](./topics/18-testing.md) | 🟡 |
| 19 | Безопасность (auth, OWASP) | [topics/19-security.md](./topics/19-security.md) | 🟡 |
| 20 | Производительность и профилирование | [topics/20-performance.md](./topics/20-performance.md) | 🟡 |
| 21 | DevOps, CI/CD, Docker, Kubernetes | [topics/21-devops-cicd.md](./topics/21-devops-cicd.md) | 🟡 |
| 22 | Наблюдаемость (логи, метрики, трейсинг) | [topics/22-observability.md](./topics/22-observability.md) | 🟡 |

### Собеседование в целом
| # | Тема | Файл | Статус |
|---|------|------|--------|
| 23 | System Design (проектирование систем) | [topics/23-system-design.md](./topics/23-system-design.md) | 🟡 |
| 24 | Алгоритмы и задачи на собеседовании | [topics/24-algorithms-coding.md](./topics/24-algorithms-coding.md) | 🟡 |
| 25 | Soft skills и поведенческие вопросы | [topics/25-soft-skills-behavioral.md](./topics/25-soft-skills-behavioral.md) | 🟡 |

## Как пользоваться

1. Идём по темам сверху вниз, отмечаем статус в таблице.
2. В каждом файле: ключевые вопросы → краткий ответ → код/пример → подводные камни.
3. Записываем сюда реальные вопросы с собеседований, чтобы не забыть.

## Вопросы с реальных собеседований

> Сюда выписываем вопросы, которые задавали, и ссылку на тему, к которой они относятся.

| Вопрос | Перейти к разбору |
|--------|------|
| Где применяются `Span<T>`/`Memory<T>` при работе с HTTP, зачем они нужны | [→ тема 11](./topics/11-request-pipeline-http-highload.md#q-span-memory-http) |
| Подробно опиши, как работает HTTP-конвейер обработки запроса в .NET Core | [→ тема 11](./topics/11-request-pipeline-http-highload.md#q-http-pipeline) |
| Как обеспечить безопасность при работе по HTTP | [→ тема 19](./topics/19-security.md#q-http-security) |
| System Design: спроектируй сервис создания коротких ссылок | [→ тема 23](./topics/23-system-design.md#q-url-shortener) |
