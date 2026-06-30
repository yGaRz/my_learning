# 12. gRPC и контракты API (REST/gRPC/GraphQL)

> Статус: 🟡 в работе · Уровень: Senior

## Что проверяют
Понимание разных стилей API, когда выбирать gRPC, как работают контракты, версионирование и обратная совместимость.

## Ключевые вопросы (чеклист)
- [x] gRPC: основа HTTP/2 + Protobuf
- [x] 4 типа вызовов
- [x] Почему gRPC быстрее REST/JSON
- [x] Версионирование protobuf
- [x] Deadlines, отмена, interceptors
- [x] gRPC-Web
- [x] gRPC vs REST — когда что
- [x] REST: уровни зрелости, идемпотентность, версионирование
- [x] GraphQL: когда, N+1, HotChocolate
- [x] Contract-first vs code-first
- [x] Совместимость, contract testing

## gRPC

### Основа: HTTP/2 + Protocol Buffers
**Краткий ответ:** gRPC — RPC-фреймворк поверх HTTP/2 с бинарной сериализацией Protocol Buffers (protobuf). Контракт описывается в `.proto`, из него генерируется код клиента и сервера.

```proto
syntax = "proto3";
service Orders {
  rpc GetOrder (GetOrderRequest) returns (OrderReply);
}
message GetOrderRequest { int32 id = 1; }
message OrderReply { int32 id = 1; string status = 2; }
```

### 4 типа вызовов
- **Unary** — один запрос → один ответ (как обычный вызов).
- **Server streaming** — один запрос → поток ответов.
- **Client streaming** — поток запросов → один ответ.
- **Bidirectional streaming** — оба потока одновременно.

### Почему gRPC быстрее REST/JSON
- Бинарный protobuf компактнее и быстрее (де)сериализуется, чем текстовый JSON.
- HTTP/2: мультиплексирование, сжатие заголовков, постоянные соединения.
- Строгий контракт → меньше накладных на валидацию/парсинг.

### Версионирование protobuf
- Поля идентифицируются **номерами**, а не именами — нельзя менять номер у существующего поля.
- Добавление новых полей с новыми номерами — обратно совместимо (старые клиенты игнорируют).
- Удалённые поля — помечать `reserved`, чтобы номер не переиспользовали.
- Не менять тип поля несовместимо.

### Deadlines, отмена, interceptors, метаданные
- **Deadline/timeout** передаётся клиентом; сервер видит оставшееся время.
- **Отмена** через `CancellationToken`.
- **Interceptors** — аналог middleware для gRPC (логирование, auth, retry).
- **Метаданные** — аналог HTTP-заголовков.

### gRPC-Web
Браузеры не дают полный контроль над HTTP/2 кадрами → нужен **gRPC-Web** (прослойка/прокси). Ограничения: client/bidi streaming поддержаны частично.

### gRPC vs REST — когда что
- **gRPC**: внутренние сервис-сервис коммуникации, высокая нагрузка, строгие контракты, streaming, полиглот-окружение.
- **REST/JSON**: публичные API, браузерные клиенты, простота, человекочитаемость, кэшируемость по HTTP.

## REST

### Уровни зрелости (Richardson) и HATEOAS
- L0: один endpoint/туннель. L1: ресурсы. L2: HTTP-методы и коды (практический стандарт). L3: HATEOAS (гиперссылки) — редко на практике.

### Идемпотентность и безопасность методов
GET/HEAD — безопасны (не меняют); GET/PUT/DELETE — идемпотентны (повтор не меняет результат); POST — нет. Для надёжных POST применяют ключи идемпотентности.

### Версионирование REST
URL (`/v1/`), header, media type. Главное — не ломать существующих клиентов.

### Пагинация/фильтрация
Offset/limit или keyset (cursor) пагинация (keyset лучше для больших объёмов/глубокой пагинации).

## GraphQL

### Когда уместен
Клиенту нужны гибкие выборки полей/агрегации из многих источников (мобильные/разнородные клиенты), чтобы избежать over/under-fetching.

### N+1 и DataLoader
Резолверы могут породить N+1 запросов; решается **DataLoader** (батчинг + кэш в рамках запроса).

### HotChocolate
Популярная GraphQL-реализация для .NET.

## Контракты в целом

### Contract-first vs code-first
- **Contract-first** — сначала схема (`.proto`/OpenAPI), из неё код. Чёткий контракт, согласование команд.
- **Code-first** — контракт генерируется из кода. Быстрее старт, риск «случайных» изменений контракта.

### Совместимость и contract testing
- Backward/forward compatibility — не ломать старых потребителей.
- **Consumer-driven contract testing (Pact)** — потребители фиксируют ожидания, провайдер проверяется против них в CI.
- OpenAPI/Swagger как контракт + кодогенерация клиентов.

## Полезные ссылки
- gRPC for .NET: https://learn.microsoft.com/aspnet/core/grpc/
- gRPC vs HTTP APIs: https://learn.microsoft.com/aspnet/core/grpc/comparison
- HotChocolate: https://chillicream.com/docs/hotchocolate