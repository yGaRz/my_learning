<!-- nav-top -->
[← К оглавлению](../../../README.md)

# C# 12 (2023, .NET 8)

<a id="top"></a>

> Primary constructors для классов, collection expressions, alias любых типов.

## Самое интересное

### 1. Primary constructors для классов и структур
Параметры конструктора объявляются прямо в заголовке типа и доступны во всём теле (раньше — только у record).

```csharp
public class OrderService(IRepo repo, ILogger<OrderService> logger)
{
    public async Task SaveAsync(Order o)
    {
        logger.LogInformation("saving");
        await repo.SaveAsync(o);
    }
}
```
- Параметры захватываются в скрытые поля (создаются по необходимости).
- ⚠️ В отличие от record, **не** генерируются публичные свойства и не меняется equality — это просто удобный DI-конструктор.

### 2. Collection expressions `[...]`
Единый синтаксис инициализации для массивов, списков, `Span` и др.

```csharp
int[] a = [1, 2, 3];
List<int> list = [1, 2, 3];
Span<int> span = [1, 2, 3];

int[] more = [0, ..a, 4];   // spread-оператор .. вставляет элементы
```

### 3. Default-значения для лямбда-параметров и `params` в лямбдах
```csharp
var greet = (string name = "World") => $"Hello, {name}";
greet();        // Hello, World
greet("Ann");   // Hello, Ann
```

### 4. Alias любых типов через `using`
Псевдоним можно дать кортежу, массиву, generic-типу, указателю.

```csharp
using Point = (int X, int Y);
using IntList = System.Collections.Generic.List<int>;

Point p = (1, 2);
```

### 5. `ref readonly` параметры
Уточняет намерение: передаём по ссылке, но менять нельзя (мост между `ref` и `in`).

```csharp
void Read(ref readonly int x) { /* x только для чтения */ }
```

### Прочее
- **Inline arrays** (`[InlineArray(N)]`) — фиксированные буферы для высокопроизводительного кода.
- **Interceptors** (экспериментально) — source generator может «перехватывать» вызовы методов.

[↑ Наверх](#top)

---

<!-- nav-bottom -->
[← К оглавлению](../../../README.md)
