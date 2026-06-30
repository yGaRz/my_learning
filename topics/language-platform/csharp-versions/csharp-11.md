<!-- nav-top -->
[← К оглавлению](../../../README.md)

# C# 11 (2022, .NET 7)

<a id="top"></a>

> Raw strings, required-члены, list patterns и generic math.

## Самое интересное

### 1. Raw string literals (`"""`)
Строки без экранирования — удобно для JSON, regex, путей, SQL.

```csharp
string json = """
    {
        "name": "Ann",
        "path": "C:\Users\Ann"
    }
    """;
```
- Минимум три `"`, можно больше, если внутри есть последовательности кавычек.
- Отступ определяется по закрывающим кавычкам — лишний отступ обрезается.
- Можно комбинировать с интерполяцией: `$$"""...{{value}}..."""` (двойные `$`/`{` чтобы не конфликтовать с `{` внутри).

### 2. `required`-члены
Свойство обязано быть инициализировано при создании объекта — гарантия на этапе компиляции без конструктора.

```csharp
public class User
{
    public required string Name { get; init; }
    public required string Email { get; init; }
}

var u = new User { Name = "Ann", Email = "a@b.c" }; // оба обязательны
// var bad = new User { Name = "Ann" };  // CS9035 — Email required
```

### 3. List patterns
Сопоставление с шаблоном для массивов/списков.

```csharp
int[] a = { 1, 2, 3 };
bool m = a is [1, 2, 3];          // true
bool startsWith1 = a is [1, ..];  // true
if (a is [var first, .., var last]) { /* first=1, last=3 */ }
```

### 4. Generic math — `INumber<T>` и static abstract в интерфейсах
Можно писать обобщённые алгоритмы над числами; интерфейсы получили **статические абстрактные члены** (в т.ч. операторы).

```csharp
T Sum<T>(IEnumerable<T> items) where T : INumber<T>
{
    T total = T.Zero;
    foreach (var x in items) total += x;
    return total;
}
```

### 5. `field` keyword? — нет, это позже; здесь — auto-default structs
В конструкторе структуры больше не обязательно инициализировать все поля — неинициализированные получают `default`.

### Прочее
- **UTF-8 string literals**: `"text"u8` → `ReadOnlySpan<byte>` без аллокаций.
- **Newlines в интерполяции** — выражения в `{}` могут занимать несколько строк.
- **`ref` поля в `ref struct`** и `scoped` для управления lifetime (основа `Span`-сценариев).
- Generic attributes: `[MyAttr<int>]`.
- `nameof` для параметров метода/типа в атрибутах.
- Checked/unchecked пользовательские операторы.

[↑ Наверх](#top)

---

<!-- nav-bottom -->
[← К оглавлению](../../../README.md)
