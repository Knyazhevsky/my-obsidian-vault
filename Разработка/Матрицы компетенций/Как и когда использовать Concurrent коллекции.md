# Как и когда использовать Concurrent коллекции

## Что это
**Concurrent-коллекции** — это специальные потоко-безопасные коллекции из пространства имён  
`System.Collections.Concurrent`.  
Они позволяют **одновременно читать и изменять данные** из разных потоков без необходимости вручную использовать `lock`.

---

## Основные типы
| Коллекция                           | Поведение                                                        | Аналог без потокобезопасности |
| ----------------------------------- | ---------------------------------------------------------------- | ----------------------------- |
| `ConcurrentDictionary<TKey,TValue>` | Словарь, безопасный для параллельных операций                    | `Dictionary<TKey,TValue>`     |
| `ConcurrentQueue<T>`                | Очередь (FIFO)                                                   | `Queue<T>`                    |
| `ConcurrentStack<T>`                | Стек (LIFO)                                                      | `Stack<T>`                    |
| `ConcurrentBag<T>`                  | Мешок — не гарантирует порядок                                   | —                             |
| `BlockingCollection<T>`             | Обёртка над другими коллекциями с поддержкой блокировки и лимита | —                             |

---

## Примеры использования

### 🔹 ConcurrentDictionary
Подходит для кэшей, подсчётов и работы с общими данными.
```csharp
var cache = new ConcurrentDictionary<string, int>();

cache.TryAdd("a", 1);
cache.AddOrUpdate("a", 1, (_, old) => old + 1);
Console.WriteLine(cache["a"]); // 2
```
Безопасно для одновременного чтения/записи из разных потоков.

---

### 🔹 ConcurrentQueue
Используется для очередей задач, логов и сообщений.
```csharp
var queue = new ConcurrentQueue<int>();
queue.Enqueue(1);

if (queue.TryDequeue(out var item))
    Console.WriteLine(item);
```
Удобно в продюсер–консьюмер паттернах.

---

### 🔹 ConcurrentBag
Подходит, если порядок не важен, но нужна высокая скорость при множестве потоков.
```csharp
var bag = new ConcurrentBag<string>();
bag.Add("test");

if (bag.TryTake(out var value))
    Console.WriteLine(value);
```
Элементы распределяются по локальным коллекциям, снижая блокировки.

---

### 🔹 BlockingCollection

Позволяет ограничить размер и **блокировать потоки**, если коллекция заполнена.  
Отлично подходит для паттерна **Producer–Consumer**.
```csharp
var tasks = new BlockingCollection<int>(boundedCapacity: 10);

Task.Run(() =>
{
    foreach (var t in tasks.GetConsumingEnumerable())
        Console.WriteLine($"Обработано: {t}");
});

for (int i = 0; i < 20; i++)
    tasks.Add(i);
    
tasks.CompleteAdding();

```
---

## Когда использовать

✅ Используй **Concurrent**-коллекции, если:

- Несколько потоков **одновременно читают и изменяют** данные.
- Не хочешь вручную синхронизировать доступ (`lock`, `Monitor`).
- Важно **минимизировать блокировки** и повысить масштабируемость.

❌ Не используй, если:

- Работа идёт в одном потоке.
- Нужен полный контроль над синхронизацией.
- Требуется порядок элементов (кроме `Queue` и `Stack`).

---

💡 **Итог:**  
Concurrent-коллекции — безопасный и эффективный способ организации совместного доступа к данным в многопоточной среде.  
Выбирай тип по задаче:

- словарь → `ConcurrentDictionary`
- очередь задач → `ConcurrentQueue`
- пул объектов → `ConcurrentBag`
- контролируемая очередь → `BlockingCollection`.

