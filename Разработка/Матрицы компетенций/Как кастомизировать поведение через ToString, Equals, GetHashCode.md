# Как кастомизировать поведение через ToString, Equals, GetHashCode

## ToString()
Метод `ToString()` возвращает строковое представление объекта.  
Полезно для логов, отладки и вывода в UI.

Пример:
```csharp
public class User
{
    public string Name { get; set; }
    public int Age { get; set; }
    
    public override string ToString() => $"{Name}, {Age} лет";
}
```
| Если не переопределить — по умолчанию возвращает имя типа (`Namespace.ClassName`).

## Equals()
Метод `Equals()` определяет, равны ли два объекта **по значению**, а не по ссылке.

Пример:

```csharp
public class Point
{
    public int X { get; }
    public int Y { get; }
    
    public Point(int x, int y) => (X, Y) = (x, y);
    
    public override bool Equals(object? obj)
        => obj is Point p && X == p.X && Y == p.Y;
}
```
Чтобы избежать лишних аллокаций, лучше реализовать `IEquatable<T>`:

```csharp
public bool Equals(Point other) => X == other.X && Y == other.Y;
```
## GetHashCode()

Используется при сравнении объектов в **хэш-структурах** (`Dictionary`, `HashSet` и т.д.).  
Два равных объекта (`Equals == true`) должны иметь одинаковый `GetHashCode`.

Современный способ:
```csharp
public override int GetHashCode() => HashCode.Combine(X, Y);
```
---
## Пример полного набора
```csharp
public readonly struct Money : IEquatable<Money>
{
    public decimal Amount { get; }
    public string Currency { get; }
    
    public Money(decimal amount, string currency)
        => (Amount, Currency) = (amount, currency);
        
    public override string ToString() => $"{Amount} {Currency}";
    public override bool Equals(object? obj) => obj is Money m && Equals(m);
    public bool Equals(Money other) => Amount == other.Amount && Currency == other.Currency;
    public override int GetHashCode() => HashCode.Combine(Amount, Currency);
}
```
💡 **Итог:**
- `ToString()` — для удобного вывода.
- `Equals()` — для сравнения по значению.
- `GetHashCode()` — для корректной работы в коллекциях.  
    Переопределяй их вместе, чтобы обеспечить согласованное поведение типа.
