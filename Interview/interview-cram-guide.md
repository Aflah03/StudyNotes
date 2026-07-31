# Interview Cram Guide — Technical Round

**How this round flows (based on your last one):**
1. Walk through your **Brownfield** work → they probe the *concepts* behind your bugs.
2. **Data structures** fundamentals.
3. **OOAD / system design** — they give you a use case (the car) and you design it live.
4. **LLD** deep-dive + software-engineering vocabulary.
5. **Greenfield** — hands-on: use AI to produce a real design in ~30 min (REST API + DB schema + UML), then defend every decision.

**The meta-skill they're testing:** *Can you understand architecture and flow even when the tech stack is new?* Lean into this. When they hear "I hadn't used SQLAlchemy, but I understood the request → route → ORM → DB → response flow," that's the exact signal they want.

---

## Part 1 — Brownfield concepts (the bug probes)

### HTTP status codes
Learn the families, then the specific ones.

| Family | Meaning | Must-know codes |
|---|---|---|
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirect | 301 Moved Permanently, 304 Not Modified |
| 4xx | **Client** error (you sent something wrong) | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 405 Method Not Allowed, 409 Conflict, 422 Unprocessable Entity, 429 Too Many Requests |
| 5xx | **Server** error (server broke) | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout |

**The classic trap:** 401 vs 403.
- **401 Unauthorized** = "I don't know who you are" (authentication failed — missing/expired/invalid token).
- **403 Forbidden** = "I know who you are, but you're not allowed" (authorization failed — valid token, wrong permissions).

### Authentication & bearer tokens
- **AuthN (authentication)** = *who you are*. **AuthZ (authorization)** = *what you're allowed to do*.
- **Bearer token**: sent in the header `Authorization: Bearer <token>`. "Bearer" means *whoever holds it can use it* — so it must travel over HTTPS and expire quickly.
- **JWT (JSON Web Token)**: three base64url parts — `header.payload.signature`. It's **stateless**: the server verifies the signature without a DB lookup. Payload holds *claims* like `sub` (subject/user), `exp` (expiry), `iat` (issued-at).
- **Session vs token auth**: sessions are stored server-side (stateful, cookie holds a session ID); tokens are self-contained (stateless).
- **OAuth 2.0** (one-liner): delegated authorization; you get an **access token** (short-lived) + a **refresh token** (used to get a new access token when it expires).
- **Common token bugs → what causes them:** expired token → 401; malformed/missing `Authorization` header → 401; valid token but missing scope/role → 403; JWT `exp` failing due to server clock skew.

### ORM & SQLAlchemy (your "new tech stack" example)
- **ORM = Object-Relational Mapping.** Maps DB **tables ↔ classes**, **rows ↔ objects**. You write Python, it generates SQL.
- **Why use one:** productivity, database-agnostic code, safe parameterized queries (helps prevent SQL injection), easier maintenance.
- **SQLAlchemy has two layers:** *Core* (SQL expression language) and *ORM* (declarative model classes). Most app code uses the ORM.
- **The flow to describe:** define models (classes with `__tablename__` and `Column`s) → create an **engine** (the DB connection) → open a **Session** (a unit of work) → add/query objects → **commit** (persist to DB). Rollback undoes a transaction.
- **Gotchas worth naming:** the **N+1 query problem** (loading related rows one-by-one in a loop instead of eager-loading), forgetting to commit, and detached objects after a session closes.
- **Your talking point:** "I didn't know SQLAlchemy's syntax, but the *flow* is the same as any ORM, so I mapped my mental model onto it." That's the transferable-skill answer.

---

## Part 2 — Data structures (fundamentals)

| Structure | Access | Search | Insert / Delete | When to use |
|---|---|---|---|---|
| Array / List | O(1) | O(n) | O(n) (end: O(1) amortized) | Indexed data, iteration |
| Linked List | O(n) | O(n) | O(1) at a known node | Frequent insert/delete, no random access |
| Stack (LIFO) | — | — | push/pop O(1) | Undo, recursion, backtracking |
| Queue (FIFO) | — | — | enqueue/dequeue O(1) | Scheduling, BFS |
| Hash Map / Dict | — | O(1) avg | O(1) avg (O(n) worst) | Fast lookups by key, counting, dedup |
| Binary Search Tree | O(log n) balanced | O(log n) | O(log n) balanced | Ordered data, range queries |
| Heap | peek O(1) | — | insert/extract O(log n) | Priority queue, top-k |
| Graph | — | BFS/DFS O(V+E) | — | Networks, dependencies, paths |

**Big-O ordering (best → worst):** O(1) < O(log n) < O(n) < O(n log n) < O(n²) < O(2ⁿ)

**Patterns interviewers love to hear:** two pointers, sliding window, hash map for O(1) lookups, BFS/DFS for graphs/trees, recursion + memoization.

---

## Part 3 — OOAD / live system design (the car)

### OOP pillars
- **Encapsulation** — bundle data + behavior, hide internals behind methods.
- **Abstraction** — expose only the essentials, hide complexity.
- **Inheritance** — a child class reuses/extends a parent (*is-a*).
- **Polymorphism** — one interface, many behaviors (`start()` differs for `ElectricCar` vs `GasCar`).

### SOLID principles
- **S** – Single Responsibility: a class has one reason to change.
- **O** – Open/Closed: open to extension, closed to modification.
- **L** – Liskov Substitution: a subtype must be usable anywhere its base type is.
- **I** – Interface Segregation: many small focused interfaces > one fat one.
- **D** – Dependency Inversion: depend on abstractions, not concrete classes.

### A repeatable OOAD method (say this out loud as you go)
1. **Clarify scope** — ask questions, pin down *functional* and *non-functional* requirements. (Interviewers reward this heavily.)
2. **Identify actors & use cases** — who uses it, to do what.
3. **Find the objects** — *nouns → classes*, *verbs → methods*.
4. **Define attributes & behaviors** for each class.
5. **Define relationships** (below).
6. **Apply a design pattern** where it fits.
7. **Extensibility & edge cases** — how does this grow, where does it break.

### Relationships (name these precisely — it signals seniority)
- **Association** — *uses-a* (loose link).
- **Aggregation** — *has-a*, independent lifecycles (a `Team` has `Players`; players outlive the team).
- **Composition** — *has-a*, dependent lifecycles (a `Car` has an `Engine`; the engine dies with the car).
- **Inheritance** — *is-a* (`ElectricCar` is a `Car`).

### Worked example: design a Car
- **Superclass:** `Vehicle` → subclasses `Car`, `Truck`, `Motorcycle` (inheritance).
- **Composition:** `Car` **has-a** `Engine`, `Transmission`, `FuelTank`/`Battery`, and 4 `Wheel`s.
- **Interfaces:** `Drivable` (`start()`, `accelerate()`, `brake()`), `Refuelable` / `Chargeable`.
- **Polymorphism:** `start()` is implemented differently by `ElectricCar` vs `GasCar`.
- **Actors:** `Driver`, `Mechanic`. **Use cases:** start, drive, brake, refuel, service.

### A few design patterns worth naming
- **Factory** — create objects without hard-coding the class.
- **Singleton** — exactly one instance (e.g., a config or connection pool).
- **Strategy** — swap interchangeable algorithms at runtime.
- **Observer** — publish/subscribe (event listeners).
- **Adapter** — make an incompatible interface fit.
- **Decorator** — add behavior to an object dynamically.

---

## Part 4 — LLD deep-dive + SE vocabulary

### HLD vs LLD
- **HLD (High-Level Design)** — bird's-eye: architecture, components, data flow, tech/DB choices, how modules talk. Diagrams: system architecture, component, ER.
- **LLD (Low-Level Design)** — zoomed-in: individual classes, methods, data structures, algorithms. Diagrams: class diagrams, sequence diagrams.

### SE terms that make you sound senior
- **Regression / regression risk** — a change that breaks something that previously worked; regression risk = the *likelihood* a change introduces such a bug. Mitigated by **regression testing** (re-running old tests after changes).
- **Coupling** — how dependent modules are on each other. You want it **low**.
- **Cohesion** — how focused/single-purpose a module is. You want it **high**.
- **Idempotency** — running the same operation repeatedly yields the same result (GET, PUT, DELETE are idempotent; POST usually isn't).
- **Race condition** — a bug where the outcome depends on the timing of concurrent operations.
- **Technical debt** — shortcuts taken now that cost more to fix later.
- **Backward compatibility** — new versions still work with older clients.
- **Scalability** — *vertical* (bigger machine) vs *horizontal* (more machines).
- **Statelessness** — the server keeps no client state between requests (enables horizontal scaling).
- **ACID** (DB transactions) — Atomicity, Consistency, Isolation, Durability.
- **CAP theorem** — under a network partition you trade off Consistency vs Availability.

---

## Part 5 — Greenfield: the 30-min AI design task

**Mindset:** *AI is your draftsman; you are the architect.* They're watching whether you can direct AI, then critically **question its output** and defend the final design. (This is exactly what you did — draft with one tool, interrogate with another.)

### How to spend the 30 minutes
1. **~3 min — Restate requirements + assumptions.** Establish scope in writing so you can't be blindsided.
2. **~5 min — Prompt AI for a first draft:** REST endpoints, DB schema, and a class/UML sketch.
3. **~10 min — Interrogate it.** For every decision ask: *Why this DB? Why this structure? What breaks at 10× scale? What's the failure mode?* Feed those challenges back in.
4. **~9 min — Refine:** add error handling, auth, edge cases, non-functional needs.
5. **~3 min — Rehearse the defense.** Be ready to justify every choice and name the alternative you rejected.

### What they expect you to produce

**1. REST API construct**
- Resources are **plural nouns**: `/users`, `/orders/{id}/items`.
- Methods map to intent: `GET` (read), `POST` (create), `PUT` (full replace), `PATCH` (partial update), `DELETE`.
- Return the right status codes (see Part 1), a consistent error body, and use versioning (`/v1/...`).
- Include pagination (`?limit=&offset=` or cursor), filtering/sorting, and the `Authorization: Bearer` header.

Example:
```
GET    /v1/orders            → 200 list (paginated)
POST   /v1/orders            → 201 created
GET    /v1/orders/{id}       → 200 / 404
PATCH  /v1/orders/{id}       → 200 / 400 / 404
DELETE /v1/orders/{id}       → 204 / 404
```

**2. DB schema**
- Entities → tables; each table has a **primary key**.
- Relationships: **1:1**, **1:many** (foreign key on the "many" side), **many:many** (a **junction/join table**).
- **Normalize** (1NF→2NF→3NF) to remove redundancy; **denormalize** deliberately only for read performance.
- Add **indexes** on frequently-queried columns (faster reads, slightly slower writes).
- Include constraints (`NOT NULL`, `UNIQUE`), sensible types, and `created_at` / `updated_at` timestamps.

**3. UML / HLD diagram**
- **Class diagram** — classes with attributes/methods and relationships. Notation: inheritance = hollow-triangle arrow; composition = filled diamond; aggregation = hollow diamond; association = plain line; dependency = dashed arrow.
- **Sequence diagram** — the order of interactions between actors/objects over time (great for showing a request flow).
- **ER diagram** — entities and their relationships (for the DB).

---

## Part 6 — Edge-case checklist (apply to EVERY design)

They said "always consider edge cases." Run this list out loud:
- **Empty / null / missing** input.
- **Invalid / malformed** input.
- **Boundary values** — 0, negative, max, overflow.
- **Duplicates & retries** — is the operation idempotent?
- **Concurrency** — two writes at once, race conditions.
- **Scale** — huge inputs → pagination, rate limiting (429).
- **Auth failures** — expired token (401), wrong permission (403).
- **Network / downstream failures** — timeouts (504), retries, circuit breakers.
- **Partial failure** — transaction rollback so you don't leave bad state.

---

## Night-before priority order (if time is short)
1. **401 vs 403, and the status-code table** — cheap points, very likely asked.
2. **The ORM flow + your SQLAlchemy story** — rehearse the "understood the flow" narrative.
3. **The OOAD method (Part 3) + the 4 relationships** — this carries the live design.
4. **HLD vs LLD + the SE vocabulary (Part 4)** — regression risk, coupling/cohesion, idempotency.
5. **The 30-min plan + edge-case checklist** — so you have a script the moment they hand you paper.

**One habit that scores across the whole round:** narrate your reasoning and the trade-offs, and ask a clarifying question before you design. They're evaluating *how you think*, not just the final artifact.
