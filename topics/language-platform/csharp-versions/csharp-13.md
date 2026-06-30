<!-- nav-top -->
[← К оглавлению](../../../README.md)

# C# 13 (2024, .NET 9)

<a id="top"></a>

> `params` для любых коллекций, новый тип `System.Threading.Lock`, partial-свойства.

## Самое интересное

### 1. `params` для коллекций (`Span`, `IEnumerable`, и др.)
Раньше `params` работал только с массивом — теперь с `ReadOnlySpan<T>`, `IEnumerable<T>`, `List<T>` и т.д. Для `Span` — без лишних аллокаций массива.

```csharp
void Log(params ReadOnlySpan<string> parts) { /* ... */ }
Log("a", "b", "c");   // без аллокации массива в куче
```

### 2. Новый тип `System.Threading.Lock`
Выделенный тип блокировки вместо `lock(object)` — быстрее и понятнее. `lock` по такому объекту использует его `EnterScope()`.

```csharp
private readonly Lock _gate = new();

void Do()
{
    lock (_gate)   // компилятор вызывает _gate.EnterScope()
    {
        // критическая секция
    }
}
```

### 3. Partial-свойства и индексаторы
Свойства теперь можно объявлять `partial` (объявление в одном файле, реализация — в другом). Полезно для source generators.

```csharp
public partial class Vm
{
    public partial string Name { get; set; }   // объявление
}
// в сгенерированном файле — реализация
```

### 4. Новый escape `\e` (ESC, символ 0x1B)
```csharp
Console.WriteLine("\e[31mred text\e[0m");   // ANSI-цвета без "\u001b"
```

### 5. Неявный индекс `^` в инициализаторах объектов
```csharp
var buffer = new Buffer { [^1] = 42 };   // последний элемент
```

### 6. `ref` и `unsafe` в итераторах и async
Ослаблены ограничения: можно использовать `ref struct`/`ref` локали в большем числе сценариев, `ref struct` может реализовывать интерфейсы и быть аргументом generic (`where T : allows ref struct`).

### Прочее
- Method group natural type — улучшение разрешения перегрузок.
- `field` keyword (preview) — доступ к backing-полю авто-свойства без явного объявления.
- Частичное снятие ограничений на `ref struct` в дженериках через `allows ref struct`.

[↑ Наверх](#top)

---

<!-- nav-bottom -->
[← К оглавлению](../../../README.md)
