<!-- nav-top -->
[← К оглавлению](../../../README.md)

# C# 9 (2020, .NET 5)

<a id="top"></a>

> Версия «эпохи records». Сильный упор на иммутабельность и лаконичность.

## Самое интересное

### 1. Records (record class)
Ссылочный тип с **value equality**, `with`-выражениями, авто-`ToString`, деконструкцией.

```csharp
public record Person(string Name, int Age);

var p1 = new Person("Ann", 30);
var p2 = p1 with { Age = 31 };       // нон-деструктивная копия
bool eq = p1 == new Person("Ann", 30); // true — сравнение по значению
Console.WriteLine(p1);  // Person { Name = Ann, Age = 30 }
```

### 2. `init`-сеттеры
Свойство можно задать только при инициализации, потом оно иммутабельно.

```csharp
public class Point
{
    public int X { get; init; }
    public int Y { get; init; }
}

var p = new Point { X = 1, Y = 2 };
// p.X = 5; // ошибка — init только при создании
```

### 3. Target-typed `new()`
Тип можно не повторять, если он понятен из контекста.

```csharp
Dictionary<string, List<int>> map = new();   // вместо new Dictionary<...>()
Point p = new() { X = 1 };
```

### 4. Top-level statements
Программа без `class Program` и `Main` — точка входа прямо в файле.

```csharp
// Program.cs
using System;
Console.WriteLine("Hello");
```

### 5. Улучшенный pattern matching
Реляционные (`>`, `<`) и логические (`and`, `or`, `not`) паттерны.

```csharp
static string Size(int n) => n switch
{
    < 0 => "negative",
    0 => "zero",
    > 0 and < 100 => "small",
    >= 100 => "big"
};

if (obj is not null) { /* ... */ }
```

### Прочее
- **Covariant return types** — переопределённый метод может вернуть более производный тип.
- `static` анонимные функции/лямбды.
- `nint`/`nuint` — native-size целые.
- Атрибуты на локальных функциях.
- Module initializers (`[ModuleInitializer]`).

[↑ Наверх](#top)

---

<!-- nav-bottom -->
[← К оглавлению](../../../README.md)
