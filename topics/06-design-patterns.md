# 06. Паттерны проектирования

> Статус: 🟡 в работе · Уровень: Senior

## Что проверяют
Знание паттернов GoF + корпоративных паттернов, умение назвать реальное применение и не «натягивать» паттерн там, где он не нужен.

## Ключевые вопросы (чеклист)
- [x] Порождающие: Singleton, Factory, Builder, Prototype
- [x] Структурные: Adapter, Decorator, Facade, Proxy, Composite
- [x] Поведенческие: Strategy, Observer, Command, Mediator, Chain of Responsibility, State, Template Method
- [x] Корпоративные: Repository + UoW, DI, Options, Result, Specification

## Порождающие

### Singleton
**Проблема:** единственный экземпляр на приложение.
**В .NET:** обычно через DI (`AddSingleton`), а не классический статический Singleton.
```csharp
public sealed class Config
{
    private static readonly Lazy<Config> _i = new(() => new Config());
    public static Config Instance => _i.Value;
    private Config() { }
}
```
**Когда НЕ применять:** глобальное мутабельное состояние = скрытые зависимости, проблемы с тестированием и потокобезопасностью. Предпочитайте DI-singleton (он управляем и тестируем).

### Factory Method / Abstract Factory
**Проблема:** инкапсулировать создание объектов, выбирать реализацию в рантайме.
**В .NET:** фабрики, `IServiceProvider`, `Func<T>`/делегаты-фабрики, `IHttpClientFactory`.

### Builder
**Проблема:** пошаговое построение сложного объекта; читаемый fluent API.
**Пример:** `StringBuilder`, `WebApplication.CreateBuilder`, конфигурация EF (`ModelBuilder`).

### Prototype
**Проблема:** создание копии существующего объекта. **В .NET:** `with` у record, `MemberwiseClone`.

## Структурные

### Adapter
**Проблема:** привести несовместимый интерфейс к ожидаемому. Часто как Anti-Corruption Layer на границе систем.

### Decorator
**Проблема:** добавить поведение объекту без изменения класса (обёртка с тем же интерфейсом).
```csharp
class CachingRepo(IRepo inner, IMemoryCache cache) : IRepo
{
    public Task<Item> Get(int id) =>
        cache.GetOrCreateAsync(id, _ => inner.Get(id));
}
```
**Применение:** кэширование/логирование/retry поверх сервиса; в DI — через декорирование регистраций (Scrutor).

### Facade
**Проблема:** упростить доступ к сложной подсистеме единым интерфейсом.

### Proxy
**Проблема:** подмена объекта для контроля доступа/ленивости/удалённого вызова. **В .NET:** EF lazy-loading proxies, динамические прокси (Castle) для перехвата (AOP).

### Composite
**Проблема:** работа с деревом объектов единообразно (узлы и листья). Пример: UI-дерево, файловая система.

## Поведенческие

### Strategy
**Проблема:** взаимозаменяемые алгоритмы. **В .NET:** интерфейс + DI, или просто `Func<>`-делегат.

### Observer
**Проблема:** уведомление подписчиков об изменениях. **В .NET:** события, `IObservable<T>`/Rx, domain events.

### Command
**Проблема:** инкапсуляция запроса как объекта (отмена, очередь, лог). **В .NET:** CQRS-команды, MediatR.

### Mediator
**Проблема:** развязать взаимодействие компонентов через посредника. **В .NET:** MediatR (центр CQRS).

### Chain of Responsibility
**Проблема:** передача запроса по цепочке обработчиков. **В .NET:** **middleware pipeline ASP.NET Core** — каноничный пример (см. тему 11), pipeline behaviors в MediatR, DelegatingHandler в HttpClient.

### State / Template Method
- **State** — поведение зависит от состояния (вместо больших switch). Пример: машина состояний заказа.
- **Template Method** — скелет алгоритма в базовом классе, шаги переопределяются.

## Корпоративные / .NET-специфичные

### Repository + Unit of Work
**Проблема:** абстракция доступа к данным + транзакционная единица.
**Нюанс:** EF Core `DbContext` уже = Unit of Work, `DbSet<T>` уже = Repository. Дополнительный Repository поверх EF часто избыточен и прячет возможности (Include, проекции). Оправдан для изоляции домена/тестов или смены провайдера.

### Dependency Injection
Паттерн внедрения зависимостей — встроен в ASP.NET Core (`IServiceCollection`). См. тему 10.

### Options pattern
Типобезопасная конфигурация через `IOptions<T>` (см. тему 10).

### Result pattern
Возврат успеха/ошибки без исключений (для ожидаемых ошибок): `Result<T>` с `IsSuccess`/`Error`. Снижает использование исключений как control flow.

### Specification pattern
Инкапсуляция критериев выборки в объекты-спецификации (комбинируемые), часто поверх `IQueryable`/EF.

## Подводные камни (общие)
- Паттерн ради паттерна = переусложнение. Сначала проблема, потом паттерн.
- Многие GoF-паттерны в C# вырождаются в делегаты/DI/встроенные механизмы.

## Полезные ссылки
- Refactoring Guru (паттерны с примерами): https://refactoring.guru/design-patterns
- .NET application architecture guides: https://learn.microsoft.com/dotnet/architecture/