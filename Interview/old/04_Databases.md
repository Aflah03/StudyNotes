# Databases (DBMS) — Conceptual Interview Guide (HPE)

> Covers relational database theory (keys, normalization, ACID, transactions, indexing), core SQL, and SQL vs NoSQL. Starred (⭐) topics are asked most. Read through, then use the Q&A bank to test yourself.

---

## 1. Database Fundamentals

- **Database** — an organized collection of related data stored electronically.
- **DBMS (Database Management System)** — software that lets you create, store, manage, and query databases (e.g., MySQL, PostgreSQL, Oracle, MongoDB).
- **RDBMS (Relational DBMS)** — a DBMS based on the relational model, storing data in **tables** (relations) made of **rows** (tuples/records) and **columns** (attributes). Examples: MySQL, PostgreSQL, Oracle, SQL Server.

**DBMS vs File System (why use a database):**
- File systems have data redundancy, inconsistency, no concurrency control, weak security, and no easy querying.
- DBMS provides: reduced redundancy, data integrity, concurrency control, security, backup/recovery, and powerful querying (SQL).

**Schema vs Instance:**
- **Schema** — the logical structure/design of the database (the blueprint of tables and relationships). Changes rarely.
- **Instance** — the actual data in the database at a given moment. Changes constantly.

**DBMS languages:**
- **DDL (Data Definition Language)** — defines structure: `CREATE`, `ALTER`, `DROP`, `TRUNCATE`.
- **DML (Data Manipulation Language)** — manipulates data: `SELECT`, `INSERT`, `UPDATE`, `DELETE`.
- **DCL (Data Control Language)** — permissions: `GRANT`, `REVOKE`.
- **TCL (Transaction Control Language)** — transactions: `COMMIT`, `ROLLBACK`, `SAVEPOINT`.

---

## 2. Keys ⭐ (very common)

Keys uniquely identify rows and establish relationships.

- **Super Key** — any set of attributes that uniquely identifies a row (may have extra attributes).
- **Candidate Key** — a minimal super key (no unnecessary attributes); there can be several.
- **Primary Key** — the candidate key chosen to uniquely identify each row. Cannot be NULL, must be unique. One per table.
- **Alternate Key** — candidate keys not chosen as the primary key.
- **Composite Key** — a primary key made of two or more columns combined.
- **Foreign Key** — an attribute in one table that references the primary key of another table, enforcing **referential integrity** (a relationship between tables).
- **Unique Key** — ensures all values in a column are unique, but *can* hold one NULL (unlike a primary key).

**Relationship among keys:** Super Key ⊇ Candidate Key ⊇ Primary Key (primary is a chosen candidate; candidate is a minimal super key).

---

## 3. Normalization ⭐ (almost always asked)

**Definition:** Normalization is the process of organizing data to reduce **redundancy** and eliminate **anomalies** (insertion, update, deletion anomalies) by dividing large tables into smaller related tables.

**The anomalies it fixes:**
- **Insertion anomaly** — can't add data because other data is missing.
- **Update anomaly** — updating redundant data in multiple places risks inconsistency.
- **Deletion anomaly** — deleting a row unintentionally removes other needed data.

**The Normal Forms (know at least 1NF–BCNF):**

| Form | Rule |
|------|------|
| **1NF** | Each column holds atomic (indivisible) values; no repeating groups or arrays. Each row is unique. |
| **2NF** | Is in 1NF **and** has no *partial dependency* — no non-key attribute depends on only part of a composite primary key. |
| **3NF** | Is in 2NF **and** has no *transitive dependency* — non-key attributes depend only on the primary key, not on other non-key attributes. |
| **BCNF** (3.5NF) | A stronger 3NF: for every functional dependency X → Y, X must be a super key. Handles certain edge cases 3NF misses. |

**Functional Dependency (FD):** X → Y means the value of X uniquely determines the value of Y. (e.g., `StudentID → StudentName`.) This is the basis for normalization.

- **Partial dependency** — a non-key attribute depends on part of a composite key (violates 2NF).
- **Transitive dependency** — a non-key attribute depends on another non-key attribute (violates 3NF).

**Denormalization** — intentionally adding redundancy back (merging tables) to improve *read* performance by avoiding joins. Trade-off: faster reads, but more storage and update complexity. Used in data warehouses / read-heavy systems.

**Normalization vs Denormalization:** Normalization reduces redundancy (good for write-heavy, consistency). Denormalization adds redundancy for read speed (good for read-heavy analytics).

---

## 4. ACID Properties ⭐ (transactions — very common)

A **transaction** is a single logical unit of work made of one or more operations, treated as all-or-nothing.

**ACID guarantees reliable transactions:**
- **A — Atomicity** — all operations in a transaction complete, or none do (all-or-nothing). If one fails, the whole thing rolls back.
- **C — Consistency** — a transaction takes the database from one valid state to another, preserving all rules/constraints.
- **I — Isolation** — concurrent transactions don't interfere; the result is as if they ran one after another.
- **D — Durability** — once committed, changes are permanent even after a crash or power loss.

**Transaction states:** Active → Partially Committed → Committed (or → Failed → Aborted).

**COMMIT** makes changes permanent. **ROLLBACK** undoes changes since the last commit. **SAVEPOINT** sets a marker to roll back to partially.

---

## 5. Concurrency Control & Isolation

When many transactions run at once, problems can occur:

**Concurrency problems:**
- **Dirty Read** — reading uncommitted data from another transaction that might roll back.
- **Non-repeatable Read** — reading the same row twice gives different values because another transaction updated it in between.
- **Phantom Read** — re-running a query returns new rows inserted by another transaction.
- **Lost Update** — two transactions update the same data and one overwrites the other.

**Isolation levels** (trade consistency for performance, weakest to strongest):
1. **Read Uncommitted** — allows dirty reads.
2. **Read Committed** — no dirty reads; non-repeatable reads possible.
3. **Repeatable Read** — no dirty/non-repeatable reads; phantom reads possible.
4. **Serializable** — fully isolated (strongest, slowest).

**Concurrency control mechanisms:**
- **Locking** — shared (read) locks and exclusive (write) locks.
- **2PL (Two-Phase Locking)** — a protocol with a growing phase (acquire locks) and a shrinking phase (release locks) to ensure serializability.
- **Deadlock in DB** — two transactions each waiting for a lock the other holds; the DBMS detects and aborts one (a "victim").

---

## 6. Indexing ⭐

**Definition:** An index is a data structure that speeds up data retrieval on a table at the cost of extra storage and slower writes (the index must be updated on insert/update/delete).

- Analogy: an index is like the index at the back of a book — you find topics fast without scanning every page.
- **Without an index**, a query does a *full table scan* (O(n)). **With an index**, lookups are much faster.

**Types:**
- **Primary index** — on the primary key (often the physical order of rows).
- **Clustered index** — determines the physical order of data in the table; only one per table (data rows *are* stored in index order).
- **Non-clustered index** — a separate structure with pointers to the data; multiple allowed per table.
- **Dense vs sparse index** — dense has an entry for every record; sparse has entries for some.

**Underlying structures:**
- **B-Tree / B+ Tree** — the most common index structure; balanced, keeps data sorted, allows efficient range queries and logarithmic lookups. In a B+ tree, all actual data is in leaf nodes linked together (great for range scans).
- **Hash index** — great for exact-match lookups (O(1)) but not for range queries.

**Trade-off:** indexes speed up reads/searches but slow down writes and use extra space — so index the columns you frequently search/join on, not everything.

---

## 7. SQL Essentials ⭐

**Basic query structure:**
```sql
SELECT column1, column2
FROM table
WHERE condition
GROUP BY column
HAVING group_condition
ORDER BY column;
```

**Logical order of execution:** FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY. (Note: SELECT is written first but runs late — this is a nice thing to mention.)

**WHERE vs HAVING:** WHERE filters individual rows *before* grouping; HAVING filters *after* grouping (used with aggregate functions).

**Aggregate functions:** `COUNT()`, `SUM()`, `AVG()`, `MIN()`, `MAX()`.

**DELETE vs TRUNCATE vs DROP:**
- **DELETE** — DML; removes rows (optionally with WHERE); can be rolled back; slower.
- **TRUNCATE** — DDL; removes *all* rows fast, resets identity; can't use WHERE; usually can't roll back.
- **DROP** — DDL; removes the entire table structure and data.

---

## 8. Joins ⭐ (very common)

A **JOIN** combines rows from two or more tables based on a related column.

- **INNER JOIN** — returns only rows with matching values in both tables.
- **LEFT (OUTER) JOIN** — all rows from the left table + matching rows from the right (NULLs where no match).
- **RIGHT (OUTER) JOIN** — all rows from the right table + matching from the left.
- **FULL (OUTER) JOIN** — all rows from both tables; NULLs where there's no match.
- **CROSS JOIN** — Cartesian product (every row of A with every row of B).
- **SELF JOIN** — a table joined with itself (e.g., employees and their managers in one table).

```sql
SELECT s.name, d.dept_name
FROM Students s
INNER JOIN Departments d ON s.dept_id = d.id;
```

---

## 9. SQL vs NoSQL ⭐ (increasingly common)

| Aspect | SQL (Relational) | NoSQL (Non-relational) |
|--------|------------------|------------------------|
| Structure | Tables, fixed schema | Flexible schema (documents, key-value, graph, column) |
| Scaling | Vertical (scale up) | Horizontal (scale out) |
| Consistency | Strong (ACID) | Often eventual (BASE) |
| Best for | Structured data, complex queries, transactions | Large-scale, unstructured/semi-structured, high throughput |
| Examples | MySQL, PostgreSQL, Oracle | MongoDB (document), Redis (key-value), Cassandra (column), Neo4j (graph) |

**NoSQL types:**
- **Document** — JSON-like documents (MongoDB).
- **Key-Value** — simple key→value pairs (Redis, DynamoDB).
- **Column-family** — data in columns (Cassandra).
- **Graph** — nodes and relationships (Neo4j).

**When to choose which:** Use SQL when you need strong consistency, complex relationships/joins, and structured data (banking, ERP). Use NoSQL when you need massive horizontal scale, flexible/changing schemas, or high-velocity data (social feeds, IoT, caching).

---

## 10. CAP Theorem & BASE (bonus, impresses)

**CAP Theorem:** A distributed data store can provide at most **two** of these three guarantees simultaneously:
- **C — Consistency** — every read gets the most recent write.
- **A — Availability** — every request gets a response (not necessarily latest).
- **P — Partition Tolerance** — the system keeps working despite network failures between nodes.

Since network partitions are unavoidable in distributed systems, you effectively choose between **C and A** during a partition. (Traditional RDBMS lean CP; many NoSQL systems lean AP.)

**BASE** (the NoSQL counterpart to ACID): **B**asically **A**vailable, **S**oft state, **E**ventual consistency — favors availability over immediate consistency.

---

## 11. Other Important Concepts

- **View** — a virtual table based on a stored query. It doesn't store data itself; it simplifies complex queries and adds a security layer.
- **Stored Procedure** — a precompiled set of SQL statements stored in the DB, executed by name. Improves performance and reusability.
- **Trigger** — SQL code that runs automatically in response to events (INSERT/UPDATE/DELETE) on a table.
- **ER Model (Entity-Relationship)** — a diagram-based way to design a database using entities, attributes, and relationships. Cardinality: one-to-one, one-to-many, many-to-many.
- **Referential integrity** — foreign key values must match an existing primary key (or be NULL), keeping relationships valid.
- **NULL** — represents missing/unknown value; it's not zero and not an empty string. Comparisons with NULL use `IS NULL` / `IS NOT NULL`.

---

## 12. Interview Q&A Bank (self-test)

**Q: What is DBMS and RDBMS?**
DBMS is software to store, manage, and query databases. RDBMS is a DBMS based on the relational model, storing data in tables with rows and columns and supporting relationships via keys.

**Q: DBMS vs File System?**
File systems have redundancy, inconsistency, poor concurrency, and weak security. A DBMS reduces redundancy and provides integrity, concurrency control, security, recovery, and powerful querying.

**Q: Primary key vs Unique key vs Foreign key?**
A primary key uniquely identifies each row, can't be NULL, and there's one per table. A unique key also enforces uniqueness but allows one NULL. A foreign key references another table's primary key to enforce referential integrity.

**Q: Candidate key vs Super key vs Primary key?**
A super key is any attribute set that uniquely identifies a row. A candidate key is a minimal super key. The primary key is the candidate key chosen to identify rows.

**Q: What is normalization and why?**
Organizing data into smaller related tables to reduce redundancy and eliminate insertion, update, and deletion anomalies, using functional dependencies.

**Q: Explain 1NF, 2NF, 3NF.**
1NF: atomic values, no repeating groups. 2NF: 1NF plus no partial dependency on part of a composite key. 3NF: 2NF plus no transitive dependency (non-key attributes depend only on the key).

**Q: What is BCNF?**
A stronger form of 3NF where for every functional dependency X→Y, X must be a super key; it handles edge cases 3NF doesn't.

**Q: What is denormalization?**
Deliberately adding redundancy back to reduce joins and speed up reads, at the cost of extra storage and update complexity — used in read-heavy systems.

**Q: What are the ACID properties?**
Atomicity (all-or-nothing), Consistency (valid state to valid state), Isolation (concurrent transactions don't interfere), Durability (committed changes survive crashes).

**Q: What is a transaction?**
A single logical unit of work made of one or more operations, treated as all-or-nothing and governed by ACID.

**Q: What is a dirty read?**
Reading uncommitted data from another transaction that may later roll back, leaving you with invalid data.

**Q: What are isolation levels?**
Read Uncommitted, Read Committed, Repeatable Read, and Serializable — from weakest (allows dirty reads) to strongest (fully serial), trading consistency against performance.

**Q: What is an index and why use it?**
A data structure (often a B+ tree) that speeds up data retrieval by avoiding full table scans, at the cost of extra storage and slower writes.

**Q: Clustered vs non-clustered index?**
A clustered index defines the physical order of the table's rows (one per table). A non-clustered index is a separate structure with pointers to rows (multiple allowed).

**Q: Why B+ trees for indexing?**
They're balanced (logarithmic lookups), keep keys sorted, and store all data in linked leaf nodes, making both point lookups and range queries efficient.

**Q: Explain the types of JOINs.**
INNER (only matches), LEFT (all left + matches), RIGHT (all right + matches), FULL (all from both), CROSS (Cartesian product), and SELF (table joined to itself).

**Q: WHERE vs HAVING?**
WHERE filters rows before grouping; HAVING filters groups after aggregation (used with aggregate functions).

**Q: DELETE vs TRUNCATE vs DROP?**
DELETE removes selected rows and can be rolled back. TRUNCATE quickly removes all rows and resets identity, usually without rollback. DROP removes the whole table structure and data.

**Q: SQL vs NoSQL?**
SQL databases are relational with fixed schemas, strong ACID consistency, and vertical scaling — good for structured data and complex queries. NoSQL databases have flexible schemas, scale horizontally, often use eventual consistency (BASE), and suit large-scale unstructured/high-velocity data.

**Q: What is the CAP theorem?**
In a distributed store you can guarantee at most two of Consistency, Availability, and Partition tolerance simultaneously; since partitions are unavoidable, you effectively trade consistency against availability.

**Q: View vs Stored Procedure vs Trigger?**
A view is a virtual table from a stored query. A stored procedure is precompiled SQL executed by name. A trigger runs automatically in response to table events like insert/update/delete.

**Q: What is a functional dependency?**
A relationship where one attribute's value uniquely determines another's (X→Y); it's the basis for normalization.

**Q: What is referential integrity?**
The rule that a foreign key value must match an existing primary key (or be NULL), keeping table relationships valid.

---

### Quick-recall cheat sheet
- Keys: Super ⊇ Candidate ⊇ Primary. FK enforces referential integrity.
- Normalization: 1NF (atomic) → 2NF (no partial dep) → 3NF (no transitive dep) → BCNF.
- ACID: Atomicity, Consistency, Isolation, Durability.
- Isolation levels: Read Uncommitted → Read Committed → Repeatable Read → Serializable.
- Index = B+ tree, fast reads, slower writes. Clustered = physical order (one per table).
- Joins: INNER, LEFT, RIGHT, FULL, CROSS, SELF.
- SQL = ACID + vertical scale; NoSQL = BASE + horizontal scale.
- CAP: pick 2 of Consistency, Availability, Partition tolerance.
