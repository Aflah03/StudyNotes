# SQL Interview Reference — Backend Engineer Edition

A focused refresher for an intermediate engineer leveling up. Everything below uses one running schema so examples build on each other. Skim the parts you know cold; slow down on **Indexing**, **Transactions & Isolation**, **Query Plans**, **Window Functions**, and **Concurrency Patterns** — those are where backend interviews separate people.

Dialect note: examples are written for **PostgreSQL** (the most common interview default). Where MySQL/SQL Server differ meaningfully, it's called out.

---

## The running schema

```sql
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    email       TEXT NOT NULL UNIQUE,
    country     TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE products (
    id       BIGSERIAL PRIMARY KEY,
    name     TEXT NOT NULL,
    category TEXT NOT NULL,
    price    NUMERIC(10,2) NOT NULL,
    stock    INT NOT NULL DEFAULT 0
);

CREATE TABLE orders (
    id         BIGSERIAL PRIMARY KEY,
    user_id    BIGINT NOT NULL REFERENCES users(id),
    status     TEXT NOT NULL,          -- 'pending' | 'paid' | 'shipped' | 'cancelled'
    total      NUMERIC(12,2) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE order_items (
    id         BIGSERIAL PRIMARY KEY,
    order_id   BIGINT NOT NULL REFERENCES orders(id),
    product_id BIGINT NOT NULL REFERENCES products(id),
    quantity   INT NOT NULL,
    unit_price NUMERIC(10,2) NOT NULL
);
```

---

## 1. Query execution order (the mental model that fixes most bugs)

SQL is written in one order but **evaluated in another**. Knowing this explains why you can't use a `SELECT` alias in `WHERE`, but you can in `ORDER BY`.

Written order: `SELECT … FROM … JOIN … WHERE … GROUP BY … HAVING … ORDER BY … LIMIT`

Logical evaluation order:

1. `FROM` / `JOIN` — build the working set of rows
2. `WHERE` — filter individual rows (no aggregates yet)
3. `GROUP BY` — collapse into groups
4. `HAVING` — filter groups (aggregates allowed)
5. `SELECT` — compute expressions and aliases
6. `DISTINCT`
7. `ORDER BY` — aliases now visible
8. `LIMIT` / `OFFSET`

**Interview payoff:** "Why can't I filter on `COUNT(*)` in `WHERE`?" → because `WHERE` runs before grouping. Use `HAVING`.

---

## 2. Joins

```sql
-- INNER: only matching rows on both sides
SELECT u.name, o.total
FROM users u
JOIN orders o ON o.user_id = u.id;

-- LEFT: all users, orders where they exist, NULLs otherwise
SELECT u.name, o.id AS order_id
FROM users u
LEFT JOIN orders o ON o.user_id = u.id;
```

Types at a glance:

| Join | Keeps | Common use |
|------|-------|-----------|
| `INNER` | rows matching on both sides | the default "give me related data" |
| `LEFT` | all left rows + matches | "users and their orders, even users with none" |
| `RIGHT` | all right rows + matches | rare; usually rewrite as LEFT |
| `FULL OUTER` | everything, matched where possible | reconciling two sources |
| `CROSS` | cartesian product | generating combinations, calendars |
| `SELF` | table joined to itself | hierarchies, comparisons within a table |

**Anti-join** — "users with no orders":

```sql
SELECT u.*
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.id IS NULL;
-- or, often clearer and index-friendly:
SELECT u.* FROM users u
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

**Semi-join** — "users who have at least one paid order" (without duplicating users per order):

```sql
SELECT u.*
FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id AND o.status = 'paid'
);
```

**Gotcha — filtering a LEFT JOIN in WHERE turns it into an INNER JOIN.** Putting `o.status = 'paid'` in `WHERE` drops users whose only matching row is NULL. Put the condition in the `ON` clause instead if you want to keep unmatched left rows:

```sql
-- keeps all users; only pulls their 'paid' orders
SELECT u.name, o.id
FROM users u
LEFT JOIN orders o ON o.user_id = u.id AND o.status = 'paid';
```

---

## 3. Aggregation, GROUP BY, HAVING

```sql
-- Revenue per country, only countries above 10k
SELECT u.country,
       COUNT(*)        AS order_count,
       SUM(o.total)    AS revenue,
       AVG(o.total)    AS avg_order
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE o.status = 'paid'
GROUP BY u.country
HAVING SUM(o.total) > 10000
ORDER BY revenue DESC;
```

Key facts interviewers probe:

- Every non-aggregated column in `SELECT` must appear in `GROUP BY` (Postgres/standard). MySQL historically allowed sloppy grouping — mention `ONLY_FULL_GROUP_BY`.
- `COUNT(*)` counts rows; `COUNT(col)` skips NULLs; `COUNT(DISTINCT col)` dedups.
- `WHERE` filters rows before grouping; `HAVING` filters after.

**ROLLUP / GROUPING SETS** — subtotals and grand totals in one pass:

```sql
SELECT u.country, o.status, SUM(o.total)
FROM orders o JOIN users u ON u.id = o.user_id
GROUP BY ROLLUP (u.country, o.status);
```

**FILTER clause** (Postgres) — conditional aggregation without `CASE`:

```sql
SELECT
  COUNT(*) FILTER (WHERE status = 'paid')      AS paid_count,
  COUNT(*) FILTER (WHERE status = 'cancelled') AS cancelled_count
FROM orders;
-- Portable equivalent: SUM(CASE WHEN status='paid' THEN 1 ELSE 0 END)
```

---

## 4. Subqueries: IN vs EXISTS vs JOIN

```sql
-- Scalar subquery: one value
SELECT name, (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) AS n_orders
FROM users u;

-- Correlated: inner query references outer row (runs per row conceptually)
SELECT * FROM orders o
WHERE o.total > (SELECT AVG(total) FROM orders WHERE user_id = o.user_id);
```

Interview-critical distinctions:

- **`IN` vs `EXISTS`:** functionally similar for existence checks. `EXISTS` short-circuits on the first match and is usually safer with large subqueries. Modern optimizers often plan them identically, but say "I'd check the plan."
- **`NOT IN` trap:** if the subquery returns *any* NULL, `NOT IN` yields no rows (because `x NOT IN (1, NULL)` is `UNKNOWN`). Prefer `NOT EXISTS`, which handles NULLs correctly.
- **Subquery vs JOIN:** a JOIN can multiply rows (one order → many items). `EXISTS` gives you filtering without fan-out.

---

## 5. CTEs (WITH) and recursion

CTEs make complex queries readable. In modern Postgres they're not automatically an optimization fence (that changed in v12 — pre-12 they were materialized).

```sql
WITH paid AS (
    SELECT user_id, SUM(total) AS spent
    FROM orders WHERE status = 'paid'
    GROUP BY user_id
)
SELECT u.name, paid.spent
FROM users u JOIN paid ON paid.user_id = u.id
WHERE paid.spent > 500;
```

**Recursive CTE** — walk a hierarchy (e.g., an employee `manager_id` tree):

```sql
WITH RECURSIVE org AS (
    SELECT id, name, manager_id, 1 AS depth
    FROM employees WHERE manager_id IS NULL      -- anchor
    UNION ALL
    SELECT e.id, e.name, e.manager_id, org.depth + 1
    FROM employees e JOIN org ON e.manager_id = org.id  -- recursive step
)
SELECT * FROM org ORDER BY depth;
```

---

## 6. Window functions (the biggest intermediate → strong jump)

Windows compute across a set of rows **without collapsing them** — unlike `GROUP BY`, you keep every row.

```sql
SELECT
  id, user_id, total, created_at,
  ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at)      AS nth_order,
  SUM(total)   OVER (PARTITION BY user_id ORDER BY created_at)      AS running_total,
  LAG(total)   OVER (PARTITION BY user_id ORDER BY created_at)      AS prev_total,
  RANK()       OVER (ORDER BY total DESC)                            AS overall_rank
FROM orders;
```

The three ranking functions — know the difference cold:

- `ROW_NUMBER()` — 1,2,3,4 — always unique.
- `RANK()` — 1,2,2,4 — ties share a rank, then it skips.
- `DENSE_RANK()` — 1,2,2,3 — ties share, no gap.

**Classic interview question — "most recent order per user" (top-N-per-group):**

```sql
SELECT * FROM (
  SELECT o.*,
         ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
  FROM orders o
) t
WHERE rn = 1;
```

**Frames** — `ROWS BETWEEN` controls the window's extent (e.g., 7-day moving average):

```sql
AVG(total) OVER (
  ORDER BY created_at
  ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
)
```

`ROWS` counts physical rows; `RANGE` counts logical peers with the same `ORDER BY` value — a subtle but favorite gotcha.

---

## 7. Schema design & normalization

Be able to name the normal forms and, more importantly, argue about when to break them.

- **1NF** — atomic values, no repeating groups (no comma-separated lists in a column).
- **2NF** — 1NF + no partial dependency on part of a composite key.
- **3NF** — 2NF + no transitive dependency (non-key attributes depend only on the key).
- **BCNF** — a stricter 3NF.

Constraints that carry design intent:

```sql
PRIMARY KEY            -- unique + not null; the row's identity
FOREIGN KEY … REFERENCES  -- referential integrity
UNIQUE                 -- alternate keys (email)
NOT NULL, CHECK (…)    -- domain rules, e.g. CHECK (quantity > 0)
DEFAULT
```

**Denormalization** is a deliberate trade: duplicate data to avoid expensive joins on hot read paths, accepting write complexity and consistency risk. Good answer: "Normalize by default for integrity; denormalize specific read-heavy paths with measurements, and keep the copy in sync via triggers, application logic, or a materialized view."

---

## 8. Indexing (study this hardest)

An index is a separate sorted structure (usually a **B-tree**) that lets the engine find rows without scanning the whole table.

```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE UNIQUE INDEX idx_users_email ON users(email);
```

**Composite indexes and the leftmost-prefix rule.** An index on `(user_id, status, created_at)` can serve queries filtering on `user_id`, or `user_id + status`, or all three — but *not* one that filters on `status` alone. Order columns by: equality filters first, then range/sort columns.

```sql
CREATE INDEX idx_orders_lookup ON orders(user_id, status, created_at);
-- Uses index:   WHERE user_id = 5 AND status = 'paid' ORDER BY created_at
-- Can't use it: WHERE status = 'paid'   (skips the leading column)
```

**Covering index** — includes every column the query needs, so the engine answers from the index alone (an "index-only scan"), never touching the table:

```sql
CREATE INDEX idx_cover ON orders(user_id) INCLUDE (total, status);  -- Postgres
```

**When indexes hurt / don't help:**

- Every index slows `INSERT`/`UPDATE`/`DELETE` (the index must be maintained) and costs storage.
- Low-cardinality columns (e.g., a boolean) rarely benefit from a plain B-tree.
- **Wrapping an indexed column in a function kills index use:** `WHERE lower(email) = …` won't use an index on `email`. Fix with an **expression index**: `CREATE INDEX ON users(lower(email))`.
- Leading wildcard `LIKE '%foo'` can't use a standard B-tree; `LIKE 'foo%'` can.
- On tiny tables a full scan is cheaper than an index lookup — the planner may (correctly) ignore your index.

Other index types worth naming: **partial** (`WHERE status = 'pending'` — index only the hot subset), **hash**, **GIN/GiST** (full-text, JSONB, arrays).

---

## 9. Reading query plans (EXPLAIN)

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 42 AND status = 'paid';
```

`EXPLAIN` shows the planned strategy; `ANALYZE` actually runs it and reports real timings and row counts. What to look for:

- **Seq Scan** on a big table with a selective filter → probably a missing index.
- **Index Scan / Index Only Scan** → good.
- **Nested Loop vs Hash Join vs Merge Join** — nested loops are great for small row counts, terrible for large unindexed ones.
- **Rows estimate vs actual** wildly off → stale statistics; run `ANALYZE` (update stats) — a strong thing to mention.
- Watch for hidden **Sort** and **Materialize** steps that spill to disk.

Say this in an interview: "I'd never guess at performance — I'd `EXPLAIN ANALYZE`, look at where time and rows concentrate, and index or rewrite from there."

---

## 10. Transactions, ACID, and isolation levels

**ACID:** Atomicity (all-or-nothing), Consistency (constraints hold), Isolation (concurrent txns don't corrupt each other), Durability (committed data survives crashes).

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;   -- or ROLLBACK;
```

**Isolation levels** and the anomalies they permit — a top-tier interview topic:

| Level | Dirty read | Non-repeatable read | Phantom read |
|-------|:---:|:---:|:---:|
| READ UNCOMMITTED | possible | possible | possible |
| READ COMMITTED | no | possible | possible |
| REPEATABLE READ | no | no | possible* |
| SERIALIZABLE | no | no | no |

- **Dirty read** — you read another txn's uncommitted change.
- **Non-repeatable read** — you read a row twice and get different values (someone updated & committed in between).
- **Phantom read** — you re-run a range query and new rows appear.
- *Postgres's REPEATABLE READ (snapshot isolation) actually prevents phantoms too; the SQL standard doesn't require that. Postgres default is **READ COMMITTED**.

---

## 11. Concurrency patterns

**Pessimistic locking — `SELECT … FOR UPDATE`.** Lock rows now so no one else can modify them until you commit. Classic for "decrement stock only if available":

```sql
BEGIN;
SELECT stock FROM products WHERE id = 10 FOR UPDATE;   -- row locked
UPDATE products SET stock = stock - 1 WHERE id = 10;
COMMIT;
```

**Optimistic locking — version column.** No lock held; detect conflicts at write time. Great for low-contention, web-request-length work:

```sql
UPDATE products
SET stock = stock - 1, version = version + 1
WHERE id = 10 AND version = 7;
-- If 0 rows updated, someone else changed it first → retry.
```

**Deadlocks** happen when two transactions lock resources in opposite order and each waits on the other. The database detects the cycle and kills one (you get a deadlock error). Prevention: **always acquire locks in a consistent order**, keep transactions short, and be ready to retry the aborted one.

---

## 12. Everyday backend patterns

**UPSERT** — insert or update on conflict:

```sql
INSERT INTO products (id, name, category, price, stock)
VALUES (10, 'Widget', 'tools', 9.99, 100)
ON CONFLICT (id)
DO UPDATE SET price = EXCLUDED.price, stock = EXCLUDED.stock;
-- MySQL: INSERT … ON DUPLICATE KEY UPDATE …
-- SQL Server: MERGE
```

**Pagination — offset vs keyset.** `OFFSET` is simple but degrades badly on deep pages (the engine still scans and discards all skipped rows) and can skip/duplicate rows if data changes mid-scroll.

```sql
-- Offset (fine for shallow pages)
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 10000;  -- slow deep

-- Keyset / "seek" (scales; needs an index on the sort key)
SELECT * FROM orders
WHERE created_at < :last_seen_created_at
ORDER BY created_at DESC
LIMIT 20;
```

**Soft deletes** — keep a `deleted_at TIMESTAMPTZ`; filter `WHERE deleted_at IS NULL`. Pair with a **partial unique index** so deleted rows don't block re-inserts:

```sql
CREATE UNIQUE INDEX ON users(email) WHERE deleted_at IS NULL;
```

**The N+1 problem** — the single most common ORM performance bug. Fetching a list, then one query per row for its children (1 + N queries). Fix by fetching in one round trip:

```sql
-- Instead of: SELECT * FROM orders;  then per-order SELECT * FROM order_items WHERE order_id = ?
SELECT o.*, i.*
FROM orders o
JOIN order_items i ON i.order_id = o.id
WHERE o.user_id = 42;
-- or a single WHERE order_id IN (…) batch. In ORMs: eager loading / JOIN fetch.
```

---

## 13. NULL handling gotchas

- NULL means "unknown," not zero or empty string.
- Any comparison with NULL yields `UNKNOWN`, not true/false: `NULL = NULL` is **not** true. Use `IS NULL` / `IS NOT NULL`.
- `COUNT(col)` ignores NULLs; `COUNT(*)` doesn't.
- `SUM`/`AVG` skip NULLs — but `AVG` divides by the non-null count, which may surprise you.
- `COALESCE(a, b, c)` returns the first non-null; `NULLIF(a, b)` returns NULL when `a = b` (handy to avoid divide-by-zero: `x / NULLIF(y, 0)`).
- `NOT IN (subquery with a NULL)` → empty result (see §4).

---

## 14. Likely interview questions + how to frame answers

1. **"What's the difference between `WHERE` and `HAVING`?"** — `WHERE` filters rows before grouping; `HAVING` filters groups after aggregation. Tie it to execution order.
2. **"`DELETE` vs `TRUNCATE` vs `DROP`?"** — `DELETE` removes rows (logged, per-row, can be filtered, fires triggers, rollback-able); `TRUNCATE` empties the whole table fast (minimal logging, resets identity, usually can't be filtered); `DROP` removes the table itself.
3. **"How would you find duplicate emails?"**
   ```sql
   SELECT email, COUNT(*) FROM users GROUP BY email HAVING COUNT(*) > 1;
   ```
4. **"Second-highest salary?"** — window function is the clean answer:
   ```sql
   SELECT DISTINCT salary FROM (
     SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS r FROM employees
   ) t WHERE r = 2;
   ```
5. **"A query got slow in production — walk me through it."** — Reproduce → `EXPLAIN ANALYZE` → find the seq scan / bad join / row-estimate miss → add or fix an index, rewrite the query, or update statistics → re-measure. Never optimize blind.
6. **"`UNION` vs `UNION ALL`?"** — both stack result sets; `UNION` removes duplicates (extra sort/hash cost), `UNION ALL` keeps everything and is faster. Use `ALL` unless you truly need dedup.
7. **"How do indexes work and when do they hurt?"** — §8. Mention write cost, storage, function-wrapping, and low cardinality.
8. **"Explain isolation levels."** — §10. Name the three anomalies and the default (READ COMMITTED).
9. **"How do you prevent two requests from double-spending stock?"** — `SELECT … FOR UPDATE` or optimistic version check (§11).
10. **"Normalize vs denormalize?"** — §7. Default to normalized; denormalize measured hot paths.

---

## 15. One-page gotcha cheat sheet

- `WHERE` can't see `SELECT` aliases; `ORDER BY` can.
- Filtering a LEFT JOIN table in `WHERE` silently makes it an INNER JOIN — use the `ON` clause.
- `NOT IN` + a NULL = zero rows. Prefer `NOT EXISTS`.
- `NULL = NULL` is not true. Use `IS NULL`.
- `COUNT(col)` skips NULLs; `COUNT(*)` doesn't.
- `func(indexed_col)` disables the index — use an expression index.
- Leading `%` in `LIKE` disables a B-tree index.
- Composite indexes obey the leftmost-prefix rule.
- Deep `OFFSET` is slow — use keyset pagination.
- `UNION` dedups (costly); `UNION ALL` doesn't.
- `ROW_NUMBER` (unique) vs `RANK` (gaps on ties) vs `DENSE_RANK` (no gaps).
- Always acquire locks in a consistent order to avoid deadlocks.
- `EXPLAIN ANALYZE` before you optimize — measure, don't guess.

---

### How to use this before the interview

Read it once end-to-end, then write out §6 (window functions), §8 (indexing), and §10–11 (transactions/concurrency) from memory — those are the topics where interviewers push for depth. For the coding round, practice §14's #4, #5, and the top-N-per-group pattern in §6 until they're automatic. Good luck.
