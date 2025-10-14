
# CancellationToken в .NET — Подробно для Middle

## 🎯 Как рассуждать на собеседовании

> “CancellationToken — это механизм отмены асинхронных операций в .NET. Он позволяет корректно прерывать задачи без принудительного завершения потоков, что особенно важно для стабильности и отзывчивости сервисов.”

---

## 🔹 Основная идея

`CancellationToken` — это **структура**, передающая сигнал “отменить операцию”.  
Её используют, чтобы **аккуратно завершить** задачу, запрос или длительный процесс.

Работает в связке с `CancellationTokenSource`, который **создаёт** и **контролирует** токен.

```csharp
var cts = new CancellationTokenSource();
var token = cts.Token;

Task.Run(async () =>
{
    while (!token.IsCancellationRequested)
    {
        Console.WriteLine("Работаю...");
        await Task.Delay(500);
    }
    Console.WriteLine("Отменено.");
});

Thread.Sleep(2000);
cts.Cancel();
```

---

## 🔹 Ключевые участники

| Класс | Назначение |
|--------|-------------|
| **CancellationTokenSource (CTS)** | Источник отмены — вызывает `Cancel()` |
| **CancellationToken** | Токен, передаваемый в операции |
| **CancellationTokenRegistration** | Подписка на событие отмены (`token.Register(...)`) |

---

## 🔹 Основные свойства и методы

| Метод / Свойство | Назначение |
|------------------|-------------|
| `IsCancellationRequested` | Проверить, был ли вызван `Cancel()` |
| `ThrowIfCancellationRequested()` | Генерирует `OperationCanceledException` |
| `Register(Action)` | Зарегистрировать колбэк при отмене |
| `Cancel()` | Отправить сигнал отмены |
| `CancelAfter(TimeSpan)` | Автоматически отменить через время |
| `Dispose()` | Освободить ресурсы CTS |

---

## 🔹 Пример: отмена с таймаутом

```csharp
var cts = new CancellationTokenSource(TimeSpan.FromSeconds(3));
try
{
    await LongOperationAsync(cts.Token);
}
catch (OperationCanceledException)
{
    Console.WriteLine("Операция отменена по таймауту");
}

async Task LongOperationAsync(CancellationToken token)
{
    for (int i = 0; i < 10; i++)
    {
        token.ThrowIfCancellationRequested();
        await Task.Delay(1000, token);
        Console.WriteLine($"Шаг {i}");
    }
}
```

✅ Важно: `Task.Delay`, `HttpClient.SendAsync`, `Stream.ReadAsync`, `DbCommand.ExecuteReaderAsync` и многие другие **уже поддерживают токены отмены**.

---

## 🔹 Пример: объединение токенов

Иногда нужно отменять из разных источников (например, таймаут + ручная отмена).  
Для этого используется **`CancellationTokenSource.CreateLinkedTokenSource`**.

```csharp
var globalCts = new CancellationTokenSource();
var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(10));

using var linked = CancellationTokenSource.CreateLinkedTokenSource(globalCts.Token, timeoutCts.Token);

await DoWorkAsync(linked.Token);

// можно отменить вручную
globalCts.Cancel();
```

---

## 🔹 Как правильно проверять отмену

- Если метод **сам управляет потоком выполнения**, проверяй `token.IsCancellationRequested` в цикле.  
- Если метод **делает ожидания или I/O**, передавай `token` в асинхронные методы (`Task.Delay`, `HttpClient`, EF Core и т.д.).

```csharp
while (!token.IsCancellationRequested)
{
    await ProcessNextAsync(token);
}
```

или

```csharp
token.ThrowIfCancellationRequested();
```

`ThrowIfCancellationRequested()` — корректный способ завершить задачу с `OperationCanceledException`, чтобы вышестоящий код понял, что это **отмена, а не ошибка**.

---

## 🔹 Отличие от Task.Wait(timeout)

| Подход | Недостаток |
|--------|-------------|
| `Task.Wait(timeout)` | Жёстко обрывает ожидание, не уведомляя задачу |
| **CancellationToken** | Позволяет задаче завершиться корректно, освободив ресурсы |

---

## 🔹 Использование с HttpClient

```csharp
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
try
{
    var response = await httpClient.GetAsync(url, cts.Token);
    Console.WriteLine(await response.Content.ReadAsStringAsync());
}
catch (TaskCanceledException)
{
    Console.WriteLine("Запрос отменён по таймауту");
}
```

🧠 `TaskCanceledException` — частный случай `OperationCanceledException`, который выбрасывается при отмене HTTP-запроса.

---

## 🔹 Отмена в BackgroundService

```csharp
public class Worker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await DoWorkAsync(stoppingToken);
            await Task.Delay(1000, stoppingToken);
        }
    }
}
```

✅ `stoppingToken` автоматически передаётся в `ExecuteAsync` — используется для мягкой остановки службы (graceful shutdown).

---

## 🔹 Ловушки и ошибки

| Ошибка | Последствие | Решение |
|---------|--------------|---------|
| Игнорирование токена в долгих операциях | “виснущие” задачи при остановке сервиса | передавать токен во все асинхронные вызовы |
| Принудительный `Thread.Abort` | некорректное завершение, утечки | использовать `Cancel()` |
| Проглатывание `OperationCanceledException` | неверное поведение при отмене | логировать, но не считать ошибкой |
| Не вызван `Dispose()` у `CancellationTokenSource` | утечка | вызывать `using` или `Dispose()` |

---

## 🔹 Пример комплексного сценария

```csharp
public async Task ProcessOrdersAsync(CancellationToken token)
{
    using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
    using var linked = CancellationTokenSource.CreateLinkedTokenSource(token, timeoutCts.Token);

    foreach (var order in await GetOrdersAsync(linked.Token))
    {
        linked.Token.ThrowIfCancellationRequested();
        await ProcessOrderAsync(order, linked.Token);
    }
}
```

✅ Такой код:
- корректно обрабатывает внешнюю отмену (например, при закрытии приложения);
- сам завершится, если превысит лимит времени;
- не оставит “висящих” задач.

---

## 🔹 Проверка отмены без исключений

Иногда бросать исключения не нужно (например, в циклах с промежуточными результатами).  
Можно просто выйти из метода:

```csharp
if (token.IsCancellationRequested)
    return;
```

---

## 🔹 Самопроверка перед интервью

- Понимаю разницу между **CancellationToken** и **CancellationTokenSource**.  
- Знаю, как **объединить токены** из разных источников.  
- Могу объяснить, **почему лучше `ThrowIfCancellationRequested()`**, а не `return`.  
- Умею **встраивать токен** в долгие операции, HTTP-запросы, EF Core и фоновые службы.  
- Знаю, что **отмена — это не ошибка**, и как обрабатывать `OperationCanceledException`.
