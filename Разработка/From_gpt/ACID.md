
# Принципы ACID в базах данных — Подробно для Middle

## 🎯 Как рассуждать на собеседовании

> “ACID — это набор свойств, гарантирующих надёжность транзакций в СУБД. Эти принципы обеспечивают целостность данных даже при ошибках, сбоях и одновременном доступе.”

---

## 🔹 Что такое транзакция

**Транзакция** — это группа операций, выполняемых как единое целое.  
Либо все операции выполняются успешно, либо **откатываются** (rollback).

**Пример:**
```sql
BEGIN TRANSACTION

UPDATE Accounts SET Balance = Balance - 100 WHERE Id = 1;
UPDATE Accounts SET Balance = Balance + 100 WHERE Id = 2;

COMMIT TRANSACTION
```

Если вторая операция не выполнится, транзакция откатится — баланс не изменится.

---

## 🔹 ACID — расшифровка

| Принцип | Название | Суть |
|----------|-----------|------|
| **A** | **Atomicity (Атомарность)** | Всё или ничего |
| **C** | **Consistency (Согласованность)** | После транзакции система остаётся в корректном состоянии |
| **I** | **Isolation (Изоляция)** | Параллельные транзакции не мешают друг другу |
| **D** | **Durability (Надёжность)** | После `COMMIT` данные не теряются, даже при сбое |

---

## 🧩 1. Atomicity (Атомарность)

**“Либо всё, либо ничего.”**  
Если одна операция из транзакции не удалась, все изменения отменяются.

**Пример (SQL Server):**
```sql
BEGIN TRANSACTION
UPDATE Accounts SET Balance = Balance - 500 WHERE Id = 1;
UPDATE Accounts SET Balance = Balance + 500 WHERE Id = 2;
IF @@ERROR <> 0
    ROLLBACK TRANSACTION;
ELSE
    COMMIT TRANSACTION;
```

**В .NET (EF Core):**
```csharp
using var transaction = await context.Database.BeginTransactionAsync();
try
{
    sender.Balance -= 500;
    receiver.Balance += 500;
    await context.SaveChangesAsync();
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
}
```

---

## 🧩 2. Consistency (Согласованность)

После завершения транзакции **данные должны оставаться валидными**.  
То есть все ограничения (primary key, foreign key, check, unique) должны выполняться.

Пример нарушения:  
Если списать деньги с одного счёта, но не зачислить на другой — баланс нарушен.  

**Согласованность гарантируется:**
- Бизнес-правилами (логикой приложения);
- Ограничениями БД (constraints, triggers);
- Типами данных и схемой.

**EF Core помогает:**
- Ограничения (`[Required]`, `[MaxLength]`, `[ForeignKey]`);
- Валидация модели перед `SaveChanges()`.

---

## 🧩 3. Isolation (Изоляция)

**Параллельные транзакции не должны мешать друг другу.**  
Например, один пользователь читает данные, пока другой их изменяет.

SQL Server поддерживает **уровни изоляции**:

| Уровень | Что допускает | Что предотвращает |
|----------|----------------|-------------------|
| **Read Uncommitted** | грязные чтения | — |
| **Read Committed** (по умолчанию) | блокирует “грязные” чтения | допускает неповторяемые чтения |
| **Repeatable Read** | запрещает неповторяемые чтения | допускает фантомные |
| **Serializable** | полная изоляция (медленно) | все аномалии предотвращены |
| **Snapshot** | версии строк (без блокировок) | высокая конкурентность |

**Пример в SQL:**
```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRANSACTION
SELECT * FROM Orders WHERE Amount > 1000;
WAITFOR DELAY '00:00:05';
COMMIT TRANSACTION;
```

В это время другая транзакция не сможет изменить те же строки.

**В EF Core:**
```csharp
await using var tx = await context.Database.BeginTransactionAsync(IsolationLevel.Serializable);
```

---

## 🧩 4. Durability (Надёжность)

После `COMMIT` данные гарантированно сохраняются даже при:
- сбое питания,
- падении сервера,
- перезапуске БД.

**SQL Server** и другие СУБД используют:
- **журнал транзакций (transaction log)** — записывает все изменения перед применением;
- **Write-Ahead Logging (WAL)** — принцип “сначала лог, потом данные”;
- **fsync/flush** — физическая запись на диск.

Если сервер падает после `COMMIT`, при следующем запуске SQL Server восстанавливает данные из журнала.

---

## 🔹 Нарушения изоляции и примеры

| Проблема | Пример | Решение |
|-----------|---------|----------|
| **Dirty Read** | одна транзакция читает незакоммиченные данные другой | Read Committed |
| **Non-repeatable Read** | при повторном чтении данные изменились | Repeatable Read |
| **Phantom Read** | между чтениями добавились новые строки | Serializable / Snapshot |

---

## 🔹 Пример аномалий

**Dirty Read:**
```sql
-- Транзакция 1
BEGIN TRANSACTION
UPDATE Accounts SET Balance = Balance - 100 WHERE Id = 1;
-- (ещё не COMMIT)

-- Транзакция 2
SELECT Balance FROM Accounts WHERE Id = 1; -- видит -100 (грязное чтение)
```

**Решение:**
```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

---

## 🔹 ACID и EF Core

EF Core по умолчанию обеспечивает **A, C, D**.  
Изоляцию можно настраивать через транзакции.

**EF Core пример:**
```csharp
await using var tx = await context.Database.BeginTransactionAsync(IsolationLevel.ReadCommitted);

await context.SaveChangesAsync();
await tx.CommitAsync();
```

EF также поддерживает **distributed transactions** (через `TransactionScope`) — несколько источников данных в одной транзакции.

```csharp
using (var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled))
{
    await db1.SaveChangesAsync();
    await db2.SaveChangesAsync();
    scope.Complete();
}
```

---

## 🔹 ACID в реальных системах

| Сценарий | Применение |
|-----------|-------------|
| **Банковские операции** | строгая ACID — атомарные переводы |
| **E-commerce корзина** | допустимо ослабить изоляцию (Read Committed) |
| **Очереди сообщений** | часто используют eventual consistency |
| **Микросервисы** | ACID на уровне одного сервиса, а между сервисами — Saga / Outbox |

---

## 🔹 ACID vs BASE

| Свойство | ACID | BASE |
|-----------|-------|------|
| Консистентность | мгновенная | со временем |
| Применение | классические SQL БД | распределённые NoSQL системы |
| Гарантии | строгие | гибкие |
| Пример | SQL Server, PostgreSQL | Cassandra, MongoDB |

---

## 🔹 Ключевые тезисы для интервью

- **A (Atomicity):** транзакция — единое целое, rollback при ошибке.  
- **C (Consistency):** данные всегда остаются корректными.  
- **I (Isolation):** параллельные транзакции не мешают друг другу.  
- **D (Durability):** после commit данные не теряются.  
- EF Core и SQL Server реализуют все четыре свойства.  
- Важно понимать **уровни изоляции** и **trade-offs между безопасностью и производительностью**.
