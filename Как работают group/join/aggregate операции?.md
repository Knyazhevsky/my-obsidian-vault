### Как работают group / join / aggregate операции

Эти операции — часть LINQ и позволяют выполнять группировку, объединение и свёртку данных, подобно SQL-запросам.

---

#### 🧩 **1. Group (группировка)**
Позволяет объединять элементы по ключу.  
Результатом является коллекция групп (`IGrouping<TKey, TElement>`), где каждая группа содержит ключ и элементы.

```csharp
var users = new[]
{
    new { Name = "Аня", City = "Алматы" },
    new { Name = "Борис", City = "Астана" },
    new { Name = "Вика", City = "Алматы" }
};

var grouped = users.GroupBy(u => u.City);

foreach (var group in grouped)
{
    Console.WriteLine($"Город: {group.Key}");
    foreach (var user in group)
        Console.WriteLine($"  {user.Name}");
}
/*
Город: Алматы
  Аня
  Вика
Город: Астана
  Борис
*/
```
👉 Используется, когда нужно посчитать, сгруппировать или агрегировать данные по признаку.

---
#### 🔗 **2. Join (объединение)**
Связывает элементы двух коллекций по совпадающим ключам, аналог SQL `INNER JOIN`.
```csharp
var users = new[]
{
    new { Id = 1, Name = "Аня" },
    new { Id = 2, Name = "Борис" }
};

var orders = new[]
{
    new { UserId = 1, Product = "Ноутбук" },
    new { UserId = 2, Product = "Мышь" }
};

var joined = users.Join(
    orders,
    user => user.Id,        // ключ в первой коллекции
    order => order.UserId,  // ключ во второй коллекции
    (user, order) => new { user.Name, order.Product } // результат
);

foreach (var item in joined)
    Console.WriteLine($"{item.Name} заказал {item.Product}");
/*
Аня заказал Ноутбук
Борис заказал Мышь
*/
```
👉 Подходит, когда нужно связать данные из разных источников по ключу.

---
#### ➕ **3. Aggregate (свёртка)**
Позволяет применить функцию накопления к коллекции, схоже с `Reduce` из других языков.
```csharp
var numbers = new[] { 1, 2, 3, 4 };

int sum = numbers.Aggregate((acc, n) => acc + n);
Console.WriteLine(sum); // 10
```
Можно задать начальное значение:
```csharp
int product = numbers.Aggregate(1, (acc, n) => acc * n);
Console.WriteLine(product); // 24
```
👉 Удобно для вычисления суммы, произведения, конкатенации строк и других агрегатов.

---
|Операция|Назначение|Аналог в SQL|Результат|
|---|---|---|---|
|`GroupBy`|Группировка по ключу|`GROUP BY`|Коллекция групп|
|`Join`|Объединение коллекций по ключу|`INNER JOIN`|Последовательность пар|
|`Aggregate`|Свёртка коллекции в одно значение|`SUM`, `COUNT`, `AVG` и др.|Одно значение|

---
✅ **Главное помнить:**
- `GroupBy` возвращает группы, которые потом можно агрегировать (`Count`, `Sum` и т.п.).
- `Join` соединяет данные по ключам.
- `Aggregate` сворачивает всё в одно значение, проходя коллекцию последовательно.
