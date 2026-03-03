# AGENTS.md — Go Hexagonal Architecture (Principal Engineer Edition)

## 🎯 Architecture Strategy: Modular Hexagonal (Clean Boundaries)

This project follows **Hexagonal Architecture (Ports & Adapters)** inside a **Modular Monolith**.

### System Boundaries

* **Public API** → Client-facing use cases
* **Internal API** → Admin/System operations
* **Domain Core** → Business logic (pure Go)
* **Infrastructure** → DB, Queue, External APIs

### Dependency Direction (Strict Rule)

```
Handler (Adapter) → Service (Use Case) → Repository (Port) → Infrastructure (Adapter)
```

**Inner layers must never depend on outer layers.**

* No framework imports inside `service`
* No SQL inside `service`
* No HTTP types inside `repository`
* No circular imports between modules

---

# 🏗️ Layer Responsibilities

## 1️⃣ Handlers (Driving Adapters)

**Responsibility:**

* Parse input
* Validate DTO
* Call service
* Map error → HTTP status
* Return JSON

**Rules:**

* Validate all inputs before calling service
* Never expose DB errors to Public API
* Do not implement business logic
* Use request-scoped context
* Keep handler thin (<50 lines preferred)

---

## 2️⃣ Services (Domain / Use Cases)

**Responsibility:**

* Coordinate business logic
* Enforce invariants
* Manage transactions (via repository abstraction)
* Call external systems through ports

**Rules:**

* Depend on interfaces only
* No framework imports
* No ORM imports
* One method = one business transaction
* If file > 500 LOC → split by use case

**Define repository interfaces where consumed (inside service package).**

---

## 3️⃣ Repositories (Driven Adapters)

**Responsibility:**

* Data persistence
* Query optimization
* Transactions
* Locking strategy

**Rules:**

* Always propagate `context.Context`
* Never ignore cancellation
* No business rules
* Keep transactions short
* Never call external APIs inside transaction

---

## 4️⃣ Models

* Pure persistence models
* No business logic
* Separate API DTOs from DB models
* Never expose DB schema directly to Public API

---

# ⚡ Performance & Safety First

## 1️⃣ Query Optimization

### ❌ Avoid

* N+1 queries
* SELECT *
* Query inside loop

### ✅ Enforce

* Preload/Join associations
* Select only needed columns
* Batch inserts/updates
* Use lightweight structs for reporting
* Prefer raw queries for complex aggregations

---

## 2️⃣ Transactions & Atomicity

### Golden Rule:

A Service method = one atomic business operation.

### Best Practices:

* Wrap multi-table updates in a transaction
* Insert background jobs inside the same transaction
* Keep transactions short
* Never do heavy compute inside transactions

---

## 3️⃣ Deadlock & Connection Safety

### Prevent Deadlocks:

* Lock rows in consistent order (sort IDs)
* Avoid long-running transactions

### Prevent Connection Exhaustion:

* Always pass context
* Configure:

  * MaxOpenConns
  * MaxIdleConns
  * ConnMaxLifetime
* No blocking calls inside repositories

---

# 🧵 Concurrency Rules

## Go Routines

### ❌ Never

```go
go func() {
    // fire and forget critical logic
}()
```

### ✅ Always

* Use `errgroup` or `WaitGroup`
* Limit concurrency using worker pools
* Use background job system for required tasks

### Rule of Thumb

| Task Type                | Solution             |
| ------------------------ | -------------------- |
| >200ms                   | Background job       |
| Critical                 | Transaction + Queue  |
| Non-critical side effect | Controlled goroutine |

---

# 📦 Background Jobs (Queue Pattern)

### Mandatory For:

* Exports
* Reports
* Heavy sync
* External integrations
* Anything slow or retryable

### Atomic Pattern

```go
func (s *Service) Execute(ctx context.Context, input Input) error {
    return s.repo.Transaction(ctx, func(r Repository) error {

        if err := r.Update(ctx, input.ID); err != nil {
            return err
        }

        if err := r.EnqueueJobTx(ctx, input.ID); err != nil {
            return err
        }

        return nil
    })
}
```

If DB fails → job must not exist.

---

# 🧩 Maintainability Principles

## 1️⃣ Logic Locality

* Keep feature code vertically grouped:

  ```
  export/
      handler.go
      service.go
      repository.go
  ```
* Avoid creating files for tiny helpers
* Extract only when reused 3+ times

---

## 2️⃣ Interface Discipline

* Define interfaces in the consumer package
* Avoid global interfaces
* Mock only:

  * DB
  * Queue
  * External APIs

Never mock simple transformers or internal structs.

---

## 3️⃣ Service Splitting

If a service:

* Has admin logic + public logic
* Has many `if role == admin`
* Exceeds 500 lines

→ Split into separate services.

---

## 4️⃣ Explicit Initialization

### ❌ Avoid

* `init()` for DB setup
* Global singletons

### ✅ Prefer

* Dependency injection via constructor
* Explicit wiring in `main.go`

---

# 🛡️ API Design Safety

## Validation Layer

* Validate DTO in handler
* Reject early (Fail Fast)
* Never trust downstream

## Error Mapping

Internal errors must be mapped:

| Internal Error | Public Response |
| -------------- | --------------- |
| Not Found      | 404             |
| Validation     | 400             |
| Conflict       | 409             |
| Unauthorized   | 401             |
| Unexpected     | 500             |

Never leak:

* SQL errors
* Stack traces
* Internal IDs

---

# 📊 Observability & Traceability

* Pass `request_id` via context
* Log structured JSON
* Include latency measurement
* Instrument DB queries
* Measure queue processing time

---

# 🧠 Engineering Philosophy

### 1️⃣ Optimize for:

* Clarity
* Atomicity
* Performance
* Operational safety

### 2️⃣ Avoid:

* Over-abstraction
* Premature microservices
* Interface bloat
* “Enterprise patterns” without need

### 3️⃣ Golden Principle

> Make illegal states unrepresentable.
>
> Make business transactions atomic.
>
> Make performance intentional.

---

# 🏁 Summary Cheat Sheet

| Problem            | Solution                   |
| ------------------ | -------------------------- |
| N+1                | Preload / Join             |
| Deadlock           | Sorted locking order       |
| Slow endpoint      | Background job             |
| Dirty data         | Validate in handler        |
| Circular import    | Strict layer dependency    |
| Large service      | Split by use case          |
| Connection full    | Context + DB tuning        |
| Data inconsistency | Transaction + atomic queue |
