# Phase 6. Capstone Project & Job-Hunting Prep (Weeks 21-26, 240h)

> 🇰🇷 Korean version: [06-phase6-capstone_kr.md](06-phase6-capstone_kr.md)

> Common 130h / Track 110h. By the end of this Phase, learners will have integrated everything built so far into a **complete service with a multi-server configuration**, and will have built a portfolio they can explain to others along with interview-ready skills.

## 0. How to Use This Document

Same structure as Phases 1-5. Follow the daily blocks in §2 (days 101-130); practices (`L6-C-xx`) are in §3, and assignment details are in §7.

Four principles of this Phase

1. **Scope changes only through an ADR.** The #1 reason capstones fail is scope creep. Lock the scope on Tuesday of week 21, and record any subsequent change as an ADR without exception.
2. **No deployment.** Run all four server types (API, matching, 2 game servers, batch worker) **simultaneously on your local PC as separate processes on different ports**. Windows service registration, remote deployment, and CI/CD are out of scope for this course. Instead, build a **PowerShell script that starts/stops everything at once**.
3. **Don't cram documentation at the end.** Starting week 24, spend 1 hour every day on documentation. Write ADRs the moment a decision is made.
4. **You must be able to explain every line of AI-generated code.** This will absolutely come up in interviews. `AI-USAGE.md` is your answer to that.

| Notation | Meaning |
|---|---|
| `L6-C-xx` | Capstone exercise (common to all tracks) |
| 🔴 Required · 🟡 Optional | Priority |

---

## 1. Overview

### 1.1 Goals

1. **Multi-server configuration**: Run an API server + matching server + 2 or more game servers + a batch worker simultaneously on a local machine, and implement inter-server communication and state sharing
2. **Session routing**: Build the flow of login token → matching → **single-use connection ticket** → game server connection
3. **Server-down handling**: Even if one game server is force-killed, matching excludes it within 15 seconds, and users successfully re-match after returning to the lobby
4. **Quality proof**: Pass 1,000 concurrent connections, 200 rooms, 10 minutes, and record p99
5. **Documentation**: Have a README, ARCHITECTURE doc, 5 ADRs, a performance report, a runbook, and an AI-usage record
6. **Job-hunting prep**: A 30-minute presentation, 3 rounds of 60-minute mock interviews, a resume, and 3 technical blog posts

### 1.2 Prerequisite Self-Check (30 min)

| # | Question | Passing criterion |
|---|---|---|
| 1 | Did the 3-1 game server pass the 100-room load test | Have `LOAD-3-1.md` |
| 2 | Did the 4-1 API server pass the concurrency test | Have test logs |
| 3 | Is the 5-C observability stack working | Dashboard·alerts |
| 4 | Are resilience patterns (retry, circuit breaker, backpressure) from 5-4 implemented | Code·tests |
| 5 | Can you run two game servers simultaneously on different ports | Confirmed running |
| 6 | Can the shared libraries (PacketLib·Rules·Pool) be referenced | Build confirmed |

**If fewer than 4 pass, reduce the capstone scope** (e.g., 2 game servers → 1 game server plus just splitting off the matching server, or drop the content expansion). Dragging along unfinished assignments means you won't finish within 6 weeks.

### 1.3 What You'll Be Able to Do After This Phase

- Explain your entire server **using a single architecture diagram** in a hiring interview
- Distinguish "what AI made vs. what I made," while still being able to **fully explain the parts AI made**
- Have a stranger succeed at running it locally within 10 minutes using only the README
- Reimplement the core of the matching server (queue, server selection, ticket) in 120 minutes
- Present the 1,000-connection load test results and server-down recovery time as concrete numbers

### 1.4 Deliverables for This Phase

```
capstone/
├─ README.md                 One-screen summary, architecture diagram, 10-minute setup steps, how to test
├─ ARCHITECTURE.md           Components, sequence, state, execution layout diagram, state placement table
├─ adr/0010~0015.md          Scope, inter-server communication, matching, tickets, state storage location, server-down policy
├─ PROTOCOL.md / API.md      Final version
├─ PERF-REPORT.md            Capstone-baseline load test results
├─ RUNBOOK.md                Matching/game/Redis/API incident response
├─ AI-USAGE.md               AI usage record (required interview material)
├─ src/
│  ├─ ApiServer/             Extension of 4-1
│  ├─ MatchServer/           New
│  ├─ GameServer/            Extension of 3-1 (same binary run twice with different port/ID)
│  ├─ BatchWorker/
│  └─ Shared/                PacketLib·Rules·Pool·Resilience
├─ scripts/
│  ├─ start-all.ps1          Starts all 4 server types (port/ID arguments)
│  ├─ stop-all.ps1           Shuts everything down
│  └─ smoke-test.ps1         Health check after startup
├─ tests/
└─ demo/                     5-minute demo video

portfolio/
├─ index (GitHub Profile README or a static page)
├─ blog/                     3 technical blog posts
├─ presentation/             15-slide presentation
├─ interviews/               Records of 3 mock interviews
└─ resume/                   Resume·cover letter
```

### 1.5 6-Week Roadmap

| Week | Main topic | Common | Track | Assignment |
|---|---|---|---|---|
| Week 21 | Scope, design, foundation | Lock scope (ADR), learn multi-server configuration, state storage location | Choose/implement inter-server communication, organize shared libraries | 6-C design, execution scripts |
| Week 22 | Matching and routing | Design matching server, ticket design, heartbeat | Implement matching server, game server status reporting | 6-1 matching + routing across 2 servers |
| Week 23 | Failures and content | Server-down handling, result-storage reliability | Content expansion (spectating, ranking, match history) | 6-1 failure handling |
| Week 24 | Integrated load and docs | 1,000 connections·200 rooms, observability, runbook update | Optimization, fixes | 6-1 quality gate passed, docs |
| Week 25 | Portfolio | Standardize repos, 3 blog posts, presentation materials, demo video | Prepare 30 track interview questions | Start 6-2, 6-3 |
| Week 26 | Interviews, wrap-up | Mock interviews daily, resume, final presentation, course retrospective | Coding-test review | 6-3, 6-4, final assessment |

---

## 2. Weekly Detailed Plan (Day by Day)

### 2.1 Week 21 — Deciding Scope and Building the Foundation

#### Day 101 (Mon) — Organizing Requirements and Drafting Scope 🔴

**Morning (2.5h)**
- Read the capstone requirements in §7.1 and write your own **Required / Optional / Excluded** table
- Calculate remaining time: 4 weeks of implementation (weeks 21-24) + 2 weeks of wrap-up (weeks 25-26). 8 hours/day, solo. **Assume actual available time is 30% less than that**
- Risk list: new matching-server development, inter-server communication, 1,000-connection load, 6 types of documents

**Afternoon (2.5h)**
- `L6-C-01` Write the scope table (required/optional/excluded per feature + estimated time)
- Draft milestones: what should be working by each week

**1 Hour Without AI**
- Decide "the single strongest thing to show an interviewer in this capstone" (build the scope around it)

**DoD**
- [ ] Scope table (required/optional/excluded + time estimate)
- [ ] Draft weekly milestones

#### Day 102 (Tue) — Locking Scope (AI PM Review) 🔴

**Morning (2.5h)**
- `L6-C-02` AI PM review (§6.1 prompt): 3 schedule-overrun risks, things that can be cut, things that must not be cut
- **I make the final call.** Record which AI opinions were accepted and which were rejected

**Afternoon (2.5h)**
- Write `adr/0010-scope.md`: scope and rationale, excluded items and why, the change process (from now on, scope changes only via ADR)
- Finalize weekly milestones + register them as issues/checklists in the repository

**1 Hour Without AI**
- Look at the milestones and predict "where you'll get stuck first," and write a contingency plan

**DoD**
- [ ] Commit ADR-0010 (scope)
- [ ] Milestones finalized; future changes only via ADR

#### Day 103 (Wed) — Learning Multi-Server Configuration 🔴

**Morning (2.5h)**
- Separating server roles: login/API (stateless), matching (queue state), game (holds room state), batch
- **Managing the server list**: static config vs. Redis registration. Share status (connection count, room count, version) via heartbeat
- Options for inter-server communication: Redis Pub/Sub (can lose messages) / gRPC (request-response) / custom TCP (reuse from 2-2). **Summarize the trade-offs**

**Afternoon (2.5h)**
- `L6-C-03` Comparison table of inter-server communication approaches + write `adr/0011-inter-server.md`
- `L6-C-04` Design the heartbeat protocol: game server → Redis every 5 seconds, `server:{id}` key (connection count, room count, version, TTL 15s)

**1 Hour Without AI**
- Explain with a calculation why "TTL 15s, heartbeat every 5s" makes sense (why 3x)

**DoD**
- [ ] ADR-0011 (inter-server communication)
- [ ] Heartbeat design document

#### Day 104 (Thu) — State Storage Location and Shared Libraries 🔴

**Morning (2.5h)**
- Write a **state placement table**: data × storage location × loss tolerance

| Data | Location | Loss tolerable | Rationale |
|---|---|---|---|
| Account, currency, items | DB | No | Ledger |
| Login token session | Redis | Partial (re-login) | Needs TTL |
| Connection ticket | Redis | Yes (re-match) | Single-use, short-lived |
| Which game server a user is on | Redis | Partial | For routing |
| Room state (match, turn) | Game server memory | Yes (game voided) | Performance first |
| Server list·load | Redis (TTL) | Yes | Regenerated via heartbeat |
| Game results | DB | No | Ranking, match history |

**Afternoon (2.5h)**
- `L6-C-05` Organize shared libraries and process skeletons: gather PacketLib, Rules, Pool, Resilience into `Shared/`; create API, Match, and Game skeletons plus two game-instance configs/PID files
- Clean project references, remove cycles, tag `v1.0`

**1 Hour Without AI**
- On the state placement table, mark "what disappears if the game server dies" and write down the impact

**DoD**
- [ ] State placement table (7+ rows)
- [ ] `Shared/` library organized and tagged

#### Day 105 (Fri) — Local Multi-Process Execution + Weekly Checkpoint 🔴

**Morning (3h)**
- `L6-C-06` Write the execution script
  ```powershell
  # scripts/start-all.ps1 (outline)
  # Build Release once, run DLLs/executables directly, and record PID files.
  Start-Process dotnet -ArgumentList "artifacts/ApiServer.dll --port 8080" -PassThru
  Start-Process dotnet -ArgumentList "artifacts/MatchServer.dll --port 8090" -PassThru
  Start-Process dotnet -ArgumentList "artifacts/GameServer.dll --port 9001 --server-id game-1" -PassThru
  Start-Process dotnet -ArgumentList "artifacts/GameServer.dll --port 9002 --server-id game-2" -PassThru
  ```
  Also include a shutdown script and a health check (smoke-test)
- Separate config: server ID, port, and metrics port via arguments or environment variables

**Afternoon (4h) — Weekly Checkpoint**
1. Reimplement without AI (60 min): heartbeat registration/lookup (Redis TTL-based)
2. Explanation test (45 min): rationale for inter-server communication choice, state placement
3. Code review (45 min): boundaries of the shared library
4. Retrospective (30 min): `W21.md`

**DoD**
- [ ] `start-all.ps1` starts all 4 process types in one run, `smoke-test.ps1` passes
- [ ] 2 game servers run simultaneously with different ports/IDs

### 2.2 Week 22 — Matching Server and Session Routing

#### Day 106 (Mon) — Matching Server Skeleton 🔴

**Morning (2.5h)**
- Matching server responsibilities: queue management, matching cycle, game server selection, ticket issuance, assignment notification, cancel/timeout
- Queue data structure: simple FIFO (first-come-first-served) or 🟡 rating buckets
- Game server selection criteria: **valid heartbeat** + version match + **fewest rooms**

**Afternoon (2.5h)**
- `L6-C-07` Create the matching server project: client connects (token verification) → receives match request → queue
- `L6-C-08` Server selection logic + 5 unit tests (normal selection, excludes server with expired heartbeat, excludes version mismatch, waits when none available, round-robin on ties)

**1 Hour Without AI**
- Write down "what disappears and how it's recovered if the matching server dies" (is the queue in memory? Redis?)

**DoD**
- [ ] Matching server verifies tokens and adds to the queue
- [ ] 5 server-selection tests pass

#### Day 107 (Tue) — Connection Ticket 🔴

**Morning (2.5h)**
- Ticket design: single-use random token → Redis `ticket:{value}` = `{userId, roomId, gameServerId}`, TTL 30s
- Ticket verification on the game server: **atomic verify + delete** (`GETDEL` or Lua) — the key to preventing reuse
- Why using only the token without a ticket doesn't work: the game server can't distinguish arbitrary connections, and there's nowhere to carry room-assignment info

**Afternoon (2.5h)**
- `L6-C-09` Implement ticket issuance/verification + 5 tests (normal, reuse rejected, expired rejected, forged rejected, ticket for a different server rejected)
- `adr/0012-ticket.md`: ticket design and reasons for rejecting alternatives (token reuse, signed tokens)

**1 Hour Without AI**
- Write down ticket-theft scenarios and defenses (short TTL, single-use, server binding)

**DoD**
- [ ] 5 ticket tests pass (especially reuse rejection)
- [ ] ADR-0012

#### Day 108 (Wed) — Game Server Heartbeat / Status Reporting 🔴

**Morning (2.5h)**
- Game server registers to Redis every 5 seconds: connection count, room count, version, metrics port, TTL 15s
- Matching server queries the server list every matching cycle → only valid servers are candidates
- **Recheck right before assignment**: after selection, before issuing the ticket, check the heartbeat once more (mitigating TOCTOU)

**Afternoon (2.5h)**
- `L6-C-10` Implement heartbeat publishing/lookup
- `L6-C-11` Connect the full flow: login (API) → connect to matching server → matching request → match found → issue ticket → connect to game server (ticket verification) → play
- Flow logging: propagate a single `traceId` from API → matching → game server to stitch the logs together

**1 Hour Without AI**
- Draw the full flow as a sequence diagram (it goes straight into the documentation)

**DoD**
- [ ] The entire section from login to before play succeeds
- [ ] Logs from all three servers are linked by traceId

#### Day 109 (Thu) — Routing Across 2 Game Servers 🔴

**Morning (2.5h)**
- With 2 game servers running simultaneously, confirm that rooms are **evenly distributed** across both when matched
- Store user location: `user_location:{userId}` = gameServerId (TTL refreshed)

**Afternoon (2.5h)**
- `L6-C-12` Verify distribution: after 100 matches, the difference in room count between servers is within 20%
- `L6-C-13` Organize the game-result storage path: game server → internal API (idempotent by gameId) → DB, ranking. Retry queue on failure

**1 Hour Without AI**
- Imagine a case where "select the server with fewest rooms" becomes unbalanced (long games vs. short games), and write a mitigation

**DoD**
- [ ] Distribution table for 100 matches (room count per server)
- [ ] Result storage works on both servers

#### Day 110 (Fri) — Matching Quality + Weekly Checkpoint

**Morning (3h)**
- Handle match cancellation/timeout (60s), handle disconnection during matching
- Add matching metrics: number waiting, average wait time, match rate, match-time p99 → dashboard panel

**Afternoon (4h) — Weekly Checkpoint**
1. Reimplement without AI (90 min): matching queue + server selection + ticket issuance (network faked)
2. Explanation test (45 min): ticket design, heartbeat TTL, server selection criteria
3. Code review (45 min): concurrency in the matching server (queue access)
4. Retrospective (30 min): `W22.md`

**DoD**
- [ ] Match cancel/timeout works
- [ ] Matching metrics shown on the dashboard

### 2.3 Week 23 — Server-Down Handling and Content

#### Day 111 (Mon) — Detecting Server-Down 🔴

**Morning (2.5h)**
- Detection: heartbeat TTL expires (15s) → matching server excludes it from candidates
- Users who were on that server: since the server is dead, **the server can't notify them** → the client detects the disconnection and returns to the lobby → re-match
- In-progress games: record as "voided" (no result saved, or the void reason is recorded)

**Afternoon (2.5h)**
- `L6-C-14` Server-down scenario script: force-kill one game server (`Stop-Process -Force`)
- Observe: excluded from matching within 15s, assignment goes only to the remaining server, client successfully re-matches, voided game is recorded
- `adr/0015-server-down.md`: policy and rationale for voiding vs. saving in-progress games

**1 Hour Without AI**
- Write 3 paragraphs on how the constraint "the server can't notify users of a dead server" affects the design

**DoD**
- [ ] Force-kill scenario passes (excluded within 15s, re-match succeeds)
- [ ] ADR-0015

#### Day 112 (Tue) — Result-Storage Reliability 🔴

**Morning (2.5h)**
- Retry queue (in game-server memory) + exponential backoff + gameId idempotency
- **If the process dies, the queue disappears** — is that acceptable? If so, justify it; if not, design a file/Redis-backed backup
- This trade-off is a classic interview question

**Afternoon (2.5h)**
- `L6-C-15` Implement/test the result-storage retry queue: stop the API for 60s → queue builds up → drains after recovery, 0 duplicate saves
- 🟡 Persist the queue (append to a file, or a Redis List)

**1 Hour Without AI**
- Write 3 sentences on "why it's OK for the retry queue to disappear" (or, if it shouldn't, why not and what to do about it)

**DoD**
- [ ] 0 loss, 0 duplicates in the 60-second API-outage scenario
- [ ] Trade-off recorded in ARCHITECTURE.md

#### Day 113 (Wed) — Matching-Server Failure and Recovery 🔴

**Morning (2.5h)**
- When the matching server goes down: the queue disappears → clients reconnect and re-request. In-progress games are unaffected (important)
- 🟡 Putting the queue in Redis makes it recoverable but adds complexity and latency — choose and justify

**Afternoon (2.5h)**
- `L6-C-16` Force-kill-then-restart scenario for the matching server: in-progress games continue, waiting users recover via re-request
- Add a matching-server-down entry to `RUNBOOK.md`

**1 Hour Without AI**
- Make a table of "what stops and what continues" when each of the three servers (API, matching, game) dies

**DoD**
- [ ] Matching-server down/recovery scenario passes
- [ ] Failure-impact table (3 servers × impact)

#### Day 114 (Thu) — Content Expansion 🔴

**Morning (2.5h)**
- Pick one content option (item 5 in §7.1): (a) Gomoku + spectating + room chat + **seasonal ranking + match history** (b) 2D action for 4 players (c) a pre-approved new game
- (a) is recommended: polishing something that already exists to raise its completeness works better for a capstone

**Afternoon (2.5h)**
- `L6-C-17` Match history system: aggregate game results per user (wins, losses, draws, win streaks), expose via profile API
- `L6-C-18` Seasonal ranking: separate season keys, save a snapshot when a season ends, query previous seasons

**1 Hour Without AI**
- Decide whether match-history aggregation should update in real time vs. via batch, and write the rationale

**DoD**
- [ ] The chosen content feature works
- [ ] Match history and season ranking queryable via API

#### Day 115 (Fri) — Integration Check + Weekly Checkpoint

**Morning (3h)**
- Full integrated smoke test: start up → login → matching → play → results → ranking/match history → server down → re-match
- Extend `smoke-test.ps1` (automated verification)

**Afternoon (4h) — Weekly Checkpoint**
1. Reimplement without AI (90 min): ticket issuance/verification (including atomic deletion)
2. Explanation test (45 min): server-down handling, result-storage reliability
3. Code review (45 min): boundaries between servers and error propagation
4. Retrospective (30 min): `W23.md`

**DoD**
- [ ] Smoke test automated (verifies all functionality within 1 minute of startup)
- [ ] Retrospective

### 2.4 Week 24 — Integrated Load Testing and Documentation

#### Day 116 (Mon) — 1,000-Connection Load Test Round 1 🔴

**Morning (2.5h)**
- Using the 5-1 tool, run the full scenario, ramp-up 20 users/sec, target 1,000 connections·200 rooms, 10 minutes
- Split the client across 2 processes (separate user-ID ranges)

**Afternoon (2.5h)**
- `L6-C-19` Round-1 measurements: connect-p99 / match-time p99 / login p99 / error rate / room distribution per server / resource usage
- Record failure points (connection failures, matching delays, memory growth, etc.)

**1 Hour Without AI**
- From the round-1 results, name 3 candidate bottlenecks and how you'd confirm each

**DoD**
- [ ] Round-1 load results table (record it even if targets weren't met)
- [ ] 3 bottleneck hypotheses

#### Day 117 (Tue) — Load Tuning and Re-Measurement 🔴

**Morning (2.5h)**
- Verify hypotheses (profiler, dashboard) → fix. Common bottlenecks: matching cycle length, number of Redis round-trips, game-server room queue, synchronous result-storage calls

**Afternoon (2.5h)**
- `L6-C-20` Re-measure after fixes (3 runs), before/after comparison table
- Target: pass 1,000 connections·200 rooms·10 minutes, keep connect-p99 at Phase 5 levels

**1 Hour Without AI**
- Cross-check that improvements didn't make other metrics worse

**DoD**
- [ ] Pass 1,000 connections·200 rooms·10 minutes (or lower to 500 connections·100 rooms and write a root-cause analysis)
- [ ] Before/after comparison table

#### Day 118 (Wed) — Observability and Runbook Update 🔴

**Morning (2.5h)**
- Add a **per-server panel** to the dashboard (instance label), new matching-server panel
- 3+ alerts: connect-p99, match time, game-server heartbeat loss

**Afternoon (2.5h)**
- `L6-C-21` Update the runbook: matching-server down / game-server down / Redis down / API down / load spike
- Actually inject each failure and respond following the runbook (real-use test of the runbook)

**1 Hour Without AI**
- Decide the **order of panels to check first** when an alert fires (diagnostic path)

**DoD**
- [ ] Per-server dashboard panels, confirmed 3 alerts fire
- [ ] All 5 runbook entries verified through real use

#### Day 119 (Thu) — Writing Documentation ① 🔴

**Morning (2.5h)**
- `ARCHITECTURE.md`: component diagram / sequence (login → play → result) / state diagram / **execution layout diagram (multi-process, ports)** / state placement table
- Organize ADRs: 0010 scope / 0011 inter-server communication / 0012 ticket / 0013 state storage location / 0014 matching approach / 0015 server-down policy

**Afternoon (2.5h)**
- `README.md`: one-screen summary → architecture diagram → **local setup steps within 10 minutes** → running tests → how to use the scripts
- `L6-C-22` README reproduction test (§6.1 prompt): have an AI "developer seeing this for the first time" describe the setup steps using only the docs → fill in the gaps where it got stuck

**1 Hour Without AI**
- Actually clone into a different folder and, using only the README, try to get it running within 10 minutes (self-verification)

**DoD**
- [ ] `ARCHITECTURE.md`, 6 ADRs
- [ ] README reproduction test passes (0-1 guessed items)

#### Day 120 (Fri) — Writing Documentation ② + Weekly Checkpoint

**Morning (3h)**
- Write `PERF-REPORT.md` (capstone baseline), `AI-USAGE.md` (format from §7.1)
- Finalize `PROTOCOL.md`/`API.md` (reflecting packets/endpoints added in the capstone)

**Afternoon (4h) — Weekly Checkpoint**
1. Reimplement without AI (90 min): server-selection logic (valid heartbeat + fewest rooms)
2. Explanation test (45 min): explain the full architecture with a single diagram
3. Code review (45 min): from the whole-capstone perspective, 3 "problems that get expensive if not fixed now"
4. Retrospective (30 min): `W24.md`

**DoD**
- [ ] All 6 documents complete (README, ARCHITECTURE, ADR, PERF, RUNBOOK, AI-USAGE)
- [ ] Demo-video shooting plan (1 min setup, 2 min play, 1 min server-down, 1 min dashboard)

### 2.5 Week 25 — Portfolio and Presentation Prep

#### Day 121 (Mon) — Demo Video and Repository Standardization 🔴

**Morning (2.5h)**
- `L6-C-23` Shoot a 5-minute demo video: ① 1 min setup explanation ② 2 min play ③ 1 min server-down/recovery ④ 1 min dashboard
- Write a script before filming (so it goes smoothly without getting stuck)

**Afternoon (2.5h)**
- `L6-C-24` Standardize README across the Phase 2-5 repos: summary / structure / setup / testing / lessons learned / metrics
- Add a license, test-execution instructions, and result logs to each repo

**DoD**
- [ ] 5-minute demo video
- [ ] README standardized across 5 repos

#### Day 122 (Tue) — Portfolio Index and Blog Post ① 🔴

**Morning (2.5h)**
- `L6-C-25` Portfolio index (GitHub Profile README or a static page): cards for the 5 repos, capstone architecture diagram, **metrics summary** (connection count, p99, improvement rate, test count)

**Afternoon (2.5h)**
- Write technical blog post 1 (1,500+ characters, including code and metrics). Example topic: "Chasing Down a Session-Close Race in an IOCP (or SAEA) Server"
- Structure: problem → reproduction → root-cause analysis → fix → metrics → lessons learned

**DoD**
- [ ] Portfolio index published
- [ ] Blog post 1 done

#### Day 123 (Wed) — Blog Posts ②③ + Presentation Materials 🔴

**Morning (2.5h)**
- Blog post 2: "Comparing Three Fixes for a Shop-Purchase Concurrency Bug" (with measured numbers)
- Blog post 3: "How I Cut p99 by 30% and the Hypotheses I Rejected"

**Afternoon (2.5h)**
- `L6-C-26` Presentation materials, about 15 slides: problem → design → **3 key decisions (ADRs)** → metrics → incident response → AI usage → retrospective
- Practice explaining the whole thing with a single diagram (3-minute version)

**DoD**
- [ ] All 3 blog posts complete
- [ ] Presentation draft

#### Day 124 (Thu) — Mock Interview Round 1 🔴

**Morning (2.5h)**
- `L6-C-27` Mock interview round 1 (60 min): 20 min portfolio deep-dive + 25 min general CS + 15 min language
- Record: questions, answer summaries, score, areas to improve

**Afternoon (2.5h)**
- Strengthen weak areas: re-study the 2 lowest-scoring topics and rewrite your answers
- Start preparing 30 track interview questions (C#: GC/async/memory/collections, C++: memory model/ownership/templates/UB)

**DoD**
- [ ] Mock interview 1 record
- [ ] Notes on strengthening 2 weak areas

#### Day 125 (Fri) — Presentation Rehearsal + Weekly Checkpoint

**Morning (3h)**
- 2 presentation rehearsals (30 min each), take questions from an AI audience
- Revise materials: remove slides without numbers, simplify diagrams

**Afternoon (4h) — Weekly Checkpoint**
1. Reimplement without AI (60 min): one component that was weak
2. Explanation test (45 min): the weakest part of the presentation content
3. Code review (45 min): first-impression review of the portfolio repos (from a stranger's perspective)
4. Retrospective (30 min): `W25.md`

**DoD**
- [ ] 2 presentation rehearsals, revised materials
- [ ] Draft self-answers for the 30 track interview questions

### 2.6 Week 26 — Interviews and Wrap-Up

#### Day 126 (Mon) — Mock Interview Round 2 + Resume 🔴

**Morning (2.5h)**
- Mock interview round 2 (60 min, strict-interviewer persona), record and note improvements
- Start reviewing coding tests: arrays, strings, hashing (2 problems/day)

**Afternoon (2.5h)**
- `L6-C-28` Write your resume: describe projects using a **"Problem → Choice → Result (with numbers)"** structure. Only verifiable claims
- Draft cover letter

**DoD**
- [ ] Mock interview round 2 record
- [ ] Resume draft (3+ projects)

#### Day 127 (Tue) — Resume Review and Coding Tests 🔴

**Morning (2.5h)**
- `L6-C-29` AI resume review (from both a recruiter's and a technical interviewer's perspective), remove exaggerations and unverifiable claims
- Write the revision

**Afternoon (2.5h)**
- Coding tests: sorting, stack/queue (2 problems), solve without AI then have AI review complexity and alternatives afterward

**DoD**
- [ ] Revised resume
- [ ] 4 cumulative coding-test problems

#### Day 128 (Wed) — Mock Interview Round 3 🔴

**Morning (2.5h)**
- Mock interview round 3 (60 min, practitioner-style interviewer), **target: last round averages 4.0+**
- Record and note improvements; summarize the score trend across the 3 rounds

**Afternoon (2.5h)**
- Coding tests: BFS/DFS, binary search (2 problems)
- 🟡 Ask a working developer for a presentation and feedback (online community, acquaintance)

**DoD**
- [ ] All 3 mock interviews complete, score-trend table
- [ ] 6 cumulative coding-test problems

#### Day 129 (Thu) — Final Presentation 🔴

**Morning (3h)**
- 30-minute final presentation (AI interviewer, including a human if possible) + Q&A
- Evaluation: average 4.0+ on a 5-point scale. **Distinguish "what AI made vs. what I made"** while explaining, and fully explain the AI-made parts too

**Afternoon (2h)**
- Final revisions to presentation materials/documents based on feedback
- Coding tests: priority queue, simple DP (2 problems)

**DoD**
- [ ] Final presentation complete, evaluation recorded
- [ ] 8 cumulative coding-test problems

#### Day 130 (Fri) — Final Assessment and Course Retrospective 🔴

**Morning (3h)**
1. **Reimplementation exam (120 min, no AI)**: §8.2 — matching-server core + pass 4 provided tests
2. Check off every item in the checklist (§8.1)

**Afternoon (3h)**
- Reflect on the whole course: what grew the most over the 26 weeks, what you'd change if you did it again
- Compile a **post-hire learning list** (§10): load balancers, cloud, containers, UDP synchronization, a second language
- Leave curriculum-improvement suggestions as an issue in the course repository (for future learners)

**DoD**
- [ ] All items in §8.1 complete
- [ ] Course retrospective + post-hire learning list

---

## 3. Lab Catalog

### 3.1 Capstone Labs (L6-C)

#### L6-C-01 Write the Scope Table (60 min) 🔴
- **Steps**: Features as rows, columns for `Required/Optional/Excluded · estimated time · rationale`. If the sum of estimated time exceeds 70% of available time, cut something
- **Acceptance criteria**: Each excluded item states "why portfolio value is preserved despite dropping it"

#### L6-C-02 AI PM Review (60 min) 🔴
- **Steps**: Using the §6.1 prompt, get 3 schedule risks, exclusion candidates, and required items, and record accept/reject for each
- **Acceptance criteria**: The final decision is recorded as **your own judgment** and reflected in ADR-0010

#### L6-C-03 Inter-Server Communication Comparison (60 min) 🔴
- **Steps**: Compare Redis Pub/Sub / gRPC / custom TCP by latency, reliability, implementation cost, debugging difficulty, and loss characteristics
- **Acceptance criteria**: A table + choice rationale. If Pub/Sub is chosen, state the **message-loss characteristics** explicitly in the ADR

#### L6-C-04 Heartbeat Design (45 min) 🔴
- **Steps**: Key structure (`server:{id}`), fields (connection count, room count, version, metrics port), 5s interval·15s TTL, handling of update failures
- **Acceptance criteria**: Calculation justifying TTL ≥ interval × 3

#### L6-C-05 Shared Libraries and Four-Process Skeleton (120 min) 🔴
- **Steps**: organize `Shared/`, remove cycles, create API/Match/Game skeletons plus two game-instance config/PID files, add version tag
- **Acceptance criteria**: API, matcher, and two game instances build Release while referencing `Shared/`

#### L6-C-06 Execution Scripts (75 min) 🔴
- **Steps**: `start-all.ps1` (port/server-ID arguments), `stop-all.ps1`, `smoke-test.ps1` (4 health checks)
- **Acceptance criteria**: One run starts all 4 types, smoke test passes, shutdown script cleans everything up
- **Common error**: Leftover processes holding ports → track PIDs in the shutdown script

#### L6-C-07 Matching Server Skeleton (120 min) 🔴
- **Steps**: Client connects → token verification (via API or direct Redis) → receive match request → register to queue → periodic matching loop
- **Acceptance criteria**: When 2 users request, a match event fires (still no ticket yet, just logs)

#### L6-C-08 Server-Selection Logic (75 min) 🔴
- **Steps**: Valid heartbeat + version match + fewest rooms. Round-robin on ties
- **Acceptance criteria**: 5 tests (normal, excludes expired, excludes version mismatch, no candidates, tie)

#### L6-C-09 Ticket Issuance/Verification (90 min) 🔴
- **Steps**: Issue (random 32 bytes, Redis TTL 30s, value holds userId/roomId/serverId) / verify (**atomic verify + delete via `GETDEL` or Lua**)
- **Acceptance criteria**: 5 tests — especially **reuse rejection** and **rejecting a ticket meant for a different server**

#### L6-C-10 Heartbeat Publish/Lookup (60 min) 🔴
- **Steps**: Game server publishes on a 5-second timer, matching server looks it up every matching cycle + **rechecks right before assignment**
- **Acceptance criteria**: Disappears from the list within 15 seconds after force-killing a game server

#### L6-C-11 Connect the Full End-to-End Flow (120 min) 🔴
- **Steps**: Login → connect to matching → match → ticket → connect to game → play → save result. Propagate `traceId`
- **Acceptance criteria**: One game's logs are linked across all three servers by traceId

#### L6-C-12 Room-Distribution Verification (45 min)
- **Steps**: Tally room counts per server after 100 matches
- **Acceptance criteria**: Difference between servers within 20%. If exceeded, revisit selection criteria

#### L6-C-13 Result-Storage Path (60 min) 🔴
- **Steps**: Game server → internal API (idempotent by gameId) → DB, ranking. Retry queue on failure
- **Acceptance criteria**: Results are saved from both game servers and reflected in the ranking

#### L6-C-14 Server-Down Scenario (90 min) 🔴
- **Steps**: Kill one game server with `Stop-Process -Force` → confirm exclusion from matching within 15s → assignment goes only to the remaining server → client re-matches successfully → voided game is recorded
- **Acceptance criteria**: Record recovery time (from kill to successful re-match) in seconds, consistent across 5 repetitions

#### L6-C-15 Result-Storage Retry Queue (75 min) 🔴
- **Steps**: Stop the API server for 60s → queue builds up → drains after restart. 0 duplicate saves (idempotent by gameId)
- **Acceptance criteria**: 0 loss, 0 duplicates, log the max queue length

#### L6-C-16 Matching-Server Down/Recovery (60 min) 🔴
- **Steps**: Force-kill the matching server → confirm no impact on in-progress games → restart → waiting users recover via re-request
- **Acceptance criteria**: 0 in-progress games interrupted, recovery time recorded

#### L6-C-17 Match-History System (75 min)
- **Steps**: Game results → aggregate wins/losses/draws/win-streaks per user, expose via profile API. Choose real-time vs. batch update
- **Acceptance criteria**: After 10 games, match history matches exactly

#### L6-C-18 Seasonal Ranking (75 min)
- **Steps**: Separate season keys (`rank:season:{n}`), save a snapshot to DB at season end, API for querying previous seasons
- **Acceptance criteria**: Previous ranking is queryable after season transition, current season resets

#### L6-C-19 1,000-Connection Load Measurement (120 min) 🔴
- **Steps**: Ramp-up 20 users/sec, target 1,000 connections·200 rooms, 10 minutes. Split clients across 2 processes
- **Acceptance criteria**: Record connect-p99, match-time p99, login p99, error rate, room distribution per server, resource usage

#### L6-C-20 Load Tuning (120 min) 🔴
- **Steps**: Verify hypotheses → fix → re-measure 3 times → before/after table
- **Acceptance criteria**: Target passed, or a lowered target + root-cause analysis document

#### L6-C-21 Runbook Update and Real-Use Test (90 min) 🔴
- **Steps**: Write entries for 5 failure types → **actually inject each failure and respond per the runbook** → fill in missing commands
- **Acceptance criteria**: Using only the runbook, narrow down the root cause within 5 minutes

#### L6-C-22 README Reproduction Test (60 min) 🔴
- **Steps**: Give an AI "developer seeing this for the first time" only the README and ask it to describe the setup steps → fill guessed items into the docs → clone into another folder and **actually get it running within 10 minutes**
- **Acceptance criteria**: 0-1 guessed items, actual setup within 10 minutes

#### L6-C-23 Demo Video (90 min)
- **Steps**: Write a script → film (1 min setup, 2 min play, 1 min server-down, 1 min dashboard) → edit
- **Acceptance criteria**: Under 5 minutes, understandable flow with subtitles/captions even with sound off

#### L6-C-24 Repository Standardization (90 min)
- **Steps**: Bring Phase 2-5 repo READMEs to the same structure (summary/structure/setup/testing/lessons learned/metrics), add licenses
- **Acceptance criteria**: The front page of each of the 5 repos shows "what was built and what numbers it achieved"

#### L6-C-25 Portfolio Index (75 min)
- **Steps**: Cards for the 5 repos + capstone architecture diagram + metrics summary (connection count, p99, improvement rate, test count)
- **Acceptance criteria**: Can navigate from the index to each repo, numbers link to their supporting documents

#### L6-C-26 Presentation Materials (120 min)
- **Steps**: About 15 slides. Problem → design → 3 key decisions → metrics → incident response → AI usage → retrospective
- **Acceptance criteria**: 0 slides without numbers, the whole thing explainable with a single diagram

#### L6-C-27 Mock Interviews (60 min × 3) 🔴
- **Steps**: Use the §6.1 prompt for a 60-minute structure (20 portfolio + 25 CS + 15 language). Record and improve each round
- **Acceptance criteria**: Last round averages 4.0+, scores rise across rounds

#### L6-C-28 Writing the Resume (90 min) 🔴
- **Steps**: 3+ projects described as "Problem → Choice → Result (with numbers)." Link supporting evidence for each claim
- **Acceptance criteria**: 0 unverifiable claims

#### L6-C-29 AI Resume Review (45 min)
- **Steps**: Request review from both a recruiter's and a technical interviewer's perspective → remove exaggeration and vague language
- **Acceptance criteria**: Before/after comparison, record the rate at which feedback was incorporated

---

## 4. Learning Items in Detail

### 4.1 Common (130h)

**Multi-Server Configuration (16h)**
- What: separating server roles, managing the server list (static vs. Redis registration), sharing state via heartbeat, choosing inter-server communication (Pub/Sub, gRPC, custom TCP), internal API authentication
- Why: the moment you go from "one server" to "several," state, routing, and failure handling all change
- How: `L6-C-03` (comparison, ADR) → `L6-C-04/10` (heartbeat) → `L6-C-06` (local multi-process execution)
- Check: server list refreshes in Redis within 15 seconds, dead servers auto-excluded

**Matching Server (16h)**
- What: why to separate it, queue data structure, matching cycle, server-selection criteria, assignment notification, cancel/timeout, matching server's own failures
- Why: matching is the first component that "coordinates multiple game servers"
- How: `L6-C-07/08` → `L6-C-09` (ticket) → `L6-C-12` (distribution verification)
- Check: rooms evenly distributed across 2 servers, forged/reused tickets rejected

**Session Routing and State Storage Location (12h)**
- What: user location (Redis), storage location and rationale for tokens/tickets/room state individually, what's OK to lose vs. not on restart, stateful vs. stateless servers
- Why: this is where the essence of scalability and failure recovery lies
- How: `L6-C-05` (state placement table) → documentation
- Check: state placement table (data × location × loss tolerance × rationale)

**Server-Down Handling (14h)**
- What: detecting heartbeat loss, handling users on a dead server (the server can't notify them → client must detect it), voiding vs. saving in-progress games, re-matching, queue for failed result saves
- Why: the real difficulty of multi-server systems lies in failure handling
- How: `L6-C-14` (game server) → `L6-C-16` (matching server) → `L6-C-15` (result storage)
- Check: force-kill scenario for the game server passes consistently 5 times, recovery time recorded

**Documentation (20h)**
- What: README (summary, architecture diagram, 10-minute setup, testing), ARCHITECTURE (components, sequence, state, execution layout diagram, state placement table), 6 ADRs, final protocol/API docs, performance report, runbook, AI-usage record
- Why: in real jobs, docs get read before code, and in interviews, doc quality is trust itself
- How: 1 hour every day starting week 24 + `L6-C-22` reproduction test
- Check: an AI "developer seeing this for the first time" can reproduce the setup steps from the README alone, actually running within 10 minutes

**Presentation and Interviews (32h)**
- What: structuring a 30-minute presentation, explaining with a single diagram, common CS topics (networking, OS, DB, concurrency, language), portfolio deep-dive prep, coding-test basics, resume writing
- How: `L6-C-26` (materials) → 2 rehearsals → `L6-C-27` (3 mock interviews) → `L6-C-28/29` (resume)
- Check: 3 mock-interview records with rising scores, final presentation averages 4.0+

**Portfolio Wrap-Up (20h)**
- What: standardize README across 5 repos, clean up tests, index page, 3 technical blog posts
- How: `L6-C-24/25` + 3 blog posts (each 1,500+ characters, including code and numbers)
- Check: can navigate to each repo from the index, blog posts include numbers and rejected-hypothesis cases

### 4.2 C# Track (110h)

| Item | Time | What | How to verify |
|---|---|---|---|
| Inter-server communication | 16h | gRPC (inter-server RPC) or Redis Pub/Sub (StackExchange.Redis), internal token auth, applying timeouts/retries | Communication-approach ADR + behavior under failure |
| Multi-host configuration | 10h | Matching, game, API, and batch each as a separate Generic Host project, referencing the shared-library project, injecting port/ID via arguments/env vars | `start-all.ps1` starts everything in one run |
| Matching server | 20h | Queue (`PriorityQueue`/list), matching loop, server selection, ticket issuance (Redis `SET EX` + `GETDEL`), cancel/timeout | 10 tests + distribution verification |
| Game server extension | 24h | Heartbeat reporting, ticket verification, result-storage retry queue (`Channel`), content expansion (match history, seasonal ranking) | Server-down and result-storage scenarios |
| Integration/optimization | 20h | 1,000 connections·200 rooms condition, fixing bottlenecks with Phase 5 tools | `PERF-REPORT.md` |
| Interview prep (language) | 20h | Self-written answers to 30 common questions: GC, async/await, memory, collections, LINQ cost, `struct` vs `class`, `Span`, thread-safe collections, etc. | Document with 30 Q&A + mock-interview scores |

### 4.3 C++ Track (110h)

| Item | Time | What | How to verify |
|---|---|---|---|
| Inter-server communication | 18h | Custom TCP protocol (reuse from 2-2) or gRPC (vcpkg), redis-plus-plus Pub/Sub, internal auth | Communication-approach ADR + behavior under failure |
| Multi-executable configuration | 10h | CMake multi-target setup, shared static library, embedded version info, argument parsing | `start-all.ps1` startup |
| Matching server | 20h | Queue, matching-loop thread, server selection, ticket issuance (Redis), cancel/timeout | 10 tests + distribution verification |
| Game server extension | 24h | Heartbeat, ticket verification, result-storage retry queue, content expansion | Server-down scenario, ASan |
| Integration/optimization | 18h | 1,000 connections·200 rooms, fixing bottlenecks with ETW/profiler | `PERF-REPORT.md` |
| Interview prep (language) | 20h | Self-written answers to 30 common questions: memory model, ownership, move semantics, templates, virtual-function cost, UB, atomics, IOCP, etc. | Document with 30 Q&A + mock-interview scores |

---

## 5. Textbook Guide (Phase 6)

### 5.1 Common

| Textbook | Chapters to read | When | How to use |
|---|---|---|---|
| Building FPS Game Matching Systems | Ch.4 (High-performance matching architecture: matching server, Redis caching, sharding, horizontal scaling), Ch.5 (C# implementation details), Ch.6 (Distributed matching queue), Ch.7 (Monitoring/tuning), 🟡 Ch.2 (Skill rating) | Days 103-110 | **The direct textbook for matching-server design.** Since the book targets FPS-scale matching, **record the judgment call to scale down to your own scope (2-player Gomoku, 2 servers) as an ADR**. Chapter 4's architecture is language-agnostic, so the C++ track should read it too |
| Developing a 2D MMORPG Game Server | Week 1 (setup, basic communication), Week 2 (API server, accounts), Weeks 3-4 (world, movement, sync), Week 8 (optimization, wrap-up) | Days 103, 114 | **Reference architecture** for a 3-part solution of game server + API server + client. Build `code/01~07` (net10.0) to read the inter-server integration code and compare against your own setup |
| A Network Learning Roadmap for Online Game Developers | Ch.8 (Advanced: load balancing, server architecture, cloud, multi-region) | Day 103 | Anything beyond this course's scope (2 servers, single machine) goes into your **"post-hire learning list."** Material for the interview answer to "how would you scale this?" |
| Learn Redis Programming in a Week | Ch.6 (Pub/Sub) | Day 103 | The direct textbook if you use Pub/Sub for inter-server communication. **State the message-loss characteristic (missed if not subscribed) explicitly in the ADR** |
| Network Theory Every Game Server Developer Should Know / Understanding Networks From the Ground Up | Full re-read (skim) | Days 124-128 | Interview review. Have AI generate 30 interview questions covering both books |
| Network Knowledge for Online Game Client Developers | Ch.6 (network characteristics by genre) | Day 126 | Prep for the interview question "what if it were a different genre?" |
| 🟡 Building an Online Game Client with MonoGame | Ch.3-4 (binary protocol, connection management), Ch.7-8 (position sync) | Day 114 | Only if you need a GUI client for the demo. Keep the client minimal in a server-developer portfolio |
| 🟡 A Practical Guide to Implementing Gacha Probability | Ch.4-6, Ch.14 | Day 114 | When adding a shop/gacha as content expansion |
| 🟡 Behavior Tree AI for Online Game Servers | Part 1 (Ch.1-4) | Day 114 | Only when expanding content into "action with NPCs." **Carries high risk of scope overrun, so not recommended by default** |
| 🟡 Building Online Games with C# and P2P Communication | Ch.1-2 (NAT, hole punching basics) | Day 126 | Concepts only, for the interview question "P2P vs. server" |

### 5.2 C# Track

| Textbook | Chapters to read | When | How to use |
|---|---|---|---|
| Developing a 2D MMORPG Game Server | Full code | Days 103-114 | Reference architecture (see above) |
| Building a C# async/await Library | Ch.13 (WhenAll/WhenAny variants), Ch.15 (async cache) | Day 108 | For when the matching server queries multiple game-server statuses in parallel |
| 🟡 ECS-Based Online Game Servers | Ch.10 (scalable structure), Ch.11-12 (mini project) | Day 114 | Compare against a different scaling approach for room actors, interview material |
| C# Design Patterns for Game Developers | Ch.17 (pattern-selection guide) | Day 126 | Prep for the interview question "which pattern did you use and why" |

### 5.3 C++ Track

| Textbook | Chapters to read | When | How to use |
|---|---|---|---|
| 🟡 Building Online Game Servers with C++ Boost.Asio | Ch.17 (scalable server design), Ch.20 (MMO server), Ch.21 (mobile server) | Days 103, 114 | Compare multi-server configuration cases. Keep the direct-IOCP-implementation path but borrow only the design ideas |
| Modern Windows Multithreading | Ch.11 re-read | Day 117 | Architecture check before final optimization |
| Safe and Elegant Programming with Modern C++ | Ch.17 (cross-platform library design) | Day 104 | Reference when splitting out shared libraries |
| 🟡 Modern C++ Programming as Safe as Rust | Ch.1, Ch.16 | Day 126 | Concepts only, for the interview question "C++ safety." No hands-on exercise |

---

## 6. AI Collaboration Guide (Phase 6)

### 6.1 Prompts

**Deciding scope (Tuesday of week 21)**
```
Here are the capstone requirements and the required/optional/excluded table I've drawn up. (attached)
Conditions: 4 weeks of implementation + 2 weeks of wrap-up, 8 hours/day, solo development, existing assets are the 3-1 game server, 4-1 API server, and 5-1 load-testing tool.
Review this from a PM's perspective.
1) 3 items with high risk of schedule overrun and why
2) Items that can be excluded without significantly hurting portfolio value
3) Conversely, items that must not be dropped
I make the final decision. Once decided, review my ADR draft.
```

**Weekly senior review (Friday)**
```
Here's this week's commit diff. (attached, or repo path)
Review it from the perspective of a senior game-server developer, and from the whole-capstone perspective
tell me the top-3 "problems that get expensive next week if not fixed now," in priority order.
Also give an estimated fix cost (in hours) for each.
```

**README reproduction test**
```
You are a developer seeing this project for the first time. Reading only the README (attached), describe the local setup steps step by step.
Do not actually run any commands — judge only from the documentation. Point out:
1) Things you have to guess because they're not in the docs  2) Steps that seem out of order or missing  3) Points that seem unlikely to finish within 10 minutes
```

**Mock interview**
```
Act as a technical interviewer hiring an entry-level game-server developer. 60-minute structure:
20 min portfolio deep-dive (based on the attached README/ARCHITECTURE), 25 min general CS (networking, OS, DB, concurrency), 15 min language (C# or C++).
Ask one question at a time, and follow up on my answers with up to 3 follow-up questions.
If I say I don't know, move on without giving hints.
At the end, produce a 5-point scorecard by category and identify the 2 weakest topics.
```

**Prep for architecture-scalability questions**
```
Given my capstone configuration (attached), if I scale the game servers from 2 to 10 and connections from 1,000 to 100,000,
tell me, in order, the top 3 things that would bottleneck first and what would need to change.
I'll answer each one myself first: "(my answer)." Only point out the gaps in my answer.
```

**Resume review**
```
Here is the project description from my resume. (attached)
From both a recruiter's and a technical interviewer's perspective, point out
whether the "Problem → Choice → Result (with numbers)" structure is followed, whether claims are verifiable, and whether there's any exaggeration.
Don't be generously positive without basis, and explicitly call out any sentences you'd recommend deleting.
```

### 6.2 What to Delegate / What Not to Delegate

| Delegate | Don't delegate |
|---|---|
| Reviewing scope/schedule risk | **Deciding scope** (final judgment is mine) |
| Weekly code review, boilerplate | Designing matching/ticket/server-down policy |
| Generating and scoring interview questions | Interview answers (in my own words) |
| Feedback on presentation-material structure | The body of the presentation/documents (write and only review with AI) |
| Playing the "README reproduction test" role | Judging what to fix in the docs |

### 6.3 `AI-USAGE.md` (Required Interview Material)

Keep this in the capstone repository and record the following. "How did you use AI" will absolutely come up in interviews.

```markdown
# AI Usage Record
## 1. Where AI Was Used (by file/feature)
| Area | AI contribution | How it was verified |
|---|---|---|
| Matching-server queue boilerplate | Initial draft | Wrote and passed 10 tests myself, designed the concurrency scenarios myself |
| Grafana PromQL | Query drafts | Verified values against real load |
| ...
## 2. Core Parts Built Without AI
Room actor, ticket verification (atomicity), server-down policy, performance-improvement decisions
## 3. Cases Where AI Was Wrong (5+)
| # | AI's claim | Reality | How it was discovered |
|---|---|---|---|
| 1 | Recommended an unbounded `Channel` | No backpressure, memory exploded | Observed memory growth during load testing |
| ...
## 4. Rules I Followed for AI Use
No committing code I can't explain / 1 hour without AI every day / generated code always comes with tests
```

---

## 7. Assignment Specifications

### 7.1 Capstone 6-1. Real-Time Multiplayer Game Server Stack (Required, 160h, Days 101-120)

**Configuration requirements**

1. **Server configuration**: 1 API server + 1 matching server + 2+ game servers (same binary, different port/ID), all as separate local processes. The batch worker is optional (🟡) and runs only one season-end snapshot job
2. **Flow**: client → API login (token) → connect to matching server (token verification) → matching request → select game server, **issue ticket**, notify address → connect to game server (ticket verification) → play → result → game server saves to API (retry queue) → update ranking → return to lobby, re-match
3. **Server-state sharing**: game server sends a Redis heartbeat every 5 seconds (connection count, room count, version, TTL 15s). The matching server selects the **server with fewest rooms** among those with a valid heartbeat, and rechecks right before assignment
4. **Server-down handling**: when one game server is force-killed, the matching server **excludes it within 15 seconds**; that server's users detect the client disconnection, return to the lobby → **re-match successfully**; the in-progress game is recorded as "voided"
5. **Game content (pick one)**: (a) Gomoku + seasonal ranking + match history (spectating and room chat 🟡) (b) 2D action for 4 players (c) a new simple game (pre-approved)
6. **Observability**: add **per-server and matching-server panels** to the Phase 5 dashboard, 3+ alerts (connect p99 / match time / heartbeat loss)
7. **Execution automation**: `start-all.ps1` (starts all 4), `stop-all.ps1`, `smoke-test.ps1` (full health check after startup)

**Quality requirements**

| Item | Criterion |
|---|---|
| Load | default **1,000 connections/200 rooms**; official reduced path **500/100**; ramp-up 20/sec, 10 minutes |
| Latency | record connect/match/login p99; connect p99 ≤ Phase 5×1.2 |
| Server-down | five force-kill runs; process stop→successful rematch ≤30 seconds |
| Result reliability | 0 loss, 0 duplicates for results even with a 60-second API outage |
| Testing | Unit/integration tests pass (rules, logic, API, matching queue), all runnable via local scripts |
| Runbook | 5 failure entries, verified through real use |

**Documentation requirements**

- `README.md`: one-screen summary, architecture diagram, **local setup steps within 10 minutes**, how to test/use scripts
- `ARCHITECTURE.md`: components, sequence (login → play → result), state, **execution layout diagram (processes, ports)**, state placement table
- **6 ADRs**: scope, inter-server communication, ticket design, matching approach, state storage location, server-down policy
- Final `PROTOCOL.md`/`API.md`, `PERF-REPORT.md`, `RUNBOOK.md`, `AI-USAGE.md`
- **5-minute demo video**: 1 min setup explanation / 2 min play / 1 min server-down/recovery / 1 min dashboard

**Deliverables**: repository, local setup instructions, demo video

**Grading**

| Item | Points | Criterion |
|---|---|---|
| Configuration/flow | 20 | Requirements 1-3, all 4 types started via the execution script, smoke test passes |
| Failure handling | 15 | Server-down passes consistently 5 times, 0 result loss/duplicates |
| Quality | 25 | 1,000 connections·200 rooms·10 minutes, 3 p99 metrics recorded, tests pass |
| Documentation | 20 | README reproduction test passes, 6 ADRs, `AI-USAGE.md` |
| Content | 10 | Completeness of the chosen content option |
| Observability/ops | 10 | Per-server dashboard, 3 alerts, runbook verified through real use |

**Milestones by phase (for self-checking)**

| Point | What should be working |
|---|---|
| End of week 21 | API, matcher, and two game servers start via script; smoke test passes |
| End of week 22 | Login → matching → ticket → play on one of 2 game servers |
| End of week 23 | Server-down and matching-server-down scenarios pass, content complete |
| End of week 24 | 1,000-connection test passes, all 6 documents complete |

### 7.2 Common Assignment 6-2. Portfolio Wrap-Up (Required, 24h, Days 121-123)

1. **Standardize 5 repositories**: Phase 2 (chat + packet library), 3 (Gomoku server), 4 (API server), 5 (load-testing tool, performance report), 6 (capstone)
   Each README should include **summary / structure / setup / testing / lessons learned / metrics**, a license, and test-execution logs
2. **Portfolio index**: cards for the 5 repos + capstone architecture diagram + **metrics summary** (max connections, p99, improvement rate, test count)
3. **3+ technical blog posts** (each 1,500+ characters, including code and metrics)
   - Example 1: "Chasing Down a Session-Close Race in an IOCP (or SAEA) Server"
   - Example 2: "Comparing Three Fixes for a Shop-Purchase Concurrency Bug"
   - Example 3: "How I Cut p99 by 30% and the **Hypotheses I Rejected**"

**Grading**: README quality 40 / index 20 / blog posts 40 (whether they include numbers, code, and rejected-hypothesis cases)

### 7.3 Common Assignment 6-3. Presentation + Mock Interviews (Required, 32h, Days 123-129)

1. **About 15 presentation slides**: problem → design → 3 key decisions (ADRs) → metrics → incident response → AI usage → retrospective
2. **60-minute mock interviews × 3+**: record each round (questions, answer summaries, score, areas to improve). **Last round averages 4.0+**
3. **30-minute final presentation**: AI interviewer (including a human if possible), average 4.0+. Explain "what AI made vs. what I made" while distinguishing them, and **fully explain the AI-made parts too**
4. 🟡 Present to and get feedback from 1 working developer (community, acquaintance). If not possible, use 2 AI-interviewer personas (strict, practitioner)

**Grading**: presentation (average score) 50 / interview records and improvement trend 50

### 7.4 Common Assignment 6-4. Resume and Cover Letter (Required, 16h, Days 126-127)

- Describe projects using the **"Problem → Choice → Result (with numbers)"** structure, only verifiable claims
- Link supporting documents (reports, repos) for every number
- History of incorporating AI review (recruiter and technical-interviewer perspectives)
- Deliverables: resume PDF, cover letter

### 7.5 Common Assignment 6-5. Coding-Test Basics Review (Required, 16h within track time, Days 126-129)

- Arrays, strings, hashing, sorting, stack/queue, BFS/DFS, binary search, priority queue, simple DP
- **2 problems/day**, in your track language, solved without AI, then AI reviews complexity and alternatives afterward
- Deliverables: a problem list and a solutions repository (8+ problems total)

---

## 8. Learning Completion Assessment (Day 130, Final Course Assessment)

### 8.1 Checklist

**Capstone**
- [ ] `start-all.ps1` starts all 4 types → smoke test passes → local demo succeeds
- [ ] Log of 1,000 connections·200 rooms·10 minutes (or a lowered target + root-cause document)
- [ ] Force-kill scenario for the game server passes 5 times, recovery-time table
- [ ] 0 loss, 0 duplicates in result storage (60-second API-outage scenario)
- [ ] README reproduction test passes + actually cloned and running within 10 minutes
- [ ] 6 ADRs, `AI-USAGE.md` (5+ recorded AI errors)
- [ ] 5-minute demo video

**Portfolio/job-hunting**
- [ ] README standardized across 5 repos + test-execution logs
- [ ] Portfolio index, 3 technical blog posts
- [ ] Presentation materials, 3 mock-interview records (last round averages 4.0+)
- [ ] Resume and cover letter
- [ ] 8+ coding-test problems

**Cumulative**
- [ ] 100+ days of study notes, 24+ weekly retrospectives

### 8.2 Reimplementation Exam (No AI, 120 min)

**Problem**: implement the core of the matching server and pass the 4 provided tests (network can be faked, Redis can be replaced with an in-memory substitute).

Required features
```
Matcher.Request(userId)            // Register to the queue
Matcher.Cancel(userId)
Matcher.Tick(nowMs)                // Matching cycle: on a 2-player match, select server + issue ticket + notify assignment
ServerRegistry.Heartbeat(serverId, rooms, version, nowMs)
TicketStore.Issue(userId, roomId, serverId) / Consume(ticket)   // Single-use
```
Provided tests
1. **2-player match**: when two users request, they're assigned to the same room and each receives a different ticket
2. **Server exclusion**: a server whose heartbeat has been silent for 15+ seconds is excluded from candidates
3. **Cancel**: a user who canceled is not matched (still waiting even if another user arrives right after canceling)
4. **Single-use ticket**: consuming the same ticket twice fails the second time

**Passing criterion**: pass all 4 within 120 minutes.

### 8.3 Final Presentation and Interview

- **30-minute presentation**: average 4.0+ on a 5-point scale. Explain the whole configuration with a single diagram, and defend the 3 key decisions with rationale
- **60-minute mock interview**: last round averages 4.0+

### 8.4 Final Oral Exam Question Bank

1. Draw the whole configuration as a single diagram and explain the data flow from login to result storage
2. What would need to change if you scaled to 10 game servers? What if the matching server became the bottleneck?
3. Why keep the ticket in Redis? What problems arise if you use only the token, with no ticket?
4. How did you prevent ticket reuse (how is atomicity guaranteed)?
5. How did you handle in-progress games when a game server died, and what were the alternatives?
6. How did the constraint "the server can't notify users of a dead server" affect the design?
7. What happens if the result-storage retry queue disappears on process termination? If you accepted that, why?
8. How did you decide on a 5-second heartbeat interval and 15-second TTL?
9. What stops and what continues when the matching server dies?
10. In the state placement table, what's the criterion for "OK to lose" vs. "must not lose"?
11. What was the bottleneck at 1,000 connections and how did you fix it? What hypotheses did you reject?
12. Where did the p99 numbers come from and how do you reproduce them?
13. In what order do you diagnose an incident on the dashboard?
14. Which AI mistake was the most dangerous, and how did you catch it?
15. What's the design decision you're least happy with in this project, and what would you do differently?
16. (C#) Where GC was a problem in the capstone and how you fixed it / (C++) Where ownership got complicated and how you cleaned it up

### 8.5 Handling Shortfalls

| Shortfall | Response |
|---|---|
| 1,000-connection target not met | Pass at a lowered target of 500 connections·100 rooms, but a **root-cause analysis + scaling plan document** is required |
| Server-down scenario fails | **Cannot pass**. 1-week extension (this is the core competency of multi-server systems) |
| Documentation incomplete | **Cannot pass** (documentation is a direct job-hunting deliverable) |
| Mock-interview average below 4.0 | Re-study 2 weak topics, then do a 4th round |
| Fewer than 3 blog posts | Lower to a minimum of 2, but each must include numbers and a rejected-hypothesis case |

---

## 9. Common Sticking Points

| Symptom | Cause | Countermeasure |
|---|---|---|
| Scope keeps growing | Adding features without an ADR | Scope changes only via ADR; decide "exclude" in the weekly review |
| Matching server assigns to a dead game server | Heartbeat TTL/check-interval mismatch, TOCTOU | TTL > interval × 3, **recheck right before assignment** |
| Tickets get reused | Verify-then-delete isn't atomic | Use Redis `GETDEL` or Lua to verify + delete in one step |
| Rooms pile up on one server | Delayed room-count updates, concurrent assignment | Shorten the heartbeat interval, or increment the counter immediately on assignment |
| Result storage gets duplicated | Retry queue without idempotency | Use gameId as the idempotency key, dedupe on the API side |
| Users stay stuck after a server goes down | No client-side disconnect detection | Client timeout + return-to-lobby logic (implement it in the test client too) |
| Local 4-process run hits port conflicts | Hardcoded ports | Inject everything via arguments/env vars from the script |
| Processes linger after shutdown | Child processes not tracked | Clean up in `stop-all.ps1` using a PID file or process-name+port lookup |
| traceId doesn't connect across servers | Propagation missing | Add a traceId field to packets/HTTP headers, standardize log fields |
| No time left to write docs | Crammed at the end | 1 hour every day starting week 24, write ADRs the moment decisions are made |
| Can't explain your own code in a mock interview | AI-generated code never reviewed | Reimplement that part without AI and record it in `AI-USAGE.md` |
| Demo video runs over 10 minutes | Filmed without a script | Write a 4-segment script before filming, only reshoot the failed segment |
| Client bottlenecks during load testing | Single process | Split user-ID ranges across 2 processes |

---

## 10. After Finishing the Course

### 10.1 Post-Hire Learning List (Deliberately Excluded from This Course)

| Topic | Why excluded | When to learn it |
|---|---|---|
| Deployment/CI-CD | To keep learning time focused on server implementation and operational quality | Follow your team's pipeline after joining |
| Docker/Kubernetes | Principle of a local Windows environment | As soon as you join a team that uses containers |
| Load balancers, cloud, multi-region | Infrastructure beyond a 2-server scale | Roadmap textbook Ch.8 → on the job |
| UDP-based sync and lag compensation | Unnecessary for Gomoku/turn-based games | When moving into FPS/action genres |
| Packet-capture-based analysis | Replaced by log/state observation | When deep network debugging is needed |
| A second language (secondary track) | Mastering the primary track comes first | After the capstone |

### 10.2 What to Keep Doing

- **Keep improving the capstone.** Register questions received from hiring assignments/interviews as issues and resolve them one by one
- **Keep writing the blog.** One post per improvement is itself a portfolio
- **Keep interview records.** Compile the questions received, your answers, and what fell short, and apply it to your next application
- **Leave anything wrong or lacking in this curriculum** as an issue in the course repository (for future learners)

### 10.3 Final Self-Check (Last Day of Week 26)

- What are the 3 skills that grew the most compared to before week 26?
- If you did it again, which Phase would you spend more time on, and where would you cut back?
- How has your way of using AI changed (early on vs. now)?
- What could you contribute in your first week if you started a job right now?

---

## 11. 2026-09-05 Revisions (this section wins on conflicts)

### 11.1 Scope and architecture contract

- Required processes are one API, one matcher, and two game servers. The **batch worker is optional (🟡)** and runs only one season-end snapshot job. Build the four required skeletons/config/PID files in `L6-C-05` on Day 104, then launch them on Day 105
- Required content is match history plus seasonal ranking; spectating and room chat are optional. The official reduced path is 500 connections/100 rooms on recommended 8-core/16GB hardware, with environment recorded
- Day 103 reserves half a day to define matcher↔client, game→API result, assignment notification, and internal API authentication in `PROTOCOL.md`/`API.md`. Prefix Redis keys with `{course}:{env}:{service}:`
- Specify matcher-client transport and embedded protocol version. In-progress reconnect is an optional design including ticket TTL, generation, and join timeout
- Add Redis Streams to the inter-server comparison and state Pub/Sub's non-durability. Record why no gateway/relay is used. A mixed path may use C++ game servers with C# API, matcher, and batch

### 11.2 Schedule and automated validation

#### L6-C-10b Extend the Load Tool Through Matching (120 min) 🔴
- **Steps**: add an async state machine for API login → matcher request → ticket → game connect → play → rematch
- **Acceptance criteria**: fixed-seed 500-connection JSON with per-stage success/failure, p99, and close reasons

- Replace Day 108 morning with `L6-C-10b Extend 5-1 tool for matching flow` (120 min), shifting following labs one day. Spread five faults across Days 117-118 and write only one half-day blog post on Day 123
- `L6-C-14` automates process stop→detection→successful rematch and prints elapsed time. Require recovery ≤30s, move p99 ≤ Phase 5×1.2, and ten automated quality-gate tests
- Build once before launching; `start-all.ps1` runs `dotnet <dll>`/executables directly and records PIDs. `stop-all.ps1` verifies PID death and port release; do not nest background `dotnet run`
- Rebudget Phase 6 as capstone 152h, portfolio 24h, presentation 24h, applications 16h, coding/interview 24h. Keep the no-AI hour on Days 121-124 and 126-128
- Project-start DoD includes committed `CLAUDE.md` T3/T3'. Write ADR-0013 for recovery and ADR-0014 for result-queue durability on their decision days

### 11.3 Hiring preparation and repository hygiene

- Add §8.4-b with 30 questions: networking10, OS8, DB7, language5, plus five behavioral questions. Use the 60-minute T9-b rubric; withhold answers during scoring and output unknown-question list
- From Week 21, use two no-AI hours per week for algorithms, reaching 32 problems including the original eight. Day 125 spends 60 minutes analyzing three target companies, stacks, motivation, and collaboration/failure stories
- 6-2 checks `.gitignore`, secret history, personal data, commit messages, license, and README AI disclosure. Slides include ten expected Q&As; the demo has one caption in each of four segments

### 11.4 AI and operations traps

- Challenge AI suggestions for `GET`+`DEL` tickets, Redis `KEYS`, Pub/Sub as reliable delivery, heartbeat TTL equal to interval, or background `Start-Process dotnet run`. Use Lua when Redis <6.2 lacks `GETDEL`
- Add troubleshooting for build locks, lingering PIDs/ports, Redis connection explosion, duplicate queueing, unclaimed ticket slots, missing server_id labels, stale tickets after matcher restart, and season-snapshot ordering
- Prerequisites are an extensible 5-1 client, internal API auth, traceId logs, Release+PDB, Redis 6.2+ or Lua, and recommended 8 cores/16GB
