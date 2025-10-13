### Как писать читаемые и эффективные LINQ-запросы

LINQ — мощный инструмент для работы с коллекциями и базами данных, но при неправильном использовании может ухудшить производительность или читаемость кода.  
Ниже — основные принципы.

---

#### 🧩 1. Используй понятные, короткие цепочки
Избегай длинных каскадов методов. Разбивай запрос на логические шаги — так проще читать и отлаживать.

```csharp
// Плохо
var result = users.Where(u => u.Age > 18 && u.IsActive)
                  .OrderBy(u => u.Name)
                  .Select(u => new { u.Name, u.Email })
                  .ToList();

// Хорошо
var adults = users.Where(u => u.Age > 18 && u.IsActive);
var sorted = adults.OrderBy(u => u.Name);
var result = sorted.Select(u => new { u.Name, u.Email }).ToList();
```

---
#### ⚡ 2. Фильтруй как можно раньше
Методы вроде `Where()` должны идти до операций `Select()`, `Join()` и `ToList()` — это сокращает объём данных.
```csharp
// Лучше фильтровать до выборки
var active = users.Where(u => u.IsActive)
                  .Select(u => u.Name);

```
---
#### 🧠 3. Не вызывай `ToList()` слишком рано

`ToList()` материализует коллекцию, прерывая **отложенное выполнение**.  
Делай это **в конце** цепочки, когда действительно нужен результат.
```csharp
// ❌ создаёт промежуточные коллекции
var data = users.ToList().Where(u => u.IsActive);

// ✅ выполняется эффективно
var data = users.Where(u => u.IsActive).ToList();
```
---
#### 🧮 4. Не дублируй вычисления
Если запрос используется несколько раз — **сохрани результат**, иначе он выполнится каждый раз заново (особенно важно для `IQueryable`).
```csharp
var query = db.Users.Where(u => u.IsActive);
var count = query.Count();
var list = query.ToList(); // выполнится один SQL-запрос, а не два разных
```
---
#### 🧱 5. Пиши проекцию в `Select`
Не возвращай целые сущности, если нужны только несколько полей — это экономит память и ускоряет запросы в БД.
```csharp
var result = db.Users
    .Where(u => u.IsActive)
    .Select(u => new { u.Id, u.Name })
    .ToList();
```
---
#### 🧾 6. Используй query-синтаксис для сложных запросов
Query-синтаксис (`from ... in ...`) часто читаемее при работе с `join`, `group` и `let`.
```csharp
var query = from u in users
            join o in orders on u.Id equals o.UserId
            where u.IsActive
            select new { u.Name, o.Total };
```
---
#### 💡 Резюме

- Сначала фильтруй (`Where`), потом сортируй (`OrderBy`), потом выбирай (`Select`).
- Не вызывай `ToList()` до конца запроса.
- Не возвращай лишние поля.
- Для сложных запросов — query-синтаксис.
- Помни про deferred execution — запрос выполняется только при обращении к результату.