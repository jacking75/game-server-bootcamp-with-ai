# Phase 4. Data & API Server (Weeks 12-15, 160h)

> 🇰🇷 Korean version: [04-phase4-data-api_kr.md](04-phase4-data-api_kr.md)

> Common 100h / Track 60h. By the end of this Phase, learners will build an HTTP API server and DB that handle account and asset data in a production-grade structure, and defend against concurrency issues (such as currency duplication) using transactions.

## 0. How to Use This Document

Same structure as Phases 1-3. Follow the daily blocks in §2; find labs (`L4-C-xx`, `L4-CS-xx`, `L4-CPP-xx`) in §3, and assignment details in §7.

The 3 principles of this Phase

1. **Put the DB behind an interface.** Inject implementations via Repository/DAO interfaces + DI. The development default is **SQLite** (no installation needed, file-based); switching to **MySQL** takes just one configuration line if you want. "Writing code so the DB can be swapped" is itself a learning goal here.
2. **Prove correctness with tests.** Demonstrate "even with 100 simultaneous purchases, the currency is deducted exactly once" not through argument but through a **passing concurrency test log**.
3. **Always keep the attacker's mindset on.** For every API, first ask "how could this be abused," then reproduce 5 or more abuse scenarios as tests and block them.

| Notation | Meaning |
|---|---|
| `L4-C-xx` / `L4-CS-xx` / `L4-CPP-xx` | Common / C# / C++ labs |
| 🔴 Required · 🟡 If time allows | Priority |
| **(SQLite)** / **(MySQL)** | Instructions that differ by DB path |

---

## 1. Overview

### 1.1 Objectives

1. **Build the flow of a game API server**: version check → login → load user data → lobby actions (inventory, mail, attendance, shop, ranking) → logout
2. **Abstract DB access**: Repository/DAO interfaces + DI. Default SQLite, optional MySQL. **Switching must be possible by changing configuration only**
3. **Understand transactions and isolation levels**: defend against double-spending and duplicate claims with conditional UPDATE / idempotency keys / locks
4. **Use Redis for its intended purposes**: sessions (TTL), rankings (ZSET), caching, and the limits of distributed locks
5. **Integrate with the game server (3-1)**: save game results → update rankings, server-to-server authentication
6. **Identify and block abuse scenarios**: currency duplication, duplicate claims, token theft, replay, races

### 1.2 DB Path (Independent of Track)

| Category | Default (recommended) | Optional |
|---|---|---|
| DB | **SQLite** — file-based, no installation, fast tests | **MySQL 8** — server-based, closer to a real production service |
| C# client | `Microsoft.Data.Sqlite` or EF Core Sqlite | `MySqlConnector` + SqlKata/Dapper |
| C++ client | vcpkg `sqlite3` or `SQLiteCpp` | MySQL Connector/C++ |
| Performance lab scale | 10,000 users / 100,000 inventory rows | 1,000,000 users / 10,000,000 inventory rows |
| Execution plan | `EXPLAIN QUERY PLAN` | `EXPLAIN` |

**Core rule**: whichever you use, **the application code knows only the interface.** DB-specific SQL exists only inside the implementation. After completing Phase 4 with SQLite, swapping in MySQL for 🟡 Assignment 4-3c lets you feel the value of this design firsthand.

### 1.3 Prerequisite Self-Check (20 minutes)

| # | Question | Passing criterion |
|---|---|---|
| 1 | Can you write a 3-table JOIN + aggregation in SQL | Able to write it on the spot |
| 2 | Can you state what a transaction guarantees, in terms of ACID | Immediate answer |
| 3 | Does the 3-1 server have a "game result hook" | Interface exists |
| 4 | Do you know the meaning of HTTP status codes 200/400/401/403/404/409/429/500 | Immediate answer |
| 5 | (C# track) Do you know the difference between the 3 DI lifetimes (Singleton/Scoped/Transient) | Immediate answer |
| 6 | Can you run Redis and add/remove keys with `redis-cli` | Confirmed execution |

If you pass fewer than 3, schedule SQL/HTTP fundamentals reinforcement in the first two days of week 12 (MySQL book Days 1-2, roadmap Ch.7).

### 1.4 What You Will Be Able to Do After This Phase

- **Prove with a test** that "with 100 simultaneous purchases, currency is deducted exactly once"
- Find slow queries via execution plans, fix them with indexes, and present before/after timings
- Reproduce isolation-level phenomena (dirty reads, non-repeatable reads, phantoms) with scripts
- Explain "why the game server goes through the API server instead of hitting the DB directly"
- Run a server built with SQLite on MySQL by changing only configuration (when 🟡 4-3c is done)

### 1.5 Deliverables of This Phase

```
phase4/
├─ SCHEMA.md                   ERD + DDL + index rationale + DB/Redis placement decision table
├─ API.md (or openapi.yaml)    20+ endpoint specs, error code table
├─ adr/0005-db-abstraction.md  Repository+DI, decision to default to SQLite / optionally use MySQL
├─ GameApi/
│  ├─ Contracts/               DTOs, error codes (zero external dependencies)
│  ├─ Domain/                  service logic (transaction boundaries)
│  ├─ Data/                    IRepository interfaces
│  ├─ Data.Sqlite/             SQLite implementation   ← default
│  ├─ Data.MySql/               MySQL implementation 🟡 ← optional
│  ├─ Cache/                   Redis access (sessions, rankings)
│  ├─ Api/                     HTTP entry point, middleware, auth
│  ├─ Worker/                  batch jobs (mail expiry, attendance reset)
│  └─ tests/                   30+ integration / 3 concurrency / 5 auth
├─ migrations/                 schema version scripts
├─ tools/seed-data.ps1         dummy data generation
├─ PERF-4-1.md                 execution plan before/after, p99 measurement
├─ SECURITY-SCENARIOS.md       Assignment 4-2
└─ labs/
```

### 1.6 4-Week Roadmap

| Week | Big topic | Common | Track | Assignment |
|---|---|---|---|---|
| 12 | API, schema, DB abstraction | HTTP/REST, game API flow, schema design, hashing | Repository+DI, SQLite implementation | 4-C, signup/login |
| 13 | Auth, Redis, transactions | Tokens vs sessions, Redis data structures, isolation levels, indexes | Auth middleware, Redis client | 4-1 accounts/inventory |
| 14 | Concurrency, content, integration | Double-spending, idempotency, ranking, server-to-server auth, batch jobs | Shop transactions, ZSET ranking | 4-1 shop/mail/attendance/ranking/integration |
| 15 | Security, performance, evaluation | Abuse scenarios, integration tests, performance verification | Execution plan tuning | 4-2, 🟡4-3, evaluation |

---

## 2. Weekly Detailed Plan (Day by Day)

Day numbers are based on the whole course (Phase 4 spans days 56-75).

### 2.1 Week 12 — API Design and DB Abstraction

#### Day 56 (Mon) — HTTP/REST and the Game API Flow 🔴

**Morning (2.5h)**
- Definitions of HTTP methods, status codes, headers, JSON, and idempotency (the same request repeated multiple times yields the same result)
- **Characteristics of game APIs**: command-style (uniformly POST) is more common than REST — because "units of action" don't map cleanly to resources. Summarize the pros and cons
- Standard flow: `version check → login (token issuance) → load user data (one bulk call) → lobby actions → logout`
- Error design: the trade-off between HTTP status + **application error code** (e.g., 200 OK + `{errorCode: 3001}`) versus pure HTTP status codes

**Afternoon (2.5h)**
- `L4-C-01` Bring up a minimal API server: health check + version check endpoints, test calls via a `.http` file
- Start Assignment 4-C: skeleton of `API.md` (placeholder for the endpoint list, placeholder for the error code table)

**1 Hour Without AI**
- List, by name only, the 20 APIs needed for the Omok lobby game (to compare against 4-C later)

**DoD**
- [ ] Health check / version check API responses confirmed (`.http` file committed)
- [ ] `API.md` skeleton

#### Day 57 (Tue) — Schema Design 🔴

**Morning (2.5h)**
- Normalization and denormalization decisions for games (inventory as rows vs a JSON column)
- Key design: AUTO_INCREMENT vs UUID/ULID (distribution, guess resistance, index locality)
- Required tables: `account`, `user`, `user_currency`, `item_master`, `user_item`, `mail`, `mail_attachment`, `attendance`, `shop_product`, `purchase_log`, `game_result`, `idempotency_key`
- Why every table should have `created_at` and `updated_at`

**Afternoon (2.5h)**
- `L4-C-02` Write the ERD (Mermaid) + a DDL draft (based on SQLite, with corresponding MySQL types noted alongside)
- `L4-C-03` Index design: attach a comment to each index describing **"which query this is for."** First write down 5 queries that would be slow without an index, then design around them

**1 Hour Without AI**
- Predict and write down 3 queries that would slow down "with 100 million users and 1 billion inventory rows" (to verify in week 15)

**DoD**
- [ ] ERD + DDL for 12+ tables
- [ ] Purpose comment on every index, list of 5 key queries

#### Day 58 (Wed) — Repository + DI (the Backbone of This Phase) 🔴

**Morning (2.5h)**
- Dependency inversion: the domain service knows only `IUserRepository`; the SQLite/MySQL implementation lives outside
- Where to place transaction boundaries: service layer (recommended) vs repository. **How to combine multiple repositories into a single transaction** (UnitOfWork or passing a connection)
- SQLite characteristics: a single file, serialized writes, using **WAL mode** to secure read concurrency, configuring `busy_timeout`

**Afternoon (2.5h)**
- `L4-C-04` Define interfaces (use the skeleton in §7.2): `IUserRepository`, `IItemRepository`, `IMailRepository`, `IShopRepository`, `IUnitOfWork`
- `L4-CS-01`/`L4-CPP-01` SQLite implementation + DI registration. Configure connection string, WAL, busy_timeout
- Migration script (`migrations/001_init.sql`) and an application tool (a simple runner)

**1 Hour Without AI**
- Write 3 reasons "why the service must not use SQL directly" (testability, replaceability, duplication)

**DoD**
- [ ] 5 interfaces + SQLite implementation, DI registration
- [ ] Migration creates an empty DB and tests use it

#### Day 59 (Thu) — Signup, Login, Hashing 🔴

**Morning (2.5h)**
- Password hashing: **Argon2id or bcrypt/PBKDF2**. SHA256 alone is forbidden (reason: fast computation favors brute force)
- Salt, iteration count, memory parameters, storage format
- Account policy: ID rules, minimum password requirements, preventing duplicate signup (unique index + exception handling)

**Afternoon (2.5h)**
- `L4-C-05` Signup API: validate → hash → store (handle unique violations) → create default user data (currency, attendance records), all in **a single transaction**
- `L4-C-06` Login API: look up account → verify hash → issue token (connected to Redis session tomorrow) → update last login time
- 6 integration tests: successful signup, duplicate ID, weak password, successful login, wrong password, nonexistent account

**1 Hour Without AI**
- Check that hash results are never printed to logs, and that error messages do not distinguish and expose "ID does not exist" vs "wrong password" (prevent account enumeration)

**DoD**
- [ ] Signup and login work, 6 integration tests pass
- [ ] Passwords are never stored in plaintext or with weak hashing

#### Day 60 (Fri) — Completing the 4-C Spec + Weekly Checkpoint

**Morning (3h)**
- Complete `SCHEMA.md`: ERD, DDL, index rationale, **DB vs Redis placement decision table** (§7.1)
- Complete `API.md`: 20+ endpoints, each with request/response, whether auth is required, error codes, and idempotency
- `adr/0005-db-abstraction.md`: Repository+DI, the decision to default to SQLite / optionally use MySQL, and alternatives

**Afternoon (4h) — Weekly Checkpoint**
1. Reimplement without AI (60 min): the signup API (validation, hashing, transaction)
2. Oral exam (45 min): hashing requirements, transaction boundaries, reasons for DB abstraction
3. Code review (45 min): Rubric item 6 (structure) — layer violations
4. Retrospective (30 min): `W12.md`

**DoD**
- [ ] 4-C submission status (`SCHEMA.md`, `API.md`, ADR)
- [ ] Ambiguous items resolved in the AI "different-team developer" review

### 2.2 Week 13 — Authentication, Redis, Transactions

#### Day 61 (Mon) — Tokens and Sessions 🔴

**Morning (2.5h)**
- Comparing token approaches: **opaque token + Redis session** (easy to invalidate) vs **JWT** (fast to verify, hard to invalidate)
- The choice for this course: opaque token (32B random) → Redis `session:{token} → userId`, TTL 1 hour, sliding renewal on every request
- Handling duplicate logins: also keep `user_session:{userId} → token`, and delete the previous token on a new login

**Afternoon (2.5h)**
- `L4-C-07` Implement Redis sessions: issuance, verification, renewal, deletion; invalidate the previous session on duplicate login
- `L4-C-08` Auth middleware/filter: token from header → verify → inject userId into request context, 401 on failure
- 5 auth tests: expired token, forged token, accessing my data with someone else's token, unauthenticated access, previous token after duplicate login

**1 Hour Without AI**
- Write down "why logout is hard with JWT alone" and the cost of workarounds (short expiry + refresh, blacklist)

**DoD**
- [ ] 5 auth tests pass
- [ ] Tokens do not appear in logs (masking confirmed)

#### Day 62 (Tue) — Redis Data Structures and Their Uses 🔴

**Morning (2.5h)**
- Game uses by data structure: String (sessions, flags), Hash (profile cache), List (queues), Set (online list), **ZSet (ranking)**
- TTL strategy, pipelining, **Lua script atomicity**, persistence concepts (RDB/AOF)
- "What must not go in Redis": ledger-like **data that must never be lost** (volatility risk + difficulty combining with transactions)

**Afternoon (2.5h)**
- `L4-C-09` Ranking ZSET: update scores, fetch top 100, fetch my rank (`ZREVRANK`), tie-breaking rules
- `L4-C-10` Cache patterns: user profile cache (fill on read, invalidate on write), preventing cache stampedes (short locks or probabilistic expiry)
- `L4-C-11` Atomic Lua operations: atomize shop stock deduction with Lua

**1 Hour Without AI**
- Write a decision table for 10 kinds of data: "DB, Redis, or both" (to be included in §7.1)

**DoD**
- [ ] All 3 ranking features work, Lua stock deduction test passes
- [ ] Data placement decision table complete

#### Day 63 (Wed) — Transactions and Isolation Levels 🔴

**Morning (2.5h)**
- ACID, the 4 isolation levels and the phenomena at each level (dirty read / non-repeatable read / phantom)
- **(SQLite)** the default is close to serializable (single writer). Read-write concurrency under WAL mode, `SQLITE_BUSY` and `busy_timeout`
- **(MySQL)** InnoDB defaults to REPEATABLE READ, record/gap/next-key locks, deadlocks and retries
- Designing transaction boundaries: keep them short, never put external calls (HTTP/Redis) inside a transaction

**Afternoon (2.5h)**
- `L4-C-12` Script to reproduce isolation-level phenomena: open two sessions and observe non-repeatable reads/phantoms (**(SQLite)** substitute by observing locking behavior)
- `L4-C-13` Reproduce a deadlock or lock conflict + retry logic (exponential backoff, max 3 attempts)

**1 Hour Without AI**
- Write in 3 sentences "what happens if you make an HTTP call inside a transaction"

**DoD**
- [ ] Logs reproducing isolation/lock phenomena
- [ ] Retry logic and its test

#### Day 64 (Thu) — Inventory and Loading User Data 🔴

**Morning (2.5h)**
- Designing a "data load" API that returns all data needed in a single call after login (reducing round trips)
- Pagination design: offset vs keyset (cursor). Why keyset is favorable for large inventories

**Afternoon (2.5h)**
- `L4-C-14` User data load API: basic info + currency + inventory summary + unread mail count
- `L4-C-15` Inventory API: list (paginated), use item (deduct quantity, delete if it reaches 0) — **prevent negatives with a conditional UPDATE**
- 8 integration tests: pagination boundaries, using a nonexistent item, using more than available quantity, using the same item twice concurrently, etc.

**1 Hour Without AI**
- Find spots where N+1 queries could occur, mark them, and write a fix

**DoD**
- [ ] Data load and inventory APIs work, 8 tests pass
- [ ] No N+1 queries (confirmed via query logs)

#### Day 65 (Fri) — Execution Plans and Indexes + Weekly Checkpoint

**Morning (3h)**
- `L4-C-16` Dummy data generation script: **(SQLite)** 10,000 users / 100,000 inventory rows; **(MySQL)** 1,000,000 / 10,000,000. Using batch INSERTs
- `L4-C-17` Check execution plans: run the 5 key queries through `EXPLAIN QUERY PLAN` (or `EXPLAIN`) and eliminate full scans. Table of execution time before/after adding indexes

**Afternoon (4h) — Weekly Checkpoint**
1. Reimplement without AI (60 min): Redis session issuance, verification, and invalidation on duplicate login
2. Oral exam (45 min): isolation levels, composite index column order, token vs session
3. Code review (45 min): Rubric item 3 (resources) — connection lifetime, transaction leaks
4. Retrospective (30 min): `W13.md`

**DoD**
- [ ] Execution plan before/after table (5 queries)
- [ ] Dummy data script committed

### 2.3 Week 14 — Concurrency, Content, Server Integration

#### Day 66 (Mon) — Defending Against Double-Spending (the Core of This Phase) 🔴

**Morning (2.5h)**
- The wrong pattern: `SELECT balance → check in the application → UPDATE` (a race)
- 3 correct patterns
  1. **Conditional UPDATE**: `UPDATE user_currency SET gold = gold - :price WHERE user_id = :id AND gold >= :price` → **check the affected row count**
  2. **Pessimistic locking**: `SELECT ... FOR UPDATE` (MySQL) / an immediate write transaction (SQLite)
  3. **Optimistic locking**: a version column + `WHERE version = :v`, retry on failure
- Chapters 11-12 of the "Gacha Probability Implementation Guide" book are the best material for this topic

**Afternoon (2.5h)**
- `L4-C-18` Implement the shop purchase transaction: currency deduction + item grant + purchase history logging, all in **a single transaction**. For products with stock, deduct via Lua or a conditional UPDATE
- `L4-C-19` **Test 100 concurrent purchases**: fire 100 requests simultaneously from the same account → 1 success, currency deducted exactly once, exactly 1 item granted

**1 Hour Without AI**
- Tabulate the costs (lock wait, retries, implementation complexity) of the three defense patterns

**DoD**
- [ ] The 100-concurrent-purchase test passes (stable across 10 repetitions)
- [ ] The affected-row-count check is present in the code

#### Day 67 (Tue) — Idempotency: Mail and Attendance 🔴

**Morning (2.5h)**
- Idempotency key design: the client generates a request ID → the server stores it → on a duplicate, **return the previous result as-is**
- Storage location and expiry (a table + TTL index, or Redis), handling key collisions
- Guaranteeing mail is claimed "only once": a status column + conditional UPDATE (`WHERE status = 'unread'`)

**Afternoon (2.5h)**
- `L4-C-20` Mailbox API: list (paginated), claim (grant attached currency/items + change status, one transaction), expiry handling (a flag for the batch job)
- `L4-C-21` Attendance API: once per day, reset at **midnight (KST)**, grant a reward. Date-boundary tests (23:59:59 / 00:00:01)
- 2 concurrency tests: 50 concurrent mail claims → only 1 succeeds, 50 concurrent attendance checks → only 1 succeeds

**1 Hour Without AI**
- Summarize "when to delete an idempotency key" and "what happens if a retry arrives before it's deleted"

**DoD**
- [ ] Mail and attendance concurrency tests pass
- [ ] Date-boundary tests pass (timezone specified)

#### Day 68 (Wed) — Rankings and Saving Game Results 🔴

**Morning (2.5h)**
- Ranking design: Redis ZSET (real-time) + DB (ledger). **How to keep the two consistent** (write order, recovery procedure)
- Season concept: include the season in the key (`rank:season:{n}`), save a snapshot to the DB when a season ends
- Tie-breaking: for equal scores, whoever reached it first ranks higher (a technique of mixing a timestamp into the fractional part)

**Afternoon (2.5h)**
- `L4-C-22` Ranking API: score update (reflecting game results), top 100, my rank and nearby ranks
- `L4-C-23` Game result save API (internal): record the `game_result` table and update the ranking together. **Idempotency key = gameId**

**1 Hour Without AI**
- Write the procedure for recovering the ranking from the DB if Redis restarts and the ranking is lost

**DoD**
- [ ] All 3 ranking features + idempotent game-result saving work
- [ ] Ranking recovery procedure documented

#### Day 69 (Thu) — Server-to-Server Integration 🔴

**Morning (2.5h)**
- Calls from the game server (3-1) to the API server: internal-only endpoints, **server-to-server authentication** (shared secret header or internal token), blocking external exposure
- Failure handling: a retry queue (on the game server side), exponential backoff, duplicate prevention (gameId idempotency)
- Verifying the game server's login token: calling the API vs querying Redis directly — comparing the trade-offs

**Afternoon (2.5h)**
- `L4-C-24` Attach an HTTP client to the 3-1 server: send results when a game ends, and on failure keep them in a local queue for retry
- `L4-C-25` Demonstrate the integration: play one game → save the result → track it through to the ranking update via logs
- Internal API auth test: calling without the secret → 401

**1 Hour Without AI**
- Summarize 4 reasons "what goes wrong if the game server hits the DB directly"

**DoD**
- [ ] Logs showing a game result flowing through the API into the DB and ranking
- [ ] The internal API cannot be called externally (auth test)

#### Day 70 (Fri) — Batch Worker + Weekly Checkpoint

**Morning (3h)**
- `L4-C-26` Batch worker: clean up expired mail (delete/archive past the expiry date), attendance reset (midnight), 🟡 season-end snapshot
- Scheduling: a separate console app + timer (without depending on the OS scheduler), preventing duplicate runs (locking)

**Afternoon (4h) — Weekly Checkpoint**
1. Reimplement without AI (90 min): the shop purchase transaction + passing the concurrency test
2. Oral exam (45 min): the 3 defenses against double-spending, idempotency, ranking consistency
3. Code review (45 min): Rubric items 1 and 4 (correctness, error handling)
4. Retrospective (30 min): `W14.md`

**DoD**
- [ ] Batch worker works (including duplicate-run prevention)
- [ ] All 10 features of 4-1 work

### 2.4 Week 15 — Security, Performance, Evaluation

#### Day 71 (Mon) — Identifying Abuse Scenarios (Assignment 4-2) 🔴

**Morning (2.5h)**
- Attacker-mindset checklist: currency duplication, duplicate claims, token theft/reuse, replay, parameter tampering (negative quantities, other users' IDs), races, enumeration (checking account existence), mass requests
- "Gacha Probability Implementation Guide" Chapter 16 (incident cases), API Game Server Lab Appendix G (security checklist)

**Afternoon (2.5h)**
- `L4-C-27` Identify **8 or more** abuse scenarios with AI (prompt in §6.1) → document a reproduction method and a defense method for each
- Reproduce 5 of them **as tests**: fail before defense (confirming the vulnerability) → pass after defense, leaving a commit history

**1 Hour Without AI**
- List every one of your APIs that can be called without authentication, and write the reasoning for why each is safe

**DoD**
- [ ] List of 8+ scenarios, 5+ reproduced as tests
- [ ] Before/after defense commit history

#### Day 72 (Tue) — Implementing Defenses and Rate Limiting 🔴

**Morning (2.5h)**
- Input validation: length, range, type, state (negative quantities, other users' userId, nonexistent products)
- SQL injection: **a full audit of parameter binding** (zero string concatenation)
- Rate limiting: limiting request counts per account/per IP (Redis counter + TTL), 429 responses

**Afternoon (2.5h)**
- `L4-C-28` Implement defenses and retest: confirm all 5 scenarios are now blocked
- `L4-C-29` Implement and test rate limiting: no impact on normal users, 429 for burst requests

**1 Hour Without AI**
- Fully audit logs to confirm no sensitive information (passwords, tokens, personal info, etc.) is leaked

**DoD**
- [ ] `SECURITY-SCENARIOS.md` complete
- [ ] 100% parameter binding, rate limiting works

#### Day 73 (Wed) — Performance Verification 🔴

**Morning (2.5h)**
- Measure login API p99: 100 concurrent requests, with **(SQLite)** 10,000 users / **(MySQL)** 1,000,000 users loaded
- Identify bottlenecks: hashing cost (intentionally expensive), connection pool, indexes, serialization

**Afternoon (2.5h)**
- `L4-C-30` Measure and tune performance: connection pool size, applying caching, query improvements. Record before/after in `PERF-4-1.md`
- Caution: lowering hashing parameters to improve p99 is a **security regression**. Document this trade-off explicitly

**1 Hour Without AI**
- Write down 3 optimizations you must not make just to lower p99

**DoD**
- [ ] Login p99 at or below 100ms (or, if not met, root-cause analysis)
- [ ] `PERF-4-1.md` complete (execution plan before/after + p99)

#### Day 74 (Thu) — Consolidating Integration Tests and 🟡 Deep Dives

**Morning (2.5h)**
- Fill out 30 integration tests: use the real DB/Redis, a dedicated test schema/key prefix, isolation between tests (cleanup before each test)
- Organize test data fixtures, reduce execution time (leverage SQLite in-memory option)

**Afternoon (2.5h)**
- 🟡 `L4-CS-02`/`L4-CPP-02` Choose one deep-dive for Assignment 4-3: admin CLI / gacha / **MySQL migration**
- **MySQL migration (4-3c) is recommended**: add a `Data.MySql` implementation, swap only the configuration, and pass the entire test suite → feel the value of abstraction

**1 Hour Without AI**
- Find where integration tests contaminate each other and improve the isolation approach

**DoD**
- [ ] 30 integration + 3 concurrency + 5 auth tests pass
- [ ] Progress on the chosen 🟡 4-3 option recorded

#### Day 75 (Fri) — Phase 4 Evaluation 🔴

**Morning (3h)**
1. **Reimplementation exam (90 min, no AI)**: §8.2 — a "currency deduction + item grant" transaction API + a concurrency test (50 concurrent → exactly 1 succeeds)
2. **Oral exam (60 min)**: 10 questions from §8.3
3. **Bug hunt (45 min)**: §8.4 — 4 out of 5 among missing transactions, N+1, unused indexes, missing token verification, races

**Afternoon (3h)**
- Review the checklist (§8.1), fill any gaps
- `W15.md` + Phase 4 retrospective
- §10 Preparing for Phase 5 (standardizing log fields, preparing metrics)

**DoD**
- [ ] All items in §8.1, oral exam average of 4.0 or above
- [ ] 3-1 integration demonstration log submitted

---

## 3. Lab Catalog

### 3.1 Common Labs (L4-C)

#### L4-C-01 Bring Up a Minimal API Server (45 min) 🔴
- **Steps**: implement a health check (`GET /health`) + version check (`POST /version` → compares against the minimum supported version), call them via a `.http` file
- **Acceptance criteria**: both endpoints return 200 and the `.http` file is in the repository
- **Note**: why the health check must not require authentication

#### L4-C-02 ERD and DDL (90 min) 🔴
- **Steps**: an ERD (Mermaid) with 12+ tables → write the DDL. Note SQLite types (`INTEGER/TEXT/REAL/BLOB`) alongside their MySQL equivalents
- **Acceptance criteria**: running the migration creates an empty DB
- **Common mistake**: SQLite has loose type affinity → compensate with `CHECK` constraints

#### L4-C-03 Index Design (60 min) 🔴
- **Steps**: first write down the 5 key queries → design indexes for each query → add a purpose comment in the DDL
- **Acceptance criteria**: every index has a "this is for this query" comment. Zero unnecessary indexes
- **Note**: criteria for deciding composite index column order (selectivity, position of range conditions)

#### L4-C-04 Define Repository Interfaces (60 min) 🔴
- **Steps**: define `IUserRepository` and 4 others + `IUnitOfWork` per the skeleton in §7.2. **SQL types must not leak into the interface**
- **Acceptance criteria**: the `Contracts`/`Data` projects reference no SQLite/MySQL package (proven by build)

#### L4-C-05 Signup API (90 min) 🔴
- **Steps**: input validation → hashing (Argon2id/bcrypt) → insert into `account` → create initial `user`/`user_currency`/`attendance` records (**one transaction**)
- **Acceptance criteria**: duplicate ID returns 409/error code, no partial creation (rollback confirmed)
- **Note**: why unique violations should be handled via an exception rather than "checking first with SELECT" (race condition)

#### L4-C-06 Login API (60 min) 🔴
- **Steps**: look up account → verify hash → issue token → update last login time
- **Acceptance criteria**: responses for a wrong password and a nonexistent account are **indistinguishable** (prevents account enumeration), and response times are also similar

#### L4-C-07 Redis Session (75 min) 🔴
- **Steps**: `session:{token} → userId` (TTL 3600), `user_session:{userId} → token`. Delete the previous token on login
- **Acceptance criteria**: 401 when requesting with the previous token after a duplicate login

#### L4-C-08 Auth Middleware (60 min) 🔴
- **Steps**: parse header → look up in Redis → sliding TTL renewal → inject userId into context. Manage a list of paths excluded from auth
- **Acceptance criteria**: 5 auth tests pass, excluded paths are managed as an explicit list

#### L4-C-09 Ranking ZSET (60 min) 🔴
- **Steps**: `ZADD rank:season:1 score userId`, top 100 (`ZREVRANGE ... WITHSCORES`), my rank (`ZREVRANK`), ±5 around me
- **Acceptance criteria**: query response under 10ms with 10,000 dummy users
- **Note**: how did you implement the tie-breaking rule (whoever reached it first wins)

#### L4-C-10 Cache Pattern (60 min)
- **Steps**: check cache on profile read → on a miss, query the DB and fill the cache → invalidate on profile change. Mitigate cache stampedes (a short lock or jittered TTL)
- **Acceptance criteria**: cache hit-rate logs, no missed invalidation (test that changes are immediately reflected)

#### L4-C-11 Lua Atomic Stock Deduction (45 min)
- **Steps**: use `EVAL` to atomically "check stock + deduct." Return 0 on insufficient stock
- **Acceptance criteria**: with 100 concurrent requests and 10 units of stock, exactly 10 succeed

#### L4-C-12 Reproduce Isolation-Level/Locking Phenomena (60 min) 🔴
- **Steps (MySQL)**: reproduce non-repeatable reads/phantoms with two sessions, observe the difference after changing isolation levels
- **Steps (SQLite)**: attempt another write during a write transaction → observe `SQLITE_BUSY` → confirm reads are still possible in WAL mode
- **Acceptance criteria**: reproduction logs + pointing out one place in your own API where this phenomenon would matter

#### L4-C-13 Retry on Lock Conflict (45 min)
- **Steps**: on conflict, exponential backoff (50ms→100ms→200ms), up to 3 retries, a clear error code once exceeded
- **Acceptance criteria**: success rate recovers via retries under concurrent load, no infinite retries

#### L4-C-14 User Data Load API (60 min) 🔴
- **Steps**: return basic info, currency, inventory summary, and unread mail count in one call. Log the query count to confirm there's no N+1
- **Acceptance criteria**: 4 or fewer queries, response time recorded

#### L4-C-15 Inventory API (75 min) 🔴
- **Steps**: list (paginated), use (deduct quantity — prevent negatives with a **conditional UPDATE**, delete if it reaches 0)
- **Acceptance criteria**: using more than the available quantity is rejected, two concurrent uses cannot go negative

#### L4-C-16 Dummy Data Generation (60 min) 🔴
- **Steps**: batch INSERTs (in batches of 1,000 rows), grouped into transactions. **(SQLite)** 10,000 users / 100,000 inventory rows; **(MySQL)** 1,000,000 / 10,000,000
- **Acceptance criteria**: generation time recorded, rerunnable (idempotent or has a reset option)
- **Note**: why is it faster to create indexes afterward

#### L4-C-17 Execution Plan Analysis (75 min) 🔴
- **Steps**: run `EXPLAIN QUERY PLAN` (SQLite) / `EXPLAIN` (MySQL) on the 5 key queries → confirm full scans → add indexes → compare before/after time
- **Acceptance criteria**: zero full scans, a before/after table (query, before time, after time, index used)

#### L4-C-18 Shop Purchase Transaction (90 min) 🔴
- **Steps**: look up product → conditionally deduct currency → grant item → record purchase history (one transaction). For stocked products, include stock deduction
- **Acceptance criteria**: affected-row-count check, full rollback on failure, before/after balance recorded in the history

#### L4-C-19 Test 100 Concurrent Purchases (60 min) 🔴
- **Steps**: 100 concurrent requests from the same account (`Task.WhenAll` or 100 threads) → confirm exactly 1 success
- **Acceptance criteria**: always exactly 1 success across 10 repetitions, currency deducted exactly once, exactly 1 item
- **Common mistake**: the test appears to pass because it actually runs sequentially → use a real simultaneous-start barrier

#### L4-C-20 Mailbox API (75 min) 🔴
- **Steps**: list (paginated), claim (grant attachments + change status, one transaction, `WHERE status='unread'`), expiry flag
- **Acceptance criteria**: only 1 succeeds out of 50 concurrent claims

#### L4-C-21 Attendance API (60 min) 🔴
- **Steps**: once per day based on midnight (KST), grant a reward, track consecutive attendance count
- **Acceptance criteria**: 23:59:59 and 00:00:01 boundary tests pass, only 1 succeeds out of 50 concurrent checks

#### L4-C-22 Ranking API (60 min) 🔴
- **Steps**: score update, top 100, my rank, ranks around me. Use a season key
- **Acceptance criteria**: rank changes are reflected immediately after a game result is applied

#### L4-C-23 Save Game Result (Internal API) (60 min) 🔴
- **Steps**: record `game_result` + update the ranking. Use **gameId as the idempotency key** to prevent duplicate saves
- **Acceptance criteria**: sending the same gameId 3 times results in only 1 saved record and the ranking updated only once

#### L4-C-24 Attach an HTTP Client to the Game Server (75 min) 🔴
- **Steps**: implement 3-1's `IGameResultSink` as an HTTP call. On failure, keep it in a local queue and retry (exponential backoff)
- **Acceptance criteria**: the game server does not crash even if the API server is briefly stopped, and queued results are sent after it restarts

#### L4-C-25 Integration Demonstration (45 min) 🔴
- **Steps**: play one game → save the result → trace it through to the ranking update via logs (filtered by gameId)
- **Acceptance criteria**: the game server log → API log → DB/Redis result form a single traceable line

#### L4-C-26 Batch Worker (75 min)
- **Steps**: clean up expired mail, reset attendance, 🟡 season-end snapshot. Prevent duplicate execution (Redis lock or a DB flag)
- **Acceptance criteria**: even running two instances simultaneously, the job runs only once

#### L4-C-27 Identify Abuse Scenarios (90 min) 🔴
- **Steps**: identify 8+ scenarios using the §6.1 prompt → document reproduction method, defense, and severity for each → reproduce 5 as tests (confirm failure before defense)
- **Acceptance criteria**: a commit history showing failure before defense → passing after defense

#### L4-C-28 Implement Defenses (90 min) 🔴
- **Steps**: strengthen input validation, fully audit parameter binding, authorization checks (block access to other users' resources), apply idempotency
- **Acceptance criteria**: all 5 scenarios are blocked, zero string-concatenated SQL

#### L4-C-29 Rate Limiting (60 min)
- **Steps**: a Redis counter (`ratelimit:{userId}:{second}` INCR + TTL) or a token bucket. 429 + `Retry-After` on excess
- **Acceptance criteria**: no impact on normal users, only bursting accounts are blocked

#### L4-C-30 Performance Measurement and Tuning (90 min) 🔴
- **Steps**: measure login API p99 with 100 concurrent requests → identify bottlenecks (hashing/connection pool/queries) → tune → remeasure
- **Acceptance criteria**: before/after table, explicit documentation that "performance was not gained by lowering security"

### 3.2 C# Track Labs (L4-CS)

#### L4-CS-01 SQLite Implementation + DI Registration (90 min) 🔴
- **Skeleton**
  ```csharp
  // Program.cs
  builder.Services.AddSingleton<IDbConnectionFactory>(_ =>
      new SqliteConnectionFactory(cfg.GetConnectionString("Db")!));   // default
  // 🟡 Switch to MySQL: replace with new MySqlConnectionFactory(...) only
  builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
  builder.Services.AddScoped<IUserRepository, UserRepository>();
  ```
  Add `Cache=Shared` to the connection string, and at startup run `PRAGMA journal_mode=WAL; PRAGMA busy_timeout=3000;`
- **Acceptance criteria**: integration tests pass against a real SQLite file (or a temp file)
- **Note**: the problems that arise if `IDbConnection` lifetime is Scoped, and how the factory pattern solves them

#### L4-CS-02 Add a MySQL Implementation (🟡 4-3c, 120 min)
- **Steps**: add a `Data.MySql` project → implement the same interface (SqlKata/Dapper) → swap only the DI registration → make the **entire test suite pass as-is**
- **Acceptance criteria**: zero lines of application code changed, 100% of tests pass, a comparison table of execution plans across both DBs

#### L4-CS-03 WebApplicationFactory Integration Tests (75 min) 🔴
- **Steps**: bring up a test server + a dedicated test DB file/schema + a Redis key prefix + reset before each test
- **Acceptance criteria**: 30 tests can run in parallel without contaminating each other (or, if sequential, isolation is still guaranteed)

#### L4-CS-04 How to Write Concurrency Tests (45 min) 🔴
- **Steps**: use a `Barrier` or `TaskCompletionSource` to launch 100 requests **at exactly the same time**
- **Acceptance criteria**: temporarily removing the defense code makes the test fail (evidence that the test truly creates a race)

### 3.3 C++ Track Labs (L4-CPP)

#### L4-CPP-01 SQLite Access Layer (120 min) 🔴
- **Steps**: connect via vcpkg `sqlite3` (or `SQLiteCpp`), prepared statements, an RAII transaction wrapper, configure WAL/busy_timeout
- **Skeleton**
  ```cpp
  class SqliteConnection {
  public:
      explicit SqliteConnection(const std::string& path);   // configures WAL, busy_timeout
      Statement Prepare(std::string_view sql);
      Transaction Begin();                                   // RAII: rolls back on destruction if not committed
  };
  ```
- **Acceptance criteria**: prove with a test that the transaction RAII rolls back on an exception path

#### L4-CPP-02 Add a MySQL Implementation (🟡, 150 min)
- **Steps**: implement the same interface using MySQL Connector/C++ + a connection pool (thread-safe, with validity checks)
- **Acceptance criteria**: swapped behind the interface, tests pass

#### L4-CPP-03 HTTP Server and Routing (120 min) 🔴
- **Steps**: routing with cpp-httplib, JSON (nlohmann), standardizing error responses, request context (injecting userId)
- **Acceptance criteria**: all 20 4-C endpoints are routed, the auth filter works

#### L4-CPP-04 Integration Tests (90 min) 🔴
- **Steps**: GoogleTest + real SQLite/Redis + calls via the cpp-httplib client, 100 concurrent threads
- **Acceptance criteria**: 3 concurrency tests pass

---

## 4. Detailed Learning Items

### 4.1 Common (100h)

**HTTP/REST and Game API Design (12h)**
- What: methods, status codes, headers, JSON, REST vs command-style, the game API flow, the error code system, versioning, DTOs
- Why: game APIs have distinctive patterns such as "bulk load after login" and "currency deduction"
- How: `L4-C-01` → write the 4-C spec → update the spec while implementing
- Verification: 20 endpoint specs + an error code table

**Authentication and Authorization (10h)**
- What: hashing (Argon2/bcrypt/PBKDF2), tokens vs sessions, issuance/verification/expiry/renewal, duplicate logins, server-to-server authentication
- Why: the first line of defense against account takeover and currency duplication
- How: `L4-C-05/06` → `L4-C-07/08` → 5 auth tests
- Verification: all tests pass for expired, forged, others'-token, unauthenticated, and duplicate-login cases

**DB Design and Queries (24h)**
- What: normalization/denormalization decisions, key design, indexes (B-tree, composite, covering), execution plans, JOINs, pagination, migrations, sharding concepts, **the differences between SQLite and MySQL** (file vs server, serialized writes, type affinity)
- Why: slow queries are the number one cause of live-service outages
- How: `L4-C-02/03` → `L4-C-16` dummy data → `L4-C-17` execution plans
- Verification: execution plan before/after table, zero full scans

**Transactions and Concurrency (20h)**
- What: ACID, isolation levels and their phenomena, locks (record/gap/next-key, or SQLite's write lock), deadlocks and retries, conditional UPDATE, optimistic locking, idempotency keys, the limits of Redis distributed locks
- Why: the "100 concurrent purchases" condition is the core evaluation of this Phase
- How: `L4-C-12/13` reproduction → `L4-C-18/19` purchases → `L4-C-20/21` idempotency
- Verification: 3 concurrency tests (purchase, mail, attendance) pass

**Using Redis (12h)**
- What: uses per data structure, TTL, pipelining, Lua atomicity, persistence (RDB/AOF), cache invalidation, "what must not go here"
- Why: a large share of a game server's real-time data lives in Redis
- How: `L4-C-07` (session) → `L4-C-09` (ranking) → `L4-C-10` (cache) → `L4-C-11` (Lua)
- Verification: 3 ranking features, session TTL renewal, Lua stock deduction

**Server Integration and Batch Jobs (8h)**
- What: internal APIs and server-to-server authentication, failure retries and duplicate prevention, batch jobs and preventing duplicate execution
- Why: training in defining the boundary between the real-time server and the data server
- How: `L4-C-23/24/25` → `L4-C-26`
- Verification: logs showing game results flowing through the API into the DB and ranking

**Integration Testing (8h)**
- What: using the real DB/Redis, test isolation (dedicated schema/file/key prefix), how to write concurrency tests, fixtures
- Why: transactions and locks cannot be verified with mocks
- How: `L4-CS-03`/`L4-CPP-04` → `L4-CS-04` (simultaneous-start barrier)
- Verification: 30 integration tests + 3 concurrency tests

**Security and Abuse Scenarios (6h)**
- What: input validation, SQL injection defense, rate limiting, currency duplication, duplicate claims, token theft, replay, sensitive info in logs
- How: `L4-C-27/28/29`
- Verification: `SECURITY-SCENARIOS.md`, defenses proven for 5 scenarios

### 4.2 C# Track / Mixed Path (60h)

| Item | Hours | What | Learning order | Verification |
|---|---|---|---|---|
| ASP.NET Core structure | 12h | middleware pipeline, DI lifetimes, controllers vs Minimal API, model binding/validation, filters, configuration, health checks | L4-C-01 → L4-C-08 | experiment with reordering middleware to break authentication |
| DB access | 14h | Repository interfaces, **default SQLite implementation** (Microsoft.Data.Sqlite), transactions/UnitOfWork, 🟡 swapping in a MySQL implementation (MySqlConnector+SqlKata) | L4-CS-01 → L4-C-18 → 🟡L4-CS-02 | shop purchase transaction, zero application code changes on DB swap |
| Redis access | 8h | StackExchange.Redis or CloudStructures, ZSet/Hash, running Lua, connection management | L4-C-07/09/11 | ranking, session |
| Auth middleware | 8h | custom token verification, context injection, auth-excluded paths | L4-C-08 | 5 auth tests |
| Testing | 10h | `WebApplicationFactory`, fixtures, concurrency tests (Barrier) | L4-CS-03 → L4-CS-04 | 30 integration + 3 concurrency |
| Logging/configuration | 8h | **Serilog** request logs (request ID, userId, elapsed time, result code), environment separation, masking sensitive info | ongoing from day 59 | log samples, token masking confirmed |

### 4.3 C++ Track (a) Pure C++ (60h)

| Item | Hours | What | Learning order | Verification |
|---|---|---|---|---|
| HTTP server | 14h | cpp-httplib (or Drogon), routing, JSON, request context, error format | L4-CPP-03 | 20 endpoints |
| DB access | 16h | **default SQLite** (sqlite3/SQLiteCpp), prepared statements, RAII transactions, 🟡 MySQL Connector/C++ + connection pool | L4-CPP-01 → 🟡L4-CPP-02 | purchase transaction, rollback on exception |
| Redis | 8h | redis-plus-plus, pipelining, Lua, connection pool | L4-C-07/09/11 | ranking, session |
| Auth | 6h | token generation (random + HMAC), verification filter, libsodium argon2 | L4-C-05~08 | auth tests |
| Testing | 10h | GoogleTest + real DB/Redis, HTTP client, 100 concurrent threads | L4-CPP-04 | 3 concurrency tests |
| Build/configuration | 6h | vcpkg manifest, config files, spdlog request logs | ongoing | local build/test scripts pass |

### 4.4 C++ Track (b) Mixed Path

- 20h in the first week of week 12: read through "Essential C# Guide for ASP.NET Core Web API" + set up a C# project and learn async basics
- After that, same as §4.2 (compressed to an effective 40h, with logging/configuration items minimized)
- Additional requirement: implement, in the C++ game server (3-1), the HTTP client that calls the API directly (cpp-httplib client or WinHTTP) — `L4-C-24`

---

## 5. Book Usage Guide (Phase 4)

### 5.1 Common

| Book | Chapters to read | When | How to use |
|---|---|---|---|
| Learn MySQL C# Programming in a Week | Day 1 (DBMS, InnoDB), Day 2 (SQL, table design), Day 5 (advanced queries, indexes) | Days 56-65 | Since this course defaults to SQLite, **take only the SQL syntax and design concepts and port the connection code to SQLite**. The MySQL installation/Workbench section of Day 1 applies only if you choose MySQL. Practice Day 5's indexes using `EXPLAIN QUERY PLAN`. Print out the Appendix A cheat sheet |
| Learn Redis Programming in a Week | Ch.1 (installation — the redis-windows section), Ch.2 (internal structure, data types) | Day 62 | Skip the Docker/WSL2/Memurai sections and install via redis-windows. Extend the Ch.2 data-structure table into a "is this data DB or Redis" decision table |
| Building an API Game Server with ASP.NET Core Web API | Appendix A (design doc), B (API spec guide), C (schema), F (custom tokens), G (security checklist) | Days 56-60 | **Direct reference for writing the 4-C spec.** Follow Appendix B's format but rewrite it for the Omok lobby. Port Appendix C's MySQL schema to SQLite DDL. Appendix G is the 4-2 checklist |
| Practical Guide to Implementing Gacha Probability for New Game Programmers | Ch.9 (server-side draw principles), Ch.10 (data design), **Ch.11 (currency deduction atomicity)**, **Ch.12 (race conditions)**, Ch.16 (incident cases) | Days 66-67 | **Chapters 11-12 are the best material on defending against double-spending.** Read them right before designing the shop purchase flow. Ch.16 is source material for the 4-2 abuse scenarios |
| Network Learning Roadmap for Online Game Developers | The authentication section of Ch.6 (security) | Day 61 | Rationale for token design |

### 5.2 C# Track / Mixed

| Book | Chapters to read | When | How to use |
|---|---|---|---|
| Essential C# Guide for ASP.NET Core Web API | Entire book | First week of week 12 (required for mixed path; review for C#) | Type out and build the examples in sections 12 and 27 directly |
| Building an API Game Server with ASP.NET Core Web API | Ch.1-3, Ch.4-7 (DB design, integration, CRUD), Ch.8-11 (tokens, login, middleware), Ch.12-14 (Redis), Ch.16-18 (inventory, mail, shop), Ch.20 (attendance), Ch.22 (ranking) | Days 56-70 | **Main textbook.** There's no code folder, so type out the text as you go and **rework the collection-RPG domain into the Omok lobby**. Replace the MySQL integration code in Ch.4-7 with SQLite behind the Repository; use the original if you chose MySQL |
| Building a Game Server with ASP.NET Core Web API | Ch.5 (DB integration), Ch.6 (Redis), Ch.7 (authentication), Ch.8 (data API), Ch.13 (async processing), Ch.16 (security) | Days 61-72 | Supplementary. Upgrade `codes/GameAPIServer_Template_01` to net10.0, build it, and write a project-structure ADR based on the Controllers/Repository/Services layout. Ch.17 (deployment) is out of scope for this course |
| Learn MySQL C# Programming in a Week | Day 3 (C# integration), Day 4 (SqlKata), Day 6 (game content), Day 7 (performance/operations) | Days 58-70 | Day 6 closely matches the 4-1 requirements. Port the connection code to SQLite; apply Day 7's performance/operations content only if you chose MySQL |
| Learn Redis Programming in a Week | Ch.3 (sessions, duplicate logins), 4 (caching), 5 (ZSet rankings), 6 (Pub/Sub — preview for Phase 6), 7 (pipelining, monitoring) | Days 61-68 | Ch.3 and 5 are direct references for 4-1 |
| Building a 2D MMORPG Game Server | Week 2 (API server & accounts) | Day 69 | Real code (net10.0) for a structure separating the game server and API server. Reflect "how the game server calls the API" into the 3-1 integration |

### 5.3 C++ Track (a)

The repository has no C++ API server book. Use official documentation instead.
- cpp-httplib README, Drogon docs 🟡
- **SQLite official documentation** (sqlite.org: WAL, busy_timeout, transactions) / 🟡 the MySQL Connector/C++ guide
- redis-plus-plus README
- Read the SQL/data-structure portions of the common books the same way, and treat "how would this be done with C++ libraries" as the translation exercise for the C# code portions

### 5.4 Supplementary Track Reading Assignment

- C# main track: Gacha guide Ch.11-12 → summarize "3 patterns of currency deduction atomicity" (already a common assignment)
- C++ main track (path a): API Game Server Lab Ch.8 (custom token design) → a 1-page comparison against your own C++ token implementation

---

## 6. AI Collaboration Guide (Phase 4 Specific)

### 6.1 Prompts

**Schema Review**
```
Here is the schema for my Omok lobby API server (default DB: SQLite, optional: MySQL). (paste the DDL)
1) Predict which queries would slow down at a scale of 100 million users, 1 billion inventory rows, and 500 million mail records (API spec attached).
2) Tell me how to verify each prediction using an execution plan.
3) Suggest indexes, and explain why that column order.
I will insert dummy data, measure before/after, and bring back the results. Do not jump to conclusions first.
```

**Identifying Abuse Scenarios**
```
Look at the following API spec from an attacker's perspective. (attached)
From the angles of currency duplication, duplicate claims, token theft, replay, parameter tampering, race conditions, account enumeration, and mass requests,
list at least 8 abuse scenarios.
For each scenario, separately write a "reproduction method" and a "defense method," and rate severity (high/medium/low).
I will reproduce 5 or more as tests and implement the defenses.
```

**Transaction Boundary Review**
```
Here is the shop purchase service code. (paste it)
Review by simulating the time ordering of two requests whether currency is deducted exactly once when 100 requests arrive concurrently.
Point out any spots where the result would differ between SQLite (serialized writes) and MySQL (REPEATABLE READ).
Also check whether any operations that should not be inside a transaction have been included.
```

**Data Placement Decisions**
```
Decide, with reasoning, whether each of the following items should go in the DB (SQLite/MySQL), Redis, or both:
gold balance, session token, ranking, mailbox, attendance record, online user list, shop stock, game result history, idempotency keys.
My own decisions are as follows: (my table). Point out only where you disagree, and skip the ones I got right.
```

**DB Abstraction Review**
```
Here is my Repository interface and its SQLite implementation. (paste it)
Point out anything that would cause problems when adding a MySQL implementation behind this interface (SQLite-specific syntax, types, differences in transaction behavior).
Flag any place where the interface leaks details specific to a particular DB.
```

### 6.2 What to Delegate / What Not to Delegate

| Delegate | Do not delegate |
|---|---|
| Controller/DTO/repository boilerplate | Deciding transaction boundaries and isolation levels |
| Dummy data generation scripts | Final decisions on schema/indexes (verify by measurement) |
| A draft list of abuse scenarios | Implementing defenses and tests |
| `.http` test files, migration script drafts | The error code system, data placement decisions |

### 6.3 Places Where AI Often Gets It Wrong

- Code that deducts currency using two queries — **`SELECT` to check balance, then `UPDATE`** (a race) → use a conditional UPDATE or `FOR UPDATE`
- Examples that hash passwords with **SHA256+salt** → should be Argon2id/bcrypt/PBKDF2
- Examples that use **JWT without a Redis session** while claiming "logout is implemented" (invalidation is impossible)
- Suggesting indexes on every column → push back by asking about write cost and index maintenance cost
- Presenting Redis distributed locks as a cure-all → push back by asking about lock expiry, client death, and clock issues
- Mixing **MySQL syntax into SQLite** (e.g., `ON DUPLICATE KEY UPDATE`) → SQLite uses `ON CONFLICT DO UPDATE`
- Code that puts HTTP calls or Redis calls inside a transaction → the transaction's duration becomes tied to external latency

---

## 7. Assignment Specifications

### 7.1 Common Assignment 4-C. Schema and API Specification (Required, 16h, days 57-60)

**Deliverable 1 — `SCHEMA.md`**

1. **ERD** (Mermaid)
2. **Per-table DDL** (based on SQLite, with MySQL types noted alongside). Minimum tables:

| Table | Key columns | Notes |
|---|---|---|
| `account` | account_id(PK), login_id(UNIQUE), password_hash, salt, created_at, last_login_at | authentication only |
| `user` | user_id(PK), account_id(FK,UNIQUE), nickname(UNIQUE), level, exp, created_at | game profile |
| `user_currency` | user_id(PK), gold, gem, updated_at | currency ledger |
| `item_master` | item_id(PK), name, type, stackable, max_stack | master data |
| `user_item` | user_item_id(PK), user_id, item_id, count, acquired_at | (user_id, item_id) index |
| `mail` | mail_id(PK), user_id, title, body, status, expire_at, created_at | (user_id, status) index |
| `mail_attachment` | mail_id, slot, item_id, count, currency_type, currency_amount | composite PK |
| `attendance` | user_id(PK), last_check_date, streak_count | once per day |
| `shop_product` | product_id(PK), item_id, price_currency, price_amount, stock, is_active | stock nullable |
| `purchase_log` | purchase_id(PK), user_id, product_id, price_amount, gold_before, gold_after, created_at | audit trail |
| `game_result` | game_id(PK, externally generated), room_id, winner_user_id, loser_user_id, reason, played_at | idempotency key |
| `idempotency_key` | key(PK), user_id, endpoint, response_json, created_at | subject to TTL cleanup |

3. **Index list**: a "this is for this query" comment on each index
4. **DB vs Redis Placement Decision Table**

| Data | Store | Rationale |
|---|---|---|
| Gold balance | DB | a ledger that must not be lost, needs transactions |
| Session token | Redis | can be volatile, needs TTL, queried frequently |
| Ranking | Redis(ZSET) + DB snapshot | real-time ordering + recoverability |
| Mailbox | DB | needs persistence |
| Attendance record | DB | persistence, audit |
| Online user list | Redis | volatile |
| Shop stock | DB(ledger) + Redis(Lua deduction) | consistency + performance |
| Idempotency key | Redis(TTL) or DB | convenient expiry management |

**Deliverable 2 — `API.md`** (20 or more endpoints)

| # | Endpoint | Auth | Idempotent | Description |
|---|---|---|---|---|
| 1 | `GET /health` | X | - | health check |
| 2 | `POST /version` | X | O | client version check |
| 3 | `POST /account/signup` | X | X | signup |
| 4 | `POST /account/login` | X | X | login, issue token |
| 5 | `POST /account/logout` | O | O | delete session |
| 6 | `POST /account/token/refresh` | O | X | refresh token |
| 7 | `GET /user/data` | O | O | one bulk load after login |
| 8 | `GET /user/profile` | O | O | profile |
| 9 | `GET /inventory?cursor=&limit=` | O | O | paginated inventory |
| 10 | `POST /inventory/use` | O | X | use item (deduct quantity) |
| 11 | `GET /mail?cursor=&limit=` | O | O | mail list |
| 12 | `POST /mail/receive` | O | **O (key required)** | claim mail |
| 13 | `POST /attendance/check` | O | **O** | attendance check |
| 14 | `GET /shop/products` | O | O | product list |
| 15 | `POST /shop/purchase` | O | **O (key required)** | purchase (transaction) |
| 16 | `GET /rank/top?count=100` | O | O | top ranking |
| 17 | `GET /rank/me` | O | O | my rank and nearby |
| 18 | `GET /notice` | X | O | notice |
| 19 | `POST /internal/game/result` | server-to-server | **O (gameId)** | save game result |
| 20 | `POST /internal/token/verify` | server-to-server | O | token verification by the game server |
| 21 | `GET /internal/health/deep` | server-to-server | O | check DB/Redis connectivity |

For each endpoint, write the **request schema / response schema / possible error codes / idempotency** in a table.

**Deliverable 3 — Error Code Table**

| Code | HTTP | Name | Client action |
|---|---|---|---|
| 0 | 200 | Success | - |
| 1001 | 400 | InvalidRequest | check input |
| 1002 | 401 | Unauthenticated | log in again |
| 1003 | 403 | Forbidden | notify that access is denied |
| 1004 | 404 | NotFound | - |
| 1005 | 429 | RateLimited | wait and retry |
| 1006 | 500 | InternalError | retry |
| 2001 | 409 | DuplicateLoginId | use a different ID |
| 2002 | 401 | InvalidCredential | check ID/password (not distinguished) |
| 2003 | 401 | SessionExpired | log in again |
| 3001 | 400 | InsufficientCurrency | notify of insufficient currency |
| 3002 | 400 | ItemNotFound | - |
| 3003 | 409 | AlreadyReceived | already claimed |
| 3004 | 409 | AlreadyCheckedToday | already checked in |
| 3005 | 409 | OutOfStock | out of stock |
| 4001 | 409 | GameResultDuplicated | ignore (normal) |

**Deliverable 4 — 2 Sequence Diagrams**: (1) login through data load, (2) shop purchase (showing the internal steps of the transaction)

**Deliverable 5 — `adr/0005-db-abstraction.md`**: Repository+DI, defaulting to SQLite / optionally MySQL. Alternatives (using an ORM throughout / calling the DB directly) and reasons for rejecting them

**Grading**: completeness 40 / rationale for data placement decisions 30 / clarity (passing the AI "different-team developer" review) 30

### 7.2 Track Assignment 4-1. Game API Server (Required, 88h, days 58-74)

**Functional Requirements**

0. **DB access via Repository/DAO interfaces + DI**. Default SQLite, and if a 🟡 MySQL implementation is added, **switching by configuration alone**
1. **Account**: signup (validation, hashing), login (token + Redis session TTL 1h), token refresh, logout, invalidate the previous session on duplicate login
2. **Version check, notices, health check** (no auth required)
3. **User data load**: basic info, currency, inventory summary, unread mail count in one call
4. **Inventory**: list (paginated), use (deduct quantity, delete if it reaches 0)
5. **Mailbox**: list, claim (grant attachments, only once), expiry (batch)
6. **Attendance**: once per day, reset at midnight (KST), consecutive attendance count
7. **Shop**: product list, purchase (currency deduction + item grant + history, **a single transaction**), one stocked product
8. **Ranking**: update score from game results (ZSET), top 100, my rank and nearby
9. **Internal API**: save game results (idempotent on gameId), token verification. Server-to-server authentication required
10. **Batch worker**: mail expiry cleanup, attendance reset, duplicate-run prevention

**Repository Interface Skeleton**

```csharp
public interface IUnitOfWork : IAsyncDisposable
{
    Task<ITransaction> BeginAsync(CancellationToken ct = default);
}

public interface IUserRepository
{
    Task<UserRow?> FindByIdAsync(long userId, ITransaction? tx = null, CancellationToken ct = default);
    Task<bool>     TryConsumeGoldAsync(long userId, long amount, ITransaction tx, CancellationToken ct = default);
    //  ↑ returns success/failure via a conditional UPDATE + affected row count. A separate API exists for "check balance".
}

public interface IItemRepository
{
    Task AddItemAsync(long userId, int itemId, int count, ITransaction tx, CancellationToken ct = default);
    Task<bool> TryConsumeItemAsync(long userId, int itemId, int count, ITransaction tx, CancellationToken ct = default);
    Task<IReadOnlyList<UserItemRow>> ListAsync(long userId, long cursor, int limit, CancellationToken ct = default);
}

public interface IMailRepository
{
    Task<bool> TryMarkReceivedAsync(long mailId, long userId, ITransaction tx, CancellationToken ct = default);
    // WHERE mail_id=? AND user_id=? AND status='unread'  → must succeed with an affected row count of 1
}
```
(For C++, use pure virtual interfaces with the same meaning + pass a `Transaction&`)

**Non-Functional Requirements**

| Condition | Default (SQLite) | Optional (MySQL) |
|---|---|---|
| 100 concurrent shop purchases | currency deducted exactly once, exactly 1 item granted | same |
| 50 concurrent mail claims/attendance checks | processed exactly once | same |
| Login p99 | under 100ms with 10,000 users loaded | under 100ms with 1,000,000 users loaded |
| Execution plan | at 10,000/100,000 scale, zero full scans on the 5 key queries | at 1,000,000/10,000,000 scale, same |
| SQL | 100% parameter binding (zero string concatenation) | same |
| Logging | request ID, userId, elapsed time, result code, tokens/passwords masked | same |

**Testing Requirements**

- **30+ integration tests**: 6 account / 5 inventory / 5 mail / 4 attendance / 5 shop / 3 ranking / 3 internal API
- **3 concurrency tests**: 100 purchases, 50 mail claims, 50 attendance checks — each must fail if the defense code is removed (proving the test's validity)
- **5 auth tests**: expired, forged, others'-token, unauthenticated, duplicate login

**Submission**: code, tests, migration scripts, dummy data script, `PERF-4-1.md`, 3-1 integration demonstration log

**`PERF-4-1.md` Template**
```markdown
## 1. Environment (DB type/version, data scale, hardware)
## 2. Execution plan before/after for the 5 key queries
| Query | Before (scan type/time) | Index | After (scan type/time) |
## 3. Login API p99 (100 concurrent)
| Attempt | p50 | p95 | p99 | Failure rate |
## 4. Bottlenecks and fixes (hashing cost is not lowered — explain why)
## 5. Remaining limitations
```

**Grading**

| Item | Points | Criteria |
|---|---|---|
| Functionality | 25 | requirements 0-10 |
| Consistency/idempotency | 25 | 3 concurrency tests pass + proof of failure when defenses are removed |
| Data design/performance | 20 | zero full scans in execution plans, login p99 target met |
| Security | 15 | 5 auth tests, 100% parameter binding, hashing policy, log masking |
| Testing/documentation | 15 | 30 integration tests, `PERF-4-1.md`, integration log |

**Common Mistakes**
- Calling Redis/HTTP inside a transaction → latency ends up holding the transaction hostage
- The concurrency test isn't actually concurrent → use a barrier for a simultaneous start
- Storing the idempotency key but **not returning the previous response** → the second request appears to fail
- `count` in inventory usage can go negative → use a conditional UPDATE + affected row count
- Leaving a long-running transaction open in SQLite causes a surge of `SQLITE_BUSY` → keep transactions short

### 7.3 Common Assignment 4-2. Abuse Scenario Defense Report (Required, 12h, days 71-72)

**Requirements**
1. Identify **8 or more** scenarios with AI (currency duplication, duplicate claims, token theft, replay, parameter tampering, races, account enumeration, mass requests)
2. **Reproduce 5 or more as tests**: fail before defense (proving the vulnerability) → implement defense → pass. Keep this as a commit history
3. `SECURITY-SCENARIOS.md`: scenario / reproduction method / severity / defense method / test link / remaining risk

**Required Example Scenarios**
- Using items with a negative quantity → quantity increases
- Claiming another user's mail by putting their `userId` in the request body
- Sending the same mail-claim request 50 times concurrently
- Reusing an expired token / reusing a token after logout
- Capturing a shop purchase request and resending it as-is (replay)
- Distinguishing whether an ID exists from the response (account enumeration)

**Grading**: reproduction evidence 40 / soundness of defenses 40 / awareness of remaining risk 20

### 7.4 Deep-Dive Assignment 4-3 (Optional 🟡)

- **4-3a Admin Tools (16h)**: user lookup, granting currency, sending mail, posting notices, **audit log** (who did what, when). CLI or a minimal web app
- **4-3b Gacha (16h)**: add one of the roulette/tier-system/pity mechanics from Ch.4-6 of the textbook to the shop. Compare against expected probability using a 100,000-run simulation per Ch.14 (probability testing)
- **4-3c MySQL Migration (8h, recommended)**: add a `Data.MySql` implementation → **swap only the DI registration** → pass the entire test suite. In `MYSQL-MIGRATION.md`, document (1) the list of files changed (2) SQL dialect differences (3) the actual impact of the concurrency model differences (4) an SQLite/MySQL performance comparison table

---

## 8. Learning Completion Assessment (Friday, Day 75)

### 8.1 Checklist

**Assignments**
- [ ] 4-C: `SCHEMA.md` (12 tables, index rationale, placement decision table), `API.md` (20+ endpoints, error codes), ADR
- [ ] 4-1: features 0-10 work, 30 integration + 3 concurrency + 5 auth tests pass
- [ ] 4-2: 8 scenarios, 5 reproduced as tests and defended
- [ ] 🟡 4-3 one option done, or reason for not doing it

**Measurements**
- [ ] `PERF-4-1.md`: execution plan before/after, login p99
- [ ] Only 1 success out of 100 concurrent purchases (stable across 10 repetitions)
- [ ] Log of a 3-1 game result → API → ranking update

**Structure**
- [ ] The domain/contracts projects reference no DB package (proven by build)
- [ ] 100% SQL parameter binding (zero results when searching for string concatenation)

**Records**
- [ ] 20 days of learning notes, 4 weekly retrospectives

### 8.2 Reimplementation Exam (No AI, 90 minutes)

**Problem**: in an empty project (including the DB), implement a single "currency deduction + item grant" API and pass the provided tests.

1. `POST /purchase` — request: `{userId, productId, idempotencyKey}`
2. On success: deduct currency, grant item, record history (one transaction)
3. Provided tests
   - 1 normal purchase
   - rejected on insufficient balance (no changes made)
   - **50 concurrent requests → exactly 1 succeeds**
   - retrying with the same idempotency key → returns the previous result (no duplicate deduction)

**Passing criterion**: pass all 4 within 90 minutes.

### 8.3 Oral Exam Question Bank (10 questions at random, average 4.0)

1. The 4 isolation levels and the phenomena at each level. What is the default for the DB you used, and what does it mean
2. 3 ways to prevent double-spending and the cost of each
3. The criteria for deciding composite index column order. What is a covering index
4. Token vs session. Why is logout hard with JWT alone
5. Why gold balances should not be kept in Redis. Are there exceptions
6. Why the game server goes through the API instead of hitting the DB directly. When is hitting it directly appropriate
7. Where is the idempotency key stored and when is it deleted. What if a retry arrives before deletion
8. Why the DB was abstracted with Repository + DI. Where did the abstraction leak
9. How the concurrency model differences between SQLite and MySQL affected your code
10. What operations must not go inside a transaction, and why
11. Why SHA256 should not be used for password hashing. How did you decide the parameters
12. What did you do to prevent account enumeration
13. How did you keep the ranking in Redis consistent with the DB
14. Where did an N+1 problem occur and how was it solved
15. Did you rate-limit per account or per IP. Why
16. (C#) What happens if a `DbConnection` with Scoped DI lifetime is injected into a Singleton service
17. (C++) When building a connection pool yourself, how did you handle thread safety and connection validity checks

### 8.4 Bug Hunt (45 minutes)

Missing transaction / N+1 / unused index / missing token verification / race — find 4 of 5 and propose fixes.

### 8.5 Response If Standards Are Not Met

| Item not met | Response |
|---|---|
| Concurrency test not passing | **Cannot pass**. Up to a 1-week extension (this is the core competency of this Phase) |
| Login p99 not met | Proceed if root-cause analysis is submitted, remeasure in Phase 5 |
| Full scans remaining in execution plans | Redesign indexes and remeasure; document the rationale for any that remain |
| Fewer than 20 integration tests | Reinforce in week 1 of Phase 5 (refactoring afterward is impossible without tests) |
| Fewer than 3 security scenarios | Redo Assignment 4-2 |

---

## 9. Common Sticking Points

| Symptom | Cause | Response |
|---|---|---|
| Occasional double-deduction in the concurrent purchase test | SELECT and UPDATE done separately | `UPDATE ... SET gold = gold - ? WHERE user_id = ? AND gold >= ?` + check affected row count |
| (SQLite) frequent `database is locked` | long write transactions, WAL not configured | `PRAGMA journal_mode=WAL`, `busy_timeout`, keep transactions short, separate reads/writes |
| (MySQL) intermittent deadlocks | two transactions locking rows in different orders | unify lock acquisition order (user → item), retry |
| Redis sessions disappear | missing TTL renewal, service restart | extend TTL on every request, verify persistence (RDB/AOF) settings |
| Dummy data insertion is too slow | row-by-row INSERTs, indexes already present | batch INSERTs (1,000 rows), group into transactions, create indexes afterward |
| Execution plan isn't using the index | column order, applying a function, type mismatch | put the condition column first, avoid `WHERE DATE(col)`, match types |
| Tests contaminate each other's data | shared DB/keys | dedicated test file/schema + key prefix + reset before each test |
| Integration tests are too slow to run | full migration on every test | copy a template DB or use in-memory, reuse fixtures |
| Duplicates processed even with an idempotency key in place | a race between storing the key and processing | make key insertion a unique constraint and process **only the request whose insert succeeded** |
| Tokens appear in logs | logging the entire request | a masking filter for sensitive fields |
| (C#) configuration doesn't change in `WebApplicationFactory` | environment name | `ASPNETCORE_ENVIRONMENT=Test`, `appsettings.Test.json` |
| Game server integration dies along with an API outage | synchronous calls, no retries | local queue + exponential backoff, set timeouts |

---

## 10. Preparing Before Moving to Phase 5

- [ ] Download the Windows binaries for Prometheus, Grafana, and windows_exporter (installation happens in Phase 5, week 17)
- [ ] Check whether the log fields in 3-1/4-1 are standardized (these become the input to Phase 5 observability): `requestId/sessionId/userId/roomId/gameId/durationMs/resultCode`
- [ ] Confirm you've left room in 4-1 for a `/metrics` endpoint (an HTTP listener)
- [ ] Prepare to add **API call scenarios** to the load tool (the 2-3/3-2 client) (login → game → check ranking)
- [ ] Record a performance baseline: current login p99, purchase p99, game-result save p99 — these become improvement targets in Phase 5
- [ ] Phase 4 retrospective: the hardest consistency problem, one risky piece of code AI suggested, a moment where DB abstraction actually helped
