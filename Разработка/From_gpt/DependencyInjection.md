
# Dependency Injection (DI) в .NET — Подробно для Middle

## 🎯 Как рассуждать на собеседовании

> “Dependency Injection — это способ передать зависимость в объект извне, а не создавать её внутри. Это снижает связанность, упрощает тестирование и делает архитектуру гибкой. В ASP.NET Core DI встроен в платформу из коробки.”

---

## 🔹 Основная идея

**Без DI:** класс сам создаёт зависимости (жёсткая связность).  
```csharp
public class OrderService
{
    private readonly OrderRepository _repository = new OrderRepository();
}
```
Если понадобится другой репозиторий (например, кэш или mock), код придётся менять.  

**С DI:** зависимости передаются снаружи — через **конструктор** или **интерфейс**.  
```csharp
public class OrderService
{
    private readonly IOrderRepository _repository;
    public OrderService(IOrderRepository repository)
    {
        _repository = repository;
    }
}
```

Теперь можно подменить реализацию, не меняя код `OrderService`.  

---

## 🔹 Преимущества DI

| Преимущество | Объяснение |
|---------------|-------------|
| **Слабая связанность** | Компоненты знают только интерфейсы, а не конкретные классы |
| **Тестируемость** | Можно внедрять mock-зависимости |
| **Расширяемость** | Новые реализации можно подменять без изменения клиентского кода |
| **Инверсия управления (IoC)** | Объекты не создают свои зависимости — контейнер делает это за них |

---

## 🔹 Виды внедрения зависимостей

| Тип | Пример | Использование |
|------|---------|----------------|
| **Constructor Injection** | через параметры конструктора | основной способ в ASP.NET Core |
| **Property Injection** | через публичные свойства | редко, в старом коде |
| **Method Injection** | через параметры метода | для одноразовых зависимостей |

**Пример Constructor Injection:**
```csharp
public class EmailService
{
    private readonly ILogger<EmailService> _logger;
    public EmailService(ILogger<EmailService> logger)
    {
        _logger = logger;
    }
}
```

---

## 🔹 IoC-контейнер в ASP.NET Core

В .NET встроен контейнер **Microsoft.Extensions.DependencyInjection**.  
Регистрация происходит в `Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);
var services = builder.Services;

services.AddTransient<IEmailService, EmailService>();
services.AddScoped<IUserRepository, UserRepository>();
services.AddSingleton<ICacheService, MemoryCacheService>();

var app = builder.Build();
```

---

## 🔹 Виды жизненного цикла (lifetime)

| Lifetime | Когда создаётся | Когда уничтожается | Типичные случаи |
|-----------|-----------------|--------------------|-----------------|
| **Transient** | при каждом запросе | сразу после использования | stateless-сервисы |
| **Scoped** | один экземпляр на HTTP-запрос | по завершении запроса | репозитории, Unit of Work |
| **Singleton** | один на всё приложение | при остановке приложения | кэш, конфигурации |

**Пример:**
```csharp
services.AddTransient<IEmailService, EmailService>(); // каждый раз новый
services.AddScoped<IOrderRepository, OrderRepository>(); // на запрос
services.AddSingleton<ICacheService, RedisCacheService>(); // общий
```

---

## 🔹 Пример работы DI в контроллере

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _service;

    public OrdersController(IOrderService service)
    {
        _service = service;
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> Get(int id)
        => Ok(await _service.GetOrderAsync(id));
}
```

DI автоматически создаёт экземпляр `OrdersController`, внедряет `IOrderService`, а тот — свои зависимости.

---

## 🔹 Вложенные зависимости

Контейнер сам строит **дерево зависимостей**.  

```plaintext
OrdersController
 └── IOrderService → OrderService
      └── IOrderRepository → OrderRepository
           └── AppDbContext
```

Если одна из зависимостей не зарегистрирована — при запуске будет исключение:  
**InvalidOperationException: Unable to resolve service for type…**

---

## 🔹 Регистрация по сборке

Если в проекте много обработчиков (например, MediatR или Handlers), удобно регистрировать автоматически.

```csharp
builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(Program).Assembly));
builder.Services.AddAutoMapper(typeof(Program));
```

Или через рефлексию:

```csharp
services.Scan(scan => scan
    .FromAssembliesOf(typeof(IApplicationMarker))
    .AddClasses()
    .AsImplementedInterfaces()
    .WithScopedLifetime());
```

---

## 🔹 DI + Clean Architecture

В Clean Architecture DI — ключевой инструмент связи между слоями.

```plaintext
Presentation → Application → Domain ← Infrastructure
```

- `Application` зависит только от **интерфейсов** (`IRepository`, `IEmailService`).
- `Infrastructure` содержит **реализации**.
- В `Program.cs` или `DependencyInjection.cs` инфраструктура подключается к приложению:

```csharp
public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(this IServiceCollection services, IConfiguration config)
    {
        services.AddScoped<IUserRepository, UserRepository>();
        services.AddScoped<IEmailService, EmailService>();
        return services;
    }
}
```

---

## 🔹 Сервисы, зависящие от настроек (IOptions)

```json
// appsettings.json
{
  "MailSettings": {
    "SmtpHost": "smtp.gmail.com",
    "Port": 587
  }
}
```

```csharp
builder.Services.Configure<MailSettings>(builder.Configuration.GetSection("MailSettings"));

public class MailService
{
    private readonly MailSettings _settings;

    public MailService(IOptions<MailSettings> options)
    {
        _settings = options.Value;
    }
}
```

---

## 🔹 Scoped внутри Singleton — антипаттерн

**Singleton не должен зависеть от Scoped-сервиса.**  
Это нарушает жизненный цикл и приведёт к ошибке при разрешении.

✅ Решения:
- Перепроектировать зависимости (например, передавать фабрику `Func<T>`).
- Или использовать `IServiceScopeFactory`:

```csharp
public class BackgroundWorker
{
    private readonly IServiceScopeFactory _scopeFactory;
    public BackgroundWorker(IServiceScopeFactory scopeFactory)
        => _scopeFactory = scopeFactory;

    public async Task DoWorkAsync()
    {
        using var scope = _scopeFactory.CreateScope();
        var repo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
        await repo.CleanupOldOrdersAsync();
    }
}
```

---

## 🔹 DI + MediatR + Pipeline Behaviors

DI позволяет подключать **pipeline behavior**-обработчики к MediatR (логирование, валидация, транзакции).

```csharp
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
```

---

## 🔹 DI в BackgroundService

```csharp
public class OrderWorker : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public OrderWorker(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var scope = _scopeFactory.CreateScope();
        var service = scope.ServiceProvider.GetRequiredService<IOrderService>();
        await service.ProcessPendingAsync(stoppingToken);
    }
}
```

---

## 🔹 Ключевые ошибки DI

| Ошибка | Причина | Решение |
|---------|----------|----------|
| “Cannot resolve service for type …” | зависимость не зарегистрирована | добавить `AddScoped<,>()` |
| Scoped в Singleton | нарушение жизненного цикла | использовать `IServiceScopeFactory` |
| Регистрация интерфейса без реализации | контейнер не знает, что внедрять | указать конкретный класс |
| Множественные реализации | контейнер не знает, какую выбрать | внедрить `IEnumerable<T>` или именованный сервис |

---

## 🔹 DI и тестирование

Для модульных тестов можно подменять зависимости с помощью Moq или NSubstitute:

```csharp
var repoMock = new Mock<IUserRepository>();
repoMock.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(new User { Id = 1 });

var service = new UserService(repoMock.Object);
```

Такой код не требует запуска контейнера DI — зависимости передаются вручную.

---

## 🔹 Ключевые тезисы для интервью

- DI — это **внедрение зависимостей извне**, а не создание их внутри.  
- Реализует принцип **Inversion of Control (IoC)**.  
- В .NET встроен контейнер **Microsoft.Extensions.DependencyInjection**.  
- Три lifetime: **Transient**, **Scoped**, **Singleton**.  
- DI — основа Clean Architecture и MediatR-паттернов.  
- Scoped нельзя внедрять в Singleton.  
- Через DI можно конфигурировать сервисы, валидаторы, pipeline behaviors и фоновые задачи.
