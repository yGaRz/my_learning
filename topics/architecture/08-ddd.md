<!-- nav-top -->
[← К оглавлению](../../README.md)

# 08. DDD (Domain-Driven Design)

<a id="top"></a>

> Статус: 🟡 в работе · Уровень: Senior

## Что проверяют
Понимание стратегического и тактического DDD, умение моделировать домен, выделять границы, а не только знать термины.

## Ключевые вопросы (чеклист)
- [x] [Ubiquitous Language](#q1)
- [x] [Bounded Context](#q2)
- [x] [Context Map (отношения контекстов)](#q3)
- [x] [Subdomains: Core/Supporting/Generic](#q4)
- [x] [Anti-Corruption Layer](#q5)
- [x] [Entity vs Value Object](#q6)
- [x] [Aggregate и Aggregate Root](#q7)
- [x] [Правила агрегатов](#q8)
- [x] [Domain Events](#q9)
- [x] [Repository (для агрегатов)](#q10)
- [x] [Domain Service vs Application Service](#q11)
- [x] [Factory, Invariants, Specification](#q12)
- [x] [DDD + EF Core](#q13)
- [x] [Anemic vs Rich](#q14)
- [x] [Когда DDD избыточен](#q15)
- [x] [Eventual consistency между агрегатами](#q16)

## Стратегический DDD

<a id="q1"></a>
### Ubiquitous Language
Единый язык предметной области, общий для разработчиков и экспертов. Термины из языка отражаются прямо в коде (классы/методы). Убирает «перевод» между бизнесом и кодом.

<a id="q2"></a>
### Bounded Context
Граница, внутри которой модель и язык согласованы и однозначны. Один и тот же термин («Клиент») в разных контекстах (Продажи, Поддержка) — разные модели. Контекст — основной кандидат на границу модуля/микросервиса.

<a id="q3"></a>
### Context Map
Карта отношений между контекстами:
- **Shared Kernel** — общая часть модели (рискованно, требует координации).
- **Customer/Supplier** — вышестоящий/нижестоящий контекст.
- **Conformist** — нижестоящий принимает модель вышестоящего как есть.
- **Anti-Corruption Layer (ACL)** — слой-переводчик, защищающий нашу модель от чужой.
- **Open Host / Published Language** — опубликованный контракт интеграции.

<a id="q4"></a>
### Subdomains
- **Core** — то, что даёт конкурентное преимущество (сюда вкладываем DDD и лучших людей).
- **Supporting** — нужно, но не уникально.
- **Generic** — типовое (платежи, аутентификация) — лучше купить/взять готовое.

<a id="q5"></a>
### Anti-Corruption Layer
Изолирует домен от внешних систем/легаси через адаптеры и переводчики, чтобы чужие концепции не «протекали» внутрь.

## Тактический DDD

<a id="q6"></a>
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

<a id="q7"></a>
### Aggregate и Aggregate Root
- **Aggregate** — кластер связанных объектов, рассматриваемый как единое целое для изменений.
- **Aggregate Root** — единственная точка входа; внешний код ссылается только на корень. Корень защищает **инварианты** всего агрегата.

<a id="q8"></a>
### Правила агрегатов
- Ссылки между агрегатами — только по **Id**, не по объектным ссылкам.
- Одна транзакция изменяет **один** агрегат (граница консистентности).
- Между агрегатами — eventual consistency (через domain events).
- Агрегаты держать маленькими.

<a id="q9"></a>
### Domain Events
События, фиксирующие значимый факт в домене («OrderPlaced»). Публикуются агрегатом, обрабатываются (часто после коммита) для побочных эффектов и согласования других агрегатов/контекстов.

<a id="q10"></a>
### Repository
Абстракция получения/сохранения **агрегатов целиком** (по корню). Один репозиторий на агрегат, не на каждую таблицу.

<a id="q11"></a>
### Domain Service vs Application Service
- **Domain Service** — бизнес-логика, не принадлежащая одной сущности (например, перевод между счетами); часть домена.
- **Application Service** — оркестратор use case: загрузить агрегат → вызвать поведение → сохранить → опубликовать события. Без бизнес-правил.

<a id="q12"></a>
### Factory, Invariants, Specification
- **Factory** — сложное создание агрегата с проверкой инвариантов.
- **Invariants** — правила, всегда истинные; защищаются внутри агрегата (в конструкторе/методах, не сеттерами).
- **Specification** — инкапсулированные критерии (для выборок/проверок).

## Практика

<a id="q13"></a>
### DDD + EF Core
- VO маппят через owned types/converters; приватные сеттеры + backing fields; конструктор для инвариантов.
- Коллекции — приватные поля, наружу `IReadOnlyCollection`.
- Избегать публичных сеттеров, ломающих инкапсуляцию.

<a id="q14"></a>
### Anemic vs Rich
См. [тему 07](./07-architecture.md). DDD предполагает rich-модель с поведением и защищёнными инвариантами.

<a id="q15"></a>
### Когда DDD избыточен
CRUD-приложения, простой домен, прототипы — DDD добавит сложности без выгоды. DDD стоит вкладывать в **Core-домен** со сложной логикой.

<a id="q16"></a>
### Eventual consistency между агрегатами
Изменение нескольких агрегатов = несколько транзакций, связанных domain events / saga; согласованность достигается со временем (см. темы 09, 15).

## Полезные ссылки
- DDD (Microsoft, microservices context): https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/
- Eric Evans / Vaughn Vernon — Implementing DDD

[↑ Наверх](#top)

---

<!-- nav-bottom -->
[← К оглавлению](../../README.md)
