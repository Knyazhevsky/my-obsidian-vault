# Когда использовать ObservableCollection

## Что это
`ObservableCollection<T>` — это коллекция, которая **уведомляет** об изменениях (добавление, удаление, обновление элементов).  
Она реализует интерфейс `INotifyCollectionChanged` и `INotifyPropertyChanged`.

Благодаря этому UI (например, WPF, MAUI, Xamarin) **автоматически обновляется**, когда коллекция меняется.

---

## Пример
```csharp
using System.Collections.ObjectModel;

public class ViewModel
{
    public ObservableCollection<string> Items { get; } = new()
    {
        "Первый", "Второй"
    };
}
```
В XAML:
```xml
<ListBox ItemsSource="{Binding Items}" />
```
Если добавить элемент: