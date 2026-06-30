

[← К оглавлению](../../README.md)

# 01. C# язык и его возможности



> Статус: 🟡 в работе · Уровень: Senior

## Что проверяют

Глубокое знание семантики языка, отличий value/reference типов, новых возможностей C#, понимание того, во что компилируется код.

## Ключевые вопросы (чеклист)

- [x] [Value type vs reference type: где живут, как передаются, boxing/unboxing](#q1)
- [x] [`struct` vs `class` vs `record` vs `record struct` — когда что](#q2)
- [ ] [`string` — иммутабельность, интернирование, `StringBuilder`, `Span<char>`](#q3)
- [ ] [`ref`, `out`, `in`, `ref readonly`, `ref struct` (`Span<T>`, почему нельзя в async)](#q4)
- [ ] [Замыкания (closures): захват переменных, типичные баги в циклах](#q5)
- [ ] [Делегаты, `Func`/`Action`/`Predicate`, события, `EventHandler`](#q6)
- [ ] [`IEnumerable` vs `IQueryable`, отложенное выполнение, `yield`](#q7)
- [ ] [`IDisposable`, `using`, `IAsyncDisposable`, finalizers](#q8)
- [ ] [Generics: ковариантность/контравариантность (`in`/`out`), ограничения (`where`)](#q9)
- [ ] [Nullable reference types, `?`, `!`, `??`, `??=`](#q10)
- [ ] [Pattern matching, switch expressions](#q11)
- [ ] [`record` — value equality, `with`-выражения, деконструкция](#q12)
- [ ] [Extension methods, как резолвятся](#q13)
- [ ] [`dynamic`, `var`, `object` — различия](#q14)
- [ ] [Что нового в C# 9/10/11/12/13 (init, file-scoped namespaces, primary constructors, collection expressions)](#q15)
- [ ] [Перегрузка операторов, неявные/явные приведения](#q16)
- [ ] [Атрибуты и рефлексия, Source Generators](#q17)



## Разбор вопросов



### Вопрос: Value type vs reference type — где живут, как передаются, boxing/unboxing

**Краткий ответ:**

- **Value type** (`struct`, `enum`, примитивы `int`/`double`/`bool`, `DateTime`, кортежи `(int, int)`): содержит **сами данные**. Переменная хранит значение. При присваивании и передаче в метод — **копируется целиком**.
- **Reference type** (`class`, `interface`, `delegate`, `string`, массивы): переменная хранит **ссылку** (указатель) на объект в управляемой куче. При присваивании копируется ссылка, оба имени указывают на один объект.
- «Value type всегда на стеке» — **неточно**. Размещение зависит от контекста: локальная value-переменная — на стеке; поле класса типа `int` лежит **в куче внутри объекта**; захваченная замыканием или забоксенная — в куче. Корректнее: value type хранится «там, где живёт его контейнер».
- **Boxing** — упаковка value type в объект на куче (`object o = 42;`): аллокация + копирование. **Unboxing** — обратное извлечение с проверкой типа. Это скрытый источник аллокаций и нагрузки на GC.

**Пример:**

```csharp
struct Point { public int X; }
void Mutate(Point p) => p.X = 99;   // меняет КОПИЮ

var a = new Point { X = 1 };
Mutate(a);
// a.X == 1 — оригинал не изменился

// Boxing/unboxing
int n = 42;
object boxed = n;        // boxing: аллокация в куче
int back = (int)boxed;   // unboxing
```

**Подводные камни:**

- Мутабельные структуры опасны: легко изменить копию вместо оригинала. Делайте структуры `readonly`.
- Большие структуры дорого копировать — передавайте через `in` (read-only ref).
- Боксинг возникает незаметно: упаковка `struct` в `object`/интерфейс, использование не-обобщённых коллекций, `enum.ToString()`, интерполяция/`string.Format` с value-аргументами.

**Почему мутабельные структуры опасны:**
Любое обращение к struct, которое не является прямой переменной, возвращает **копию**. Это значит, что мутирующий вызов меняет временную копию, которая тут же выбрасывается, — компилятор при этом не выдаёт ошибки, и баг тихо живёт в коде.

```csharp
struct Counter { public int Value; public void Inc() => Value++; }

// 1. Элемент списка возвращается КОПИЕЙ
var list = new List<Counter> { new Counter() };
list[0].Inc();        // меняет копию -> list[0].Value == 0
// list[0] = new Counter { Value = list[0].Value + 1 };  // так правильно

// 2. readonly-поле: вызов метода работает с защитной копией
class Holder { public readonly Counter C = new(); }
new Holder().C.Inc(); // тоже меняет копию

// 3. foreach даёт копии — элемент вообще нельзя мутировать
foreach (var c in list) c.Inc(); // CS1654 / меняет копию
```

Решение — делать структуры неизменяемыми: `readonly struct`. Тогда компилятор запрещает мутацию полей, а методы не создают лишних защитных копий (производительность).

```csharp
readonly struct Point
{
    public int X { get; }              // только get
    public Point(int x) => X = x;
    public Point WithX(int x) => new(x); // «изменение» = новый экземпляр
}
```

**Где боксинг возникает незаметно** (каждый случай — аллокация в куче + нагрузка на GC):

```csharp
// 1. Присваивание в object или интерфейс
object o = 42;                    // boxing
IComparable c = 42;              // boxing (struct -> интерфейс)

// 2. Не-обобщённые коллекции (ArrayList, Hashtable, очереди из System.Collections)
var al = new System.Collections.ArrayList();
al.Add(42);                       // boxing каждого элемента

// 3. enum.ToString() и форматирование enum
DayOfWeek d = DayOfWeek.Monday;
string s = d.ToString();          // boxing enum

// 4. Интерполяция / string.Format со value-аргументами
//    (в string.Format всегда; в $"" — зависит от версии/перегрузок,
//     современный DefaultInterpolatedStringHandler многое убирает)
string msg = string.Format("x = {0}", 42); // 42 boxing в object[]

// 5. Вызов метода интерфейса через ограничение без where T : struct
//    или вызов не переопределённого метода Object на struct
```

Как избегать: использовать обобщённые коллекции (`List<T>`, `Dictionary<K,V>`), вызывать `Enum`-перегрузки/`nameof` где можно, для логирования брать структурные логгеры (Serilog/`ILogger` с шаблонами), для собственных типов переопределять `ToString()`, `Equals`, `GetHashCode` и реализовывать `IEquatable<T>`, чтобы избежать боксинга при сравнении.

[↑ Наверх](#top)

---



### Вопрос: `struct` vs `class` vs `record` vs `record struct` — когда что

**Краткий ответ:**

- `class` — ссылочный тип, reference equality по умолчанию, наследование. Дефолтный выбор для сущностей с идентичностью и поведением.
- `struct` — значимый тип, value equality (но дефолтный `Equals` через рефлексию — медленный, лучше переопределять). Подходит для маленьких (≈≤16 байт), иммутабельных, часто создаваемых значений без идентичности (координаты, деньги). Нет наследования.
- `record` **(record class)** — ссылочный тип с автоматическим **value equality** (по значениям свойств), `with`-выражениями, деконструкцией, удобным `ToString`. Для DTO, immutable-моделей, value objects.
- `record struct` — значимый тип с теми же удобствами record (value equality, `with`), но семантика копирования значения. `readonly record struct` — иммутабельный значимый.

**Пример:**

```csharp
public record Person(string Name, int Age);          // ссылочный, value equality
public readonly record struct Money(decimal Amount, string Currency); // значимый, immutable

var p1 = new Person("Ann", 30);
var p2 = p1 with { Age = 31 };   // копия с изменением одного поля
bool eq = p1 == new Person("Ann", 30); // true — сравнение по значениям
```

**Подводные камни:**

- `record` всё ещё ссылочный тип (аллокации в куче) — это не «бесплатная» иммутабельность.
- Value equality у record сравнивает все публичные поля/свойства — для коллекций-полей это reference equality (две одинаковые по содержимому коллекции не равны).
- `struct` по умолчанию мутабельный — это частый источник багов; делайте `readonly struct`.

**Что такое value equality (равенство по значению):**

Есть два принципиально разных смысла «равенства»:

- **Reference equality (равенство по ссылке)** — два объекта равны, только если это **один и тот же объект** в куче (один адрес). Это поведение `class` по умолчанию: `==` и `Equals` сравнивают ссылки.
- **Value equality (равенство по значению)** — два объекта равны, если **совпадает их содержимое** (значения полей/свойств), даже если это физически разные экземпляры в памяти.

```csharp
// class — reference equality по умолчанию
class PointC { public int X, Y; }
var a = new PointC { X = 1, Y = 2 };
var b = new PointC { X = 1, Y = 2 };
bool e1 = a == b;        // false — разные объекты, хоть и одинаковое содержимое
bool e2 = a == a;        // true  — одна ссылка

// record — value equality автоматически
record PointR(int X, int Y);
var c = new PointR(1, 2);
var d = new PointR(1, 2);
bool e3 = c == d;        // true — содержимое совпадает
bool e4 = ReferenceEquals(c, d); // false — это всё равно разные объекты в куче
```

**Кто и как использует value equality:**

- `struct` и `record`/`record struct` дают value equality «из коробки».
- У обычного `class` его нет — нужно вручную переопределить `Equals(object)`, `GetHashCode()` и (по желанию) `==`/`!=`, реализовать `IEquatable<T>`.

**Что компилятор генерирует для** `record` **(важно понимать на собеседовании):**

- `bool Equals(T other)` — сравнивает тип (`EqualityContract`) и все публичные поля/автосвойства через `EqualityComparer<T>.Default`.
- `GetHashCode()` — комбинирует хеши тех же членов.
- Перегрузку операторов `==` и `!=`.
- Реализацию `IEquatable<T>`.

```csharp
// Примерно так record разворачивается компилятором
public bool Equals(PointR other) =>
    other is not null &&
    EqualityContract == other.EqualityContract &&
    EqualityComparer<int>.Default.Equals(X, other.X) &&
    EqualityComparer<int>.Default.Equals(Y, other.Y);
```

**Важные нюансы value equality:**

- Сравнение идёт по **полям/свойствам**, а если поле — ссылочный тип (например `List<T>`, массив), то сравнивается **ссылка этого поля**, а не его содержимое. Поэтому два record с разными, но содержательно одинаковыми списками — **не равны**.

```csharp
record Order(int Id, List<string> Items);
var o1 = new Order(1, new() { "a" });
var o2 = new Order(1, new() { "a" });
bool eq = o1 == o2;   // false! списки — разные объекты (reference equality поля)
```

- `GetHashCode` **обязан быть согласован с** `Equals`: если `a.Equals(b)` → `true`, то `a.GetHashCode() == b.GetHashCode()`. Иначе ломаются `Dictionary`/`HashSet`.
- Хеш-код **нельзя** строить на изменяемых полях, если объект используется как ключ словаря: измените поле — потеряете объект в `HashSet`/`Dictionary`. Поэтому value equality естественно сочетается с **иммутабельностью**.
- У `record` с наследованием в сравнение входит `EqualityContract` (тип), поэтому `BaseRecord` и `DerivedRecord` с одинаковыми полями **не равны** — это защищает от «срезки типа».
- Дефолтный `ValueType.Equals` для `struct` работает через **рефлексию и боксинг** — медленный; для часто сравниваемых структур переопределяйте `Equals`/`GetHashCode` и реализуйте `IEquatable<T>` вручную.

**Как работает дефолтный** `Equals`**/**`GetHashCode` **у struct (и почему он медленный):**

Любой `struct` неявно наследуется от `System.ValueType`, который переопределяет `Equals(object)` и `GetHashCode()` за вас. Но реализация — общая для всех структур, поэтому работает «вслепую», не зная конкретных полей на этапе компиляции:

1. **Боксинг аргумента.** Сигнатура — `Equals(object obj)`. Чтобы передать туда другую структуру, её надо упаковать (boxing) → аллокация в куче.
2. **Рантайм-проверка типа.** Сравнивается, что `obj` той же структуры.
3. **Сравнение полей.**
  - Если структура «blittable» (содержит только простые value-поля без ссылок и без «дырок» от выравнивания) — CLR может сравнить память **байт-в-байт** (быстрый путь).
  - Иначе CLR идёт по полям **через рефлексию** (`FieldInfo`), читает значение каждого поля, **боксит** его и вызывает `Equals` рекурсивно. Это медленно и снова аллоцирует.

```csharp
struct Point { public int X; public double Y; public string Label; }

var a = new Point { X = 1, Y = 2.0, Label = "p" };
var b = new Point { X = 1, Y = 2.0, Label = "p" };

bool eq = a.Equals(b);
// Что происходит под капотом при дефолтной реализации:
//  - b упаковывается в object (boxing)
//  - есть ссылочное поле (string Label) -> blittable-путь невозможен
//  - ValueType.Equals идёт рефлексией по полям X, Y, Label,
//    боксит каждое значение и сравнивает -> медленно + мусор
```

**Почему нельзя «просто сравнивать байты всегда»:**

- Поля-ссылки (`string`, `object`) надо сравнивать через их `Equals`, а не по адресу.
- В памяти структуры бывают **padding-байты** (выравнивание) с мусорными значениями — побайтовое сравнение дало бы ложное «не равно».
- `0.0` и `-0.0`, а также `NaN` имеют особую семантику в `double`, которую байты не отражают корректно.

`GetHashCode()` **по умолчанию ещё коварнее:** исторически `ValueType.GetHashCode` для производительности берёт хеш **только первого поля** (если оно не подходит — комбинирует поля рефлексией). Если первое поле у всех экземпляров одинаковое, все они попадут в один bucket словаря → деградация `Dictionary`/`HashSet` до O(n).

**Правильно — переопределить вручную и реализовать** `IEquatable<T>` (типизированный `Equals` без боксинга):

```csharp
readonly struct Point : IEquatable<Point>
{
    public int X { get; }
    public double Y { get; }
    public string Label { get; }

    public Point(int x, double y, string label) => (X, Y, Label) = (x, y, label);

    // Типизированный Equals — без boxing, без рефлексии
    public bool Equals(Point other) =>
        X == other.X && Y == other.Y && Label == other.Label;

    // Переопределяем и object-версию, чтобы поведение было согласованным
    public override bool Equals(object? obj) => obj is Point p && Equals(p);

    public override int GetHashCode() => HashCode.Combine(X, Y, Label);

    public static bool operator ==(Point a, Point b) => a.Equals(b);
    public static bool operator !=(Point a, Point b) => !a.Equals(b);
}
```

> Именно поэтому `record struct` так удобен: компилятор сам генерирует типизированный `IEquatable<T>.Equals`, согласованный `GetHashCode` и операторы — без рефлексии и боксинга.

[↑ Наверх](#top)

---



### Вопрос: `string` — иммутабельность, интернирование, `StringBuilder`, `Span<char>`

**Краткий ответ:**

- `string` — **иммутабельный** ссылочный тип. Любая «модификация» создаёт новую строку. Поэтому конкатенация в цикле = много мусора в куче.
- **Интернирование (string interning)**: строковые литералы хранятся в общем пуле, одинаковые литералы ссылаются на один объект. `string.Intern`/`IsInterned` — ручное управление (осторожно: интернированные строки живут до конца домена).
- `StringBuilder` — мутабельный буфер для построения строк; снижает аллокации при множественной конкатенации.
- `Span<char>` **/** `ReadOnlySpan<char>` — работа с подстроками **без аллокаций**: `Slice`, парсинг, `string.Create`. `AsSpan()` не создаёт новую строку, в отличие от `Substring`.

**Пример:**

```csharp
// Плохо: N промежуточных строк
string s = "";
for (int i = 0; i < 1000; i++) s += i;

// Хорошо:
var sb = new StringBuilder();
for (int i = 0; i < 1000; i++) sb.Append(i);
string result = sb.ToString();

// Без аллокаций — срез вместо Substring
ReadOnlySpan<char> span = "2026-06-30".AsSpan();
ReadOnlySpan<char> year = span.Slice(0, 4);   // никаких новых string
int y = int.Parse(year);
```

**Подводные камни:**

- `StringBuilder` оправдан при многих конкатенациях; для 2-3 склеек обычная интерполяция/`+` читабельнее и не хуже.
- `==` для строк сравнивает по значению (содержимому), а не по ссылке — это часто удивляет на фоне остальных ссылочных типов.
- Сравнение строк культурозависимо: для ключей/идентификаторов используйте `StringComparison.Ordinal`.

[↑ Наверх](#top)

---



### Вопрос: `ref`, `out`, `in`, `ref readonly`, `ref struct`

**Краткий ответ:**

- `ref` — передача по ссылке: метод видит и может менять переменную вызывающего; должна быть инициализирована до вызова.
- `out` — передача по ссылке для **возврата** значения; обязана быть присвоена внутри метода (до вызова можно не инициализировать).
- `in` — передача по ссылке **только для чтения**: избегаем копии большой структуры без права изменения.
- `ref readonly` — возврат/хранение read-only ссылки (например, элемент большого массива структур без копирования).
- `ref struct` (`Span<T>`, `ReadOnlySpan<T>`) — тип, который **обязан жить только на стеке**. Нельзя положить в поле класса, забоксить, использовать в `async`/итераторах/лямбдах, захватывающих его.

**Почему** `Span<T>` **нельзя в** `async`**:** async-метод компилируется в стейт-машину, локальные переменные «поднимаются» в поле объекта-состояния (в куче), а `ref struct` запрещён в куче. Поэтому `Span<T>` нельзя держать через `await`. Решение — `Memory<T>` (его можно), а `Span` получать из него синхронными участками.

**Пример:**

```csharp
void Swap(ref int a, ref int b) { (a, b) = (b, a); }

bool TryParse(string s, out int value) => int.TryParse(s, out value);

decimal Sum(in BigStruct s) => s.A + s.B;  // без копии, без права менять s

async Task BadAsync(Memory<byte> mem)
{
    await SomethingAsync();
    Span<byte> span = mem.Span;  // ок: получили после await, не держим через await
}
```

**Подводные камни:**

- `in` может незаметно создавать **defensive copy**, если структура не `readonly` (компилятор страхуется от мутаций) — теряется выигрыш. Делайте `readonly struct`.
- `ref`/`out` усложняют API; для возврата нескольких значений часто лучше кортеж или record.

[↑ Наверх](#top)

---



### Вопрос: Замыкания (closures) — захват переменных, баги в циклах

**Краткий ответ:**

- Лямбда/анонимный метод **захватывает переменные по ссылке** (точнее — переменная «поднимается» в скрытый класс замыкания, аллоцируемый в куче). Захватывается **переменная, а не её значение** на момент создания делегата.
- Классический баг — захват переменной цикла. В современном C# `foreach` имеет отдельную переменную на итерацию (исправлено с C# 5), а вот переменная классического `for` — общая на все итерации.

**Пример:**

```csharp
// for: одна переменная i на все итерации -> все делегаты печатают 3
var actions = new List<Action>();
for (int i = 0; i < 3; i++) actions.Add(() => Console.Write(i));
foreach (var a in actions) a();   // 3 3 3

// Фикс: локальная копия в теле цикла
for (int i = 0; i < 3; i++) { int copy = i; actions.Add(() => Console.Write(copy)); } // 0 1 2
```

**Подводные камни:**

- Замыкание аллоцирует объект в куче — на горячем пути это давление на GC; статические лямбды (`static () => ...`) запрещают захват и помогают избегать случайных аллокаций.
- Захват `this` в лямбде продлевает жизнь объекта (потенциальная утечка через долгоживущие подписки на события).

[↑ Наверх](#top)

---



### Вопрос: Делегаты, `Func`/`Action`/`Predicate`, события

**Краткий ответ:**

- **Делегат** — типобезопасный указатель на метод (по сути класс, наследник `MulticastDelegate`). Может быть multicast (список методов).
- `Action<...>` — возвращает `void`; `Func<...,TResult>` — возвращает значение; `Predicate<T>` — `Func<T,bool>`.
- **Событие (**`event`**)** — обёртка над делегатом, ограничивающая доступ: извне можно только `+=`/`-=`, нельзя вызвать или присвоить `=`. Защищает паблишера.
- Соглашение: `EventHandler`/`EventHandler<TArgs>` с сигнатурой `(object? sender, TArgs e)`.

**Пример:**

```csharp
public event EventHandler<OrderEventArgs>? OrderCreated;

protected void OnOrderCreated(Order o)
    => OrderCreated?.Invoke(this, new OrderEventArgs(o));  // ?. — потокобезопаснее, чем проверка на null
```

**Подводные камни:**

- **Утечка памяти через события**: подписчик живёт, пока паблишер держит ссылку на него через делегат. Не забывайте `-=` (или weak-events).
- Multicast-делегат с возвращаемым значением вернёт результат только последнего метода.

[↑ Наверх](#top)

---



### Вопрос: `IEnumerable` vs `IQueryable`, отложенное выполнение, `yield`

**Краткий ответ:**

- `IEnumerable<T>` — последовательность в памяти; LINQ-операторы (`LINQ to Objects`) выполняются как **делегаты в C#**.
- `IQueryable<T>` — запрос как **дерево выражений** (`Expression`), которое провайдер (EF Core) транслирует в SQL и выполняет на сервере БД.
- **Отложенное выполнение (deferred execution)**: LINQ-запрос не выполняется в момент описания — только при перечислении (`foreach`, `ToList`, `Count` и т.п.).
- `yield return` — ленивый итератор: значения генерируются по требованию, без материализации всей коллекции.

**Пример:**

```csharp
// IEnumerable: Where выполнится в памяти (после загрузки всех строк!)
IEnumerable<User> q1 = db.Users.AsEnumerable().Where(u => u.Age > 18);

// IQueryable: Where уйдёт в SQL (WHERE Age > 18)
IQueryable<User> q2 = db.Users.Where(u => u.Age > 18);

IEnumerable<int> Naturals()
{
    int i = 1;
    while (true) yield return i++;   // бесконечная ленивая последовательность
}
```

**Подводные камни:**

- Преждевременный `AsEnumerable()`/`ToList()` тянет всю таблицу в память — частая причина тормозов с EF.
- **Множественный перебор** отложенного запроса = повторное выполнение (и повторный запрос к БД). Материализуйте через `ToList()`, если перебираете несколько раз.
- Захват изменяемой переменной в отложенном запросе даёт «неожиданное» значение на момент выполнения.

[↑ Наверх](#top)

---



### Вопрос: `IDisposable`, `using`, `IAsyncDisposable`, финализаторы

**Краткий ответ:**

- `IDisposable.Dispose()` — детерминированное освобождение **неуправляемых** ресурсов (файлы, сокеты, соединения) и подписок.
- `using` гарантирует вызов `Dispose()` даже при исключении. `using var x = ...;` — до конца области видимости.
- **Финализатор (**`~Class`**)** — страховка на случай, если забыли `Dispose`; вызывается GC недетерминированно, замедляет сборку (объект переживает лишний цикл GC). Нужен только при прямом владении неуправляемым ресурсом.
- `IAsyncDisposable.DisposeAsync()` + `await using` — для асинхронной очистки (например, flush в поток).

**Пример (Dispose pattern):**

```csharp
public class Resource : IDisposable
{
    private bool _disposed;
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);   // отменяем финализацию — уже всё освободили
    }
    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;
        if (disposing) { /* освободить управляемые */ }
        /* освободить неуправляемые */
        _disposed = true;
    }
}
```

**Подводные камни:**

- Забытый `Dispose` у соединений → утечки/исчерпание портов; но `HttpClient` как раз НЕ нужно оборачивать в `using` (см. тему 11).
- Финализатор без `GC.SuppressFinalize` в `Dispose` оставляет объект на лишний цикл GC.

[↑ Наверх](#top)

---



### Вопрос: Generics — ковариантность/контравариантность, ограничения

**Краткий ответ:**

- **Дженерики** дают типобезопасность без боксинга и дублирования кода. Для value-типов CLR генерирует специализированный код (нет боксинга), для ссылочных — переиспользует.
- **Ковариантность (**`out T`**)** — можно использовать более производный тип как менее производный: `IEnumerable<string>` → `IEnumerable<object>`. Только для **возвращаемых** позиций.
- **Контравариантность (**`in T`**)** — наоборот: `IComparer<object>` → `IComparer<string>`. Только для **входных** позиций.
- Вариантность поддерживается только интерфейсами и делегатами (не классами) и только для ссылочных типов.
- **Ограничения (**`where`**)**: `where T : class`, `struct`, `new()`, `IInterface`, `BaseClass`, `notnull`, `unmanaged`.

**Пример:**

```csharp
interface IProducer<out T> { T Get(); }     // ковариантный
interface IConsumer<in T> { void Set(T v); } // контравариантный

IProducer<string> ps = ...;
IProducer<object> po = ps;   // ок благодаря out

T Max<T>(T a, T b) where T : IComparable<T>
    => a.CompareTo(b) >= 0 ? a : b;
```

**Подводные камни:**

- Вариантность не работает для value-типов: `IEnumerable<int>` НЕ приводится к `IEnumerable<object>`.
- `out`/`in` имеют ограничения на позиции использования параметра — компилятор это проверяет.

[↑ Наверх](#top)

---



### Вопрос: Nullable reference types, `?`, `!`, `??`, `??=`

**Краткий ответ:**

- **NRT** (с C# 8, по умолчанию в новых проектах) — статический анализ на этапе компиляции: `string` считается non-nullable, `string?` — nullable. Это **подсказки компилятора**, в рантайме типы те же (не путать с `Nullable<T>`/`int?` для value-типов, который реальный тип-обёртка).
- `?.` — null-conditional (вернёт null, если левая часть null).
- `??` — null-coalescing (значение по умолчанию), `??=` — присвоить, если null.
- `!` — null-forgiving оператор: «я гарантирую, что не null», подавляет предупреждение (но не проверку в рантайме!).

**Пример:**

```csharp
string? name = GetName();
int len = name?.Length ?? 0;          // null-safe
name ??= "default";                   // присвоить, если был null
string sure = name!;                  // подавить предупреждение (на свой риск)
```

**Подводные камни:**

- `!` ничего не проверяет в рантайме — злоупотребление маскирует реальные `NullReferenceException`.
- NRT не защищает на границах с кодом без аннотаций (рефлексия, десериализация, старые библиотеки) — там null может «просочиться».

[↑ Наверх](#top)

---



### Вопрос: Pattern matching и switch expressions

**Краткий ответ:**

- Сопоставление с образцом: type patterns, property patterns, relational (`> 5`), logical (`and`/`or`/`not`), list patterns (C# 11), positional (деконструкция).
- `switch`-выражение возвращает значение, компактнее классического `switch`.

**Пример:**

```csharp
decimal Discount(Customer c) => c switch
{
    { Orders: > 100, IsVip: true } => 0.2m,
    { Orders: > 100 }              => 0.1m,
    null                           => throw new ArgumentNullException(),
    _                              => 0m
};

string Describe(object o) => o switch
{
    int n and > 0  => $"positive {n}",
    int            => "non-positive int",
    string s       => $"string of {s.Length}",
    _              => "other"
};
```

**Подводные камни:**

- Неполный `switch`-expression без `_` может бросить `SwitchExpressionException` в рантайме — компилятор предупреждает о неисчерпанности.

[↑ Наверх](#top)

---



### Вопрос: `record` — value equality, `with`, деконструкция

**Краткий ответ:**

- Компилятор для record генерирует: `Equals`/`GetHashCode` по значениям, `==`/`!=`, `ToString`, `Deconstruct` (для позиционных), копирующий конструктор и поддержку `with`.
- `with` создаёт **неглубокую копию** с изменением части свойств (использует init-сеттеры).

**Пример:**

```csharp
public record Point(int X, int Y);

var p = new Point(1, 2);
var (x, y) = p;              // деконструкция
var moved = p with { Y = 5 };
Console.WriteLine(p == new Point(1, 2)); // True
```

**Подводные камни:**

- `with` делает **shallow copy** — вложенные ссылочные объекты разделяются между копиями.
- Value equality сравнивает поля-коллекции по ссылке, а не поэлементно.

[↑ Наверх](#top)

---



### Вопрос: Extension methods — как резолвятся

**Краткий ответ:**

- Статический метод в статическом классе с `this` у первого параметра. Компилятор переписывает `obj.Ext()` в `StaticClass.Ext(obj)`.
- Видимы только при наличии `using` нужного пространства имён. **Метод экземпляра всегда приоритетнее** одноимённого extension-метода.

**Пример:**

```csharp
public static class StringExtensions
{
    public static bool IsNullOrEmpty(this string? s) => string.IsNullOrEmpty(s);
}
// "abc".IsNullOrEmpty();
```

**Подводные камни:**

- Можно вызывать на `null` (это просто аргумент) — поэтому extension-метод должен сам обрабатывать null.
- Конфликты имён из разных namespace разрешаются по `using`; это может приводить к неожиданной перегрузке.

[↑ Наверх](#top)

---



### Вопрос: `dynamic`, `var`, `object` — различия

**Краткий ответ:**

- `var` — статическая типизация, тип выводится компилятором на этапе компиляции. Никакой динамики, чисто синтаксис.
- `object` — базовый тип всех типов; доступны только члены `object`, для остального нужен каст/боксинг.
- `dynamic` — обход проверки типов на этапе компиляции; разрешение членов откладывается на рантайм (DLR). Ошибки вылезают в рантайме, есть оверхед.

**Пример:**

```csharp
var i = 5;          // int (на этапе компиляции)
object o = 5;       // boxing, члены int недоступны без каста
dynamic d = 5;
d.Foo();            // компилируется, но упадёт в рантайме (RuntimeBinderException)
```

**Подводные камни:**

- `dynamic` отключает IntelliSense и проверки — использовать точечно (COM, JSON-как-dynamic, интеграции).

[↑ Наверх](#top)

---



### Вопрос: Что нового в C# 9–13 (ключевое для Senior)

**Краткий ответ (по версиям):**

- **C# 9**: `record`, init-сеттеры, target-typed `new()`, top-level statements, улучшенный pattern matching.
- **C# 10**: `record struct`, global usings, file-scoped namespaces, const-интерполяция.
- **C# 11**: required-члены, raw string literals (`"""`), list patterns, generic math (`INumber<T>`).
- **C# 12**: primary constructors для классов/структур, collection expressions (`[1, 2, 3]`), `ref readonly` параметры, alias любых типов.
- **C# 13**: `params` для коллекций, `Lock` тип, `\e` escape, partial-свойства, неявный индекс `^` в инициализаторах.

**Пример:**

```csharp
// primary constructor + collection expression (C# 12)
public class Service(IRepo repo)
{
    private readonly int[] _defaults = [1, 2, 3];
    public Task Do() => repo.SaveAsync(_defaults);
}
```

**Подводные камни:**

- Primary constructor у класса захватывает параметры в скрытые поля — легко случайно «оставить» mutable-зависимость; не путать с record-семантикой.

[↑ Наверх](#top)

---



### Вопрос: Перегрузка операторов и приведения

**Краткий ответ:**

- Операторы определяются как `public static` методы (`operator +`, `==` и т.д.). При переопределении `==` обязательно переопределять `!=`, а также `Equals`/`GetHashCode`.
- Приведения: `implicit` (безопасные, без потери данных) и `explicit` (требуют каста, возможна потеря).

**Пример:**

```csharp
public readonly struct Celsius
{
    public double Value { get; }
    public Celsius(double v) => Value = v;
    public static implicit operator double(Celsius c) => c.Value;
    public static explicit operator Celsius(double d) => new(d);
}
```

**Подводные камни:**

- Неявные приведения, теряющие данные/семантику, делают код хрупким — для рискованных используйте `explicit`.

[↑ Наверх](#top)

---



### Вопрос: Атрибуты, рефлексия, Source Generators

**Краткий ответ:**

- **Атрибуты** — метаданные на типах/членах, читаются через рефлексию (`[Obsolete]`, `[JsonPropertyName]`, кастомные).
- **Рефлексия** — инспекция типов и динамический вызов в рантайме; гибко, но медленно и мешает trimming/AOT.
- **Source Generators** — генерация кода **на этапе компиляции** (Roslyn): альтернатива рефлексии без рантайм-оверхеда. Применяется в `System.Text.Json`, логировании, DI, мапперах.

**Пример:**

```csharp
[JsonSerializable(typeof(Person))]
internal partial class AppJsonContext : JsonSerializerContext { } // source-generated
```

**Подводные камни:**

- Рефлексия ломает Native AOT/trimming (типы могут быть вырезаны) — для AOT предпочитайте source generators.
- Рефлексия на горячем пути — узкое место; кэшируйте `MemberInfo`/делегаты.

[↑ Наверх](#top)

---



## Полезные ссылки

- C# language reference: [https://learn.microsoft.com/dotnet/csharp/](https://learn.microsoft.com/dotnet/csharp/)
- What's new in C#: [https://learn.microsoft.com/dotnet/csharp/whats-new/](https://learn.microsoft.com/dotnet/csharp/whats-new/)
- Span и память: [https://learn.microsoft.com/dotnet/standard/memory-and-spans/](https://learn.microsoft.com/dotnet/standard/memory-and-spans/)

[↑ Наверх](#top)

---



[← К оглавлению](../../README.md)