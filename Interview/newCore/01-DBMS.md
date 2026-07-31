# DBMS — Technical Interview Prep Guide

> Goal: Walk into any DBMS round and be able to *reason*, not just recite. Every section below is written the way you'd want to say it out loud to an interviewer.

---

## 0. How DBMS gets tested in interviews

Three flavors:
1. **Conceptual rapid-fire** — "What is a transaction? What are ACID properties? Difference between clustered and non-clustered index?"
2. **SQL problem-solving** — "Write a query to find the 2nd highest salary." "Find employees earning more than their manager."
3. **Design reasoning** — "You have a table with 100M rows and this query is slow. What do you do?" "Would you use SQL or NoSQL here, and why?"

The people who pass sound like they've *operated* a database, not memorized a textbook. So for each concept, know the **what**, the **why it exists**, and **when it bites you in production**.

---

## 1. The mental model: what a database actually is

A DBMS is software that sits between your application and raw disk, and it guarantees things raw files can't: concurrent access without corruption, crash recovery, fast lookups, and query-ability.

**Relational model in one line:** Data is stored in tables (relations). Each row is a tuple, each column is an attribute. Relationships between tables are expressed through keys, not pointers.

---

## 2. Keys (know these cold — very common opener)

| Key | Definition | Example (Employee table) |
|-----|-----------|--------------------------|
| **Super key** | Any set of columns that uniquely identifies a row | `{emp_id}`, `{emp_id, name}`, `{email}` |
| **Candidate key** | A *minimal* super key (no redundant column) | `{emp_id}`, `{email}` |
| **Primary key** | The candidate key you *chose* to be the main identifier; unique + NOT NULL | `emp_id` |
| **Alternate key** | Candidate keys you didn't pick as primary | `email` |
| **Foreign key** | A column that references the primary key of another table | `dept_id` referencing `Department.dept_id` |
| **Composite key** | A primary key made of 2+ columns | `{order_id, product_id}` in an order-items table |

**Interview trap:** "Can a foreign key be NULL?" → Yes. A NULL foreign key means "no relationship yet" (e.g., an employee not yet assigned to a department). "Can a primary key be NULL?" → Never.

---

## 3. Normalization (the single most-asked DBMS topic)

**Why it exists:** to eliminate *redundancy* and *anomalies* (insert, update, delete anomalies). Storing the same fact in many places means updates can go inconsistent.

### The anomalies (say these to show you understand the *point*)
Imagine one giant table `StudentCourse(student_id, student_name, course_id, course_name, instructor)`:
- **Update anomaly:** Instructor changes → you must update every row for that course or data goes inconsistent.
- **Insert anomaly:** Can't add a new course until at least one student enrolls (course_name has no home otherwise).
- **Delete anomaly:** Last student drops a course → you lose the course's existence entirely.

### The normal forms

**1NF — Atomic values.** No repeating groups, no multi-valued cells.
- ❌ `phone_numbers = "9998887777, 8887776666"`
- ✅ Separate rows or a separate `PhoneNumbers` table.

**2NF — 1NF + no partial dependency.** Every non-key column depends on the *whole* composite key, not part of it.
- Table `Enrollment(student_id, course_id, course_name)`. Here `course_name` depends only on `course_id` (part of the key) → partial dependency → violates 2NF. Split `course_name` into a `Course` table.

**3NF — 2NF + no transitive dependency.** Non-key columns must not depend on other non-key columns.
- Table `Employee(emp_id, dept_id, dept_name)`. `dept_name` depends on `dept_id`, which depends on `emp_id` → transitive → violates 3NF. Move `dept_name` to a `Department` table.

**BCNF (Boyce-Codd, "3.5NF") — stricter 3NF.** For every functional dependency X → Y, X must be a super key.
- Handles edge cases where a table has multiple overlapping candidate keys. Rare in interviews but know the name and that it's stricter than 3NF.

**One-liner to memorize (Bill Kent's version):** *"Every non-key attribute must depend on the key, the whole key, and nothing but the key — so help me Codd."*
- "the key" → 1NF, "the whole key" → 2NF, "nothing but the key" → 3NF.

### Denormalization (the sophisticated follow-up)
Normalization reduces redundancy but *adds joins*, which cost read performance. In read-heavy systems (analytics, dashboards, feeds) you deliberately **denormalize** — duplicate data to avoid joins.
- **Example:** A `Posts` table storing `author_name` directly instead of joining to `Users` every time the feed loads. Trade-off: faster reads, but you must update `author_name` in many rows if a user renames.
- **Say this:** "Normalize for correctness by default, denormalize deliberately for read performance when profiling shows join cost is the bottleneck."

---

## 4. ACID properties (guaranteed to be asked)

A **transaction** is a unit of work that must happen completely or not at all. The classic example: transferring ₹1000 from A to B = debit A **and** credit B. Both, or neither.

| Property | Meaning | What breaks without it |
|----------|---------|------------------------|
| **Atomicity** | All operations succeed or all roll back | Money debited from A but never credited to B |
| **Consistency** | Transaction moves DB from one valid state to another (constraints hold) | Balance goes negative, foreign key points to nothing |
| **Isolation** | Concurrent transactions don't interfere; result = as if run one at a time | Two people withdraw simultaneously, both see old balance |
| **Durability** | Once committed, survives crashes/power loss | Server crashes after "transfer complete", money vanishes |

**How they're implemented (bonus points):**
- Atomicity + Durability → **write-ahead log (WAL)**. Changes are logged to disk *before* being applied, so on crash the DB replays (redo) or reverts (undo).
- Isolation → **locking** or **MVCC** (below).

---

## 5. Concurrency, isolation levels, and the read phenomena

When many transactions run at once, three "bad reads" can happen. Isolation levels are defined by which ones they prevent.

**The three phenomena:**
- **Dirty read:** You read data another transaction wrote but hasn't committed (it might roll back).
- **Non-repeatable read:** You read a row twice in one transaction and get different values (someone updated it in between).
- **Phantom read:** You run the same range query twice and get different *rows* (someone inserted/deleted matching rows).

**The four isolation levels (SQL standard):**

| Level | Dirty read | Non-repeatable | Phantom |
|-------|-----------|----------------|---------|
| Read Uncommitted | ❌ possible | ❌ possible | ❌ possible |
| Read Committed | ✅ prevented | ❌ possible | ❌ possible |
| Repeatable Read | ✅ | ✅ | ❌ possible |
| Serializable | ✅ | ✅ | ✅ (fully isolated) |

Higher isolation = more correctness, less concurrency/throughput. **Read Committed** is the default in PostgreSQL/Oracle; **Repeatable Read** is MySQL InnoDB's default.

**MVCC (Multi-Version Concurrency Control) — modern answer:** Instead of readers blocking writers, the DB keeps multiple versions of a row. Readers see a consistent snapshot; writers create new versions. This is why "readers don't block writers and writers don't block readers" in Postgres. Mentioning MVCC signals real depth.

---

## 6. Locks and deadlocks

- **Shared lock (S):** for reading; multiple transactions can hold it.
- **Exclusive lock (X):** for writing; only one holder, blocks all others.

**Deadlock:** T1 holds lock on row A and wants row B; T2 holds row B and wants row A. Neither proceeds. Same **four Coffman conditions** as OS (mutual exclusion, hold-and-wait, no preemption, circular wait).

**How DBs handle it:** deadlock detection (build a wait-for graph, find a cycle, kill the "victim" transaction and roll it back). The app should catch the error and retry.

---

## 7. Indexing (the highest-leverage practical topic)

**Analogy that always lands:** An index is the index at the back of a textbook. Without it, finding "photosynthesis" means reading every page (full table scan, O(n)). With it, you jump to the page (O(log n)).

**Data structure:** Most indexes are **B+ trees** — balanced, high fan-out, so even billions of rows are ~3–4 levels deep. Leaf nodes are linked → great for range queries (`WHERE age BETWEEN 20 AND 30`). Hash indexes give O(1) equality lookups but *can't* do ranges.

### Clustered vs non-clustered (extremely common)
- **Clustered index:** determines the *physical order* of rows on disk. One per table (data can only be sorted one way). The primary key is usually clustered. Looking up by it is fastest — the data *is* the leaf.
- **Non-clustered index:** a separate structure holding indexed-column values + a pointer back to the actual row. You can have many. Requires an extra lookup ("bookmark lookup") to fetch the full row.

**Analogy:** Clustered = a phone book sorted by last name (the data itself is ordered). Non-clustered = a separate index card catalog that points you to a page.

### The cost of indexes (shows maturity)
Indexes speed up **reads** but slow down **writes** (every INSERT/UPDATE/DELETE must also update every index) and consume disk. So you don't index everything — you index columns used in `WHERE`, `JOIN`, `ORDER BY`, and foreign keys.

### Composite index + leftmost prefix rule
An index on `(last_name, first_name)` helps queries filtering by `last_name` alone, or `last_name + first_name`, but **not** `first_name` alone. Order matters — put the most selective / most-filtered column first.

**Production war-story answer:** "A slow query on a 100M-row table? First `EXPLAIN` the query to see if it's doing a full scan. Add a covering index on the filtered columns. A **covering index** includes all columns the query needs, so the DB never touches the table at all."

---

## 8. Joins (know the diagrams and when each is used)

Given `Employees` and `Departments`:

- **INNER JOIN** — only rows with matches in both tables. (Employees who have a department.)
- **LEFT (OUTER) JOIN** — all left rows + matched right rows; NULLs where no match. (All employees, even deptless ones.)
- **RIGHT JOIN** — mirror of LEFT.
- **FULL OUTER JOIN** — all rows from both, matched where possible. (Everyone and every department, matched up.)
- **CROSS JOIN** — Cartesian product, every combination. (Rarely intended; usually a bug when you forget the join condition.)
- **SELF JOIN** — a table joined to itself. Classic use: employees and their managers (both in the same table).

```sql
-- Self join: find each employee and their manager's name
SELECT e.name AS employee, m.name AS manager
FROM Employees e
LEFT JOIN Employees m ON e.manager_id = m.emp_id;
```

---

## 9. SQL vs NoSQL (the design-reasoning question)

| | SQL (Relational) | NoSQL |
|---|---|---|
| Schema | Fixed, defined upfront | Flexible / schema-less |
| Scaling | Vertical (bigger machine) primarily; harder to shard | Horizontal (add nodes) by design |
| Consistency | Strong (ACID) | Often eventual (BASE) — tunable |
| Best for | Structured data, complex queries, transactions (banking, ERP) | Huge scale, unstructured/varied data, high write throughput (feeds, logs, IoT) |
| Examples | PostgreSQL, MySQL, Oracle | MongoDB (document), Cassandra (wide-column), Redis (key-value), Neo4j (graph) |

**How to actually answer "SQL or NoSQL?":** Don't say one is better. Say: *"It depends on the access patterns. If I need multi-row transactions and complex joins with strong consistency — like a payments system — I'd use a relational DB. If I need to scale writes horizontally to millions/sec with flexible schema — like an activity feed or logging pipeline — a NoSQL store like Cassandra fits. Many real systems use both (polyglot persistence)."*

### CAP Theorem (comes with NoSQL discussion)
In a distributed system, during a **network Partition** you can guarantee at most one of **Consistency** or **Availability** — not both.
- **CP** (consistency over availability): refuse requests rather than serve stale data. E.g., HBase, MongoDB (default).
- **AP** (availability over consistency): always respond, may be stale, reconcile later. E.g., Cassandra, DynamoDB.
- **CA** isn't really achievable in a distributed system because partitions *will* happen; a single-node DB is "CA" trivially.

**BASE** (the NoSQL counterpart to ACID): **B**asically **A**vailable, **S**oft state, **E**ventual consistency.

---

## 10. Stored procedures, triggers, views (quick hits)

- **View:** a saved query that acts like a virtual table. Simplifies complex queries, adds a security layer (expose only some columns). A **materialized view** stores the result physically and refreshes periodically (faster reads, stale data risk).
- **Stored procedure:** precompiled SQL logic stored in the DB, called by name. Reduces network round-trips, centralizes logic.
- **Trigger:** SQL that fires *automatically* on an event (INSERT/UPDATE/DELETE). Use for audit logs, enforcing complex rules. Use sparingly — hidden logic is hard to debug.

---

## 11. Sharding, replication, partitioning (scaling vocabulary)

- **Replication:** copy the same data to multiple servers. **Read replicas** serve reads to offload the primary; the primary handles writes. Improves read scale + availability.
- **Partitioning:** split one table into pieces. **Horizontal** (by rows, e.g., users A–M on one partition, N–Z on another) vs **vertical** (by columns).
- **Sharding:** horizontal partitioning across *different servers*. Each shard holds a subset (e.g., by `user_id % N`). Scales writes but complicates joins and transactions across shards.

---

## 12. Common SQL interview questions (with solutions)

**Q1. Second-highest salary.**
```sql
-- Approach 1: subquery
SELECT MAX(salary) FROM Employees
WHERE salary < (SELECT MAX(salary) FROM Employees);

-- Approach 2: window function (also gives Nth easily)
SELECT DISTINCT salary FROM (
  SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
  FROM Employees
) t WHERE rnk = 2;
```
*Why DENSE_RANK over RANK:* handles ties correctly (two people at top salary still leaves the next distinct value as "2nd").

**Q2. Find duplicate emails.**
```sql
SELECT email, COUNT(*) AS cnt
FROM Users
GROUP BY email
HAVING COUNT(*) > 1;
```
*Key insight:* `WHERE` filters rows *before* grouping; `HAVING` filters *after* aggregation.

**Q3. Employees earning more than their manager.**
```sql
SELECT e.name
FROM Employees e
JOIN Employees m ON e.manager_id = m.emp_id
WHERE e.salary > m.salary;
```

**Q4. Department with the highest average salary.**
```sql
SELECT dept_id, AVG(salary) AS avg_sal
FROM Employees
GROUP BY dept_id
ORDER BY avg_sal DESC
LIMIT 1;
```

**Q5. Nth highest without LIMIT (tests window functions).**
```sql
SELECT salary FROM (
  SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) rnk
  FROM Employees
) t WHERE rnk = N;
```

**Q6. Running total of sales by date.**
```sql
SELECT sale_date, amount,
       SUM(amount) OVER (ORDER BY sale_date) AS running_total
FROM Sales;
```

**Q7. Delete duplicate rows keeping one.**
```sql
DELETE FROM Users
WHERE id NOT IN (
  SELECT MIN(id) FROM Users GROUP BY email
);
```

---

## 13. Rapid-fire concept questions (be ready to answer in 1–2 sentences)

1. **DELETE vs TRUNCATE vs DROP?** DELETE removes rows (logged, can have WHERE, can rollback). TRUNCATE removes all rows fast (minimal logging, resets identity, no WHERE). DROP removes the whole table structure.
2. **WHERE vs HAVING?** WHERE filters individual rows before grouping; HAVING filters groups after aggregation.
3. **UNION vs UNION ALL?** UNION removes duplicates (extra sort cost); UNION ALL keeps everything (faster).
4. **Primary key vs Unique key?** Both enforce uniqueness. PK: one per table, NOT NULL. Unique: many allowed, permits one NULL.
5. **What is a covering index?** An index that contains all columns a query needs, so the query is answered from the index alone without touching the table.
6. **What is a functional dependency?** X → Y means the value of X determines the value of Y (e.g., `emp_id → name`).
7. **What does EXPLAIN / EXPLAIN ANALYZE do?** Shows the query execution plan — whether it uses an index or full scan, join methods, estimated cost. First tool for debugging slow queries.
8. **Optimistic vs pessimistic locking?** Pessimistic: lock the row upfront, others wait. Optimistic: don't lock; check at commit time if data changed (version column) and retry if so. Optimistic wins under low contention.
9. **What is an ORM and one downside?** Object-Relational Mapper (Hibernate, Django ORM) maps objects to tables. Downside: the "N+1 query problem" — lazily loading related rows in a loop fires one query per iteration.
10. **What is the N+1 query problem?** Fetching a list (1 query) then looping to fetch each item's relation (N queries). Fix with a JOIN or eager loading.

---

## 14. Practice questions (do these yourself)

**Conceptual:**
1. Explain why BCNF can require splitting a table that's already in 3NF. Give an example.
2. Your `Orders` table query filtering by `customer_id AND order_date` is slow. Walk through your debugging steps.
3. When would you choose Read Committed over Serializable, and what risk do you accept?
4. Design the schema for an e-commerce site (Users, Products, Orders, OrderItems, Payments). Draw the relationships and mark keys.
5. Explain how a write-ahead log gives you both atomicity and durability.

**SQL (write the queries):**
6. Find customers who have placed orders in every month of 2024.
7. For each product, find its rank by total revenue within its category.
8. Find the median salary per department.
9. Find pairs of employees who share the same manager and the same salary.
10. Given a `Logins(user_id, login_date)` table, find users who logged in on 3+ consecutive days.

---

## 15. The 60-second "why should we trust you with a database" summary

> "A relational DB gives me ACID guarantees so concurrent operations stay correct and committed data survives crashes. I normalize to 3NF to kill redundancy and anomalies, then denormalize selectively when profiling shows join cost hurts read-heavy paths. I index the columns in WHERE/JOIN/ORDER BY — usually B+ trees — knowing indexes trade write speed for read speed. For slow queries I start with EXPLAIN. When one machine isn't enough I scale reads with replicas and writes with sharding, and I pick SQL vs NoSQL based on whether I need strong transactional consistency or horizontal write scale."
