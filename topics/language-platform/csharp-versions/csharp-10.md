<!-- nav-top -->
[← К оглавлению](../../../README.md)

# C# 10 (2021, .NET 6)

<a id="top"></a>

> Версия «меньше шаблонного кода»: global usings, file-scoped namespaces, record struct.

## Самое интересное

### 1. `record struct` и `readonly record struct`
Значимый тип со всеми удобствами record (value equality, `with`, `ToString`), но семантикой копирования по значению.

```csharp
public readonly record struct Money(decimal Amount, string Currency);

var a = new Money(10, "USD");
var b = a with { Amount = 20 };
bool eq = a == new Money(10, "USD"); // true, без боксинга и рефлексии
```

### 2. Global usings
Один раз объявил — доступно во всём проекте.

```csharp
// GlobalUsings.cs
global using System;
global using System.Collections.Generic;
```
Плюс `<ImplicitUsings>enable</ImplicitUsings>` в `.csproj` подключает базовый набор автоматически.

### 3. File-scoped namespaces
Меньше вложенности — namespace на весь файл без фигурных скобок.

```csharp
namespace MyApp.Services;   // вместо namespace MyApp.Services { ... }

public class Foo { }
```

### 4. Constant interpolated strings
Интерполяция в `const`, если все части — константы.

```csharp
const string Name = "App";
const string Title = $"{Name} v1";   // ок
```

### 5. Extended property patterns
Можно «проваливаться» в вложенные свойства без скобок.

```csharp
if (person is { Address.City: "Moscow" }) { /* ... */ }
// раньше: { Address: { City: "Moscow" } }
```

### 6. Record struct + улучшения record
- `record class` можно писать явно (`record` = `record class`).
- `ToString` у record теперь можно «запечатать» (`sealed override`).

### Прочее
- Lambda improvements: естественный тип у лямбд (`var f = () => 1;`), атрибуты и явный возвращаемый тип на лямбдах.
- `with` для **анонимных типов** и обычных `struct`.
- Деконструкция и присваивание в одном выражении: `(x, y) = (1, 2);` с уже объявленными переменными.
- `[CallerArgumentExpression]` — получить текст выражения аргумента (удобно для guard-проверок).

[↑ Наверх](#top)

---

<!-- nav-bottom -->
[← К оглавлению](../../../README.md)
