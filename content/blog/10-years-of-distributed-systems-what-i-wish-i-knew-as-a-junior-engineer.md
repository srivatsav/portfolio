---
title: "10 Years of Distributed Systems: What I Wish I Knew as a Junior Engineer"
date: 2025-03-23
---

I've been building distributed systems for a decade now. Looking back, I see patterns—decisions that worked, mistakes I repeated, lessons that took years to learn.

This post is a letter to my junior self. Maybe it helps you too.

## 2015: The "It Works on My Machine" Fallacy

My first job at Cognizant, I built a backend service for Toyota dealers. Tested it locally. Passed all tests. Deployed to staging.

**It failed**.

The issue? Database connection pool size. On my laptop, 5 connections were plenty. In production, with 200 concurrent users, we ran out.

```java
// What I wrote
DataSource ds = new DriverManager.getConnection(url, user, pass);

// What I should have written
HikariConfig config = new HikariConfig();
config.setMaximumPoolSize(100);  // Not 5
config.setMinimumIdle(10);
DataSource ds = new HikariDataSource(config);
```

**Lesson**: Production is not your laptop. Configure for production load, not development convenience.

## 2017: The Monolith That Ate My Weekends

At ROKA, I built the Travel Desk solution as a monolith. Everything in one codebase:

- Flight booking engine
- Hotel booking engine
- Bus booking engine
- Approval workflows
- Notifications
- Reporting

**The problem**:

A bug in flight booking could break hotel bookings. A code change to notifications required redeploying everything. When onboarding a new travel aggregator, I risked breaking existing integrations.

**Refactoring took 6 months**. We extracted services one by one:

```
Monolith
  ├─ Booking Service (extracted first)
  ├─ Notification Service (next)
  ├─ Reporting Service (last)
  └─ Remaining Monolith (shrinking)
```

**Lesson**: Strangler Fig pattern is painful but necessary. Don't let monoliths grow until they're impossible to dismantle.

## 2019: The Async Trap

At Phenom, we switched to async processing. "Everything should be async" was the mantra.

We made everything async:

```java
@Async
public Future<Booking> createBooking(BookingRequest request) {
    return bookingService.createAsync(request);
}
```

**The problem**: We lost observability. When a booking failed, we didn't know why—the error was lost in a Future. Debugging became a nightmare.

We rolled back:

```java
// Selective async, not blanket
public Booking createBooking(BookingRequest request) {
    // Sync path
    if (request.isUrgent()) {
        return bookingService.createSync(request);
    }
    // Async path
    return bookingService.createAsync(request);
}
```

**Lesson**: Async is a tool, not a religion. Use it where it helps, not everywhere.

## 2021: The Schema Migration Disaster

At Phenom, we decided to break a monolithic Postgres database into microservices, each with its own database.

**The design**:

```
Orders DB (Postgres) ──┐
Users DB (Postgres)   ├─> Orders Service
Payments DB (Postgres) ──┘
```

**The migration strategy**: Change table by change scripts. 200+ scripts.

**What went wrong**:

Script 47 failed mid-way. Orders were in Orders DB, but payments were still in Payments DB. Inconsistency. We couldn't roll back—the old schema was gone.

**Recovery**: 2 days of manual data repair.

**What I learned**:

### 1. Backwards Compatibility

Migrate data, but keep old schema readable:

```sql
-- Old schema still works
SELECT id, amount FROM orders WHERE status = 'pending';

-- New schema works too
SELECT id, amount FROM orders_v2 WHERE status = 'pending';

-- View bridges both
CREATE VIEW orders AS SELECT * FROM orders_v2;
```

### 2. Change Data, Not Just Schema

We migrated the database structure but didn't migrate the data access layer. Code still assumed monolith schema.

**Lesson**: Treat schema migrations as full-stack changes, not just database tasks.

### 3. Feature Flags, Not Migrations

Instead of migrating on deploy day, we should have used feature flags:

```java
if (featureFlags.isEnabled("new_orders_schema")) {
    // Use new schema
} else {
    // Use old schema
}
```

This lets you migrate gradually, not in one day.

**Lesson**: Database migrations are product releases, not technical tasks. Treat them as such.

## 2023: The "It's Just a Small Refactor" Lie

I've seen this repeatedly. Engineers (including me) say:

> "This is a quick refactor, just cleaning up code. Won't change behavior."

**Every time, it breaks something.**

At Warner Bros., my team refactored a data validation module. "Just renaming functions, same logic."

Two weeks later:

```
Production incident: Data validation returning false negatives.
Root cause: Refactored function didn't handle null case that previous version handled.
```

**What I learned**:

### Treat Refactor as Risky as Feature Addition

Refactoring changes code paths. Every changed path needs:
- Unit tests
- Integration tests
- Manual verification

**Never refactor without tests**.

### Branch by Abstraction, Not by File

```java
// Bad: Refactor by touching files that look "messy"
class DataValidator {
    public void refactorThis() { ... }  // Why just this?
}

// Good: Refactor by changing behavior
class DataValidator {
    // Extracted to support new requirement
    public ValidationResult validateWithRules(List<Rule> rules) { ... }
}
```

**Lesson**: Refactoring is architectural change. Treat it with the same rigor as feature work.

## 2024: The Observability Gap

At Warner Bros., we built a financial reporting pipeline. Worked in dev. Passed QA.

**Three weeks after production**: Users reported missing reports.

We checked logs:

```
2024-03-15 10:23:12 INFO Report generated successfully
2024-03-15 10:23:15 INFO Report sent to database
2024-03-15 10:23:18 INFO Report completed
```

Everything looked fine.

**The problem**: We logged success, but not what success meant. Was the report empty? Did it have the right columns?

We added structured logging:

```java
logger.info("Report generated",
    Map.of(
        "reportId", report.id,
        "rowCount", report.rows.size(),
        "columns", report.columns,
        "user", report.userId
    )
);
```

Suddenly, we could search logs for "rowCount=0" and find the bug immediately.

**Lesson**: Log what matters, not that something happened. "Success" without context is useless.

## Patterns That Work

After 10 years, these patterns consistently work:

### 1. Circuit Breakers Everywhere

```java
@CircuitBreaker(
    failureThreshold = 5,
    timeout = Duration.ofSeconds(10),
    resetTimeout = Duration.ofMinutes(1)
)
public Result callDownstream(DownstreamService service) {
    return service.execute();
}
```

Every external dependency gets a circuit breaker. Cascading failures are unacceptable.

### 2. Idempotent Operations

Every write operation is idempotent:

```java
// Good
public void updateOrderStatus(String orderId, Status status) {
    repository.updateWhere(
        "id = ? AND status != ?",
        orderId, status
    );
}

// Bad (updates every time called)
public void updateOrderStatus(String orderId, Status status) {
    repository.updateById(orderId, status);
}
```

Retries work because operations are idempotent.

### 3. Feature Flags for Everything

We don't deploy code directly to production:

```
Code → Feature flag (default OFF) → Canary (5%) → Production (100%)
```

This has saved us multiple times from production incidents.

### 4. Architectural Decision Records (ADRs)

Every significant decision gets an ADR:

```markdown
# ADR-001: Use Kafka for Event Streaming

## Context
We need to decouple integration platform from downstream systems.

## Decision
Use Apache Kafka as event streaming platform.

## Consequences
### Positive
- Decouples producers from consumers
- Supports replay for debugging
- Scales horizontally

### Negative
- Adds operational complexity (Kafka cluster management)
- Learning curve for team

## Alternatives Considered
- RabbitMQ (rejected: less scalable)
- AWS Kinesis (rejected: vendor lock-in)
```

Six months later, when asked "why Kafka?", the ADR answers. No debate, no re-litigation.

## What I'd Tell My Junior Self

### Technical Advice

1. **Write code for the next reader, not yourself**
   - Variable names should explain "why"
   - Comments should explain "what we can't change from code"
   - Tests should be documentation

2. **Measure before optimizing**
   - Don't guess. Profile first.
   - The bottleneck is rarely where you think it is.

3. **Design for failure**
   - Assume networks will fail
   - Assume databases will be slow
   - Assume downstreams will be down

4. **Treat configuration as code**
   - Don't hardcode URLs, timeouts, limits
   - Version control config changes

### Career Advice

1. **Own your failures**
   - "I broke production" is acceptable if followed by "and here's what I learned"
   - Teams that hide failures don't improve

2. **Teach what you learn**
   - Writing makes you think deeper
   - Blog posts become your knowledge base
   - You don't truly understand something until you can explain it

3. **Balance breadth and depth**
   - Don't chase every new framework
   - Pick 2-3 technologies and go deep
   - Depth compounds; breadth dilutes

## The Future

Distributed systems are getting harder, not easier:
- Microservices multiplying operational complexity
- Event-driven architectures making reasoning harder
- AI tools changing how we write code

But fundamentals stay the same:
- Reliability > speed
- Simplicity > cleverness
- Observability > invisibility
- Empathy for the next developer

---

**If you're early in your career: Ask questions. Make mistakes. Learn. You have decades ahead of you.**

**If you're experienced: Mentor. Share what you know. Your scars are valuable lessons for others.**

---

What lessons have you learned the hard way? I'd love to hear your stories.
