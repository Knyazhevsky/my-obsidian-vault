### Как использовать SelectMany и проекцию сложных данных

#### 📘 Что делает SelectMany
`SelectMany` — это оператор, который **“расплющивает” вложенные коллекции** (flattening).  
Он превращает коллекцию коллекций в **один общий поток элементов**.

---

#### 🧩 Простой пример
```csharp
var users = new[]
{
    new { Name = "Аня", Orders = new[] { "Ноутбук", "Мышь" } },
    new { Name = "Борис", Orders = new[] { "Клавиатура" } }
};

// Без SelectMany: коллекция массивов заказов
var nested = users.Select(u => u.Orders);
// nested: IEnumerable<string[]>

// С SelectMany: единый список заказов
var flat = users.SelectMany(u => u.Orders);
// flat: IEnumerable<string>

foreach (var o in flat)
    Console.WriteLine(o);
/*
Ноутбук
Мышь
Клавиатура
*/
```
👉 `SelectMany` полезен, когда нужно обработать **все вложенные элементы** как одну коллекцию.

---

#### 🧱 С проекцией данных
Можно использовать `SelectMany` не только для "расплющивания", но и для создания **проекций**, которые связывают родительские и дочерние данные.
```csharp
var result = users.SelectMany(
    u => u.Orders,
    (u, order) => new { u.Name, Order = order }
);

foreach (var item in result)
    Console.WriteLine($"{item.Name} заказал {item.Order}");
/*
Аня заказал Ноутбук
Аня заказал Мышь
Борис заказал Клавиатура
*/
```
📌 Здесь второй параметр — **функция проекции**, которая комбинирует элементы из внешней и внутренней коллекции.

---
🧠 Пример с классами:
```csharp
class User
{
    public string Name { get; set; }
    public List<Order> Orders { get; set; }
}
class Order
{
    public string Product { get; set; }
    public decimal Price { get; set; }
}

var users = new List<User>
{
    new User { Name = "Аня", Orders = new() { new Order { Product = "Ноутбук", Price = 500 } } },
    new User { Name = "Борис", Orders = new() { new Order { Product = "Мышь", Price = 50 }, new Order { Product = "Клавиатура", Price = 100 } } }
};

var query = users.SelectMany(
    u => u.Orders,
    (u, o) => new { u.Name, o.Product, o.Price }
);

foreach (var item in query)
    Console.WriteLine($"{item.Name} — {item.Product}: {item.Price}");
```

---
#### 💡 Резюме

| Метод        | Назначение                         | Результат                  |
| ------------ | ---------------------------------- | -------------------------- |
| `Select`     | Преобразует каждый элемент         | Коллекция той же структуры |
| `SelectMany` | “Расплющивает” вложенные коллекции | Единый поток элементов     |
**Когда использовать:**

- Когда есть коллекция коллекций (`List<List<T>>`, `IEnumerable<Child[]>`).
- Когда нужно сделать **плоскую проекцию** сложных объектов.
- Когда важно связать родительские и дочерние данные в одном запросе.

---
📎 **Запомни:**  
`Select` возвращает один результат на элемент,  
`SelectMany` — **много результатов** (flatten).