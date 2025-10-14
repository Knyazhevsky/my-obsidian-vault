
# CQRS (Command Query Responsibility Segregation) — Подробно для Middle

## 🎯 Как рассуждать на собеседовании

> “CQRS — это архитектурный паттерн, разделяющий операции чтения и записи. Он повышает читаемость, тестируемость и масштабируемость приложения. В .NET чаще всего реализуется через MediatR и Clean Architecture.”

---

## 🔹 Основная идея

**Command Query Responsibility Segregation (CQRS)** — принцип, согласно которому **команды (commands)** и **запросы (queries)** разделяются на разные модели и обработчики.

> Один метод **изменяет состояние** (Command), другой **читает** (Query).  
> Оба никогда не делают и того, и другого.

---

## 🔹 Проблема, которую решает CQRS

В типичном сервисе слой Application содержит методы вроде:

```csharp
public class OrderService
{
    public void CreateOrder(CreateOrderDto dto) { ... }
    public IEnumerable<OrderDto> GetOrdersByUser(int userId) { ... }
}
```

С течением времени код становится монолитным: логика чтения и записи перемешивается, тестировать трудно.

**CQRS решает это,** разделяя команды и запросы в разные классы и потоки данных.

---

## 🔹 Компоненты CQRS

| Компонент | Назначение |
|------------|------------|
| **Command** | Представляет действие, которое изменяет состояние системы |
| **CommandHandler** | Содержит бизнес-логику для выполнения команды |
| **Query** | Запрос данных (без изменений состояния) |
| **QueryHandler** | Возвращает DTO или результат чтения |
| **Mediator** | (необязательно) координирует взаимодействие между ними |

---

## 🔹 Пример базовой структуры

```plaintext
Application/
 ├── Commands/
 │    └── CreateOrder/
 │         ├── CreateOrderCommand.cs
 │         └── CreateOrderHandler.cs
 ├── Queries/
 │    └── GetOrdersByUser/
 │         ├── GetOrdersByUserQuery.cs
 │         └── GetOrdersByUserHandler.cs
```

---

## 🔹 Пример реализации с MediatR

**Команда (Command):**

```csharp
public record CreateOrderCommand(int UserId, List<int> ProductIds) : IRequest<int>;
```

**Обработчик (CommandHandler):**

```csharp
public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, int>
{
    private readonly IApplicationDbContext _context;

    public CreateOrderHandler(IApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<int> Handle(CreateOrderCommand request, CancellationToken cancellationToken)
    {
        var order = new Order { UserId = request.UserId };
        _context.Orders.Add(order);
        await _context.SaveChangesAsync(cancellationToken);
        return order.Id;
    }
}
```

**Запрос (Query):**

```csharp
public record GetOrdersByUserQuery(int UserId) : IRequest<List<OrderDto>>;
```

**Обработчик (QueryHandler):**

```csharp
public class GetOrdersByUserHandler : IRequestHandler<GetOrdersByUserQuery, List<OrderDto>>
{
    private readonly IApplicationDbContext _context;
    private readonly IMapper _mapper;

    public GetOrdersByUserHandler(IApplicationDbContext context, IMapper mapper)
    {
        _context = context;
        _mapper = mapper;
    }

    public async Task<List<OrderDto>> Handle(GetOrdersByUserQuery request, CancellationToken cancellationToken)
    {
        var orders = await _context.Orders
            .Where(o => o.UserId == request.UserId)
            .ProjectTo<OrderDto>(_mapper.ConfigurationProvider)
            .ToListAsync(cancellationToken);

        return orders;
    }
}
```

**Контроллер:**
```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IMediator _mediator;

    public OrdersController(IMediator mediator) => _mediator = mediator;

    [HttpPost]
    public async Task<IActionResult> Create(CreateOrderCommand command)
        => Ok(await _mediator.Send(command));

    [HttpGet("{userId}")]
    public async Task<IActionResult> Get(int userId)
        => Ok(await _mediator.Send(new GetOrdersByUserQuery(userId)));
}
```

---

## 🔹 Преимущества CQRS

| Преимущество | Объяснение |
|---------------|-------------|
| **Изоляция логики** | Команды и запросы не зависят друг от друга |
| **Чистота архитектуры** | Проще читать, тестировать и сопровождать |
| **Масштабирование** | Разные модели чтения и записи можно масштабировать независимо |
| **Совместимость с Event Sourcing** | CQRS часто используется с событиями |
| **Явная структура** | Каждое действие системы — отдельный обработчик |

---

## 🔹 Недостатки

| Недостаток | Решение |
|-------------|----------|
| Увеличение количества классов | Автоматизация с шаблонами (Snippets, Generators) |
| Потенциальная избыточность кода | Комбинировать Handlers через Generic Base |
| Не подходит для очень маленьких CRUD | Можно частично применять CQRS |

---

## 🔹 Расширенная реализация: разделение контекстов

Иногда вводят два контекста:

```plaintext
Infrastructure/
 ├── Persistence/
 │    └── WriteDbContext.cs   (для команд)
 │    └── ReadDbContext.cs    (для запросов, может быть readonly replica)
```

- **WriteDbContext** — использует полную модель с навигациями и валидацией.  
- **ReadDbContext** — лёгкий, оптимизированный под `SELECT`, может подключаться к replica DB.

---

## 🔹 Комбинация CQRS + Clean Architecture

Часто CQRS — это **верхний уровень Application слоя**.

```plaintext
Presentation → Application (CQRS) → Domain → Infrastructure
```

- Commands / Queries не обращаются напрямую к EF.  
- Через **интерфейсы репозиториев или DbContext**, переданные через DI.  
- Domain остаётся “чистым” — без зависимостей от фреймворков.

---

## 🔹 CQRS и Validation

Обычно валидация команд выполняется через **FluentValidation**:

```csharp
public class CreateOrderCommandValidator : AbstractValidator<CreateOrderCommand>
{
    public CreateOrderCommandValidator()
    {
        RuleFor(x => x.UserId).GreaterThan(0);
        RuleFor(x => x.ProductIds).NotEmpty();
    }
}
```

Регистрация через pipeline:

```csharp
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
```

---

## 🔹 Pipeline Behaviors (MediatR)

CQRS удобно расширяется через **pipeline behaviors** — промежуточные слои для всех команд/запросов.

Примеры:
- **ValidationBehavior** — валидация через FluentValidation.
- **LoggingBehavior** — логирование запроса и времени выполнения.
- **PerformanceBehavior** — измерение метрик.
- **TransactionBehavior** — откат транзакции при исключении.

---

## 🔹 Пример ValidationBehavior

```csharp
public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
        => _validators = validators;

    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken cancellationToken)
    {
        var context = new ValidationContext<TRequest>(request);
        var failures = _validators
            .Select(v => v.Validate(context))
            .SelectMany(r => r.Errors)
            .Where(f => f != null)
            .ToList();

        if (failures.Any())
            throw new ValidationException(failures);

        return await next();
    }
}
```

---

## 🔹 Когда использовать CQRS

✅ Использовать:
- Сложные доменные модели (много логики, сценариев).  
- Высокие требования к масштабируемости.  
- Много разных проекций данных (разные API под одни данные).  
- Микросервисная архитектура.  

❌ Необязательно использовать:
- Простые CRUD-приложения без бизнес-логики.  
- Когда код станет сложнее, чем проблема.  

---

## 🔹 Ключевые тезисы для интервью

- CQRS разделяет **команды и запросы** → чистота, масштабируемость, тестируемость.  
- Часто реализуется через **MediatR** и **pipeline behaviors**.  
- Поддерживает **FluentValidation**, **логирование**, **транзакции**.  
- Легко комбинируется с **Clean Architecture** и **Event Sourcing**.  
- CQRS ≠ Event Sourcing, но они **хорошо сочетаются**.

