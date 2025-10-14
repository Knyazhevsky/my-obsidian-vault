
# Backend (C#, .NET, ASP.NET Core) — Middle

### 🎯 Как рассуждать на собеседовании
- **Я начну с контекста и требований**: что критичнее — производительность, читаемость или простота поддержки? От этого зависят выбор коллекций, LINQ, кэш, транзакции.
- **Опишу компромиссы**: где уместен LINQ, а где лучше императивный код; когда DI-абстракции оправданы; где кэш даст выигрыш, а где создаст сложности с консистентностью.
- **Покажу практику**: как я бы написал, протестировал и задеплоил решение (логирование, метрики, обработка ошибок, ретраи).

---

## 1) Типы, равенство, иммутабельность, value vs reference

**Зачем:** производительность, предсказуемость, отсутствие лишних аллокаций и боковых эффектов.

- **Value vs Reference:**
  - Value-типы (struct) хранятся *по значению*, копируются целиком; Reference-типы (class) — *по ссылке*, аллоцируются в heap.
  - **Критично:** лишние копии крупных `struct` ухудшают перф; боковые эффекты в `class` ухудшают тестируемость.

- **Иммутабельность (на уровне дизайна):**
  - Упрощает многопоточность, предсказуемость и кэширование.
  - Для классов — **readonly-свойства**, init-only setters, инкапсуляция мутаций.

- **Равенство и хэш-код:**
  - Переопределяйте **`Equals`/`GetHashCode`** согласованно; используйте `EqualityComparer<T>` и `HashCode.Combine`.
  - Для value-объектов (DDD) определяйте семантическое равенство по полям.

**Мини-пример:** корректное равенство + ToString
```csharp
public sealed class Money : IEquatable<Money>
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency) =>
        (Amount, Currency) = (amount, currency);

    public override string ToString() => $"{Amount:0.##} {Currency}";

    public bool Equals(Money? other) =>
        other is not null && Amount == other.Amount && Currency == other.Currency;

    public override bool Equals(object? obj) => obj is Money m && Equals(m);
    public override int GetHashCode() => HashCode.Combine(Amount, Currency);
}
```

**Когда struct уместен:** маленькие, часто передаваемые «значения», например координаты, деньги (с оговорками), идентификаторы. Используйте `readonly struct` для неизменяемости.

---

## 2) Коллекции и выбор структур данных

**Частые решения:**
- **List<T>** — динамический массив; вставки в конец O(1), поиск O(n).
- **Dictionary<TKey,TValue>** — быстрый поиск O(1) амортизированно; думайте о корректном `GetHashCode`.
- **HashSet<T>** — проверки вхождения O(1), удобно для дедупликации.
- **ConcurrentDictionary/Bag/Queue** — для мультипоточности; но оцените contention, иногда быстрее локальные коллекции + объединение.

**Советы:**
- Не создавайте лишних коллекций в LINQ-цепочках (`ToList` только там, где нужен материализованный результат).
- Для частого «проверить, есть ли id» — `HashSet<int>`.
- Для больших коллекций и частых вставок в середину — подумайте о других структурах (Queue, LinkedList — с осторожностью).

---

## 3) LINQ для продакшена (IEnumerable vs IQueryable, отложенное выполнение)

- **Отложенное vs немедленное:** LINQ-операторы над `IEnumerable<T>` **ленивые**. Материализация — `ToList/ToArray/First/Count`.
- **IQueryable**: в EF Core провайдер преобразует выражение в SQL; **не тащите** в память то, что можно отфильтровать на стороне БД.
- **Паттерн:** формируйте запросы **пошагово**, но **материализуйте один раз** по месту использования.

**Читаемость и перф:**
- Не злоупотребляйте анонимными сложными проекциями — лучше отдельные DTO.
- Для «группировок и агрегаций» в БД — используйте `IQueryable` и серверные агрегаты.

**Мини-пример:** безопасная композиция запросов
```csharp
IQueryable<Order> query = db.Orders.Where(o => o.Status == Status.Active);
if (fromDate is not null) query = query.Where(o => o.Date >= fromDate);
if (toDate   is not null) query = query.Where(o => o.Date <  toDate);
var page = await query
    .OrderByDescending(o => o.Date)
    .Skip((pageIndex-1)*pageSize).Take(pageSize)
    .Select(o => new OrderDto(o.Id, o.Date, o.Total))
    .ToListAsync(ct);
```

---

## 4) Async/await, Task и параллелизм

- **`ConfigureAwait(false)`** — используйте в библиотечном/бэкграунд-коде, где не нужно возвращаться в контекст синхронизации.
- **Обработка ошибок:** исключения из `Task` всплывут при `await`; **не теряйте** `await` (fire-and-forget только под контролем).
- **Параллельная агрегация:** `Task.WhenAll` для независимых запросов; дозируйте параллелизм через `SemaphoreSlim`.

**Мини-пример:** параллельные запросы с лимитом
```csharp
var throttler = new SemaphoreSlim(8);
var tasks = items.Select(async item =>
{
    await throttler.WaitAsync(ct);
    try { return await ProcessAsync(item, ct); }
    finally { throttler.Release(); }
});
var results = await Task.WhenAll(tasks);
```

- **`Task` vs `ValueTask`**: `ValueTask` уменьшает аллокации при *очень* частых синхронно-завершающихся путях; в API по умолчанию лучше `Task`.

---

## 5) Исключения, логирование, глобальная обработка

- Ловите **узко** и **близко к месту** восстановления; не глушите все ошибки.
- В ASP.NET Core:
  - Для продакшена — **`UseExceptionHandler`** + единый формат ошибок (problem details).
  - Логируйте `ex.ToString()` (со stack trace), добавляйте **контекст** (requestId, userId, параметры).
- **Политики повторов** (retry, timeout, circuit breaker) — через Polly на уровне HTTP/БД-клиентов.

**Мини-пример:** ProblemDetails middleware (идея)
```csharp
app.UseExceptionHandler(builder =>
{
    builder.Run(async ctx =>
    {
        var ex = ctx.Features.Get<IExceptionHandlerFeature>()?.Error;
        var problem = new ProblemDetails
        {
            Status = StatusCodes.Status500InternalServerError,
            Title = "Unexpected error",
            Detail = ex?.Message,
            Instance = ctx.TraceIdentifier
        };
        await Results.Problem(problem).ExecuteAsync(ctx);
    });
});
```

---

## 6) Dependency Injection (жизненные циклы, фабрики, тестируемость)

- **Жизненные циклы:** `Transient` (каждый раз новый), `Scoped` (на запрос), `Singleton` (на приложение). Не «поднимайте» зависимости случайно.
- **Паттерны:** `Func<T>`/`IServiceProvider` для фабрик, **Options** (`IOptions*`) для конфигураций.
- **Соблюдайте границы scoped:** не инжектируйте `Scoped` в `Singleton`.
- **Тестируемость:** зависимости через **интерфейсы**, побочные эффекты — за абстракциями (IClock, IDateProvider, ICache).

**Мини-пример:** фабрика клиента
```csharp
services.AddHttpClient<IMyApi, MyApi>(client => client.BaseAddress = new Uri(cfg.Url))
        .AddPolicyHandler(PollyPolicies.RetryWithJitter);
```

---

## 7) Контроллеры, минимальные API, фильтры, middleware

- **Границы ответственности:** контроллер тонкий → бизнес-логика в сервисах/хендлерах (MediatR — опционально).
- **Фильтры vs Middleware:** фильтры ближе к MVC (валидация, кросс-срезы на уровне действий); middleware — **сквозные** вещи (логирование, кореляции, CORS).
- **Минимальные API** уместны для лёгких ендпоинтов, где MVC избыточен.

**Мини-пример:** пагинация + сортировка
```csharp
[HttpGet]
public async Task<PagedResult<OrderDto>> Get([FromQuery] OrderQuery q, CancellationToken ct)
{
    var baseQuery = repo.QueryActive();
    if (!string.IsNullOrWhiteSpace(q.Search))
        baseQuery = baseQuery.Where(o => o.Number.Contains(q.Search));
    var total = await baseQuery.CountAsync(ct);
    var items = await baseQuery.OrderByDescOrAsc(q.SortBy, q.Desc)
                               .Skip(q.Skip).Take(q.Take)
                               .ProjectToDto()
                               .ToListAsync(ct);
    return new(items, total);
}
```

---

## 8) HttpClientFactory + Polly

- **Почему важно:** переиспользование сокетов, централизованный конфиг, отказоустойчивость.
- Используйте **Typed/Named clients**, добавляйте **retry/timeout/breaker**.
- Не забудьте **логирование** запросов/ответов и **корреляцию** (trace id).

**Мини-пример:** таймаут + повтор
```csharp
services.AddHttpClient("billing", c => c.BaseAddress = new Uri(cfg.BillingUrl))
    .AddTransientHttpErrorPolicy(p => p.WaitAndRetryAsync(3, i => TimeSpan.FromMilliseconds(200 * i)))
    .AddPolicyHandler(Policy.TimeoutAsync<HttpResponseMessage>(TimeSpan.FromSeconds(5)));
```

---

## 9) BackgroundService, таймеры, отмена

- Для периодических задач используйте `BackgroundService` + `PeriodicTimer`.
- Пропихивайте **`CancellationToken`** на всех уровнях, аккуратно с логикой повторов.
- Логи, метрики, idempotency (не обрабатывать одно и то же событие дважды).

**Мини-пример:**
```csharp
public sealed class CleanupService : BackgroundService
{
    private readonly PeriodicTimer _timer = new(TimeSpan.FromMinutes(5));
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (await _timer.WaitForNextTickAsync(ct))
        {
            await DoCleanupAsync(ct); // try/catch + логирование внутри
        }
    }
}
```

---

## 10) GC и управление памятью (для Middle)

**Что знать на собеседовании:**
- **Поколения GC:** Gen0, Gen1, Gen2; частые короткие объекты умирают в Gen0; долгоживущие — поднимаются до Gen2.
- **LOH (Large Object Heap):** крупные объекты (85+ KB) попадают в LOH; сборки реже, фрагментация дороже. Избегайте частых больших аллокаций/копий.
- **Когда запускается GC:** при нехватке памяти, аллокациях, ручном вызове (не рекомендуется), фоновые сборки в Server/Workstation режимах.
- **Async/allocations:** не создавайте лишних замыканий/итераторов, избегайте бокового `ToList()`; используйте пулы (`ArrayPool`/объектные пулы) по необходимости.

**Типовые источники утечек:**
- Долгоживущие статики/кеши, удерживающие ссылки на объекты; неподписанные события (event handler утечки); таймеры, не очищенные `IDisposable`.
- Замыкания, захватывающие «толстые» контексты.

**Практика:**
- Профилируйте (dotMemory, PerfView), смотрите **alloc flame graph**.
- Используйте **`using`/`await using`** для `IDisposable/IAsyncDisposable`.
- Не злоупотребляйте `string` конкатенацией в циклах — используйте `StringBuilder` или интерполяцию вне «горячих» путей.
- Подумайте о **`ReadOnlySpan<T>`**/**`Memory<T>`** для парсинга/форматирования в «узких» местах, но без преждевременной оптимизации.

**Мини-чеклист по памяти:**
- Избегаю больших краткоживущих объектов → LOH.
- Не держу ссылки в статических кэшах «навсегда» → TTL/Size cap.
- Закрываю подписки на события/Timers/HttpResponseMessage/Streams.
- Не материализую LINQ без нужды.

---

## 11) IDisposable / IAsyncDisposable и паттерн Dispose

**Когда нужен финалайзер:** когда владеете **неуправляемым ресурсом** напрямую (handle, native buffer). Иначе — избегайте финалайзера, он дорог.

**Шаблон:**
```csharp
public class ResourceHolder : IDisposable, IAsyncDisposable
{
    private bool _disposed;
    private readonly Stream _stream;
    public ResourceHolder(Stream stream) => _stream = stream;

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        _stream.Dispose();
        GC.SuppressFinalize(this);
    }

    public async ValueTask DisposeAsync()
    {
        if (_disposed) return;
        _disposed = true;
        await _stream.DisposeAsync();
        GC.SuppressFinalize(this);
    }
}
```

---

## 12) Конфигурация и Options

- Мэппьте секции в `record`-типы, валидируйте при старте.
- Отличия: **`IOptions`** (фиксируется при старте), **`IOptionsSnapshot`** (scoped-обновление), **`IOptionsMonitor`** (подписка на изменения).

**Мини-пример:**
```csharp
builder.Services.AddOptions<StorageOptions>()
    .Bind(builder.Configuration.GetSection("Storage"))
    .ValidateDataAnnotations()
    .Validate(o => Uri.IsWellFormedUriString(o.Endpoint, UriKind.Absolute), "Endpoint must be URL")
    .ValidateOnStart();
```

---

## 13) Логирование (ILogger/NLog/Serilog идеи)

- **Структурные логи:** не строка, а поля: `logger.LogInformation("User {UserId} created order {OrderId}", userId, orderId)`.
- Уровни и фильтры по средам; корреляция `X-Request-ID`.
- Логируйте **ключевые события**, ошибки с контекстом, но избегайте «шумовых» логов.

---

## 14) Практические паттерны (кратко)

- **CQRS (внутри сервиса):** команды/запросы → читаемость и тестируемость.
- **Specification / Predicate builder** для композиции фильтров в EF/Dapper.
- **Policy-обёртки** вокруг клиентских вызовов.
- **Кэширование** «на краях» (HTTP/Redis) для read-heavy сценариев с TTL.

---

## 15) Короткий самопроверочный список (перед интервью)

- Могу объяснить разницу `IEnumerable` vs `IQueryable` и где материализую.
- Знаю, как спроектировать **обработку ошибок** (ProblemDetails) + логирование.
- Умею настроить `HttpClientFactory` с Polly и **объяснить** политики.
- Понимаю **поколения GC, LOH и источники утечек** в .NET.
- Могу уверенно говорить про **DI-жизненные циклы** и scoped-границы.
- Понимаю, где фильтры, где middleware, где минимальные API.
- Знаю, когда и как распараллелить работу, не теряя `await` и контроль.
