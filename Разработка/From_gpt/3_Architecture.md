
# Архитектура и паттерны — Middle

### 🎯 Как рассуждать на собеседовании
- **Начинаю с принципов:** SOLID, слои, зависимости сверху вниз, изоляция домена от инфраструктуры.
- **Показываю понимание компромиссов:** «Тут я бы выбрал CQRS ради читаемости, но не стал бы дробить сервис без острой нужды».
- **Аргументирую:** почему структура упрощает поддержку, тесты и масштабирование.

---

## 1) Принципы SOLID (кратко и по существу)

- **S — Single Responsibility:** класс или модуль отвечает за одну причину изменения.
  - Нарушение: сервис делает и валидацию, и логирование, и работу с БД.
- **O — Open/Closed:** расширяем без изменения кода.
  - Через интерфейсы, стратегии, DI.
- **L — Liskov Substitution:** наследники не должны ломать поведение базового типа.
- **I — Interface Segregation:** лучше несколько маленьких интерфейсов, чем один огромный.
- **D — Dependency Inversion:** верхний слой зависит от абстракций, а не от реализаций.

**Мини-пример:**
```csharp
public interface INotifier { Task SendAsync(string msg); }
public class EmailNotifier : INotifier { /* ... */ }
public class SmsNotifier : INotifier { /* ... */ }

// Расширяем без изменения существующего кода
```

---

## 2) Clean Architecture

**Слои:**
1. **Domain** — сущности, value objects, интерфейсы репозиториев.
2. **Application (Core)** — use cases, DTO, интерфейсы сервисов, MediatR-хендлеры.
3. **Infrastructure** — реализация интерфейсов (БД, HTTP, кэш).
4. **API (Presentation)** — контроллеры, endpoints, DI.

**Зависимости:** направлены **внутрь** (в Domain).  
Domain ничего не знает о внешнем мире.

**Преимущества:**
- Легко тестировать бизнес-логику.
- Можно менять инфраструктуру (БД, кэш, API) без переписывания домена.

---

## 3) DDD (Domain-Driven Design)

**Основные идеи:**
- Модель предметной области — в центре.
- Общий язык (Ubiquitous Language) между кодом и бизнесом.
- **Value Object** — по значению, неизменяемый.
- **Entity** — по идентификатору, изменяемый.
- **Aggregate Root** — гарантирует целостность внутри себя.
- **Repository** — слой доступа к хранилищу для агрегатов.

**Мини-пример:**
```csharp
public class Order
{
    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items;
    public void AddItem(Product p, int qty)
    {
        if (qty <= 0) throw new DomainException("Invalid qty");
        _items.Add(new OrderItem(p, qty));
    }
}
```

---

## 4) Основные архитектурные паттерны

| Паттерн | Когда применять | Пример |
|----------|----------------|--------|
| **Factory** | Когда нужно централизовать создание объектов | `IServiceFactory.Create<T>()` |
| **Strategy** | Для взаимозаменяемых алгоритмов | расчет скидки, сортировка |
| **Decorator** | Добавить функциональность без изменения кода | логирование запросов |
| **Mediator (MediatR)** | Уменьшить связанность между обработчиками | CQRS запросы |
| **Observer (Pub/Sub)** | Реакция на события | обработка уведомлений |
| **Chain of Responsibility** | Пошаговая обработка | пайплайн фильтров |

---

## 5) CQRS и MediatR

**CQRS (Command Query Responsibility Segregation):**
- Разделяем операции **чтения** и **изменения** данных.
- Команды не возвращают данные, только результат выполнения.
- Запросы (queries) — только для чтения.

**MediatR-хендлеры:**
```csharp
public record CreateOrderCommand(Guid UserId) : IRequest<Guid>;

public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, Guid>
{
    private readonly IOrderRepository _repo;
    public CreateOrderHandler(IOrderRepository repo) => _repo = repo;

    public async Task<Guid> Handle(CreateOrderCommand req, CancellationToken ct)
    {
        var order = new Order(req.UserId);
        await _repo.AddAsync(order);
        return order.Id;
    }
}
```

---

## 6) Тестируемость и архитектура

**Принцип:** чем ближе код к домену, тем проще тестировать без внешней среды.

- Domain тестируется юнит-тестами (никаких зависимостей).
- Application тестируется моками (репозиторий, API).
- Infrastructure проверяется интеграционными тестами.

**Инструменты:** xUnit, FluentAssertions, Moq.

---

## 7) Отказоустойчивость и устойчивость к сбоям

- **Ретраи**: через Polly или собственные политики.
- **Circuit Breaker:** временно отключает зависимость при частых ошибках.
- **Bulkhead:** ограничивает число параллельных вызовов.
- **Timeout:** обязательный в каждом внешнем вызове.

**Пример Polly:**
```csharp
var policy = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, retry => TimeSpan.FromSeconds(retry));
```

---

## 8) Логирование, трассировка и метрики

- **Корреляция:** RequestId, TraceId, SpanId — помогают отследить цепочку вызовов.
- **Метрики:** Prometheus/Grafana или Application Insights.
- **Здоровье:** `HealthChecks` (DB, Redis, API).

**Мини-пример:**
```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(cfg.ConnectionString)
    .AddRedis(cfg.Redis);
```

---

## 9) Микросервисы и интеграция

- Каждый сервис имеет **свой контекст и БД**.
- Общение — через REST, gRPC, очередь (Kafka, RabbitMQ).
- **Иденпотентность:** повтор вызова не должен менять результат.
- **Версионирование API:** при изменениях не ломать клиентов.

**Паттерны коммуникации:**
- Request/Response — синхронное взаимодействие.
- Event-driven — асинхронное взаимодействие.

---

## 10) Антипаттерны (что лучше избегать)

- **God object** — один сервис знает всё.
- **Anemic domain model** — сущности без логики, всё в сервисах.
- **Shared DB** между микросервисами.
- **Слои без пользы:** если слой ничего не добавляет — удалите.

---

## 11) Технологические решения в .NET мире

- **MediatR** — CQRS и хендлеры команд/запросов.
- **FluentValidation** — валидация на уровне use-case.
- **Mapster** — лёгкий маппинг между DTO и моделями.
- **Serilog** — структурное логирование.
- **Quartz.NET** — планирование задач.
- **Redis / MemoryCache** — кэширование на уровне приложения.

---

## 12) Архитектурные компромиссы

- Простота > “чистота”, если нет реальной выгоды от усложнения.
- DRY — но не в ущерб читаемости.
- “Слои ради слоёв” — не цель, а инструмент.

**Как отвечать на собеседовании:**
> “Я обычно придерживаюсь Clean Architecture, но без фанатизма — если сервис небольшой, достаточно разделить контроллер, сервис и репозиторий. В больших проектах выношу домен отдельно, добавляю use cases и CQRS.”

---

## 13) Самопроверка

- Могу описать **Clean Architecture** на примере.
- Понимаю **границы слоёв** и направленность зависимостей.
- Могу объяснить, зачем нужен **CQRS** и **Mediator**.
- Знаю разницу между **Value Object** и **Entity**.
- Понимаю, когда стоит использовать **Polly**, **HealthChecks**, **Mapster**.
- Могу привести пример, когда **архитектура упрощает тестирование**.
