# Transactional Outbox Pattern
### PostgreSQL + Dapper + MassTransit

A complete guide to solving the **dual-write problem** (saving to DB + publishing to a message broker) using the Transactional Outbox pattern.

---

## Table of Contents
1. [The Problem](#the-problem)
2. [Architecture Overview](#architecture-overview)
3. [Database Schema](#database-schema)
4. [Project Setup](#project-setup)
5. [Models](#models)
6. [Reusable Outbox Writer](#reusable-outbox-writer)
7. [Domain Handler (Producer)](#domain-handler-producer)
8. [The Outbox Relay](#the-outbox-relay)
9. [Dead-Letter Table](#dead-letter-table)
10. [Why FOR UPDATE SKIP LOCKED](#why-for-update-skip-locked)
11. [Idempotency on the Consumer Side](#idempotency-on-the-consumer-side)
12. [Optional: Lower-Latency with LISTEN/NOTIFY](#optional-lower-latency-with-listennotify)
13. [Testing Tips](#testing-tips)
14. [Operational Checklist](#operational-checklist)

---

## The Problem

```
Without Outbox:
┌─────────────────┐
│ Begin Tx        │
│ Save to DB   ✓  │  ← If this succeeds but...
│ Publish      ✗  │  ← ...this fails, you have inconsistency
│ Commit Tx       │
└─────────────────┘
```

```
With Outbox:
┌─────────────────┐
│ Begin Tx        │
│ Save to DB   ✓  │  Atomic
│ Save to Outbox ✓│  Atomic
│ Commit Tx       │
└─────────────────┘
         ↓
   Background Relay picks up from Outbox → Publishes to MassTransit
```

---

## Architecture Overview

```
┌──────────────────┐         ┌──────────────────┐
│   Application    │         │   OutboxRelay    │
│                  │         │ (BackgroundSvc)  │
│ 1. INSERT order  │         │                  │
│ 2. INSERT outbox │         │ 1. SELECT FOR    │
│ 3. COMMIT        │         │    UPDATE SKIP   │
└────────┬─────────┘         │    LOCKED        │
         │                   │ 2. Publish via   │
         │                   │    MassTransit   │
         │                   │ 3. UPDATE         │
         │                   │    processed_at   │
         ▼                   └────────┬──────────┘
   ┌──────────┐                       │
   │ Postgres │ ◄─────────────────────┘
   │  + Outbox│
   └────┬─────┘
        │
        ▼
   ┌──────────┐       ┌──────────────┐
   │ RabbitMQ │ ────► │  Consumers   │
   └──────────┘       └──────────────┘
```

---

## Database Schema

```sql
-- 1. Main outbox table
CREATE TABLE outbox_messages (
    id              UUID PRIMARY KEY,
    message_type    TEXT NOT NULL,            -- e.g., "OrderCreated"
    payload         JSONB NOT NULL,           -- serialized event
    correlation_id  UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at    TIMESTAMPTZ,
    attempts        INT NOT NULL DEFAULT 0,
    last_error      TEXT,
    next_attempt_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Partial index keeps it tiny and fast for the relay
CREATE INDEX idx_outbox_pending
    ON outbox_messages (next_attempt_at)
    WHERE processed_at IS NULL;

-- 2. Idempotency table (for consumers)
CREATE TABLE processed_messages (
    id              UUID PRIMARY KEY,
    processed_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 3. Dead-letter table
CREATE TABLE outbox_dead_letters (
    id              UUID PRIMARY KEY,
    message_type    TEXT NOT NULL,
    payload         JSONB NOT NULL,
    correlation_id  UUID,
    created_at      TIMESTAMPTZ,
    attempts        INT,
    last_error      TEXT,
    dead_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Project Setup

```bash
dotnet add package Npgsql
dotnet add package Dapper
dotnet add package MassTransit
dotnet add package MassTransit.RabbitMQ
```

```csharp
// Program.cs
using Npgsql;

var builder = WebApplication.CreateBuilder(args);

// 1. Npgsql data source (registered as singleton)
var dataSource = new NpgsqlDataSourceBuilder(
    builder.Configuration.GetConnectionString("Default")).Build();
builder.Services.AddSingleton(dataSource);

// 2. MassTransit
builder.Services.AddMassTransit(x =>
{
    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host("localhost", "/", h =>
        {
            h.Username("guest");
            h.Password("guest");
        });
        cfg.ConfigureEndpoints(ctx);
    });
});

// 3. Outbox relay as a hosted service
builder.Services.AddHostedService<OutboxRelay>();

var app = builder.Build();
app.Run();
```

---

## Models

```csharp
public class OutboxMessage
{
    public Guid Id { get; set; }
    public string MessageType { get; set; } = default!;
    public string Payload { get; set; } = default!;
    public Guid? CorrelationId { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? ProcessedAt { get; set; }
    public int Attempts { get; set; }
    public string? LastError { get; set; }
    public DateTime NextAttemptAt { get; set; }
}

// Example event published to MassTransit
public record OrderCreatedEvent(
    Guid OrderId,
    Guid CustomerId,
    decimal Amount,
    DateTime CreatedAt);

// Example business command
public record CreateOrderCommand(
    Guid CustomerId,
    decimal Amount,
    Guid? CorrelationId = null);
```

---

## Reusable Outbox Writer

```csharp
using System.Text.Json;
using Dapper;
using Npgsql;

public static class OutboxWriter
{
    private static readonly JsonSerializerOptions JsonOpts = new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase
    };

    public static async Task EnqueueAsync(
        NpgsqlConnection conn,
        NpgsqlTransaction tx,
        object evt,
        Guid? correlationId,
        CancellationToken ct)
    {
        const string sql = @"
            INSERT INTO outbox_messages
                (id, message_type, payload, correlation_id, created_at, next_attempt_at)
            VALUES
                (@Id, @MessageType, @Payload::jsonb, @CorrelationId, @CreatedAt, @NextAttemptAt)";

        await conn.ExecuteAsync(new CommandDefinition(sql, new
        {
            Id            = Guid.NewGuid(),
            MessageType   = evt.GetType().FullName!,
            Payload       = JsonSerializer.Serialize(evt, evt.GetType(), JsonOpts),
            CorrelationId = correlationId,
            CreatedAt     = DateTime.UtcNow,
            NextAttemptAt = DateTime.UtcNow
        }, transaction: tx, cancellationToken: ct));
    }
}
```

---

## Domain Handler (Producer)

The business write and the outbox write happen in the **same transaction**.

```csharp
public interface IOrderService
{
    Task<Guid> CreateOrderAsync(CreateOrderCommand cmd, CancellationToken ct);
}

public class OrderService : IOrderService
{
    private readonly NpgsqlDataSource _dataSource;

    public OrderService(NpgsqlDataSource dataSource) => _dataSource = dataSource;

    public async Task<Guid> CreateOrderAsync(CreateOrderCommand cmd, CancellationToken ct)
    {
        var orderId = Guid.NewGuid();
        var evt = new OrderCreatedEvent(
            orderId, cmd.CustomerId, cmd.Amount, DateTime.UtcNow);

        await using var conn = await _dataSource.OpenConnectionAsync(ct);
        await using var tx = await conn.BeginTransactionAsync(ct);

        try
        {
            // 1. Business write
            const string insertOrder = @"
                INSERT INTO orders (id, customer_id, amount, created_at)
                VALUES (@Id, @CustomerId, @Amount, @CreatedAt)";

            await conn.ExecuteAsync(new CommandDefinition(insertOrder, new
            {
                Id        = orderId,
                cmd.CustomerId,
                cmd.Amount,
                CreatedAt = DateTime.UtcNow
            }, transaction: tx, cancellationToken: ct));

            // 2. Outbox row in the SAME transaction
            await OutboxWriter.EnqueueAsync(conn, tx, evt, cmd.CorrelationId, ct);

            // 3. Single commit
            await tx.CommitAsync(ct);
            return orderId;
        }
        catch
        {
            await tx.RollbackAsync(ct);
            throw;
        }
    }
}
```

---

## The Outbox Relay

The relay is a `BackgroundService` that:
1. Claims a batch of rows with `FOR UPDATE SKIP LOCKED`
2. Publishes each via `IPublishEndpoint`
3. Marks successful ones as processed
4. Applies exponential backoff to failed ones
5. Moves exhausted messages to the dead-letter table

```csharp
using System.Text.Json;
using Dapper;
using MassTransit;
using Npgsql;

public class OutboxRelay : BackgroundService
{
    private readonly NpgsqlDataSource _dataSource;
    private readonly IPublishEndpoint _publishEndpoint;
    private readonly ILogger<OutboxRelay> _logger;

    private static readonly TimeSpan PollInterval = TimeSpan.FromMilliseconds(500);
    private const int BatchSize  = 50;
    private const int MaxAttempts = 8;

    public OutboxRelay(
        NpgsqlDataSource dataSource,
        IPublishEndpoint publishEndpoint,
        ILogger<OutboxRelay> logger)
    {
        _dataSource = dataSource;
        _publishEndpoint = publishEndpoint;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                var processed = await ProcessBatchAsync(stoppingToken);
                if (processed == 0)
                    await Task.Delay(PollInterval, stoppingToken);
            }
            catch (OperationCanceledException) { /* shutdown */ }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Outbox relay loop failed");
                await Task.Delay(TimeSpan.FromSeconds(2), stoppingToken);
            }
        }
    }

    private async Task<int> ProcessBatchAsync(CancellationToken ct)
    {
        await using var conn = await _dataSource.OpenConnectionAsync(ct);
        await using var tx = await conn.BeginTransactionAsync(ct);

        // 1. Atomically claim a batch. Other workers SKIP locked rows.
        const string fetchSql = @"
            SELECT id, message_type, payload, correlation_id,
                   created_at, processed_at, attempts, last_error, next_attempt_at
            FROM outbox_messages
            WHERE processed_at IS NULL
              AND next_attempt_at <= now()
            ORDER BY created_at
            LIMIT @BatchSize
            FOR UPDATE SKIP LOCKED";

        var batch = (await conn.QueryAsync<OutboxMessage>(
            new CommandDefinition(fetchSql,
                new { BatchSize },
                transaction: tx,
                cancellationToken: ct))).ToList();

        if (batch.Count == 0)
        {
            await tx.CommitAsync(ct);
            return 0;
        }

        // 2. Publish each, track results in-memory
        var now = DateTime.UtcNow;
        var successIds = new List<Guid>(batch.Count);
        var failedUpdates = new List<(Guid Id, int Attempts, string Err, DateTime NextAt)>();

        foreach (var msg in batch)
        {
            try
            {
                var type = Type.GetType(msg.MessageType, throwOnError: true)!;
                var evtObj = JsonSerializer.Deserialize(msg.Payload, type)!;

                await _publishEndpoint.Publish(evtObj, ctx =>
                {
                    if (msg.CorrelationId.HasValue)
                        ctx.CorrelationId = msg.CorrelationId;
                    ctx.Headers.Set("outbox-id", msg.Id);
                }, ct);

                successIds.Add(msg.Id);
            }
            catch (Exception ex)
            {
                var nextAttempt = msg.Attempts + 1;
                var backoff = TimeSpan.FromSeconds(
                    Math.Pow(5, Math.Min(nextAttempt, 5)));

                failedUpdates.Add((msg.Id, nextAttempt, ex.Message, now + backoff));
                _logger.LogWarning(ex,
                    "Outbox publish failed. Id={Id} Attempt={Attempt}",
                    msg.Id, nextAttempt);
            }
        }

        // 3a. Mark successes
        if (successIds.Count > 0)
        {
            const string markDone = @"
                UPDATE outbox_messages
                SET processed_at = @Now
                WHERE id = ANY(@Ids)";
            await conn.ExecuteAsync(new CommandDefinition(markDone,
                new { Now = now, Ids = successIds.ToArray() },
                transaction: tx, cancellationToken: ct));
        }

        // 3b. Mark failures with backoff
        foreach (var f in failedUpdates)
        {
            const string markFail = @"
                UPDATE outbox_messages
                SET attempts        = @Attempts,
                    last_error      = @Err,
                    next_attempt_at = @NextAt
                WHERE id = @Id";

            await conn.ExecuteAsync(new CommandDefinition(markFail, new
            {
                f.Id, f.Attempts, Err = f.Err, NextAt = f.NextAt
            }, transaction: tx, cancellationToken: ct));
        }

        // 4. Move truly dead messages to dead-letter table
        const string deadLetter = @"
            INSERT INTO outbox_dead_letters
                (id, message_type, payload, correlation_id,
                 created_at, attempts, last_error)
            SELECT id, message_type, payload, correlation_id,
                   created_at, attempts, last_error
            FROM outbox_messages
            WHERE id = ANY(@Ids) AND attempts >= @Max";

        var deadIds = failedUpdates
            .Where(x => x.Attempts >= MaxAttempts)
            .Select(x => x.Id)
            .ToArray();

        if (deadIds.Length > 0)
        {
            await conn.ExecuteAsync(new CommandDefinition(deadLetter,
                new { Ids = deadIds, Max = MaxAttempts },
                transaction: tx, cancellationToken: ct));

            const string removeDead =
                "DELETE FROM outbox_messages WHERE id = ANY(@Ids)";
            await conn.ExecuteAsync(new CommandDefinition(removeDead,
                new { Ids = deadIds },
                transaction: tx, cancellationToken: ct));
        }

        await tx.CommitAsync(ct);
        return batch.Count;
    }
}
```

---

## Dead-Letter Table

Already shown in the schema. The relay automatically:
- Inserts into `outbox_dead_letters` after `MaxAttempts` failures
- Deletes from `outbox_messages`
- Preserves the full payload for inspection or replay

---

## Why `FOR UPDATE SKIP LOCKED`

```
Relay 1 (pod A)                   Relay 2 (pod B)
─────────────────                 ─────────────────
BEGIN TX                          BEGIN TX
SELECT ... FOR UPDATE             SELECT ... FOR UPDATE SKIP LOCKED
  rows: A, B, C (locked)            rows: A, B, C → SKIPPED
                                    rows: D, E, F (locked)
publish A, B, C                    publish D, E, F
UPDATE processed_at                UPDATE processed_at
COMMIT                            COMMIT
```

Each row is claimed by **exactly one** worker. Safe horizontal scaling, no double-publish, no lost rows.

---

## Idempotency on the Consumer Side

Because the relay provides **at-least-once** delivery, consumers must dedupe using the `outbox-id` header.

```csharp
using Dapper;
using MassTransit;
using Npgsql;

public class OrderCreatedConsumer : IConsumer<OrderCreatedEvent>
{
    private readonly NpgsqlDataSource _dataSource;

    public OrderCreatedConsumer(NpgsqlDataSource dataSource)
        => _dataSource = dataSource;

    public async Task Consume(ConsumeContext<OrderCreatedEvent> ctx)
    {
        var outboxId = ctx.Headers.Get<Guid>("outbox-id");

        await using var conn = await _dataSource.OpenConnectionAsync();
        await using var tx = await conn.BeginTransactionAsync();

        // 1. Try to claim the message
        const string claimSql = @"
            INSERT INTO processed_messages (id, processed_at)
            VALUES (@Id, now())
            ON CONFLICT (id) DO NOTHING
            RETURNING id";

        var inserted = await conn.ExecuteScalarAsync<Guid?>(
            new CommandDefinition(claimSql,
                new { Id = outboxId },
                transaction: tx));

        if (inserted is null)
        {
            // Already processed before — no-op
            await tx.RollbackAsync();
            return;
        }

        // 2. Actual business side-effects (e.g., send email, update read model)
        // ... do work ...

        await tx.CommitAsync();
    }
}
```

---

## Optional: Lower-Latency with `LISTEN/NOTIFY`

Skip the 500ms polling latency. Add a trigger and have the relay subscribe.

```sql
CREATE OR REPLACE FUNCTION notify_outbox() RETURNS trigger AS $$
BEGIN
    PERFORM pg_notify('outbox_new', '');
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_outbox_notify
    AFTER INSERT ON outbox_messages
    FOR EACH STATEMENT EXECUTE FUNCTION notify_outbox();
```

```csharp
// In OutboxRelay.ExecuteAsync override:
private readonly System.Threading.Channels.Channel<bool> _signal =
    System.Threading.Channels.Channel.CreateBounded<bool>(new(1));

protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    // LISTEN task runs in parallel
    _ = Task.Run(async () =>
    {
        await using var conn = await _dataSource.OpenConnectionAsync(stoppingToken);
        conn.Notification += (_, _) => _signal.Writer.TryWrite(true);

        await using var cmd = new NpgsqlCommand("LISTEN outbox_new", conn);
        await cmd.ExecuteNonQueryAsync(stoppingToken);

        // Keep connection alive
        while (!stoppingToken.IsCancellationRequested)
            await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
    }, stoppingToken);

    // Main loop — wait for signal or fallback timer
    while (!stoppingToken.IsCancellationRequested)
    {
        try
        {
            await ProcessBatchAsync(stoppingToken);
        }
        catch (OperationCanceledException) { break; }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Outbox relay loop failed");
        }

        // Wait for a notification OR a 5s safety timer
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(stoppingToken);
        cts.CancelAfter(TimeSpan.FromSeconds(5));
        try
        {
            await _signal.Reader.ReadAsync(cts.Token);
        }
        catch (OperationCanceledException) { /* timer fired, loop again */ }
    }
}
```

---

## Testing Tips

### 1. Unit test the handler
```csharp
[Fact]
public async Task CreateOrder_BothInsertsSucceed()
{
    // Use Testcontainers for Postgres
    // Call service, then verify both orders and outbox_messages have a row
}
```

### 2. Test the relay
```csharp
[Fact]
public async Task Relay_PublishesPendingMessages()
{
    // Insert 3 outbox rows directly
    // Start the relay for 2 seconds
    // Assert all 3 are marked processed_at
}
```

### 3. Test failure handling
```csharp
[Fact]
public async Task Relay_FailedPublishAppliesBackoff()
{
    // Make a row with a non-existent message_type
    // Run the relay
    // Assert attempts = 1, next_attempt_at is in the future
}
```

---

## Operational Checklist

| Concern | Solution |
|---|---|
| Long-held locks | Keep batch small (50 rows), commit fast |
| Stuck messages | `attempts` cap; route to `dead_letter` after N tries |
| Hot index | Partial index `WHERE processed_at IS NULL` keeps it small |
| Idempotency on consumer | Use `outbox-id` header as message ID; consumer dedupes |
| Ordering | `ORDER BY created_at` + `SKIP LOCKED` preserves FIFO per worker |
| Hot row contention | The outbox rows are *not* the business rows — no contention with user traffic |
| Horizontal scaling | Run N replicas of the relay; `SKIP LOCKED` makes it safe |
| Monitoring | Track: outbox lag, dead-letter count, publish duration, batch size |
| Replay | `INSERT INTO outbox_messages SELECT ... FROM outbox_dead_letters` to retry |
| Connection pooling | `NpgsqlDataSource` is connection-pooled by default |

---

## TL;DR

```
[Handler]
   └─ conn.BeginTransactionAsync()
   └─ Dapper: INSERT order
   └─ Dapper: INSERT outbox_messages
   └─ tx.CommitAsync()        ← atomic

[OutboxRelay (1..N replicas)]
   └─ Dapper: SELECT ... FOR UPDATE SKIP LOCKED
   └─ IPublishEndpoint.Publish
   └─ Dapper: UPDATE processed_at (or backoff)
   └─ tx.CommitAsync()

[Consumer]
   └─ Check processed_messages by outbox-id
   └─ Do work idempotently
```

Result: **at-least-once** delivery, **no dual-write inconsistency**, **safe horizontal scaling**, **fully visible** outbox state in Postgres.
