# CS Technical Interview Prep Pack — Start Here

A complete, interview-focused study pack for a 4th-year CS student. Six guides, each written the way you'd want to *speak* the answer in a room — concepts, real-world examples, gotchas, and practice questions.

## What's in this pack

| File | Subject | What it covers |
|------|---------|----------------|
| `01-DBMS.md` | Databases | Keys, normalization, ACID, isolation levels, indexing, joins, SQL vs NoSQL, CAP, sharding, + SQL problems |
| `02-OOPS.md` | Object-Oriented Programming | 4 pillars, overloading vs overriding, abstract class vs interface, composition vs inheritance, SOLID, mechanics |
| `03-LLD.md` | Low-Level Design | 6-step framework, SOLID with code, all key design patterns, UML, fully worked problems (Parking Lot, Vending Machine, Splitwise), concurrency |
| `04-CN.md` | Computer Networks | OSI/TCP-IP, TCP vs UDP, 3-way handshake, IP/subnetting, HTTP/HTTPS, TLS, DNS, sockets, devices |
| `05-OS.md` | Operating Systems | Process vs thread, scheduling, mutex/semaphore, deadlock, memory/paging, page replacement, IPC, classic sync problems |
| `06-How-X-Works.md` | Conceptual systems | Type-a-URL journey, DNS, Google Search, back/forward button, HTTPS, autocomplete, CDN, load balancer, email, caching, URL shortener |

Every guide ends with a **60-second summary** you can memorize as your "default answer" for that subject, plus a **practice questions** section.

---

## How to use this (a realistic plan)

### If you have ~2 weeks
- **Days 1–2:** DBMS + do the SQL problems on paper.
- **Days 3–4:** OS — memorize the four deadlock conditions and do the scheduling/page-fault numericals.
- **Days 5–6:** CN — be able to draw the 3-way handshake and TLS flow from memory.
- **Days 7–8:** OOPS — attach a real example to each of the four pillars and each SOLID principle.
- **Days 9–11:** LLD — code the three worked problems yourself from scratch, then attempt two from the practice list.
- **Day 12:** How-X-Works — practice the "type a URL" journey out loud until it's fluent.
- **Days 13–14:** Revise every 60-second summary; redo the practice questions you got wrong.

### If you have ~3 days (crunch)
Read every **60-second summary** first (they're your safety net), then the rapid-fire sections in each doc, then the How-X-Works doc (highest ROI per minute). Skim the worked LLD problems.

---

## The rule that makes you pass

**Don't recite — reason.** For every concept, an interviewer can ask "why?" or "give an example." The people who pass:
1. Give the definition,
2. **immediately attach a concrete example**, and
3. mention a trade-off or when it breaks in production.

Example — weak vs strong:
- ❌ "An index makes queries faster."
- ✅ "An index is like a book's back-index — a B+ tree that turns a full-table scan into an O(log n) lookup. I'd add one on the columns in WHERE/JOIN clauses, knowing it speeds up reads but slows writes and uses disk, so I don't index everything."

---

## High-frequency questions to guarantee you can answer

These come up constantly across companies. If you can nail all of these cold, you're in good shape:

- **DBMS:** Explain ACID. Normalization to 3NF with an example. Clustered vs non-clustered index. SQL vs NoSQL — when each. Write the 2nd-highest-salary query.
- **OOPS:** Four pillars with examples. Overloading vs overriding. Abstract class vs interface. Why favor composition over inheritance.
- **LLD:** Explain SOLID. Design a parking lot. Singleton + one other pattern. When would you use Strategy vs State.
- **CN:** TCP vs UDP. 3-way handshake. How does HTTPS work. What happens when you type a URL.
- **OS:** Process vs thread. Mutex vs semaphore. Four conditions of deadlock. Paging vs segmentation. Explain thrashing.
- **Systems:** How DNS works. How Google Search works. How the back button works. How caching works.

---

## A note on the cross-links

The subjects overlap — that's a feature, and interviewers love when you connect them:
- **Caching** appears in DBMS (query cache), OS (page cache, TLB), and CN (CDN, DNS cache) — the same idea everywhere.
- **Deadlock's four conditions** are identical in OS and DBMS transactions.
- **LRU** is both an OS page-replacement policy and a cache eviction policy and a classic LLD problem.
- **The OSI layers** explain the "type a URL" journey.
- **SOLID and the design patterns** connect OOPS and LLD directly.

When you can say "this is the same principle as X in another subject," you sound like someone who actually understands computing rather than someone who crammed five separate syllabi.

Good luck — go build things and reason out loud.
