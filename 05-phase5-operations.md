# Phase 5. Operational Quality: Performance, Observability, Security (Weeks 16-20, 200h)

> 🇰🇷 Korean version: [05-phase5-operations_kr.md](05-phase5-operations_kr.md)

> Common 140h / Track 60h. By the end of this Phase, learners will improve the server they built via **Measure → Hypothesize → Fix → Re-measure**, and by reproducing and responding to failures, turn it into an "operable server."

## 0. How to Use This Document

Same structure as Phases 1-4. Follow the daily blocks in §2 (Days 76-100), find labs (`L5-C-xx`, `L5-CS-xx`, `L5-CPP-xx`) in §3, and assignment details in §7.

The 4 principles of this Phase

1. **Never optimize without measuring.** Every improvement goes through the sequence "baseline → hypothesis → experiment → re-measure." **Keep rejected hypotheses in the report too** (this is proof of real skill).
2. **Observability is design.** Don't attach metrics afterward. First decide "if something goes wrong, which metric moves first," then instrument accordingly.
3. **A server that dies predictably and recovers on its own beats one that never dies.** Prove this through fault injection.
4. **This course does not cover deployment.** The server runs locally as a process. Instead, spend that time on **resilience, observability, and security**.

| Notation | Meaning |
|---|---|
| `L5-C-xx` / `L5-CS-xx` / `L5-CPP-xx` | Common / C# / C++ labs |
| 🔴 Required · 🟡 If time allows | Priority |

---

## 1. Overview

### 1.1 Objectives

1. **Reproduce load**: reliably measure TPS, p99, and error rate with a scenario-based dummy client (within 10% deviation across 3 runs)
2. **Find bottlenecks**: analyze CPU, allocation, and lock contention with a profiler, and quantifiably improve at least 2 of them
3. **Observe**: view game server, API server, and system metrics on dashboards with Prometheus/Grafana, and set up alerts
4. **Withstand**: operate with graceful degradation instead of crashing under DB failure, Redis latency, and network blocking, and recover automatically
5. **Block**: establish basic security with rate limiting, input validation, and secrets management
6. **Analyze**: open crash dumps to pinpoint the root-cause line

### 1.2 Prerequisite Self-Check (20 minutes)

| # | Question | Passing criterion |
|---|---|---|
| 1 | Can the 3-1 game server and 4-1 API server run simultaneously, integrated with each other? | Confirmed running |
| 2 | Can you apply load with the 2-3/3-2 test client? | Scenario runnable |
| 3 | Do your logs include `requestId/userId/roomId/durationMs`? | Log sample confirmed |
| 4 | Can you run a benchmark with a Release build and record the variance? | Phase 1 report |
| 5 | Can you explain the difference between p50/p95/p99? | Immediate answer |

If fewer than 3 pass, reinforce them in the first two days of Week 16. In particular, if #3 is not in place, observability labs are completely blocked.

### 1.3 What You Will Be Able to Do After This Phase

- Explain "why p99 spikes" using profiler results and metrics
- State **why** each panel on the dashboard was added
- Have the server survive 3 injected failures and present recovery time as a number
- Write a report describing a performance improvement as "hypothesis → experiment → adopt/reject → re-measure"
- Reimplement a token bucket rate limiter within 60 minutes

### 1.4 Deliverables of This Phase

```
phase5/
├─ LoadTool/                   Assignment 5-1: production-grade dummy client (full scenario)
│  └─ results/                 Results of 3 repeated runs (JSON+MD)
├─ observability/
│  ├─ prometheus.yml           Scrape configuration
│  ├─ alerts.yml               2+ alert rules
│  ├─ alertmanager.yml         or Grafana contact-point config
│  ├─ webhook.ps1              local delivery log/toast
│  ├─ dashboard.json           Grafana dashboard export
│  └─ screenshots/             Captures during load and alert firing
├─ resilience/                 Directly implemented retry, circuit breaker, backpressure
├─ chaos/                      Fault injection scripts (DB, Redis, network)
├─ PERF-REPORT.md              Assignment 5-2
├─ RUNBOOK.md                  Assignment 5-4
├─ postmortem/                 1 postmortem
├─ SECURITY-CHECK.md           Assignment 5-6
└─ labs/
```

### 1.5 5-Week Roadmap

| Week | Main topic | Common | C# track | C++ track | Assignment |
|---|---|---|---|---|---|
| Week 16 | Load and diagnostic tools | Load design, USE/RED, performance analysis procedure | `dotnet-counters/trace`, PerfView | VS Profiler, ETW/WPA | 5-1 |
| Week 17 | Observability | Metric/log design, Prometheus/Grafana, alerts | prometheus-net/OpenTelemetry | prometheus-cpp | 5-C |
| Week 18 | Performance improvement | Bottleneck hypotheses/experiments, hot paths, classifying lock/allocation/I/O | GC tuning, Span, ThreadPool | Allocation optimization, false sharing, batching | 5-2 |
| Week 19 | Resilience/security design | Retry, circuit breaker, backpressure, timeout, rate limiting, secrets | Polly or direct implementation | Direct implementation | Start 5-4, start 5-6 |
| Week 20 | Fault injection, wrap-up | Inject 3 failure types, runbook, postmortem, crash dump | `dotnet-dump`/SOS | WinDbg/minidump | Complete 5-4/5-6, 🟡5-5, evaluation |

---

## 2. Weekly Detailed Plan (Day by Day)

### 2.1 Week 16 — Load Design and Diagnostic Tools

#### Day 76 (Mon) — Load Test Design 🔴

**Morning (2.5h)**
- Scenario design: connect → login (API) → matchmaking → play → result → check ranking → repeat
- Parameters: ramp-up (N users/sec), target concurrent connections, steady-state duration, teardown
- Metric definitions: TPS, per-stage p50/p95/p99, error rate, concurrent connections, games completed
- **Measurement reliability**: target within 10% deviation across 3 runs. Don't misdiagnose a client bottleneck (CPU, ports) as a server bottleneck
- USE (Utilization, Saturation, Errors) / RED (Rate, Errors, Duration) methodologies

**Afternoon (2.5h)**
- `L5-C-01` Write a baseline measurement plan: fix in a table what to measure, how many times, and under what conditions
- `L5-C-02` Check client bottleneck: measure client CPU/port usage and identify limits (max load with client alone, no server)

**1 Hour Without AI**
- Write down "5 reasons this measurement could become unreliable" and a control method for each

**DoD**
- [ ] Measurement plan (conditions, repetitions, metrics)
- [ ] Recorded client's own limit numbers

#### Day 77 (Tue) — 5-1 Tool Extension ① 🔴

**Morning (2.5h)**
- Upgrade the 3-2 client to production grade: **full scenario** of API login → connect to game server with token → matchmaking → play → check ranking
- Measure per-stage latency separately (login, matchmaking, move, ranking) — you must isolate which stage is slow before you can improve it

**Afternoon (2.5h)**
- `L5-C-03` Implement full scenario, insert per-stage timers
- `L5-C-04` Statistics engine: histogram-based p50/p95/p99 (verify accuracy against the sort-based method), error counts by type

**1 Hour Without AI**
- Decide how to set histogram bucket boundaries and write the rationale (dense around the 50ms target)

**DoD**
- [ ] Full scenario runs for 5 minutes with 100 connections
- [ ] p99 calculation verification test passes

#### Day 78 (Wed) — 5-1 Tool Extension ② 🔴

**Morning (2.5h)**
- Real-time stats output (every 5s): connection count, TPS, per-stage latency, errors
- Final report: JSON + markdown table, script for 3 repeated runs and deviation calculation
- Distributed execution: split user ID ranges via arguments to run across 2 processes

**Afternoon (2.5h)**
- `L5-C-05` Implement report output and 3-run repetition script
- `L5-C-06` **Baseline measurement**: measure the game server (300 connections, 100 rooms) and API (100 concurrent logins) 3 times each to fill the baseline table in `PERF-REPORT.md`

**1 Hour Without AI**
- If the 3-run deviation exceeds 10%, find the cause (background processes, power settings, GC, cache state)

**DoD**
- [ ] Results and deviation table from 3 repeated runs
- [ ] Baseline finalized (the comparison target for all future improvements)

#### Day 79 (Thu) — Track Diagnostic Tools 🔴

**C# Track (5h)**
- `L5-CS-01` `dotnet-counters`: observe GC collection count, heap size, ThreadPool queue length, exception count during load
- `L5-CS-02` `dotnet-trace` + PerfView (or VS): collect CPU samples/allocation profile → table of top 5 hot functions
- `L5-CS-03` `dotnet-dump` + SOS: check the types occupying the most heap using `!dumpheap -stat`, `!threads`

**C++ Track (5h)**
- `L5-CPP-01` VS Performance Profiler: visualize CPU usage, memory, concurrency → table of top hot functions
- `L5-CPP-02` ETW/WPA: collect with Windows Performance Recorder → analyze context switching and lock waits in Analyzer
- `L5-CPP-03` Measure lock contention: observe SRWLock wait time and spinning

**1 Hour Without AI**
- For the top 5 functions in the profiler results, explain in your own words "why is this expensive" while reading the code

**DoD**
- [ ] Table of top 5 hot functions + captures
- [ ] (C#) GC/ThreadPool counter observations / (C++) lock wait time table

#### Day 80 (Fri) — Establish Performance Analysis Procedure + Weekly Checkpoint

**Morning (3h)**
- Bottleneck classification scheme: CPU / memory (allocation, GC) / lock contention / I/O (disk, network) / DB / external calls
- Fix a hypothesis template: `Hypothesis / Rationale / Verification method / Expected improvement / Result / Adopt or reject`
- `L5-C-07` Write **5 hypotheses** from the baseline data (do not fix anything yet)

**Afternoon (4h) — Weekly Checkpoint**
1. Reimplement without AI (60 min): histogram-based p99 calculator
2. Oral exam (45 min): load design, p99, USE/RED
3. Code review (45 min): accuracy of the measurement tool code
4. Retrospective (30 min): `W16.md`

**DoD**
- [ ] 5 hypotheses (including verification methods)
- [ ] 5-1 tool complete (meets requirements in §7.2)

### 2.2 Week 17 — Observability Stack (Assignment 5-C)

#### Day 81 (Mon) — Metric Design 🔴

**Morning (2.5h)**
- Metric types: counter (monotonically increasing) / gauge (current value) / histogram (distribution)
- **Cardinality explosion**: putting userId/roomId into labels causes time series to explode. Decide the allowed label list first
- Histogram bucket design: dense around the target (50ms for a move) — `1,5,10,25,50,100,250,500,1000ms`
- Build a table of **which metric moves first when something goes wrong** (failure → leading indicator mapping)

**Afternoon (2.5h)**
- `L5-C-08` Metric list design document: 8 for the game server, 6 for the API server, 4 for the system. For each metric, specify type, labels, purpose, and whether it triggers an alert
- Book "Monitoring Game Servers with Prometheus" Chapters 1-2

**1 Hour Without AI**
- For each metric, write down "the failure that could not be caught without this metric" (if there isn't one, drop that metric)

**DoD**
- [ ] Metric design document (18+ metrics, labels and purpose specified)
- [ ] Failure → leading indicator mapping table

#### Day 82 (Tue) — Installing Prometheus/Grafana and Scraping 🔴

**Morning (2.5h)**
- Install Prometheus, Grafana, windows_exporter on Windows (book Chapters 3-4)
- Write `prometheus.yml`: scrape targets (game server `/metrics`, API server `/metrics`, windows_exporter), interval (15s)

**Afternoon (2.5h)**
- `L5-C-09` Install, start, and confirm target status (UP on the `/targets` page)
- `L5-CS-04`/`L5-CPP-04` Attach an instrumentation library: prometheus-net or OpenTelemetry `Meter` for C#, prometheus-cpp (or a simple self-built `/metrics`) for C++
- Since the socket server has no HTTP listener, open a **lightweight metrics-only HTTP endpoint** on a separate port

**1 Hour Without AI**
- Read the `/metrics` output directly and interpret the meaning of each line (type, labels, value)

**DoD**
- [ ] Prometheus collects 3 targets, all UP
- [ ] Game server and API server expose `/metrics`

#### Day 83 (Wed) — Server Instrumentation 🔴

**Morning (2.5h)**
- Game server metrics: connection count (gauge), room count (gauge), packets received/sent (counter, packet ID label **top 20 only**), move processing latency (histogram), max send queue length (gauge), sessions closed by reason (counter), unreturned pool objects (gauge), room queue length (gauge)

**Afternoon (2.5h)**
- `L5-C-10` Implement game server instrumentation (8 metrics)
- `L5-C-11` Implement API server instrumentation (6 metrics): requests/errors/latency histogram per endpoint, DB query latency, Redis command latency, connection pool utilization
- Measure instrumentation overhead: compare throughput with instrumentation on/off (should be within 1% normally)

**1 Hour Without AI**
- Calculate label cardinality (e.g., 20 endpoints × 6 status codes = 120 time series)

**DoD**
- [ ] 14+ metrics exposed, values change during load
- [ ] Recorded instrumentation overhead measurement

#### Day 84 (Thu) — Dashboards and Alerts 🔴

**Morning (2.5h)**
- PromQL basics: `rate()`, `histogram_quantile()`, `sum by()`, `increase()`
- Dashboard layout: rows for overview / game server / API server / DB·Redis / system. Select instance via variable
- Panel principle: you should be able to **judge "is it healthy right now" within 5 seconds** from one screen

**Afternoon (2.5h)**
- `L5-C-12` Build dashboard (5+ rows, 15+ panels). Add explanatory annotations to each panel
- `L5-C-13` 2+ alerts: move p99 >100ms for 1 minute / error rate >1% for 1 minute, delivered through Alertmanager or Grafana to a local webhook
- Apply load and capture both the graph moving and the alert actually firing

**1 Hour Without AI**
- For each panel, write one line on "what would be missed without this panel"

**DoD**
- [ ] Dashboard JSON export, captures during load
- [ ] Logs/captures of 2 alerts firing

#### Day 85 (Fri) — Log Pipeline + Weekly Checkpoint

**Morning (3h)**
- Write structured logs to files with daily rolling (C#: Serilog / C++: spdlog)
- 🟡 Practice log collection with fluent-package (Windows) — not required, but "what is a log pipeline" is in scope for the oral exam
- Division of roles between logs and metrics: metrics tell "what is happening," logs tell "why it happened"
- `L5-C-14` Organize the procedure for analyzing a specific failure window in logs via grep/PowerShell

**Afternoon (4h) — Weekly Checkpoint**
1. Reimplement without AI (60 min): expose a histogram metric (including bucket design)
2. Oral exam (45 min): cardinality, bucket design, RED/USE
3. Code review (45 min): does the instrumentation code burden the hot path?
4. Retrospective (30 min): `W17.md`

**DoD**
- [ ] Assignment 5-C complete (80+ points on the §7.1 rubric)
- [ ] Log analysis procedure documented

### 2.3 Week 18 — Performance Improvement (Assignment 5-2)

#### Day 86 (Mon) — Deep Profiling 🔴

**Morning (2.5h)**
- Types of profiling: CPU sampling / allocation / lock contention / I/O wait. When to use each
- Collection principle: collect for **30 seconds after entering steady state** (the ramp-up window is contaminated)

**Afternoon (2.5h)**
- `L5-C-15` Collect profiles under load (game server and API server separately) → tables of top functions, top allocation types, lock waits
- Cross-check against the 5 hypotheses from Day 80: verify whether the predictions were correct

**1 Hour Without AI**
- From the profile results, distinguish "things that are naturally expensive" from "things that are suspiciously expensive"

**DoD**
- [ ] 3 kinds of profile results (CPU, allocation, lock)
- [ ] Hypothesis comparison table (predicted vs. measured)

#### Day 87 (Tue) — Bottleneck Improvement ① 🔴

**Morning (2.5h)**
- Fix the single most promising hypothesis. Representative candidates
  - Repeated serialization in broadcast → serialize once and share
  - Synchronous log writes → async sink
  - Allocation per request → pooling/`Span`/struct-ify (C#) / arena/`reserve` (C++)
  - Room queue waiting → change worker assignment method
  - DB query N+1 → join/batch

**Afternoon (2.5h)**
- **Re-measure 3 times under identical conditions** after the fix → before/after table
- Check side effects: memory, CPU, code complexity

**1 Hour Without AI**
- If the improvement is smaller than expected, analyze why (Amdahl's Law: what % of the total was that section?)

**DoD**
- [ ] 1 improvement complete, before/after 3-run measurement table
- [ ] Side effects recorded

#### Day 88 (Wed) — Bottleneck Improvement ② + Rejected Case 🔴

**Morning (2.5h)**
- Fix the second hypothesis and re-measure
- **Create a rejected case**: record at least 1 hypothesis that looked plausible but measured no effect (e.g., increasing thread count, setting affinity, swapping a particular collection)

**Afternoon (2.5h)**
- Track-specific deep dive
  - C#: `L5-CS-05` GC tuning (Server GC, `GCHeapCount`, avoiding LOH), compare GC pause/p99 before and after, `L5-CS-06` diagnose and fix ThreadPool starvation
  - C++: `L5-CPP-05` remove false sharing (`alignas(64)`), relax memory order (seq_cst → acq/rel), then measure, `L5-CPP-06` apply thread-local pools

**1 Hour Without AI**
- Explain the rejected hypothesis in terms of "why it had no effect" (backed by measurements)

**DoD**
- [ ] 2 improvements complete (goal: 30%+ improvement in move p99, or 50%+ increase in connections at the same p99, or 30%+ improvement in API TPS)
- [ ] 1+ rejected hypothesis recorded

#### Day 89 (Thu) — Writing the Report 🔴

**Morning (2.5h)**
- Write `PERF-REPORT.md` (template in §7.3): environment → baseline → hypothesis table → experiment results → final before/after → side effects → remaining bottlenecks
- Attach graphs (Grafana captures comparing before/after improvement windows)

**Afternoon (2.5h)**
- Ensure reproducibility: confirm 3 runs under the same conditions stay within 10% deviation
- 🟡 Explore further optimization opportunities (a list to continue in Phase 6)

**1 Hour Without AI**
- Write the report's conclusion paragraph without AI, then only ask AI to point out "factors you missed"

**DoD**
- [ ] `PERF-REPORT.md` complete (including rejected hypotheses)
- [ ] Analysis of whether the target was reached, or why not

#### Day 90 (Fri) — Performance Wrap-up + Weekly Checkpoint

**Morning (3h)**
- Full regression test to confirm improvements didn't break existing tests
- Add before/after comparison panels to the dashboard

**Afternoon (4h) — Weekly Checkpoint**
1. Reimplement without AI (60 min): eliminate allocations by applying an object pool (small example)
2. Oral exam (45 min): bottleneck classification, improvement rationale, reasons for rejection
3. Code review (45 min): how much did the optimization hurt readability?
4. Retrospective (30 min): `W18.md`

**DoD**
- [ ] Regression tests pass
- [ ] Assignment 5-2 complete

### 2.4 Week 19 — Resilience and Security Design

#### Day 91 (Mon) — Failure Modes and Timeouts 🔴

**Morning (2.5h)**
- Failure mode design: fail fast vs. wait, graceful degradation
- **Timeout hierarchy**: separate timeouts for connection, request, and overall operation. Upper timeout > sum of lower timeouts
- Basis for deciding timeout values: p99 × margin multiplier, user-perceptible limits

**Afternoon (2.5h)**
- `L5-C-16` Implement timeout hierarchy: game server → API call, API → DB/Redis, separately
- `L5-C-17` Timeout tests: verify each layer gives up in time using artificial delay (sleep injection)

**1 Hour Without AI**
- Draw out, in chronological order, the cascading failure that occurs without timeouts

**DoD**
- [ ] 3-layer timeout implemented and tested
- [ ] Table of timeout values and rationale

#### Day 92 (Tue) — Retry and Backoff 🔴

**Morning (2.5h)**
- The 3 principles of retry: **cap**, **exponential backoff**, **jitter** (prevent a surge of simultaneous retries)
- What must not be retried: non-idempotent requests, 4xx-class errors
- Retry queue: on immediate retry failure, hold in a local queue (e.g., saving game results)

**Afternoon (2.5h)**
- `L5-C-18` **Directly implement** a retry policy (no library): cap of 3, backoff 100→200→400ms, jitter ±30%
- `L5-C-19` Retry test: measure success rate and total latency in a 50% failure-rate environment, compare with and without jitter

**1 Hour Without AI**
- Draw a scenario where "retries make the failure worse" (retry storm)

**DoD**
- [ ] Retry implementation and test, jitter effect measured
- [ ] Documented list of things that must not be retried

#### Day 93 (Wed) — Circuit Breaker and Backpressure 🔴

**Morning (2.5h)**
- 3 circuit breaker states: Closed → (failure rate exceeds threshold) → Open → (cooldown) → Half-Open → Closed on success
- Threshold design: failure rate, minimum request count, cooldown time
- Backpressure: queue cap, connection limit, request rejection (429 / server-overloaded response)

**Afternoon (2.5h)**
- `L5-C-20` **Directly implement** a circuit breaker + 5 state transition tests (stay closed, transition to open, immediate failure while open, half-open trial request, reopen on half-open failure)
- `L5-C-21` Backpressure: cap on game server room queue/send queue, limit on API concurrent processing. Confirm policy behavior when exceeded
- 🟡 Configure the same behavior with Polly (C#) and compare with the direct implementation

**1 Hour Without AI**
- Write what to show the user while the circuit is open (graceful degradation design)

**DoD**
- [ ] 5 circuit breaker state transition tests pass
- [ ] Backpressure behavior confirmed (memory stays within cap)

#### Day 94 (Thu) — Security Basics (Start Assignment 5-6) 🔴

**Morning (2.5h)**
- Rate limiting: token bucket algorithm, packets/sec per connection, connection attempts/sec per IP
- Input validation: length, range, state. Unauthenticated session timeout (basis of flood defense)
- Secrets management: DB connection strings, inter-server tokens as environment variables or **permission-restricted files**. Zero secrets in the repository
- TLS concepts: TLS recommended for API, optional for sockets. Handshake cost

**Afternoon (2.5h)**
- `L5-C-22` Implement a token bucket rate limiter (thread-safe) + tests (allow burst, reject over limit, recover over time)
- `L5-C-23` Secrets check: scan the entire repository (regex for key/password patterns), verify log masking
- `L5-C-24` Dependency vulnerability scan: `dotnet list package --vulnerable` / check vcpkg updates

**1 Hour Without AI**
- Write a scenario where rate limiting only "per account" gets bypassed (multi-account, distributed IPs)

**DoD**
- [ ] Rate limiter implemented and tested
- [ ] 0 secrets found in scan, list of vulnerable packages

#### Day 95 (Fri) — Flood Defense + Weekly Checkpoint

**Morning (3h)**
- `L5-C-25` Flood test: 1,000 connection attempts/sec + 100 normal concurrent users → **table of normal-user p99 change**
- Defenses: shorten unauthenticated session timeout, cap connection count, limit connection attempts per IP, tune accept queue
- Table of the boundary between what the application can do / what infrastructure must do (firewall, DDoS protection)

**Afternoon (4h) — Weekly Checkpoint**
1. Reimplement without AI (60 min): token bucket rate limiter
2. Oral exam (45 min): 3 circuit breaker states, jitter, backpressure
3. Code review (45 min): failure paths in resilience code
4. Retrospective (30 min): `W19.md`

**DoD**
- [ ] Table of normal-user p99 before/after flood defense
- [ ] Defense responsibility boundary table

### 2.5 Week 20 — Fault Injection and Wrap-up

#### Day 96 (Mon) — Fault Injection ①: DB 🔴

**Morning (2.5h)**
- Fault scenario design: what to break, when, and for how long. **Write down the expected behavior first**
- DB failure: **(MySQL)** stop the service for 30 seconds → restart / **(SQLite)** another process locks the file for 30 seconds with a long write transaction

**Afternoon (2.5h)**
- `L5-C-26` Write and run the DB fault injection script
- Observe: 0 crashes, API returns a clear error code, game server holds results in a queue, circuit breaker activates, automatic normalization and queue drain after recovery
- Confirm the failure window is visible on the dashboard, confirm alerts fire

**1 Hour Without AI**
- Measure recovery time (failure end → back to normal) and analyze what consumed the time

**DoD**
- [ ] DB failure scenario passes (0 crashes, automatic recovery)
- [ ] Recovery time recorded, dashboard capture

#### Day 97 (Tue) — Fault Injection ②③: Redis, Network 🔴

**Morning (2.5h)**
- Redis failure: stop the service for 30 seconds, or repeat 3-second delays. **Decide the policy in advance** for session validation failure (reject login vs. allow temporarily)
- Network blocking: block the firewall outbound rule between game server ↔ API server for 60 seconds

**Afternoon (2.5h)**
- `L5-C-27` Inject and observe Redis failure
- `L5-C-28` Inject and observe network blocking: game server queues result saves, sends after recovery, in-progress games continue
- 🟡 Disk-full / 90% CPU utilization scenario

**1 Hour Without AI**
- Write 3 lines each on the "symptoms visible to the user" for the three failures

**DoD**
- [ ] All 3 failure types: 0 crashes, recovery time table
- [ ] Graceful degradation works as designed

#### Day 98 (Wed) — Runbook and Postmortem 🔴

**Morning (2.5h)**
- Write `RUNBOOK.md` (template T7): for each failure, **Detect (alert/symptom) → Diagnose (check commands/dashboard panel) → Respond → Confirm recovery → Follow-up**
- Runbook quality bar: can **"someone seeing this for the first time at 3am"** follow it as-is?

**Afternoon (2.5h)**
- `L5-C-29` Write runbooks (3 failure types + server process down + load spike)
- `L5-C-30` Write 1 postmortem (template T8): a real failure you experienced, as timeline, root cause, and improvement items
- Ask AI to review the runbook in the role of a "3am on-call operator" → reinforce the points where it got stuck

**1 Hour Without AI**
- Follow your own runbook step by step and fill in missing commands/paths (a live self-test of your runbook)

**DoD**
- [ ] `RUNBOOK.md` with 5 failure entries
- [ ] 1 postmortem, AI review incorporated

#### Day 99 (Thu) — Crash Dumps + Security Wrap-up

**Morning (2.5h) — 🟡 Assignment 5-5**
- `L5-C-31` Set up automatic crash dump collection: Windows Error Reporting LocalDumps registry key or procdump
- C#: `dotnet-dump analyze` / WinDbg+SOS, C++: `SetUnhandledExceptionFilter` + `MiniDumpWriteDump`
- Inject 3 intentional crash types (null reference, memory corruption, stack overflow) → collect dumps → pinpoint the root-cause line

**Afternoon (2.5h)**
- Complete `SECURITY-CHECK.md` (Assignment 5-6): rate limiting, flood test results, input validation, secrets, dependency scan
- Separate the list of remaining risks from items "out of scope for this course" (e.g., infrastructure DDoS protection)

**1 Hour Without AI**
- Read the stack from a dump and explain the crash cause in your own words, 3 lines

**DoD**
- [ ] 🟡 Analysis record of 3 dump types
- [ ] `SECURITY-CHECK.md` complete

#### Day 100 (Fri) — Phase 5 Evaluation 🔴

**Morning (3h)**
1. **Reimplementation exam (60 min, no AI)**: §8.2 — token bucket rate limiter + 4 provided tests
2. **Oral exam (60 min)**: 10 questions from §8.3
3. **Bug hunt (45 min)**: §8.4 — find 4 of 5 among infinite retry, missing timeout, cardinality explosion, secrets in logs, missing cleanup

**Afternoon (3h)**
- Review the checklist (§8.1), fill any gaps
- `W20.md` + Phase 5 retrospective
- §10 Prepare for Phase 6 (confirm local multi-process execution)

**DoD**
- [ ] All items in §8.1, oral exam average of 4.0 or above
- [ ] Confirmed 2 game servers can run simultaneously on different ports

---

## 3. Lab Catalog

### 3.1 Common Labs (L5-C)

#### L5-C-01 Measurement Plan (45 min) 🔴
- **Steps**: fix in a table the target (game/API), scenario, connection count, ramp-up, hold duration, repetition count, metrics, and control conditions (power settings, killing background processes)
- **Acceptance criteria**: someone else could reproduce the same measurement from this document alone

#### L5-C-02 Measuring Client Limits (45 min) 🔴
- **Steps**: max load with client alone (no server, or against an echo server) → check CPU/port usage/max connections
- **Acceptance criteria**: you know, as a number, "the point where the client dies first." From then on, only load up to 70% of that value

#### L5-C-03 Implement Full Scenario (120 min) 🔴
- **Steps**: API login → connect to game server with token → matchmaking → play → result → check ranking → repeat. Insert **per-stage timers**
- **Acceptance criteria**: per-stage latency is aggregated separately (login/matchmaking/move/ranking)

#### L5-C-04 Histogram Statistics Engine (90 min) 🔴
- **Steps**: calculate p50/p95/p99 with a fixed-bucket histogram. Compare error against sort-based calculation (10,000 samples)
- **Acceptance criteria**: unit tests against a known distribution pass, error within 5%

#### L5-C-05 Report and Repeated Runs (60 min) 🔴
- **Steps**: JSON + markdown table output, 3-run repetition script, deviation (max-min)/median calculation
- **Acceptance criteria**: 3-run deviation within 10%, or the cause is recorded if it exceeds that

#### L5-C-06 Baseline Measurement (90 min) 🔴
- **Steps**: measure the game server (300 connections, 100 rooms) and API (100 concurrent logins) 3 times each
- **Acceptance criteria**: the baseline table in `PERF-REPORT.md` is filled in. This value becomes the comparison target for all future improvements

#### L5-C-07 Write 5 Hypotheses (60 min) 🔴
- **Steps**: `Hypothesis / Rationale / Verification method / Expected improvement` table. Don't fix anything yet
- **Acceptance criteria**: each hypothesis is paired with an **observable method**

#### L5-C-08 Metric Design Document (75 min) 🔴
- **Steps**: 18+ metrics as `name/type/labels/purpose/alerting`. Calculate expected label cardinality
- **Acceptance criteria**: no userId/roomId in labels (preventing cardinality explosion), 1 "failure that can't be caught without this" per metric

#### L5-C-09 Install Prometheus/Grafana (90 min) 🔴
- **Steps**: install and start the 3 binaries, write `prometheus.yml` (3 targets, 15s), connect the Grafana data source
- **Acceptance criteria**: all UP on `/targets`, queries run successfully in Grafana
- **Common mistake**: collection fails because the metrics port is bound to localhost → check the bind address

#### L5-C-10 Game Server Instrumentation (120 min) 🔴
- **Steps**: expose 8 metrics (connections, rooms, packet counters, move-latency histogram, send queue, close reason, unreturned pool, room queue)
- **Acceptance criteria**: values change under load and histogram buckets are dense around the target value

#### L5-C-11 API Server Instrumentation (90 min) 🔴
- **Steps**: requests/errors/latency per endpoint, DB query latency, Redis command latency, connection pool utilization
- **Acceptance criteria**: endpoint labels are normalized (path parameters don't explode the label set)

#### L5-C-12 Build Dashboard (120 min) 🔴
- **Steps**: 5 rows (overview/game/API/DB·Redis/system), 15+ panels, instance variable
- **Acceptance criteria**: load start/end is clearly visible in the graphs. Each panel has an explanatory annotation

#### L5-C-13 Set Up Alerts (60 min) 🔴
- **Steps**: move p99 > 100ms sustained for 1 minute / error rate > 1% sustained for 1 minute. Log or notify when triggered
- **Acceptance criteria**: log/capture of the alert **actually firing** under artificial load

#### L5-C-14 Log Analysis Procedure (45 min)
- **Steps**: document a procedure for aggregating errors in a specific time window with PowerShell (`Select-String`, grouping)
- **Acceptance criteria**: a procedure that extracts the top 3 error types from a failure-window log within 5 minutes

#### L5-C-15 Collect Profile Under Load (90 min) 🔴
- **Steps**: collect for 30 seconds after entering steady state (CPU, allocation, lock). Exclude the ramp-up window
- **Acceptance criteria**: tables of top 5 hot functions / top allocation types / lock waits

#### L5-C-16 Timeout Hierarchy (75 min) 🔴
- **Steps**: game server → API (2s total, 500ms connect), API → DB (1s), API → Redis (200ms). Verify the "upper > sum of lower" rule
- **Acceptance criteria**: each layer gives up in time and propagates a clear error upward

#### L5-C-17 Timeout Tests (45 min)
- **Steps**: inject artificial delay (test middleware/wrapper) to confirm each timeout fires
- **Acceptance criteria**: with a 3-second injected delay, the caller gives up at 2 seconds

#### L5-C-18 Direct Retry Implementation (75 min) 🔴
- **Steps**: cap of 3, exponential backoff (100/200/400ms), jitter ±30%, no-retry conditions (4xx, non-idempotent)
- **Acceptance criteria**: works without a library, 5 unit tests

#### L5-C-19 Measure Retry Effect (45 min)
- **Steps**: compare success rate, total latency, and server load with and without jitter in a 50% failure-rate environment
- **Acceptance criteria**: numerically show jitter mitigates a surge of simultaneous retries

#### L5-C-20 Direct Circuit Breaker Implementation (90 min) 🔴
- **Steps**: Closed/Open/Half-Open, failure-rate threshold, minimum request count, cooldown. State transition log
- **Acceptance criteria**: 5 transition tests pass, only 1 trial request passes through in half-open

#### L5-C-21 Backpressure (60 min) 🔴
- **Steps**: cap room queue/send queue, limit API concurrency (semaphore), 429 or connection close when exceeded
- **Acceptance criteria**: memory stays within the cap under overload, normal requests are not completely starved

#### L5-C-22 Token Bucket Rate Limiter (75 min) 🔴
- **Steps**: capacity and refill rate, thread safety, applied per account and per IP
- **Acceptance criteria**: tests pass for burst allowed / over-limit rejected / recovers after time passes

#### L5-C-23 Secrets Check (45 min) 🔴
- **Steps**: pattern-scan the entire repository (`password=`, `secret`, key formats), verify log masking, move config to environment variables/permission-restricted files
- **Acceptance criteria**: 0 secrets in the repository, no tokens exposed in logs

#### L5-C-24 Dependency Vulnerability Scan (30 min)
- **Steps**: `dotnet list package --vulnerable --include-transitive` / check vcpkg package versions
- **Acceptance criteria**: list of vulnerabilities and remediation (update or accepted reason)

#### L5-C-25 Flood Test (75 min) 🔴
- **Steps**: 1,000 connection attempts/sec + 100 normal concurrent users → measure normal-user p99 change → re-measure after applying defenses
- **Acceptance criteria**: p99 degradation for normal users is meaningfully reduced after defenses

#### L5-C-26 DB Fault Injection (75 min) 🔴
- **Steps**: **(MySQL)** stop the service for 30s / **(SQLite)** lock for 30s via a long write transaction → observe → recover
- **Acceptance criteria**: 0 crashes, clear error codes, automatic normalization after recovery, recovery time recorded

#### L5-C-27 Redis Fault Injection (60 min) 🔴
- **Steps**: 30s stop or inject delay → confirm session-validation-failure policy → recover
- **Acceptance criteria**: in-progress games continue, new logins fail clearly, normalized after recovery

#### L5-C-28 Network Blocking Injection (60 min) 🔴
- **Steps**: block game server → API for 60s via firewall outbound rule → confirm result-save queue buildup → confirm queue drains after removing the rule
- **Acceptance criteria**: 0 loss, 0 duplicate saves (idempotency), recovery time recorded

#### L5-C-29 Write Runbook (90 min) 🔴
- **Steps**: cover 5 failure types (DB, Redis, network, process down, load spike) as detect/diagnose/respond/confirm-recovery/follow-up
- **Acceptance criteria**: passes an AI "3am operator" review, and you personally followed it and filled in missing commands

#### L5-C-30 Postmortem (45 min)
- **Steps**: document a real failure you experienced as timeline, root cause, reason for detection delay, and improvement items
- **Acceptance criteria**: improvement items have an owner (yourself) and a deadline

#### L5-C-31 Crash Dump Analysis (🟡 90 min)
- **Steps**: set up LocalDumps registry or procdump → 3 intentional crash types → open the dump and pinpoint the root-cause line
- **Acceptance criteria**: crash line and stack explanation for each of the 3 types

### 3.2 C# Track Labs (L5-CS)

#### L5-CS-01 dotnet-counters Observation (60 min) 🔴
- **Steps**: `dotnet-counters monitor -p <pid> --counters System.Runtime,Microsoft.AspNetCore.Hosting`
- **Acceptance criteria**: captures and 3-line interpretation of GC collection count, heap size, ThreadPool queue length, exception count during load

#### L5-CS-02 dotnet-trace + PerfView (90 min) 🔴
- **Steps**: `dotnet-trace collect -p <pid> --profile cpu-sampling --duration 00:00:30` → open with PerfView/VS → analyze top functions/allocations
- **Acceptance criteria**: table of top 5 functions, top 3 allocation types

#### L5-CS-03 dotnet-dump + SOS (60 min)
- **Steps**: `dotnet-dump collect` → `analyze` → `!dumpheap -stat`, `!threads`, `!clrstack`
- **Acceptance criteria**: top heap-occupying types and 1 suspected point identified

#### L5-CS-04 prometheus-net Instrumentation (90 min) 🔴
- **Skeleton**
  ```csharp
  static readonly Counter   PacketsRecv = Metrics.CreateCounter("game_packets_recv_total", "Packets received", new CounterConfiguration{ LabelNames = new[]{"packet"} });
  static readonly Gauge     Sessions    = Metrics.CreateGauge("game_sessions", "Current connection count");
  static readonly Histogram PlaceLatency= Metrics.CreateHistogram("game_place_latency_seconds", "Move processing latency",
      new HistogramConfiguration{ Buckets = new[]{0.001,0.005,0.01,0.025,0.05,0.1,0.25,0.5,1.0} });
  // Since it's a socket server, start the metrics server on a separate port
  var server = new KestrelMetricServer(port: 9101); server.Start();
  ```
- **Acceptance criteria**: `/metrics` exposed, label cardinality controlled (top 20 packets only)

#### L5-CS-05 GC Tuning (75 min)
- **Steps**: Server GC on/off, `GCHeapCount`, avoid LOH allocations, apply `ValueTask`/`Span`, then compare GC pause/p99 before and after
- **Acceptance criteria**: table of GC count/pause/p99 before and after. If there's no effect, that's a valid conclusion too

#### L5-CS-06 Diagnose ThreadPool Starvation (60 min)
- **Steps**: reproduce starvation by injecting synchronous waits (`.Result`) → observe queue length → confirm recovery after converting to `await`
- **Acceptance criteria**: before/after graph of queue length

### 3.3 C++ Track Labs (L5-CPP)

#### L5-CPP-01 VS Performance Profiler (75 min) 🔴
- **Steps**: collect under load with the CPU usage/memory usage/concurrency visualization tools
- **Acceptance criteria**: table of top 5 functions, identified thread wait sections

#### L5-CPP-02 ETW/WPA Analysis (120 min) 🔴
- **Steps**: collect CPU/context switch/disk with WPR → analyze thread wait causes (Ready Thread) in WPA
- **Acceptance criteria**: top 3 causes of context switching, lock wait time table

#### L5-CPP-03 Measure Lock Contention (60 min)
- **Steps**: instrument SRWLock/mutex acquisition wait time (timer in a RAII wrapper), top 3 contention points
- **Acceptance criteria**: contention ranking table and identified improvement candidates

#### L5-CPP-04 prometheus-cpp Instrumentation (90 min) 🔴
- **Steps**: use vcpkg `prometheus-cpp` or a self-built `/metrics`, expose counters/gauges/histograms
- **Acceptance criteria**: Prometheus collects it, histogram bucket design is justified

#### L5-CPP-05 False Sharing / Memory Order (75 min)
- **Steps**: separate hot counters with `alignas(64)`, relax seq_cst → acquire/release, then measure
- **Acceptance criteria**: before/after throughput/p99 table (record it as a conclusion even if there's no effect)

#### L5-CPP-06 Thread-Local Pool (75 min)
- **Steps**: switch to a 2-tier scheme of per-thread buffer pool + global pool, compare allocations per second under load
- **Acceptance criteria**: before/after allocation count and p99

---

## 4. Learning Items in Detail

### 4.1 Common (140h)

**Load Test Design (16h)**
- What: scenarios, ramp-up, steady state, metrics (TPS, p50/p95/p99, error rate), isolating client bottlenecks, reproducibility
- Why: if the measurement isn't trustworthy, the improvement isn't trustworthy either
- How: `L5-C-01/02` → `L5-C-03~06` → verify 3-run deviation
- Verification: p99 deviation within 10% across 3 runs under the same conditions

**Observability: Metrics, Logs, Alerts (32h)**
- What: counter/gauge/histogram, RED/USE, label design and **cardinality explosion**, PromQL basics (`rate`, `histogram_quantile`, `sum by`), Grafana panels/variables, alert rules (condition, duration), structured log collection, division of roles between logs and metrics, distributed tracing concepts, dashboard layering
- Why: you must know "what's happening right now" within 5 seconds during operations
- How: `L5-C-08` (design) → `L5-C-09` (install) → `L5-C-10/11` (instrumentation) → `L5-C-12/13` (dashboard/alerts)
- Verification: load windows are visible on graphs and logs show alerts actually firing

**Performance Analysis Methodology (28h)**
- What: forming bottleneck hypotheses (CPU, memory, lock, I/O, DB), observe→hypothesize→experiment→reject/adopt loop, identifying hot paths, allocation/lock profiling, prioritizing multiple bottlenecks, "no optimization without measurement," report structure
- Why: this is the core of 5-2 and the most persuasive story in interviews
- How: `L5-C-07` (hypotheses) → `L5-C-15` (profile) → improvements on Days 87-88 → `PERF-REPORT.md`
- Verification: **at least 1 rejected hypothesis** in the report

**Failure Response (32h)**
- What: fault injection (DB, Redis, network, disk, CPU), failure mode design, **retry, exponential backoff, jitter**, **circuit breaker**, timeout hierarchy, **backpressure**, graceful degradation, runbooks, postmortems, blast radius
- Why: the goal is "a server that, even when it dies, dies predictably and recovers on its own"
- How: `L5-C-16~21` (implement patterns) → `L5-C-26~28` (inject) → `L5-C-29/30` (runbook, postmortem)
- Verification: 0 crashes across 3 failure types, recovery times recorded

**Security Basics (20h)**
- What: rate limiting (token bucket), input validation, flood defense (connection limits, unauthenticated timeout), TLS concepts and scope of application, secrets management, dependency vulnerabilities
- How: `L5-C-22~25`
- Verification: minimized p99 degradation for normal users during flooding, 0 secrets found

**Crash Analysis (12h)**
- What: automatic dump collection (LocalDumps, procdump), opening dumps (WinDbg/VS), checking stack/threads/heap, symbol (PDB) management
- How: `L5-C-31`
- Verification: root-cause line pinpointed in at least 1 dump

### 4.2 C# Track (60h)

| Item | Hours | What | Learning order | Verification |
|---|---|---|---|---|
| Diagnostic tools | 14h | `dotnet-counters`, `dotnet-trace` + PerfView/VS, `dotnet-dump` + SOS | L5-CS-01 → 02 → 03 | Table of top 5 hot functions |
| GC tuning | 13h | Server/Workstation, Concurrent, `GCHeapCount`, `TieredPGO`, avoiding LOH, reducing allocations | L5-CS-05 | GC count/pause before and after |
| ThreadPool | 6h | Diagnosing starvation (queue length), removing synchronous waits, the meaning of `MinThreads` | L5-CS-06 | Reproduce and resolve starvation |
| Instrumentation | 10h | prometheus-net or OpenTelemetry `Meter`, metrics server, histogram buckets | L5-CS-04 | 5-C |
| Resilience | 11h | Directly implement retry, circuit breaker, timeout, then 🟡 compare with Polly, `HttpClient` timeout | L5-C-18/20 | 5-4 |
| Crash | 6h | LocalDumps, `dotnet-dump analyze`, WinDbg + SOS | L5-C-31 | 🟡 5-5 |

### 4.3 C++ Track (60h)

| Item | Hours | What | Learning order | Verification |
|---|---|---|---|---|
| Profiler | 14h | VS Performance Profiler, ETW/WPA (WPR→WPA) for context switching/lock waits | L5-CPP-01 → 02 | Table of top functions/lock waits |
| Allocation optimization | 13h | Pools/arenas, thread-local pools, avoiding container reallocation (`reserve`), moves | L5-CPP-06 | Allocations per second before and after |
| Lock/cache | 10h | Measuring lock contention, removing false sharing, relaxing memory order, batch processing | L5-CPP-03 → 05 | Context switching/p99 before and after |
| Instrumentation | 8h | prometheus-cpp or self-built `/metrics`, counters/histograms | L5-CPP-04 | 5-C |
| Resilience | 9h | Directly implement retry/backoff/circuit breaker, timeout | L5-C-18/20 | 5-4 |
| Crash | 6h | `SetUnhandledExceptionFilter` + `MiniDumpWriteDump`, WinDbg, PDB management, sanitizers always on | L5-C-31 | 🟡 5-5 |

---

## 5. Textbook Guide (Phase 5)

### 5.1 Common

| Book | Chapters to read | When | How to use |
|---|---|---|---|
| Monitoring Game Servers with Prometheus | All (12 chapters + appendix) | Week 17 | **The direct textbook for 5-C.** Based on Windows installation, so it matches this course exactly. Order: Ch.3 (installation, windows_exporter) → Ch.4 (Grafana) → Ch.6 (API metrics) → Ch.8 (socket server metrics) → Ch.9 (custom exporter) → Ch.10 (alerts) → Ch.11 (dashboards) → Ch.12 (operational scenarios). **Instrument your own 3-1/4-1 instead of the example server.** Keep Appendix B (Grafana queries) at hand during dashboard work |
| Log Collection and Shipping with fluentd | Ch.1 (concepts), Ch.2 (fluent-package for Windows), Ch.3 (first example), Ch.4 (Serilog integration), Ch.5 (socket server logs), Ch.13 (incident response) | Day 85 (reading required, lab 🟡) | Follow only the Windows fluent-package sections. The lab is optional and file logging + PowerShell analysis is also acceptable. However, "what is a log pipeline" is in scope for the oral exam. Upgrade .NET 8-based code to net10.0 to build |
| Network Learning Roadmap for Online Game Developers | Ch.5 (performance optimization), Ch.6 (security: DDoS, anti-cheat) | Days 86, 94 | Ch.5 as methodology reference. After reading Ch.6's DDoS defense, **build a table distinguishing what the application can do from what infrastructure must do** (included in 5-6) |
| Building an API Game Server with ASP.NET Core Web API | Ch.23 (logging/monitoring), Ch.24 (configuration/environment separation) | Days 85, 94 | Re-review the 4-1 server's logs/config against this chapter's standard |
| Building a Game Server with ASP.NET Core Web API | Ch.14 (performance optimization), Ch.16 (hardening security) — Ch.17 (deployment) is not covered in this course | Days 86-94 | Apply Ch.14's optimization items to 4-1 and **measure**. Apply Ch.16 to 5-6 |
| C# Socket Programming for Game Server Development | Ch.8 (server performance testing and optimization), appendix (optimization checklist) | Days 76, 86 | Design reference for the 5-1 tool and the 5-2 checklist. The methodology also applies to the C++ track |
| Practical Guide to Implementing Gacha Probability for New Game Programmers | Ch.15 (live monitoring) | Day 81 | The "business metrics as metrics" perspective — rationale for putting purchase/ranking metrics on the dashboard |

### 5.2 C# Track

| Book | Chapters to read | When | How to use |
|---|---|---|---|
| Mastering C# Async/Await | Ch.5 (async/await on the server — thread starvation), Appendix B (async debugging cheat sheet) | Days 79, 88 | Use the cheat sheet with `dotnet-counters`/`dotnet-dump` when diagnosing ThreadPool starvation under load |
| Building Async/Await Libraries in C# | Ch.14 (rate limiter, retry, circuit breaker), Ch.19 (memory tracking, async leaks), Ch.20 (microbenchmarking) | Days 88-95 | **Ch.14 is the textbook for directly implementing the 5-4 resilience patterns** (even if you use Polly, implement it once yourself to understand the internals). Ch.19 for leak investigation. Build the samples |
| C# Design Patterns for Game Developers | Ch.3 (revisiting Object Pool), Ch.16 (anti-patterns) | Day 87 | Judging the scope of pool application during optimization |

### 5.3 C++ Track

| Book | Chapters to read | When | How to use |
|---|---|---|---|
| Modern Windows Multithreading | Ch.10 (performance analysis and debugging — WPT/ETW), Appendix B (measurement results), Appendix C (troubleshooting) | Days 79, 86 | **The direct textbook for the ETW/WPA lab.** Reproduce the book's measurement scenarios on your own server and compare against Appendix B's numbers |
| Modern Win32 API Programming for Game Server Developers | Ch.7 (performance counters and profiling), Appendix C (optimization checklist) — Ch.10 (services) is not covered in this course | Days 79, 83 | Use Ch.7's performance counters (PDH) to expose your own metrics |
| Practical Thread-Local Storage | Ch.4 (per-thread memory pools, avoiding false sharing, collecting stats), Ch.5 (advanced patterns), Ch.6 (comprehensive example) | Day 88 | The textbook for `L5-CPP-06`. Measurement before/after application is mandatory |
| The Complete Guide to the C++23 Memory Model | Part 3 (spinlocks, lock-free, DCL, fences), Part 4 (platform differences, optimization) | Day 88 | Measure whether relaxing seq_cst → acquire/release actually affects performance. If not, that's a conclusion too |
| TCP/IP Windows Socket Programming Every Game Server Developer Should Know | Ch.12 (zero-copy), Ch.13 (revisiting buffer pooling), Ch.14 (revisiting batch processing) | Day 87 | Candidates for network-layer optimization. Measure the p99 change after applying each technique |
| Safe and Elegant Programming with Modern C++ | Ch.18 (performance benchmarking), Ch.14 (parallel algorithms) | Day 76 | Re-review benchmarking methodology |
| Docker for Game Server Developers | Not read | — | This course does not use Docker |

### 5.4 Secondary-Track Reading

- C# main track: Modern Windows Multithreading Ch.10 → 1 page on "what do you see when you look at a .NET server through ETW" (PerfView is also ETW-based)
- C++ main track: Mastering C# Async/Await Ch.5 → 1 page on "the relationship between .NET thread starvation and IOCP worker shortage"

---

## 6. AI Collaboration Guide (Phase 5)

### 6.1 Prompts

**Bottleneck Hypotheses (No Conclusions Allowed)**
```
Here are load test results: at 300 connections/100 rooms, move p99 is 120ms (target 50ms), CPU is 35%,
GC (or context switching) metrics are (attached), dashboard screenshot (attached).
Give me 3 or more bottleneck hypotheses, and for each, an observation method (profiler view, counter, log) to verify it.
I will verify each hypothesis experimentally and bring back the results. Don't jump to a conclusion first.
```

**Interpreting Profiler Results (My Interpretation First)**
```
Profiler top-function table: (paste)
Build configuration: Release / Collection window: 30s at steady state
My interpretation: "(3 sentences)"
Point out what's wrong with my interpretation, and what cannot be known from this table alone (measurement overhead, sampling limits, inlining).
```

**Metric Design Review**
```
Here is my metric list (name, type, labels, purpose). (paste)
1) Point out labels at risk of cardinality explosion (including an estimate of the number of time series).
2) Tell me if there are failure types this list cannot detect.
3) Evaluate whether the histogram bucket boundaries are appropriate for the target (50ms move).
```

**Generating Failure Scenarios**
```
Given my server setup (game server + API server + DB (SQLite or MySQL) + Redis, all as local Windows processes),
suggest one failure scenario that could realistically happen.
Condition: it must be reproducible, and give the reproduction method (stop service, file lock, firewall rule, delay injection) along with it.
I'll bring back the runbook after responding. Then review the runbook from the perspective of "an operator seeing this for the first time at 3am."
```

**Resilience Code Review**
```
Here is my directly implemented retry/circuit breaker code. (paste)
1) Are there conditions under which a thundering herd of retries could occur?
2) Is there a path where the circuit could stay open forever?
3) What is the worst-case latency when timeout and retry multiply together (calculate it)?
Don't give me fixed code, just point out the problem spots.
```

### 6.2 What to Delegate / What Not to Delegate

| Delegate | Do not delegate |
|---|---|
| Instrumentation boilerplate, draft Grafana PromQL | Choosing which metrics go on the dashboard, deciding bucket boundaries |
| Suggesting failure scenario ideas | The runbook body (write it yourself, only get it reviewed) |
| Retry/circuit breaker skeletons | Deciding timeout values and failure modes |
| Summarizing profile results | Judging bottlenecks, prioritizing optimizations, deciding rejections |

### 6.3 Places Where AI Often Gets It Wrong

- **"This code is slow, so change it this way" without measurement** → demand before/after numbers
- Leaving histogram buckets at defaults, making p99 meaningless → design around the target value (50ms)
- Suggesting infinite/immediate retries → cap, backoff, jitter
- Leaving the circuit breaker open and **forgetting the recovery condition (half-open)**
- Examples that put userId/roomId into metric labels → cardinality explosion
- Collecting and interpreting profiler results during the ramp-up window → collect after reaching steady state
- Oversimplifying lock contention as "just remove the lock" → demand specifics on what to split and how (sharding, actors)

---

## 7. Assignment Specifications

### 7.1 Common Assignment 5-C. Build an Observability Stack (Required, 24h, days 81-85)

**Requirements**

1. **Install**: install Prometheus, Grafana, windows_exporter on your dev PC, scrape 3 targets via `prometheus.yml` (15-second interval)
2. **Game Server (3-1) — 8 Metrics**

| Metric | Type | Labels | Purpose |
|---|---|---|---|
| `game_sessions` | Gauge | - | Current connection count |
| `game_rooms` | Gauge | state(waiting/playing) | Room count |
| `game_packets_total` | Counter | packet(top 20 only), dir(recv/send) | Traffic |
| `game_place_latency_seconds` | Histogram | - | Move processing latency (buckets 1/5/10/25/50/100/250/500ms) |
| `game_send_queue_max` | Gauge | - | Max send queue length |
| `game_room_queue_len` | Gauge | - | Room queue length (max) |
| `game_session_closed_total` | Counter | reason(timeout/error/normal/cheat) | Close reason |
| `game_pool_outstanding` | Gauge | pool(buffer/job) | Unreturned pool objects (leak detection) |

3. **API Server (4-1) — 6 Metrics**: requests/errors/latency histogram per endpoint, DB query latency, Redis command latency, connection pool utilization
4. **System**: CPU, memory, network, disk from windows_exporter
5. **1 Dashboard**: 5 rows (overview / game server / API server / DB·Redis / system), 15+ panels, instance variable. **Explain "why it was added" on each panel**
6. **2+ Alerts**: move p99 >100ms for 1 minute / error rate >1% for 1 minute. Require Alertmanager/Grafana→webhook delivery and a captured receipt log
7. **Logs**: write structured logs to files with daily rolling. 🟡 lab: collect with fluent-package

**Submission**: `prometheus.yml`, `alerts.yml`, `alertmanager.yml` or Grafana contact-point config, `webhook.ps1`, `dashboard.json`, load/delivery captures, metric design document

**Grading**

| Item | Points | Criteria |
|---|---|---|
| Metric design | 40 | 18+ metrics, cardinality controlled, buckets justified, "undetectable failure" mapping |
| Dashboard usefulness | 30 | Can judge healthy/unhealthy within 5 seconds, panels have explanations |
| Alert behavior | 30 | Evidence of actual firing, false-positive-minimizing design (duration conditions) |

### 7.2 Track Assignment 5-1. Scenario-Based Dummy Client (Required, 24h, days 77-80)

Extend the 3-2 client into a production-grade tool.

**Requirements**

1. **Full scenario**: API login (4-1) → connect to game server with token → matchmaking → play → result → check ranking via API → repeat
2. **Parameters**: ramp-up (N users/sec), target connection count, hold duration, message interval, teardown after finish
3. **Real-time stats (5s)**: connection count, TPS, **per-stage** p50/p95/p99 (login, matchmaking, move, ranking), error counts by type
4. **Final report**: JSON + markdown table. **Script for 3 repeated runs** and deviation calculation
5. **Client's own bottleneck indicator**: output client CPU/port usage together (distinguished from server bottlenecks)
6. **Distributed execution**: can split user ID ranges via arguments to run concurrently across 2+ processes
7. Track language (C++ track may also use C# — for tool code)

**Submission**: code, results report for a 1,000-connection ramp-up (20 users/sec), 3-run deviation table

**Grading**: scenario/statistics accuracy 40 / reproducibility (within 10% deviation) 30 / usability/report 30

### 7.3 Common Assignment 5-2. Performance Improvement Report (Required, 40h, days 86-90)

**Requirements**

1. **Baseline**: measure the game server (300 connections/100 rooms, 🟡 1,000 connections/300 rooms) and API (100 concurrent logins) 3 times using 5-1
2. **Profiling**: collect top functions for CPU, allocation, and lock with the track tool (30s at steady state)
3. Improve **at least 2 bottlenecks** through "hypothesis → experiment → adopt/reject → fix → re-measure." **Include at least 1 rejected hypothesis**
4. **Goal (at least one of the following)**: 30%+ improvement in game server move p99, **or** 50%+ increase in connections at the same p99, **or** 30%+ improvement in API login TPS
5. **Check side effects**: changes in memory, CPU, code complexity
6. Write **`PERF-REPORT.md`**

**`PERF-REPORT.md` Template**
```markdown
## 1. Environment
CPU/RAM/OS, build configuration (Release), commit hash, power settings, background control
## 2. Scenario
Tool, scenario name, connection count, ramp-up, hold duration, repetition count
## 3. Baseline (3 runs)
| Metric | Run 1 | Run 2 | Run 3 | Median | Deviation |
| TPS / p50 / p95 / p99 / error rate / CPU / memory |
## 4. Profile Results
Top 5 functions / top allocation types / top lock waits
## 5. Hypotheses and Experiments
| # | Hypothesis | Rationale | Verification method | Result | Adopt/Reject |
(At least 1 rejection required, with the rejection reason backed by numbers)
## 6. Changes Applied
Change 1: what / why / commit hash / impact on code complexity
## 7. Final Before/After (3 runs)
| Metric | Before | After | % improvement |
## 8. Side Effects and Costs
## 9. Remaining Bottlenecks and Next Candidates (to continue in Phase 6)
```

**Grading**

| Item | Points | Criteria |
|---|---|---|
| Measurement reliability | 25 | Release, 3-run deviation, environment recorded, client bottleneck excluded |
| Analysis | 30 | Profiler-backed evidence, **rejected hypothesis included**, hypothesis-vs-measurement comparison |
| Improvement | 30 | Target reached, side effects recorded |
| Documentation | 15 | Reproducible by a third party |

### 7.4 Common Assignment 5-4. Fault Injection + Runbook (Required, 48h, days 91-98)

**Part A — Directly Implement Resilience Patterns (No Library)**

1. **Retry**: cap of 3, exponential backoff (100/200/400ms), jitter ±30%, no-retry conditions (non-idempotent, 4xx)
2. **Circuit breaker**: Closed/Open/Half-Open, failure-rate threshold, minimum request count, cooldown, state transition log
3. **Timeout hierarchy**: game→API (2s), API→DB (1s), API→Redis (200ms). Upper > sum of lower
4. **Backpressure**: cap on room queue/send queue, limit API concurrency, policy when exceeded
5. 🟡 Afterward, swap in Polly etc. and compare **behavioral differences from the direct implementation**

**Part B — Inject 3+ Failure Types**

| Failure | Reproduction method | Expected behavior |
|---|---|---|
| DB failure | (MySQL) stop service for 30s / (SQLite) 30s file lock | 0 crashes, clear error code, circuit opens, automatic normalization after recovery |
| Redis failure | Stop for 30s or repeat 3s delays | In-progress games continue, new logins fail clearly, normalized after recovery |
| Network blocking | Game→API outbound firewall rule for 60s | Result-save queue builds up, drains after recovery, 0 loss/0 duplicates |
| 🟡 Disk full / 90% CPU | Dummy files / load generation | Log failure handling, latency increases but 0 crashes |

**Common pass conditions**: 0 crashes / operates with graceful degradation / normalizes after recovery **without manual intervention** / failure window shown on dashboard/alerts

**Part C — Runbook and Postmortem**

- `RUNBOOK.md`: for 5 failure types (the above 3 + server process down + load spike), as **Detect → Diagnose (check commands/panels) → Respond → Confirm recovery → Follow-up**. Commands should be copy-paste ready
- 1 postmortem (template T8): timeline, root cause, reason for detection delay, improvement items (owner, deadline)

**Submission**: resilience implementation code + tests, fault injection scripts, dashboard captures during failures, recovery time table, `RUNBOOK.md`, postmortem

**Grading**

| Item | Points | Criteria |
|---|---|---|
| Survival/recovery | 35 | 0 crashes across 3 failure types, automatic recovery, recovery time recorded |
| Resilience pattern implementation/understanding | 25 | 4 patterns directly implemented + tested, worst-case latency calculated |
| Runbook quality | 25 | Passes AI "3am operator" review, self-tested in practice |
| Postmortem | 15 | Root cause/detection delay analysis, improvement items |

### 7.5 Track Assignment 5-5. Crash Dump Analysis (Advanced 🟡, 12h, Day 99)

- Insert 3 intentional crash types (null reference / memory corruption / stack overflow)
- Set up automatic dump collection (LocalDumps or procdump), organize symbol (PDB) retention rules
- Pinpoint the root-cause line with a debugger + 3 stack-interpretation notes
- Submission: 3 dumps, analysis notes, symbol management rules

### 7.6 Common Assignment 5-6. Security Check (Required, 28h, days 94-99)

**Requirements**

1. **Rate limiting**: packets/sec per connection, connection attempts/sec per IP, requests/min per API account. Directly implement token bucket + 3 tests
2. **Flood test**: 1,000 connection attempts/sec + 100 normal concurrent users → **table of normal-user p99 change** (before/after defense)
3. **Input validation check**: table of length/range/state validation for every endpoint/packet (0 gaps)
4. **Secrets management**: DB connection info and inter-server tokens as environment variables or permission-restricted files. 0 findings in repository scan, log masking confirmed
5. **Dependency vulnerabilities**: `dotnet list package --vulnerable --include-transitive` / vcpkg check results and remediation
6. **Responsibility boundary table**: what the application can block / infrastructure territory (out of scope for this course)

**Submission**: `SECURITY-CHECK.md` (the 6 items above), rate limiter code + tests, before/after flood table

**Grading**: rate limiting implementation/tests 30 / flood test results 25 / validation/secrets check 25 / boundary awareness and remaining risks 20

---

## 8. Learning Completion Assessment (Day 100, Friday)

### 8.1 Checklist

**Assignments**
- [ ] 5-C: dashboard/alerts working, 18 metrics, evidence of alerts actually firing
- [ ] 5-1: full scenario, within 10% deviation across 3 runs, distributed execution works
- [ ] 5-2: `PERF-REPORT.md` reaches the target (or analyzes why not), **includes a rejected hypothesis**
- [ ] 5-4: 4 resilience patterns implemented, 0 crashes across 3 failure types, `RUNBOOK.md`, postmortem
- [ ] 5-6: `SECURITY-CHECK.md`, rate limiting, before/after flood table
- [ ] 🟡 5-5 or a reason for not doing it

**Measurement**
- [ ] Baseline → post-improvement before/after table (3 runs)
- [ ] Recovery time table per failure
- [ ] Degree of normal-user p99 degradation during flooding

**Records**
- [ ] 25 days of learning notes, 5 weekly retrospectives

### 8.2 Reimplementation Exam (No AI, 60 minutes)

**Problem**: implement a token bucket rate limiter and pass the 4 provided tests.

```
class TokenBucket {
  TokenBucket(int capacity, double refillPerSecond);
  bool TryConsume(int tokens = 1);   // Thread-safe
}
```
Provided tests
1. Burst allowed: with capacity 10, 10 immediate successes, the 11th fails
2. Refill: with 5/sec refill, 5 more successes after waiting 1 second
3. Cap over time: waiting 10 seconds still doesn't exceed the capacity (10)
4. Concurrency: total allowed consumption doesn't exceed the cap even with 8 threads consuming simultaneously

**Pass criteria**: pass all 4 within 60 minutes.

### 8.3 Oral Exam Question Bank (10 questions at random, average 4.0)

1. Why p99 matters more than the average. Why lowering p99 and lowering the average require different methods
2. How did you decide the histogram buckets. What is cardinality explosion
3. The difference between RED and USE. When to use each
4. 5 causes of large deviation across 3 measurement runs and how to control them
5. Explain one rejected hypothesis. Why did it seem plausible, and why was it wrong
6. The three circuit breaker states and their transition conditions. What is the worst-case latency when combined with retry
7. Why add jitter to retries
8. What happens without backpressure. Where did you put the caps
9. How did you design the timeout hierarchy. What happens if the upper timeout is shorter than the lower one
10. What did you keep and what did you give up during "graceful degradation" in a failure
11. How did you ensure automatic normalization after recovery
12. Explain why you added each dashboard panel
13. The boundary between what the application can do and what infrastructure must do in flood defense
14. How did you manage secrets. Where does log masking happen
15. (C#) The relationship between GC pause and p99. What changed after switching to Server GC
16. (C#) How did you diagnose and fix ThreadPool starvation
17. (C++) How did you measure lock contention and false sharing, and what did you change
18. (C++) What could you see with ETW. How is it different from a profiler

### 8.4 Bug Hunt (45 minutes)

Infinite retry / missing timeout / metric cardinality explosion / secrets in logs / no cleanup on process exit — find 4 of 5 + propose fixes.

### 8.5 Response If Standards Are Not Met

| Item not met | Response |
|---|---|
| Performance target not met | Pass if the analysis is sound (the report is what matters). Retry in the Phase 6 capstone |
| Crash during fault injection | **Cannot pass**. Extend up to 1 week to eliminate the cause |
| Alert doesn't fire | Redesign threshold/duration and retest |
| 3-run deviation exceeds 20% | Control the measurement environment and re-measure (measurement reliability is the foundation of this Phase) |
| Runbook fails in live use | Follow it yourself, fill in missing commands, and get it re-reviewed |

---

## 9. Common Sticking Points

| Symptom | Cause | Response |
|---|---|---|
| Prometheus can't scrape a target | Firewall, localhost binding | Allow inbound on the metrics port (internal only), bind to `0.0.0.0` or an internal IP |
| Dashboard graph is stair-stepped/empty | Scrape interval and panel interval mismatch, `rate()` window shorter than the interval | Use `rate(...[1m])` or longer, adjust the panel's minimum interval |
| p99 comes out suspiciously low | Histogram's upper bucket boundary too low, so overflow bunches up | Set the upper bucket (just before +Inf) to 10x the target |
| Profiler results captured with no load | Collected during ramp-up | Collect for 30 seconds after reaching steady state |
| Results differ every time you measure | Background processes, power settings, cache state | High-performance power plan, close apps, warm up before measuring, use the median of 3 runs |
| p99 unchanged after improvement | The bottleneck is elsewhere (Amdahl's Law) | First calculate what % of the total that section represents |
| Retries make the failure worse | Surge of immediate retries | Backoff, jitter, cap, circuit breaker |
| Circuit doesn't close after the failure ends | Missing half-open logic | Allow 1 trial request after cooldown, close on success |
| Server dies during fault injection | No exception-handling boundary | Catch exceptions only at handler entry points, propagate failure as a value |
| Doesn't normalize even after recovery | Connection pool holding dead connections | Validate connections, reset the pool, reconnect with exponential backoff |
| Dummy client dies first | Client CPU/port limits | Distribute across 2 processes, widen the dynamic port range |
| Logs fill up the disk | No rolling/retention policy | Daily rolling + retention period, adjust log level |
| Instrumentation slows things down | String formatting/locking on the hot path | Precompute labels, use atomic counters, sample |

---

## 10. Preparing for Phase 6

- [ ] Read the capstone requirements (document 06 §7.1) and prepare for the scope-decision meeting (AI PM review) in the first two days of Week 21
- [ ] Confirm you can **run 2 game servers simultaneously on different ports on your local PC** (port and server ID separated via config file)
- [ ] Prepare the skeleton of a PowerShell script that launches all 4 process types (API, matching-to-be, 2 game servers) at once
- [ ] Confirm the observability stack can distinguish servers by instance label (per-server panels needed in the capstone)
- [ ] Books: "Building an FPS Game Matchmaking System" Ch.4-7, "2D MMORPG Game Server Development" clone
- [ ] Phase 5 retrospective: the most valuable rejected hypothesis, a problem caught quickly thanks to observability, one case where AI was wrong

---

## 11. 2026-09-05 Revisions (this section wins on conflicts)

### 11.1 Measurement validity and observability contract

- Day 76 adds 40 minutes on closed/open models, coordinated omission, same-host interference, `ProcessorAffinity`, and Windows dynamic ports/TIME_WAIT. Record same-host status, core allocation, DB backend, and client CPU
- Before connection floods, record `netsh int ipv4 show dynamicport tcp`; if needed, apply `start=10000 num=50000` and `TcpTimedWaitDelay=30` as administrator, with rollback steps
- Add 30 minutes on SLI/SLO/error budget; require `instance` and `server_id`. Standardize histogram buckets in seconds as `0.001,0.005,0.01,0.025,0.05,0.1,0.25,0.5,1.0`
- Judge instrumentation overhead as no significant regression beyond repeat variance. Require normal-user success ≥95%, defended p99 ≤ baseline×1.5, and automatic recovery ≤30s
- Standard log fields are `timestamp,level,service,instance,requestId,traceId,sessionId,userId,roomId,event,durationMs,resultCode`, with seven-day/cap retention. Propagate HTTP `X-Request-Id` and packet traceId

### 11.2 Alerts, resilience, and shutdown

#### L5-C-29b Graceful Shutdown (45 min) 🔴
- **Steps**: reject new sessions → notify rooms → apply in-progress-game policy → flush results → exit
- **Acceptance criteria**: exit within five seconds, zero lost/duplicate results, PID and port released

- Require either Prometheus rule→Windows Alertmanager→local webhook or Grafana Alerting contact point→local webhook. `L5-C-13` passes only with captured webhook file-log/toast delivery
- Decide a file-durable or memory result queue in an ADR, require Phase 4 idempotency, and script a 30-second SQLite lock, process stop, and firewall block of new connections
- Day 98 adds `L5-C-29b Graceful shutdown` (45 min): reject new sessions, notify rooms, flush results, exit. Day 97 adds a 30-minute-to-two-hour soak with heap/handle trend
- C++ adds 45 minutes of CRT/VS handle-memory tracking and an HTTP client decision. C# crash injection uses only explicit `Environment.FailFast` or `Marshal` scenarios

### 11.3 Schedule, deliverables, security

- The required 5-1 submission is reproducible 500 connections; 1,000 is recommended. CLI/JSON includes think time, seed, message size, graceful-close ratio, and the T17 schema
- Move `L5-C-11` to Day 84 morning and spread `L5-C-12` across Day 84 afternoon/Day 85 morning. Read fluentd Chapters 1-2; use the rest as reference. Only unauthenticated timeout and per-IP limit are required on Day 95
- Rebudget 5-1/5-4/5-6 as 14h/32h/10h and reserve a weekly half-day buffer. Keep 5-3 as **removed deployment assignment (number reserved)**
- `SECURITY-CHECK.md` covers token expiry/replay, packet userId distrust/session binding, SQL parameters, password hashing, secrets, and TLS. Add 30 minutes for environment config plus port/serverId separation
- Performance reports include per-connection CPU/memory and one-host capacity, three hypotheses with one rejected, Release+PDB, raw data, and commit hash

### 11.4 AI, interview, and troubleshooting checks

- Verify PromQL `sum by(le)`, rate windows ≥2× scrape, and counter resets. Do not copy Docker/Linux `tc`/`iptables`, Polly v7 APIs, or missing-package `KestrelMetricServer` suggestions
- Add questions on load models, same-host distortion, TIME_WAIT, SLO/alerts, request IDs, safe shutdown, leak vs GC, retry idempotency, and capacity. Add troubleshooting for 10048/10055, histogram NaN, undelivered firing alerts, datasource UID, PDB symbols, huge traces, keep-alive firewall behavior, and flaky clocks
