<!-- nav-top -->
[← К оглавлению](../../../README.md)

# C# 8 (2019, .NET Core 3.0 / .NET Standard 2.1)

<a id="top"></a>

> Первая версия, многие фичи которой **требуют .NET Core 3.0+** (не работают на .NET Framework полноценно).

## Самое интересное

### 1. Nullable reference types (NRT)
Компилятор начинает различать `string` (не должен быть null) и `string?` (может быть null) и предупреждает о потенциальных `NullReferenceException`.

```csharp
#nullable enable
string notNull = null;   // CS8600 warning
string? maybe = null;    // ок
int len = maybe.Length;  // CS8602 — возможен null
```
- Включается через `<Nullable>enable</Nullable>` в `.csproj`.
- Это **только статический анализ** — в рантайме `?` ничего не меняет.
- Операторы: `?` (nullable), `!` (null-forgiving — «я знаю, что не null»).

### 2. Async streams — `IAsyncEnumerable<T>` + `await foreach`
Асинхронная ленивая последовательность: каждый элемент можно ждать.

```csharp
async IAsyncEnumerable<int> GenerateAsync()
{
    for (int i = 0; i < 3; i++)
    {
        await Task.Delay(100);
        yield return i;
    }
}

await foreach (var x in GenerateAsync())
    Console.WriteLine(x);
```
Идеально для постраничного чтения из БД/API без загрузки всего в память.

### 3. Switch expressions
Компактный `switch` как выражение, возвращающее значение.

```csharp
string Describe(int x) => x switch
{
    0 => "zero",
    > 0 => "positive",
    < 0 => "negative"
};
```

### 4. Pattern matching: property, tuple, positional patterns
```csharp
static decimal Discount(Order o) => o switch
{
    { Total: > 1000 } => 0.1m,
    { Items.Count: 0 } => 0m,
    _ => 0.05m
};
```

### 5. Ranges и индексы — `^` и `..`
```csharp
int[] a = { 0, 1, 2, 3, 4 };
int last = a[^1];        // 4 (с конца)
int[] mid = a[1..4];     // {1, 2, 3}
int[] tail = a[2..];     // {2, 3, 4}
```

### 6. `using`-declaration (без фигурных скобок)
```csharp
void Process()
{
    using var file = File.OpenRead("data.txt"); // Dispose в конце метода
    // ...
}
```

### 7. Default interface methods
Интерфейс может содержать реализацию по умолчанию — можно добавлять методы, не ломая существующих реализаторов.

```csharp
interface ILogger
{
    void Log(string msg);
    void LogError(string msg) => Log("ERROR: " + msg); // default
}
```

### Прочее
- `static` локальные функции (не захватывают переменные → без аллокаций).
- `??=` — null-coalescing assignment: `list ??= new();`
- `readonly`-члены структуры (метод гарантированно не мутирует struct).
- Disposable ref structs (`ref struct` может иметь `Dispose`).

[↑ Наверх](#top)

---

<!-- nav-bottom -->
[← К оглавлению](../../../README.md)
