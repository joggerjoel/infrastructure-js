# Backend OOP & Structure (Node/Express)

**Scope:** `backend/src` (Express server, controllers, services, models)  
**Purpose:** Define a clear OOP/structure so the server stays testable, maintainable, and consistent.

---

## 1. Target Structure (Well-Defined Layers)

```
backend/src/
├── server.ts           # Bootstrap: app, middleware, route mounting
├── routes/             # HTTP → controller binding only (no business logic)
├── controllers/       # Request/response handling; delegate to services
├── services/          # Application & domain logic (use-cases, orchestration)
├── models/            # Domain entities & data access (optional; can live in services)
├── middleware/        # Cross-cutting (auth, errors, logging)
├── utils/             # Pure helpers (logging, compression); no domain
├── sql/               # Raw SQL files (migrations, one-off scripts)
└── types/             # Shared TypeScript types (no runtime behavior)
```

**Rules:**

- **Controllers:** Thin. Parse input (query, body), call one or more services, shape response. No SQL, no business rules.
- **Services:** Where behavior lives. Use classes for **stateful or multi-method use-cases**; use **plain functions** for stateless, single-responsibility operations (e.g. `calculateScore`, `buildQuery`).
- **Models:** Optional. Use when you have clear **domain entities** with identity and behavior; otherwise services can own queries and DTOs.
- **Dependencies:** Prefer **constructor injection** for services (DB, other services) so tests can inject mocks. Avoid global `getConnection()` / `getRedis()` inside services where it blocks testability.

---

## 2. When to Use Classes vs Functions

| Use | Prefer | Example |
|-----|--------|--------|
| **Use-case / application service** (multi-step, stateful, or needs injected deps) | **Class** | `UserService`, `OrderService`, `PaymentService` |
| **Stateless single operation** (pure or I/O behind injectable abstraction) | **Function** | `calculateScore()`, `buildQuery()`, `formatDate()` |
| **Domain entity** (identity + invariants) | **Class** (optional) | `User`, `Order`, `Product` as classes with validation |
| **Controller** | Either | Class: `UserController`; functions: `getUsers`, `createUser` — **pick one style per module** and stick to it |
| **Infrastructure adapter** (DB, Redis, HTTP client) | **Class** or **factory function** | `DatabaseClient` (class); `query()` (function wrapping pool) |

**Class guidelines (from existing guidance):**

- One class = one responsibility.
- Constructors inject dependencies (no `getConnection()` inside if it can be passed in).
- Avoid god objects and flag-driven behavior.
- Understand current behavior before refactoring; consider unit tests at boundaries.

**Function guidelines:**

- One function = one job (e.g. build a query, calculate a score, format a date).
- Prefer explicit parameters over reading globals or `req` inside service-layer functions.

---

## 3. Dependency Injection & Testability

**Goal:** Services and controllers should be testable without a real DB or Redis.

| Current | Recommended |
|--------|-------------|
| Services call `getConnection()`, `query()`, `getRedis()` directly | Inject `Database` (or pool) and `Redis | null` (or interface) in constructor |
| Controllers `import { query } from '../services/database'` | Controllers receive a service instance (or a factory) that already has DB/Redis injected |
| `getQueryTracker(req)` / `createBackgroundJobTracker()` inside services | Pass a `QueryTracker` (or null) as argument or via injected context |

**Minimal improvement (no big refactor):**

- For **new** services: accept `{ db?, redis?, queryTracker? }` in the constructor and use them instead of globals.
- For **existing** services: keep as-is until touched; when refactoring, introduce an interface (e.g. `IDatabase`) and inject it so tests can stub it.

---

## 4. Common Patterns & Anti-Patterns

### 4.1 Controllers

**Anti-pattern:** Controllers with inline SQL and business logic.

**Pattern:** Thin controllers that delegate to services:
- Parse input (query, body)
- Call one or more services
- Shape response
- No SQL, no business rules

**Example:**
```typescript
// ❌ Bad: Controller doing too much
export async function getUsers(req: Request, res: Response) {
  const conn = await getConnection();
  const users = await conn.query('SELECT * FROM users WHERE ...');
  // business logic here
  res.json(users);
}

// ✅ Good: Controller delegates to service
export async function getUsers(req: Request, res: Response) {
  const users = await userService.getUsers(req.query);
  res.json(users);
}
```

### 4.2 Services

**Anti-pattern:** Services using global `getConnection()`, `query()`, `getRedis()`.

**Pattern:** Services with constructor injection:
- Accept dependencies in constructor
- Use injected interfaces (e.g., `IDatabase`, `IRedis`)
- Testable with mocks

**Example:**
```typescript
// ❌ Bad: Global dependency
class UserService {
  async getUser(id: number) {
    const conn = await getConnection();
    return conn.query('SELECT * FROM users WHERE id = ?', [id]);
  }
}

// ✅ Good: Dependency injection
class UserService {
  constructor(private db: IDatabase) {}
  async getUser(id: number) {
    return this.db.query('SELECT * FROM users WHERE id = ?', [id]);
  }
}
```

### 4.3 Models

**Pattern:** Models are optional. Use when you have clear domain entities with identity and behavior. Otherwise, services can own queries and DTOs.

**Anti-pattern:** Empty placeholder model classes. Either implement them or remove them.

### 4.4 Consistency

- **Naming:** Services that are classes: `XxxService`. Functions: `doSomething` or `buildSomething`. Controllers: `XxxController` (class) or `getXxx` / `postXxx` (function).
- **Exports:** Prefer **one class or one cohesive set of functions per file**. Avoid a file that exports both a class and many unrelated functions unless they are clearly grouped (e.g. class + factory).
- **Error handling:** Use middleware (`errorHandler`) for HTTP errors. Services throw domain or application errors; controllers or middleware map them to status codes.

---

## 5. Summary: Is the Structure Well-Defined?

| Aspect | Well-defined? | Note |
|--------|----------------|------|
| **Layers (routes → controllers → services)** | Partially | Layers exist; controllers often do too much (SQL, logic). |
| **When to use class vs function** | Partially | Mix of classes and functions without a single written rule; this doc defines it. |
| **Dependency injection** | No | Most code uses global `query()` / `getConnection()` / `getRedis()`; little constructor injection. |
| **Testability** | Partially | Some services are hard to unit-test without a real DB when using globals. |
| **Models** | Varies | Depends on project needs; some projects use models, others use services + DTOs. |
| **Controller style** | No | One controller is a class; others are procedural functions. |

**Where it can be improved:**

1. **Controllers:** Make them thin: parse input → call service → send response. Move SQL and business logic into services.
2. **Services:** Prefer constructor injection for DB/Redis/tracker so tests can stub them. Use classes for use-cases, functions for stateless helpers.
3. **Models:** Either remove the placeholder or introduce real domain/data-access methods and use them from services.
4. **Single style per area:** Decide “controllers as classes with DI” vs “controllers as functions” and apply consistently; document here.
5. **Naming and one-responsibility:** Avoid god controllers; split large controllers into smaller services (e.g. `UserService`, `OrderService`, `PaymentService`) and keep files focused.

---

## 6. Related Docs

- **AI_SLOP_FIX.md** (OOP Guidance): Use classes for domain entities and use-cases; one responsibility; constructors inject dependencies; consider unit tests before refactoring.
- **Infrastructure.md** (plans): Circuit breaker, Redis, resilience—align new infrastructure with the DI approach above so it can be injected into services.
