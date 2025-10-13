# Делегаты и события в C#

## Делегаты
**Delegate** — это тип, который хранит ссылку на метод (или несколько методов). Делегаты позволяют передавать методы как параметры и вызывать их позже.

Пример:
```csharp
public delegate void MyDelegate(string message);

void Print(string msg) => Console.WriteLine(msg);

MyDelegate del = Print;
del("Hello"); // вызовет Print("Hello")
```

Делегаты бывают **одноадресные** (один метод) и **многоадресные** (через `+=` можно добавить несколько).
```csharp
del += AnotherMethod;
del("Test"); // вызовется оба метода
```

## События (events)

**Event** — это "обертка" над делегатом, ограничивающая доступ: подписчики могут добавлять/удалять методы, но не вызывать событие напрямую (только владелец может).

Пример:

```csharp
public class Button
{
    public event Action Clicked;

    public void Click()
    {
        Clicked?.Invoke(); // вызов события
    }
}

// Подписка
button.Clicked += () => Console.WriteLine("Нажато!");
```
## Главное различие

- **Delegate** — просто ссылка на метод.
    
- **Event** — механизм уведомлений на основе делегатов (паттерн "наблюдатель").
    

---

💡 **Типичный сценарий** — события UI, нотификации, реакция на изменения состояния.
```csharp
public event EventHandler DataLoaded;
```
