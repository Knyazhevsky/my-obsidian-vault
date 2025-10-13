# Как писать свои value-типы

## Основы
**Value-типы** в C# создаются с помощью ключевого слова `struct`.  
Они хранятся в стеке (или внутри объекта), передаются **по значению** и не требуют сборки мусора.

Пример простого value-типа:
```csharp
public struct Point
{
    public int X { get; }
    public int Y { get; }
    
    public Point(int x, int y)
    {
        X = x;
        Y = y;
    }
    
    public double DistanceTo(Point other)
        => Math.Sqrt(Math.Pow(X - other.X, 2) + Math.Pow(Y - other.Y, 2));
}
```

## Правила хорошего `struct`

1. **Иммутабельность**
   Все поля лучше делать `readonly`, а свойства — только с геттерами.

```csharp
public readonly struct Money
{
    public decimal Amount { get; }
    public string Currency { get; }
    
    public Money(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency;
    }
}
```
2. **Размер** 
    Не делай `struct` слишком большим — оптимально до ~16 байт.  
    Большие value-типы копируются целиком, что снижает производительность.
    
3. **Переопредели `Equals`, `GetHashCode`, `ToString`**
    Чтобы корректно сравнивать и логировать значения.
    
```csharp
    public override string ToString() => $"{Amount} {Currency}";
    public override int GetHashCode() => HashCode.Combine(Amount, Currency);
```
4. **Реализуй интерфейсы при необходимости**  
    Например, `IEquatable<T>`, чтобы улучшить сравнение и избежать боксов.
    
---
## Когда использовать

- Для маленьких, логически цельных значений: `Point`, `Money`, `DateRange`, `RgbColor`.
- Когда важна **экономия памяти** и **предсказуемость поведения** (без GC).
    
💡 **Итог:**  
Пиши value-типы как компактные, иммутабельные структуры с четкой семантикой значения.