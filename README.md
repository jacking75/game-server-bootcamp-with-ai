# Entry-Level Online Game Server Developer Curriculum

> 🇰🇷 Korean version: [README_kr.md](README_kr.md)
>
> Version 0.4 (2026-09-05 review integrated)
> Audience: learners who know basic C# or C++ and want to enter online-game server development
> Duration: 26 weeks, five 8-hour days per week, approximately 1,040 hours
> Platform: Windows 11, Visual Studio 2026, .NET 10, local scripts, no Docker or CI

This self-study curriculum centers on implementation, measurement, explanation, and deliberate AI use. AI may teach, review, generate exercises, and pair-program; the learner must design first, verify every claim, and reimplement core work without AI.

## Documents

| English (default) | Korean | Scope |
|---|---|---|
| `README.md` | `README_kr.md` | course goals, tracks, policy, roadmap, validation |
| `01-phase1-language-tools.md` | `01-phase1-language-tools_kr.md` | Phase 1: language and tooling |
| `02-phase2-network.md` | `02-phase2-network_kr.md` | Phase 2: networking |
| `03-phase3-realtime-server.md` | `03-phase3-realtime-server_kr.md` | Phase 3: real-time architecture |
| `04-phase4-data-api.md` | `04-phase4-data-api_kr.md` | Phase 4: data and API servers |
| `05-phase5-operations.md` | `05-phase5-operations_kr.md` | Phase 5: performance, observability, security |
| `06-phase6-capstone.md` | `06-phase6-capstone_kr.md` | Phase 6: capstone and hiring preparation |
| `07-books-guide.md` | `07-books-guide_kr.md` | textbook modes and reading budget |
| `08-templates.md` | `08-templates_kr.md` | notes, design, ADR, reports, runbooks, portfolio |

Each Phase follows: usage → goals and prerequisites → day-by-day plan → Lab Catalog → Learning Items in Detail → Textbook Guide → AI collaboration → assignments → completion assessment and Oral exam → troubleshooting → preparation for the next Phase. Korean is the source for curriculum changes; update English in the same commit while preserving heading hierarchy, tables, code fences, and links.

## 0. Outcomes and Tracks

After six months, a learner should be able to:

- implement a Windows TCP game server in the chosen language and keep it stable under hundreds to roughly one thousand local connections
- build an account/lobby/asset API with SQLite by default, optional MySQL, and Redis
- prove improvements with load tests, profiles, logs, metrics, and reproducible reports
- inject and recover from failures using timeouts, retries, circuit breakers, idempotency, and runbooks
- explain architecture, tradeoffs, and measured limits in reviews and technical interviews

Choose one primary track in Week 1. C# emphasizes service tooling, APIs, and production speed; C++ emphasizes IOCP, ownership, memory, and low-level performance. A secondary track is limited to four reading hours per week. Doing both full tracks extends the course to eight or nine months. A mixed path—C++ game server plus C# API/matcher/batch—is explicitly supported.

| Phase | Weeks | Hours | Common | Track |
|---|---:|---:|---:|---:|
| 1. Language and tools | 1-3 | 120 | 40 | 80 |
| 2. Networking | 4-7 | 160 | 70 | 90 |
| 3. Real-time architecture | 8-11 | 160 | 90 | 70 |
| 4. Data and API | 12-15 | 160 | 100 | 60 |
| 5. Operational quality | 16-20 | 200 | 140 | 60 |
| 6. Capstone and hiring | 21-26 | 240 | 130 | 110 |
| **Total** | **26** | **1,040** | **570** | **470** |

## 1. Environment

| Area | Standard |
|---|---|
| OS/IDE | Windows 11 (10 acceptable), Visual Studio 2026 Community, VS Code |
| C# | .NET 10 SDK pinned by `global.json` |
| C++ | MSVC C++20/23, CMake 3.28+, vcpkg manifest mode |
| Tests | xUnit or GoogleTest; local `scripts/build-and-test.ps1` |
| Data | SQLite default; optional MySQL 8 through Repository+DI; redis-windows 6.2+ or Lua fallback |
| Observability | Prometheus, Grafana, windows_exporter, Alertmanager/Grafana Alerting |
| Logging | Serilog for C#, spdlog for C++ |
| AI | any coding agent/chat AI; Claude Code and Codex CLI are examples, not requirements |

## 2. AI Learning Contract

1. Do not commit code you cannot explain.
2. Attempt the problem for 15-20 minutes before asking.
3. Spend one hour every day reimplementing core code or solving scheduled algorithms without AI.
4. Verify AI against official documentation and executable tests; record errors.
5. Create a project instruction file from T3/T3' for every assignment.
6. Keep one daily Markdown note with learning, confusion, AI errors, time use, and next steps.

Daily routine: review/quiz 30m, concepts 2h, implementation 2.5h, no-AI work 1h, review/refactor 40m, notes 20m, with breaks. Friday afternoon is a four-hour checkpoint: 60-120m no-AI reimplementation, a 60m Oral exam, code review/fixes, and retrospective. Every Phase reserves a half-day weekly buffer before expanding scope.

## 3. Roadmap

| Phase | Common Focus | C# | C++ | Main Deliverable |
|---|---|---|---|---|
| 1 | Git, tests, debugging, memory/concurrency, benchmarking | async internals, GC, Span/ArrayPool, Channel | RAII, atomics, Win32 synchronization, ASan/CRT heap | log queue, object pool, reports |
| 2 | TCP semantics, framing, serialization, sessions | Socket/SAEA, Pipelines, MemoryPack, SuperSocketLite comparison | Winsock, select, overlapped I/O, IOCP | chat server, packet library, load client |
| 3 | room actor, fixed tick, authority, reconnect | Channel actors, timers, Serilog | room jobs, worker assignment, pools, spdlog | multi-room Omok server |
| 4 | HTTP, auth, SQLite/MySQL, Redis, transactions | ASP.NET Core, Repository+DI, Dapper | cpp-httplib/Drogon, sqlite3, redis-plus-plus | account/lobby/mail/shop API |
| 5 | load validity, profiles, SLI/SLO, alerts, resilience, security | dotnet-counters/trace, PerfView | VS Profiler, ETW/WPA, CRT/ASan boundaries | load tool, performance report, runbook |
| 6 | contracts, matching, scale-out, E2E, portfolio, interviews | integrated or mixed path | integrated or mixed path | capstone, demo, portfolio, mock interview |

## 4. Textbook Map

Books come from `jacking75/programming-books-with-ai`; [the full guide](07-books-guide.md) is authoritative. Load only designated chapter files into AI context.

| Phase | Required Reading Shape |
|---|---|
| 1 | Async/Await core, Modern C++, Windows multithreading/Win32 memory, memory model |
| 2 | networking foundations; C# Socket Ch.1-7,9,10 or TCP/IP Windows Socket Ch.1-6,8-11,13 |
| 3 | actor/state/game-loop chapters, Omok reference, matching Ch.1/3 |
| 4 | API/DB/Redis core, API lab appendices, consistency-focused gacha chapters |
| 5 | Prometheus; fluentd Ch.1-2 read-through and remaining chapters as reference; profiling books |
| 6 | matching Ch.4-7 and 2D MMORPG as reference architecture |

## 5. Validation

| Item | Method | Pass Condition |
|---|---|---|
| Assignment | required checklist plus correctness/performance logs | all mandatory items complete |
| Reimplementation | empty project, 60-120 minutes, no AI | supplied tests pass |
| Oral exam | 60-minute AI interviewer using T9/T9-b | average at least 4.0/5 |
| Defect finding | injected code with five defects | at least four found |
| Documentation | design, README, ADR, reports | another developer reproduces in ten minutes |

Phase 3 and 5 may extend by at most one week if the core gate fails. Missing work receives eight hours in the next Phase's first week. Capstone targets may be reduced only through the documented 500-connection/100-room path, never by silently lowering correctness gates.

## 6. Ports and Processes

| Phase | Process/Purpose | Default Port |
|---|---|---|
| 2 | chat server | TCP 9000 |
| 3 | game servers/tool | TCP 5000/5001, tool 5050 |
| 4 | API | HTTP 8080; use 8081 when the capstone reserves 8080 |
| 5 | metrics | HTTP 9101 |
| 6 | API, matcher, game servers | 8080, 8090, 9001, 9002 |

Ports and `serverId` must be configurable. Client ephemeral ports are separate and must be checked with `netsh int ipv4 show dynamicport tcp` before connection-flood tests.

## 7. ADR Registry

| ADR | Decision | Phase/Day |
|---|---|---|
| 0001 | serialization | Phase 2 / Day 20 |
| 0002 | room threading model | Phase 3 / Day 37 |
| 0003 | cross-room messaging | Phase 3 / Day 39 |
| 0004 | reconnect authentication | Phase 3 / Day 46 |
| 0005 | DB abstraction | Phase 4 / Day 57 |
| 0006 | password hashing | Phase 4 / Day 59 |
| 0007 | session-token rotation | Phase 4 / Day 61 |
| 0008 | currency/stock source of truth | Phase 4 / Day 66 |
| 0009 | result-queue durability | Phase 5 / Day 91 |
| 0010-0012 | inter-server transport, matching, tickets | Phase 6 / Days 103-107 |
| 0013 | failure recovery | Phase 6 / Day 112 |
| 0014 | result durability | Phase 6 / Day 116 |
| 0015 | capstone scope freeze | Phase 6 / Day 120 |

## 8. Repository Policy

- English documents are the default public entry points; `_kr.md` files are Korean counterparts.
- Apply content changes to Korean first, then mirror only the diff into English in the same commit.
- Keep headings, code fences, tables, checkboxes, and cross-links structurally aligned.
- Use **Lab Catalog**, **Track Labs**, **Learning Items in Detail**, **Textbook Guide**, **Acceptance criteria**, **Expected result**, **Note**, **Common mistake**, **Oral exam**, **Secondary-track reading**, and **Preparing for Phase N+1** consistently in English.
- Markdown uses LF via `.gitattributes`.
