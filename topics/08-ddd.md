# 08. DDD (Domain-Driven Design)

> Статус: 🟡 в работе · Уровень: Senior

## Что проверяют
Понимание стратегического и тактического DDD, умение моделировать домен, выделять границы, а не только знать термины.

## Ключевые вопросы (чеклист)
- [x] Ubiquitous Language
- [x] Bounded Context
- [x] Context Map (отношения контекстов)
- [x] Subdomains: Core/Supporting/Generic
- [x] Anti-Corruption Layer
- [x] Entity vs Value Object
- [x] Aggregate и Aggregate Root
- [x] Правила агрегатов
- [x] Domain Events
- [x] Repository (для агрегатов)
- [x] Domain Service vs Application Service
- [x] Factory, Invariants, Specification
- [x] DDD + EF Core
- [x] Anemic vs Rich
- [x] Когда DDD избыточен
- [x] Eventual consistency между агрегатами

## Стратегический DDD

### Ubiquitous Language
Единый язык предметной области, общий для разработчиков и экспертов. Термины из языка отражаются прямо в коде (классы/методы). Убирает «перевод» между бизнесом и кодом.

### Bounded Context
Граница, внутри которой модель и язык согласованы и однозначны. Один и тот же термин («Клиент») в разных контекстах (Продажи, Поддержка) — разные модели. Контекст — основной кандидат на границу модуля/микросервиса.

### Context Map
Карта отношений между контекстами:
- **Shared Kernel** — общая часть модели (рискованно, требует координации).
- **Customer/Supplier** — вышестоящий/нижестоящий контекст.
- **Conformist** — нижестоящий принимает модель вышестоящего как есть.
- **Anti-Corruption Layer (ACL)** — слой-переводчик, защищающий нашу модель от чужой.
- **Open Host / Published Language** — опубликованный контракт интеграции.

### Subdomains
- **Core** — то, что даёт конкурентное преимущество (сюда вкладываем DDD и лучших людей).
- **Supporting** — нужно, но не уникально.
- **Generic** — типовое (платежи, аутентификация) — лучше купить/взять готовое.

### Anti-Corruption Layer
Изолирует домен от внешних систем/легаси через адаптеры и переводчики, чтобы чужие концепции не «протекали» внутрь.

## Тактический DDD

### Entity vs Value Object
- **Entity** — имеет **идентичность** (Id), живёт во времени, может менять состояние. Равенство — по Id.
- **Value Object** — определяется **значением**, иммутабелен, без идентичности (Money, Address, DateRange). Равенство — по значениям. В C# удобно `record`/`readonly record struct`.

```csharp
public readonly record struct Money(decimal Amount, string Currency)
{
    public Money Add(Money other) =>
        other.Currency == Currency
            ? this with { Amount = Amount + other.Amount }
            : throw new InvalidOperationException("Currency mismatch");
}
```

### Aggregate и Aggregate Root
- **Aggregate** — кластер связанных объектов, рассматриваемый как единое целое для изменений.
- **Aggregate Root** — единственная точка входа; внешний код ссылается только на корень. Корень защищает **инварианты** всего агрегата.

### Правила агрегатов
- Ссылки между агрегатами — только по **Id**, не по объектным ссылкам.
- Одна транзакция изменяет **один** агрегат (граница консистентности).
- Между агрегатами — eventual consistency (через domain events).
- Агрегаты держать маленькими.

### Domain Events
События, фиксирующие значимый факт в домене («OrderPlaced»). Публикуются агрегатом, обрабатываются (часто после коммита) для побочных эффектов и согласования других агрегатов/контекстов.

### Repository
Абстракция получения/сохранения **агрегатов целиком** (по корню). Один репозиторий на агрегат, не на каждую таблицу.

### Domain Service vs Application Service
- **Domain Service** — бизнес-логика, не принадлежащая одной сущности (например, перевод между счетами); часть домена.
- **Application Service** — оркестратор use case: загрузить агрегат → вызвать поведение → сохранить → опубликовать события. Без бизнес-правил.

### Factory, Invariants, Specification
- **Factory** — сложное создание агрегата с проверкой инвариантов.
- **Invariants** — правила, всегда истинные; защищаются внутри агрегата (в конструкторе/методах, не сеттерами).
- **Specification** — инкапсулированные критерии (для выборок/проверок).

## Практика

### DDD + EF Core
- VO маппят через owned types/converters; приватные сеттеры + backing fields; конструктор для инвариантов.
- Коллекции — приватные поля, наружу `IReadOnlyCollection`.
- Избегать публичных сеттеров, ломающих инкапсуляцию.

### Anemic vs Rich
См. [тему 07](./07-architecture.md). DDD предполагает rich-модель с поведением и защищёнными инвариантами.

### Когда DDD избыточен
CRUD-приложения, простой домен, прототипы — DDD добавит сложности без выгоды. DDD стоит вкладывать в **Core-домен** со сложной логикой.

### Eventual consistency между агрегатами
Изменение нескольких агрегатов = несколько транзакций, связанных domain events / saga; согласованность достигается со временем (см. темы 09, 15).

## Полезные ссылки
- DDD (Microsoft, microservices context): https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/
- Eric Evans / Vaughn Vernon — Implementing DDD