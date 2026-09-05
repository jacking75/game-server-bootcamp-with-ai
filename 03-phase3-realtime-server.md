# Phase 3. Real-Time Game Server Architecture (Weeks 8-11, 160h)

> 🇰🇷 Korean version: [03-phase3-realtime-server_kr.md](03-phase3-realtime-server_kr.md)

> Common 90h / Track 70h. By the end of this Phase, learners will **design a real-time server with game logic starting from the thread model** and complete it with the design document and code in full agreement.

## 0. How to Use This Document

Same structure as Phase 1/2. Keep the day block in §2 open each day and work through it; find labs (`L3-C-xx`, `L3-CS-xx`, `L3-CPP-xx`) in §3, and assignment details in §7.

The three core principles of this Phase

1. **Write the design first.** `DESIGN.md` comes before code. Write the design document in week 8, and update the "deviations from the design" in week 11. This is the thing new hires are worst at in the field.
2. **No locks in room logic.** All room access passes through the Job queue. If even one lock remains, this Phase cannot be passed.
3. **The server makes the judgment.** The client only sends "requests." Win/loss, coordinates, and turns are all validated by the server. All 5 cheat scenarios must be rejected.

| Notation | Meaning |
|---|---|
| `L3-C-xx` / `L3-CS-xx` / `L3-CPP-xx` | Common / C# / C++ lab |
| 🔴 Required · 🟡 If time allows | Priority |
| **DoD** | Definition of Done for the day |

---

## 1. Overview

### 1.1 Goals

1. **Choose a thread model and document the rationale.** Compare 4 models and write an ADR explaining why the room actor model was chosen for the Omok server.
2. **Implement the room actor.** Each room has its own Job queue, and only one thread executes a room at a time. Timers, packets, and inter-room messages all enter as Jobs.
3. **Design and implement state machines.** States and transitions for session, room, and matching respectively, and illegal transition handling.
4. **Implement server authority.** Move validity, turn verification, rate limiting, reproducible randomness.
5. **Separate layers.** Three layers: Rules (pure functions) / Logic (room, matching) / Network (session). The 30 rule tests run in under 1 second with no network dependency.
6. Prove by measurement that **move latency p99 is 50ms at 100 rooms / 300 connections**.

### 1.2 Prerequisite Self-Check (20 minutes)

| # | Question | Pass Criteria |
|---|---|---|
| 1 | Can the 2-2 packet library be referenced as an independent project? | Builds in a different solution |
| 2 | Did the 500-connection chat server run for 10 minutes with zero loss? | Have `LOAD-2-1.md` |
| 3 | Can you explain within 30 seconds how you prevented session-close races? | Immediate answer |
| 4 | Can you add scenarios to the 2-3 test client? | Know the structure |
| 5 | Are you using Phase 1's log queue/object pool attached to the server? | Reference exists in code |

If fewer than 3 pass, address the incomplete Phase 2 items first. In particular, if items 1 and 2 are not met, you cannot satisfy this Phase's load conditions.

### 1.3 What You Can Do After This Phase

- Explain "why is a room processed on a single thread, and what are the alternatives" through a comparison of 4 models
- **Reimplement the room actor in 120 minutes** (Job queue + single consumer loop + timer Jobs + inter-room messages)
- Measure move latency p99 on a server running 100 rooms concurrently and identify the bottleneck
- Write the design document so that someone else can explain the server structure just by reading it
- Demonstrate through tests that the server rejects all 5 cheat scenarios

### 1.4 Deliverables of This Phase

```
phase3/
├─ DESIGN.md                  Assignment 3-C: design document (+ "deviations from design" section)
├─ adr/
│  ├─ 0002-thread-model.md    thread model choice
│  ├─ 0003-room-session-ref.md room-session reference approach
│  └─ 0004-matching.md         matching approach
├─ OmokServer/
│  ├─ Rules/                  Omok rules — zero external dependencies (30 tests)
│  ├─ Logic/                  room actor, user, matching, lobby (25 tests)
│  ├─ Net/                    session, dispatch (references PacketLib)
│  ├─ Host/                   entry point, config, logging, shutdown
│  └─ tests/
├─ LoadClient/                2-3 extension: omok/spectate/reconnect/cheat scenarios
├─ LOAD-3-1.md                100-room load report
└─ labs/
```

### 1.5 4-Week Roadmap

| Week | Main Topic | Common | C# Track | C++ Track | Assignment |
|---|---|---|---|---|---|
| Week 8 | Design and room actor | 4 thread models, room actor, tick/timer, design doc & ADR | `Channel<IJob>` actor, `PeriodicTimer` | Room Job queue + worker assignment, timer thread | 3-C draft, actor prototype |
| Week 9 | State machines and rules | Session/room/matching states, server authority, reproducible randomness | Handler dispatch, rule tests | Function table dispatch, rule tests | 3-1 lobby, matching, entry |
| Week 10 | Gameplay and operations | Timeout, reconnect, spectating, structured logging, shutdown | Serilog, Generic Host | spdlog, shutdown sequence | 3-1 game completion, 3-2 |
| Week 11 | Load and consistency | Integration tests, 100-room load, design update | Bottleneck fixes, library analysis | Bottleneck fixes, 🟡 Asio comparison | 3-1 performance pass, evaluation |

---

## 2. Weekly Detailed Plan (Day by Day)

Day numbers are based on the entire course (Phase 3 spans days 36-55).

### 2.1 Week 8 — Design and Room Actor

#### Day 36 (Mon) — Comparing 4 Thread Models 🔴

**Morning (2.5h)**
- Summarize the 4 models
  1. **Thread per session**: simple, but collapses at 1,000 connections (already measured in Phase 2)
  2. **I/O thread + single logic thread**: no locks, simple to implement, uses only 1 core → limited when there are many rooms
  3. **Worker pool + shared-state locking**: uses all cores, but lock contention, deadlocks, and debugging hell
  4. **Room/entity-level actor**: a queue per room, room logic runs on a single thread → parallel without locks. The choice for this course
- Build your own table rating each model's throughput, latency, complexity, scalability, and debugging difficulty on a 5-point scale

**Afternoon (2.5h)**
- `L3-C-01` Write a 4-model comparison table: which model to choose and why for each of the three cases "Omok 100 rooms," "MMO zone server," and "FPS match server"
- Draft `adr/0002-thread-model.md` (context, decision, 3 alternatives, reasons for rejection, consequences)

**1 Hour Without AI**
- Write down 3 drawbacks of the room actor model yourself (inter-room communication cost, when a single room becomes heavy, tracing flow during debugging)

**DoD**
- [ ] Comparison table of 4 models × 3 cases
- [ ] ADR-0002 draft (including 3 alternatives and reasons for rejection)

#### Day 37 (Tue) — Room Actor Prototype 🔴

**Morning (2.5h)**
- Actor principles: (1) only the room's own thread touches room state (2) outside callers request only via Jobs (3) inter-room communication is also a Job (4) timers are also Jobs
- Job design: `IJob { void Execute(Room room); }` or a tagged union. Minimize allocation (struct Job + pool)
- Queue choice: unbounded vs bounded. Decide on a **bounded queue + overflow policy** (drop the oldest / disconnect the session)

**Afternoon (2.5h)**
- `L3-CS-01` (C#) `Channel<IJob>`-based room actor prototype / `L3-CPP-01` (C++) MPSC queue + running flag (atomic)
- Requirement: create 100 rooms and post 10,000 Jobs to each, then verify that **every Job executes in order, on exactly one thread at a time** (assertions + counters)

**1 Hour Without AI**
- Write in 3 sentences how you guaranteed in code that "two workers never run the same room at the same time," and imagine one scenario where it could break

**DoD**
- [ ] Run 100 rooms × 10,000 Jobs, verify order guarantee and zero concurrent execution
- [ ] Queue bound and overflow policy exist in code

#### Day 38 (Wed) — Tick, Timer, and Game Loop 🔴

**Morning (2.5h)**
- Tick-based vs event-based. Event-based + timeout timers are sufficient for Omok (you should be able to explain why)
- Windows timer precision: default 15.6ms, the effects and side effects of `timeBeginPeriod`
- Timer Job principle: **never touch room state directly from a timer callback.** Post a Job to the room queue instead

**Afternoon (2.5h)**
- `L3-C-02` Measure timer precision: a table of the actual firing-interval distribution (1,000 runs) for 1ms/10ms/100ms period timers
- `L3-C-03` Implement timer → room Job posting: 3 types — turn timeout (30s), heartbeat check (5s), reconnect grace period (60s)
- Timer cancellation: ensure no scheduled timer survives room destruction (cancellation token or generation number)

**1 Hour Without AI**
- Write down, in chronological order, the bugs that could occur if a timer callback directly modified room state

**DoD**
- [ ] Timer precision table (3 intervals × mean/max/standard deviation)
- [ ] All 3 timer Job types work; confirm timer cleanup on room destruction

#### Day 39 (Thu) — State Machine Design 🔴

**Morning (2.5h)**
- Session states: `Connected → Authenticated(nickname) → Lobby → Matching → InGame → Result → Lobby`
- Room states: `Created → Waiting → Playing → Ended → Destroying`
- Matching states: `Requested → Queued → Matched / Cancelled / TimedOut`
- Build a table of each state × event → next state, and decide the **illegal transition handling policy** (ignore / error response / terminate session)

**Afternoon (2.5h)**
- `L3-C-04` Write 3 state transition tables (session, room, matching) → insert into `DESIGN.md`
- `L3-C-05` Design the packet list: extend the 5000-range game packets from 2-C (see the §7.1 table, 30+ types)
- Write 3 sequence diagrams: matching success / move → judgment → notification / disconnect → reconnect

**1 Hour Without AI**
- List 5 illegal transitions such as "what if a move packet arrives while in the lobby state?" and decide the handling policy for each

**DoD**
- [ ] 3 state transition tables, 3 sequence diagrams
- [ ] List of 30 game packet types

#### Day 40 (Fri) — Finalize Design Document + Weekly Checkpoint

**Morning (3h)**
- Complete `DESIGN.md` (all items in §7.1): goals & non-goals, constraints, component diagram, thread model diagram, data ownership table, state transition tables, packet list, 3 sequence diagrams, risks
- Complete 3 ADRs (thread model / room-session reference / matching approach)
- **AI "developer seeing this for the first time" review** (§6.1 prompt): have it explain the structure from the document alone, then treat any misunderstood parts as document defects and fix them

**Afternoon (4h) — Weekly Checkpoint**
1. Reimplement without AI (60 min): minimal room actor version (Job queue + single consumer loop)
2. Explanation exam (45 min): 4-model comparison, actor principles, timer rules
3. Code review (45 min): focused on rubric item 6 (structure)
4. Retrospective (30 min): `W08.md`

**DoD**
- [ ] `DESIGN.md` complete + AI review feedback incorporated
- [ ] 3 ADRs committed

### 2.2 Week 9 — State Machines and Game Rules

#### Day 41 (Mon) — Omok Rules Library 🔴

**Morning (2.5h)**
- Finalize rules: 15×15 board, black moves first, whether exactly 5-in-a-row wins (6-or-more counts as a win, or overlines are invalid) — **pick one and document it**
- Data structure choice: `byte[225]` 1D vs 2D, or a bitboard (optional). Win-check algorithm (4-directional scan based on the last move)
- The Rules project has **zero external dependencies** (no sockets, logging, or time). Pure functions only

**Afternoon (2.5h)**
- `L3-C-06` Implement rules: `PlaceResult Place(Board b, int x, int y, Stone s)`, `WinCheck(Board, lastX, lastY)`, `IsDraw(Board)`
- Write 30 tests (exactly the §7.2 rule test list)

**1 Hour Without AI**
- Reimplement win-checking without AI and pass all 30 tests

**DoD**
- [ ] All 30 rule tests pass, run time under 1 second
- [ ] Rules project does not reference any other project (proven by build)

#### Day 42 (Tue) — Matching System 🔴

**Morning (2.5h)**
- Matching queue design: first-come, first-served pairing of 2 players. Cancellation notice if waiting exceeds 60 seconds
- Matching cadence: immediate event-driven matching vs periodic batching (100ms) — record the choice and rationale in an ADR
- 🟡 Rating-range matching (book "Building an FPS Game Matching System," Ch. 3): expand the allowed range over time

**Afternoon (2.5h)**
- `L3-C-07` Implement the matching queue: request/cancel/timeout, and on a successful pairing of 2, create a room + notify both sides
- 8 matching logic tests: pairing 2 players, no pairing after cancellation, timeout notification, preventing duplicate concurrent requests, rejecting a re-request while already matching, handling cancellation right after a match, rollback on room creation failure, queue ordering (FIFO)

**1 Hour Without AI**
- Draw the timeline for "what if a match success and a cancel request arrive at the same time?" and design a defense

**DoD**
- [ ] All 8 matching tests pass
- [ ] Matching statistics log (number waiting, average wait time)

#### Day 43 (Wed) — Lobby and Room Entry Flow 🔴

**Morning (2.5h)**
- Lobby: nickname login (replaced with authentication in Phase 4), lobby user list, lobby chat (reuse of 2-1)
- Room entry flow: match success → room creation → send `GameStartNotify` to both sides (room ID, opponent info, who moves first) → session state becomes `InGame`

**Afternoon (2.5h)**
- `L3-C-08` Implement handler dispatch: packet ID → handler map, **per-session-state permission check**, packet → Job conversion
- `L3-C-09` Implement lobby and room entry + 5 illegal-transition tests (making a move from the lobby, requesting a match while in-game, etc.)

**1 Hour Without AI**
- Draw the dispatch table by hand and fill in the allowed state for each packet

**DoD**
- [ ] Successfully complete login through matching through room entry with the client
- [ ] All 5 illegal-transition tests pass

#### Day 44 (Thu) — Server Authority and Cheat Defense 🔴

**Morning (2.5h)**
- Server authority principle: the client only makes "requests." The server validates turn, coordinates, rules, and timing entirely
- Validation checklist: is it my turn / coordinate range / is the cell empty / is the game in progress / am I a participant in this room / rate limit (10 per second)
- Reproducible randomness: fix a seed for decisions like who moves first (record the seed in logs → reproducible)

**Afternoon (2.5h)**
- `L3-C-10` Implement move validation + 5 cheat tests: (1) moving out of turn (2) out-of-range coordinates (3) an already-occupied cell (4) packets targeting a different room (5) a burst of 50 requests per second
- Implement the policy of terminating the session after 3 cumulative violations

**1 Hour Without AI**
- In 3 sentences, describe what the server should do "if the client sends a win result" (never trust it)

**DoD**
- [ ] All 5 cheat types rejected, reason and session ID recorded in logs
- [ ] Confirm termination behavior after 3 cumulative violations

#### Day 45 (Fri) — Complete Gameplay + Weekly Checkpoint

**Morning (3h)**
- Move → judgment → notify both sides → turn switch → win/loss/draw determination → result notification → room destruction
- Turn timeout of 30 seconds: automatic loss handled via a timer Job
- Resignation handling, rematching (return to lobby after the result)

**Afternoon (4h) — Weekly Checkpoint**
1. Reimplement without AI (90 min): matching queue (pairing, cancellation, timeout)
2. Explanation exam (45 min): server authority, state transitions, matching races
3. Code review (45 min): focused on rubric items 1 and 6
4. Retrospective (30 min): `W09.md`

**DoD**
- [ ] One full game completes the full cycle: lobby → matching → play → result → lobby
- [ ] Turn timeout and resignation work

### 2.3 Week 10 — Reconnection, Spectating, and Operational Features

#### Day 46 (Mon) — Separating Session and User, Reconnection 🔴

**Morning (2.5h)**
- **Core concept**: separate the session (connection) from the user (game state). Even if the connection drops, the user stays in the room
- Reconnection flow: detect disconnection → mark the user `Disconnected` + a 60-second grace timer → on reconnection with the same nickname, attach the new session to the user → **send a board-state snapshot** → resume the game
- Exceeding the grace period: automatic loss, then close the room

**Afternoon (2.5h)**
- `L3-C-11` Refactor to separate session and user: make the user object's session reference replaceable
- `L3-C-12` Implement reconnection + 6 tests: normal reconnection, loss after exceeding the grace period, opponent moving during reconnection, double reconnection attempts, attempting to reconnect to a different room, reconnecting after the game has ended

**1 Hour Without AI**
- Make a list of what should go into the "board state snapshot" sent on reconnection (board, turn, remaining time, opponent info)

**DoD**
- [ ] All 6 reconnection tests pass, 100% board state recovery
- [ ] Grace timer is cleaned up on room destruction

#### Day 47 (Tue) — Spectating Feature 🔴

**Morning (2.5h)**
- Spectating design: list of in-progress rooms, spectator entry (max 10 per room), receiving move notifications, spectator chat
- Spectators cannot make moves (permission separation); the impact of spectator count on room performance (increased broadcast targets)

**Afternoon (2.5h)**
- `L3-C-13` Implement spectating + 5 tests: enter/leave, rejecting when over the cap, rejecting a spectator's move attempt, cleaning up spectators when the game ends, room destruction while spectating
- `L3-C-14` Broadcast optimization: share a single serialized buffer with both participants and spectators (reference counting)

**1 Hour Without AI**
- Calculate what the room thread does with 100 spectators and write down potential bottlenecks

**DoD**
- [ ] All 5 spectating tests pass
- [ ] Single-serialization shared broadcast implemented (proven by allocation measurement)

#### Day 48 (Wed) — Structured Logging, Configuration, Shutdown 🔴

**Morning (2.5h)**
- Structured log field standard: `timestamp, level, sessionId, userId, roomId, packetId, event, durationMs`
- Log level policy: Debug (development), Info (state transitions, matching, game results), Warn (cheats, timeouts), Error (exceptions)
- C# uses **Serilog** (structured logging + async sink + daily rolling), C++ uses spdlog (async sink)

**Afternoon (2.5h)**
- `L3-CS-02`/`L3-CPP-02` Configure logging: Serilog/spdlog setup, apply the field standard, log file rolling
- `L3-C-15` Configuration file: port, worker count, room cap, turn time, timeout, log level. Separate files per environment
- `L3-C-16` Graceful shutdown: block new connections → post a shutdown Job to every room → notify in-progress games as "voided" → clean up sessions → join threads (within 5 seconds)

**1 Hour Without AI**
- Check whether you can trace "the entire course of one specific game" from logs alone, and add any missing fields

**DoD**
- [ ] The entire course of one game can be traced through logs (filtering by roomId)
- [ ] Shutdown completes within 5 seconds, and the step order is recorded in the logs

#### Day 49 (Thu) — Extending the Scenario Client (Assignment 3-2) 🔴

**Morning (2.5h)**
- Add bot logic to the 2-3 client: a bot that references the rules library and places **only legal moves** (a random empty cell)
- Scenario `omok`: connect → lobby → matching → game (0.5-2s per turn) → result → repeat rematching

**Afternoon (2.5h)**
- `L3-C-17` Add 3 scenarios: `spectate` (count notifications after entering as a spectator), `reconnect` (forced disconnect after N seconds → reconnect after 30s → verify board consistency), `cheat` (attempt 4 cheat types and aggregate the rejection rate)
- Extend statistics: number of rooms, completed games, move→notification p50/p95/p99, reconnection success rate, cheat rejection rate

**1 Hour Without AI**
- Verify how you guaranteed the bot "only places legal moves," and in 3 lines describe the benefits of reusing the rules library

**DoD**
- [ ] All 4 scenarios (omok/spectate/reconnect/cheat) work
- [ ] Statistics output includes p99, reconnection success rate, and cheat rejection rate

#### Day 50 (Fri) — Integration and Weekly Checkpoint

**Morning (3h)**
- Integration run: verify all features simultaneously (matching, play, spectating, reconnection, cheat defense) at a 10-room scale
- Fix discovered bugs, fill out the 25 logic tests

**Afternoon (4h) — Weekly Checkpoint**
1. Reimplement without AI (90 min): session-user separation + reconnection snapshot
2. Explanation exam (45 min): reconnection design, spectator broadcast, shutdown sequence
3. Code review (45 min): focused on rubric items 2 and 3 (room destruction and timer cleanup)
4. Retrospective (30 min): `W10.md`

**DoD**
- [ ] The 10-room integration scenario runs without incident
- [ ] All 25 logic tests pass

### 2.4 Week 11 — Load and Consistency

#### Day 51 (Mon) — First 100-Room Load Test 🔴

**Morning (2.5h)**
- Set the load conditions: 100 rooms (200 playing + 100 spectating = 300 connections), 10 minutes, target of move p99 50ms
- Measurement items: move→notification latency p50/p95/p99, max room queue length, memory (1 min vs 10 min), CPU, number of completed games

**Afternoon (2.5h)**
- Run the first pass and record results. Establish 3 bottleneck hypotheses (also get AI predictions using the §6.1 prompt for comparison)
- Common bottlenecks: too many Tasks/threads per room, repeated broadcast serialization, synchronous log writes, timer thread overload, GC (C#)

**1 Hour Without AI**
- Look at the measurement results, identify the "3 most suspicious spots" yourself, and write down how to verify each

**DoD**
- [ ] Complete one 100-room, 10-minute run, and record the results table
- [ ] 3 bottleneck hypotheses + a verification method for each

#### Day 52 (Tue) — Fixing Bottlenecks and Remeasuring 🔴

**Morning (2.5h)**
- Verify hypotheses: check using counters, logs, and profilers (C#: `dotnet-counters`, C++: VS Profiler)
- Fix only the accepted hypotheses. **Also record the rejected hypotheses in the report** (important)

**Afternoon (2.5h)**
- Remeasure after fixing, and write a before/after comparison table
- Representative optimizations: sharing broadcast buffers, pooling Job objects, making logging asynchronous, changing the room-assignment method (round robin → runnable room queue)

**1 Hour Without AI**
- Based on the before/after numbers, write "what the bottleneck was" in 3 paragraphs

**DoD**
- [ ] At least 1 of the 3 hypotheses accepted, fixed, and remeasured
- [ ] At least 1 rejected hypothesis recorded

#### Day 53 (Wed) — Mass Reconnection and Room Destruction Scenario 🔴

**Morning (2.5h)**
- Scenario: 100 sessions disconnect simultaneously during games → all reconnect after 30 seconds → check the board-state recovery rate
- Scenario: packets arrive during room destruction (move/chat/spectator entry) → zero crashes, logged as ignored

**Afternoon (2.5h)**
- `L3-C-18` Write an automation script for the two scenarios above and repeat 5 times
- C++: pass once with an ASan build. C#: confirm zero unreturned pool buffers

**1 Hour Without AI**
- Trace the code path for "what happens to Jobs left in the queue after a room is destroyed" and write it down

**DoD**
- [ ] 100% reconnection recovery rate, zero crashes from room-destruction races
- [ ] C++ ASan pass log

#### Day 54 (Thu) — Updating Design-Code Consistency 🔴

**Morning (2.5h)**
- Write a **"deviations after implementation"** section in `DESIGN.md`: a table of item / design / actual / reason
- Update diagrams (reflect any changes to the thread model or components), commit the change history

**Afternoon (2.5h)**
- Complete `LOAD-3-1.md` (in the §7.2 format)
- C#: `L3-CS-03` Read the library analyses (FastSocketLite, LiteNetwork, laster40Net) → a thread-model comparison table against your own server
- C++: `L3-CPP-03` Find "where the logic thread is" in the `IocpNetLib` chat server → a comparison table. 🟡 Compare with Asio strands

**1 Hour Without AI**
- Among the reasons design and reality diverged, distinguish and write down "cases where the design was wrong" versus "cases where the implementation compromised"

**DoD**
- [ ] Updated `DESIGN.md` (deviations table with 5+ rows)
- [ ] `LOAD-3-1.md` complete

#### Day 55 (Fri) — Phase 3 Evaluation 🔴

**Morning (3h)**
1. **Reimplementation exam (120 min, no AI)**: §8.2 — room actor (Job queue + single consumer + timer Jobs + inter-room messages) + 4 provided tests
2. **Explanation exam (45 min)**: 10 questions from §8.3
3. **Defect hunting (45 min)**: §8.4 — find 4 of 5 room logic defects

**Afternoon (3h)**
- Review the checklist (§8.1), capture a screenshot of a search confirming zero locks in room logic (`grep -r "lock\|mutex" Logic/Room*`)
- `W11.md` + Phase 3 retrospective
- §10 Preparation for Phase 4

**DoD**
- [ ] All items in §8.1; explanation exam average of 4.0 or higher
- [ ] Screenshot proving zero locks in room logic

---

## 3. Lab Catalog

### 3.1 Common Labs (L3-C)

#### L3-C-01 Thread Model Comparison Table (60 min) 🔴
- **Goal**: turn design decisions into "evidence" rather than "preference"
- **Steps**: a 5-point-scale table of 4 models × 5 criteria (throughput, latency, complexity, scalability, debugging) → 3 lines of choice and rationale for each of 3 cases (Omok 100 rooms / MMO zone / FPS match)
- **Acceptance criteria**: each choice states explicitly "because of which characteristic of this case"
- **Note**: 3 drawbacks of the room actor model

#### L3-C-02 Timer Precision Measurement (45 min)
- **Steps**: fire timers 1,000 times at 1ms/10ms/100ms periods and measure the actual interval's mean/max/standard deviation. Compare before/after applying `timeBeginPeriod(1)` (also note the increased power consumption when applied)
- **Acceptance criteria**: a table of 3 intervals × before/after
- **Note**: does this precision matter for a 30-second turn timeout (if not, why)

#### L3-C-03 Timer → Room Job Posting (60 min) 🔴
- **Steps**: implement the timer callback so it **never touches room state** and only posts a Job to the queue. 3 types: turn timeout / heartbeat / reconnection grace
- **Acceptance criteria**: even if a timer fires after room destruction, it is ignored without crashing (generation number or cancellation token)
- **Common mistake**: the timer holds a strong reference to the room object, delaying destruction → use an ID reference instead

#### L3-C-04 3 State Transition Tables (60 min) 🔴
- **Steps**: write a table for each of session, room, and matching (current state × event → next state + action + illegal-case handling)
- **Acceptance criteria**: every cell is filled in (impossible combinations are explicitly marked, e.g. "illegal-ignore")

#### L3-C-05 Game Packet Design (60 min) 🔴
- **Steps**: define 30+ game packets in the 5000 range (see the §7.1 table). Each packet's fields, allowed state, and response relationship
- **Acceptance criteria**: added to `PROTOCOL.md` and serializable with the 2-2 library

#### L3-C-06 Omok Rules Implementation (150 min) 🔴
- **API**
  ```
  enum class PlaceResult { Ok, OutOfRange, Occupied, NotYourTurn, GameOver }
  PlaceResult Place(Board&, int x, int y, Stone);
  WinState  CheckWin(const Board&, int lastX, int lastY);  // None/BlackWin/WhiteWin
  bool      IsDraw(const Board&);
  ```
- **Acceptance criteria**: all 30 §7.2 rule tests pass, complete within 1 second, zero external dependencies

#### L3-C-07 Matching Queue (120 min) 🔴
- **Steps**: first-come 2-player matching, cancellation, 60-second timeout, room creation and notification on success
- **Acceptance criteria**: all 8 matching tests pass (list in §2 Day 42)
- **Common mistake**: one side disconnects at the moment of matching → missing logic to return the opponent to the queue

#### L3-C-08 Handler Dispatch (90 min) 🔴
- **Steps**: packet ID → handler table, per-state permission check, packet → Job conversion (buffer ownership transfer)
- **Acceptance criteria**: all 5 illegal transitions are rejected and the reason is logged

#### L3-C-09 Lobby and Room Entry (120 min) 🔴
- **Steps**: nickname login, lobby user list, lobby chat, match success → room entry → `GameStartNotify`
- **Acceptance criteria**: succeeds end-to-end from login to game start with the client

#### L3-C-10 Move Validation and Cheat Defense (90 min) 🔴
- **Steps**: implement 6 checks (turn, range, empty cell, game state, room membership, rate limit), reject with a response + log on violation, terminate after 3 cumulative violations
- **Acceptance criteria**: all 5 cheat tests are rejected

#### L3-C-11 Session-User Separation (90 min) 🔴
- **Steps**: separate `User` (game state, room membership) from `Session` (connection); make the user's session reference replaceable
- **Acceptance criteria**: the user object and room state persist even when the session disconnects

#### L3-C-12 Reconnection (120 min) 🔴
- **Steps**: detect disconnection → mark `Disconnected` + 60-second grace → swap the session on reconnection → send a snapshot → resume
- **Acceptance criteria**: all 6 reconnection tests pass, 100% board-state match

#### L3-C-13 Spectating (90 min)
- **Steps**: list of in-progress rooms, spectator entry (cap 10), receiving notifications, spectator chat, rejecting spectator moves
- **Acceptance criteria**: all 5 spectating tests pass

#### L3-C-14 Broadcast Optimization (60 min) 🔴
- **Steps**: share a single serialized buffer with participants and spectators (reference counting or minimized copying)
- **Acceptance criteria**: with 10 spectators, serialization happens exactly once (proven by a counter); before/after allocation comparison

#### L3-C-15 Configuration File (45 min)
- **Steps**: port, worker count, room cap, turn time, timeout, log level; separate files per environment; validate on startup
- **Acceptance criteria**: startup fails with a clear error on invalid configuration (e.g., a negative port)

#### L3-C-16 Graceful Shutdown (60 min) 🔴
- **Steps**: block new connections → post a shutdown Job to every room → notify in-progress games as "voided" → clean up sessions → join threads
- **Acceptance criteria**: completes within 5 seconds, logs each step, ignores packets arriving during shutdown

#### L3-C-17 Extending the Scenario Client (150 min) 🔴
- **Steps**: the 4 types `omok`/`spectate`/`reconnect`/`cheat` + statistics (p99, recovery rate, rejection rate)
- **Acceptance criteria**: a result file is generated from a 100-room run

#### L3-C-18 Mass Reconnection and Room-Destruction Race (90 min) 🔴
- **Steps**: 100 sessions disconnect simultaneously → reconnect after 30 seconds, repeated 5 times / a packet flood during room destruction
- **Acceptance criteria**: 100% recovery rate, zero crashes, (C++) ASan passes

### 3.2 C# Track Labs (L3-CS)

#### L3-CS-01 Channel-Based Room Actor (150 min) 🔴
- **Skeleton**
  ```csharp
  public interface IJob { void Execute(Room room); }

  public sealed class Room
  {
      private readonly Channel<IJob> _jobs =
          Channel.CreateBounded<IJob>(new BoundedChannelOptions(1024)
          { FullMode = BoundedChannelFullMode.Wait, SingleReader = true });

      public bool Post(IJob job) => _jobs.Writer.TryWrite(job);   // false when full; caller applies policy

      public async Task RunAsync(CancellationToken ct)            // one loop per room
      {
          await foreach (var job in _jobs.Reader.ReadAllAsync(ct))
              try { job.Execute(this); } catch (Exception ex) { Log.Error(ex, "job failed"); }
      }
  }
  ```
- **Acceptance criteria**: 100 rooms running concurrently, Job order guaranteed per room, zero `lock`s in room logic
- **Note**: the benefit of `SingleReader=true`, the rationale for the chosen Bounded overflow policy

#### L3-CS-02 Serilog Structured Logging (60 min) 🔴
- **Steps**: `Log.Logger = new LoggerConfiguration().Enrich.WithProperty("app","omok").WriteTo.Async(a => a.File("logs/log-.txt", rollingInterval: RollingInterval.Day)).CreateLogger();`
  Log calls should be **structured into fields** in the form `Log.Information("place {RoomId} {UserId} {X} {Y} {DurationMs}", ...)`
- **Acceptance criteria**: an entire game can be traced in the log file by `RoomId`; the async sink minimizes latency impact

#### L3-CS-03 Network Library Analysis (90 min)
- **Steps**: from the FastSocketLite, LiteNetwork, and laster40Net analysis documents, read only the receive flow, send flow, and thread model, then write a 4-row comparison table against your own server

### 3.3 C++ Track Labs (L3-CPP)

#### L3-CPP-01 Room Job Queue + Worker Assignment (180 min) 🔴
- **Skeleton**
  ```cpp
  class Room {
  public:
      bool Post(Job job);           // post to the MPSC queue, apply policy when over the cap
      void Drain();                 // called by a worker: drains the queue while executing Jobs
      std::atomic<bool> running_{false};   // only one worker per room
  };
  // Scheduler: after Post, if the running_ CAS succeeds, push into the "runnable room queue," and a worker pops it and calls Drain
  ```
- **Acceptance criteria**: two workers never Drain the same room concurrently (assert + counter), zero `mutex`es in room logic
- **Note**: what happens when the `running_` CAS fails (another worker is already processing → return as-is)

#### L3-CPP-02 spdlog Async Logging (60 min) 🔴
- **Steps**: `spdlog::init_thread_pool`, async file sink, daily rolling, unify field format (`[roomId=..][userId=..]`)
- **Acceptance criteria**: p99 does not spike due to logging under load (compare log on/off)

#### L3-CPP-03 IocpNetLib Logic Thread Analysis (90 min)
- **Steps**: trace through the code of the book's chat server to find "which thread the logic runs on" → a comparison table against your own room actor

---

## 4. Learning Items in Detail

### 4.1 Common (90h)

**Thread Model (14h)**
- What: the throughput, latency, complexity, scalability, and debugging difficulty of the 4 models (thread-per-session / I/O + single logic thread / worker pool + locks / room actor)
- Why: it's the first decision in game server design, a common interview topic, and choosing wrong means rewriting everything
- How: `L3-C-01` comparison table → write ADR-0002 → verify with a prototype
- Verification: rationale table for the 3-case choices, the ADR

**Room Actor Pattern (12h)**
- What: per-room Job queue, single consumption, inter-room messages, scheduling (worker assignment), timer Jobs, queue-cap policy
- Why: game logic can be written without locks, and data races are structurally eliminated
- How: `L3-CS-01`/`L3-CPP-01` prototype → verify with 100 rooms → apply to the real server
- Verification: zero locks in room logic (code search), zero concurrent execution (assert), Job order guaranteed across 100 rooms

**Game Loop, Tick, Timer (8h)**
- What: tick-based vs event-based, timer precision (default 15.6ms), delayed Jobs, cancellation
- Why: turn timeouts, heartbeats, and reconnection grace periods are all timers
- How: `L3-C-02` measurement → `L3-C-03` Job-posting structure
- Verification: precision table, timer harmlessness after room destruction

**State Machine (10h)**
- What: session/room/matching states and transitions, allowed packets, illegal-transition handling
- Why: without an explicit spec, the server misbehaves on inputs like "making a move from the lobby"
- How: `L3-C-04` build the table → reflect it in `L3-C-08` dispatch → lock it in with tests
- Verification: 3 transition tables + 5 illegal-transition tests

**Server Authority and Validation (8h)**
- What: 6 checks (turn, range, empty cell, game state, room membership, rate), reproducible randomness, cheat scenarios
- Why: the foundation of the online game trust model
- How: `L3-C-10` → 5 cheat tests → cumulative violation policy
- Verification: all 5 cheat types rejected + logged

**Synchronization Concepts (8h)**
- What: state synchronization vs input synchronization, snapshots & deltas, interpolation, prediction & rewind (conceptual), tick rate and bandwidth, AOI
- Why: unnecessary for Omok, but needed for the Phase 6 extension and interviews
- How: book roadmap Ch.4 → the key exercise is building a **"needed for Omok or not"** judgment table
- Verification: bandwidth calculation for "30Hz, 100 players, 16B position" and 3 ways to reduce it

**Design Documents and ADRs (10h)**
- What: design document structure, ADR format, diagrams (component, sequence, state), post-implementation updates
- Why: in the field, it's the deliverable reviewed before code, and it's a pillar of the capstone evaluation
- How: write on days 39-40 → AI "developer seeing this for the first time" review → update on day 54
- Verification: AI accurately re-explains the structure from the document alone; "deviations" table with 5+ rows

**Layer Separation and Testing (10h)**
- What: 3 layers — Rules (pure) / Logic (room, matching) / Network (session), abstracting the network via an interface, fake sessions
- Why: to catch rule bugs in 1 second without a server
- How: keep the Rules project at zero dependencies → Logic only knows about `ISessionSender`
- Verification: all 30 rule tests run within 1 second with no network reference

**Logging, Configuration, Shutdown (10h)**
- What: structured log field standard, level policy, configuration file/environment separation, shutdown order
- Why: the foundation for Phase 5 observability and operations
- How: `L3-CS-02`/`L3-CPP-02` → `L3-C-15` → `L3-C-16`
- Verification: a game traceable by roomId, shutdown completes within 5 seconds

### 4.2 C# Track (70h)

| Item | Time | What | Learning Order | Verification Method |
|---|---|---|---|---|
| Channel-based room actor | 16h | Per-room `Channel<IJob>` (Bounded), single consumer loop, room Task lifetime, cancellation, inter-room messages | L3-CS-01 → applied in 3-1 | 100 rooms run independently, zero locks in room logic |
| Timer | 6h | `PeriodicTimer`, posting timer Jobs, cancellation tokens, cleanup on room destruction | L3-C-02 → L3-C-03 | Turn timeout test |
| Handler dispatch | 8h | Packet ID → handler (dictionary/source generator), per-state permission check, packet → Job | L3-C-08 | 5 illegal transitions |
| Logic-network separation | 8h | `ISessionSender`, logic never references sockets, fake sessions | Refactor after L3-C-09 | Logic tests compile without referencing Net |
| Hosting, config, logging | 10h | Generic Host, `IHostedService`, `IOptions<T>`, **Serilog** structured logging, `IHostApplicationLifetime` | L3-CS-02 → L3-C-15/16 | Shutdown sequence log |
| Omok rules | 6h | Board representation, win checking (4 directions), draw, 6-in-a-row policy | L3-C-06 | 30 rule tests |
| Library analysis | 8h | Thread models and flows of FastSocketLite/LiteNetwork/laster40Net | L3-CS-03 | Comparison table |
| Load & fixes | 8h | 100-room scenario, bottleneck hypotheses, fixes, remeasurement | Days 51-52 | `LOAD-3-1.md` |

### 4.3 C++ Track (70h)

| Item | Time | What | Learning Order | Verification Method |
|---|---|---|---|---|
| Room Job queue and scheduling | 16h | Per-room MPSC queue (reuse of 1-1), running flag (atomic CAS), runnable room queue, whether IOCP workers and logic workers are separated | L3-CPP-01 | Zero concurrent execution assert, zero mutexes |
| Timer | 8h | Timer wheel or a `priority_queue` thread, posting to the room queue on expiry, high resolution | L3-C-02 → L3-C-03 | Precision table, timeout test |
| Handler dispatch | 6h | Packet ID → function table, state check, buffer ownership transfer | L3-C-08 | Illegal transition tests |
| Object lifetime | 10h | Session/user/room reference style (ID reference recommended), room destruction timing, cleanup order, applying memory pools | L3-C-11 → L3-C-18 | Room-destruction race passes ASan |
| Logging, config, shutdown | 8h | spdlog async, toml++/nlohmann config, shutdown via `SetConsoleCtrlHandler` | L3-CPP-02 → L3-C-16 | Shutdown sequence log |
| Omok rules | 6h | Fixed-array board, win checking, draw | L3-C-06 | 30 GoogleTest cases |
| Comparative reading 🟡 | 8h | Mapping Asio strands ↔ the room actor | After L3-CPP-03 | Comparison table |
| Load & fixes | 8h | 100 rooms, bottleneck fixes | Days 51-52 | `LOAD-3-1.md` |

---

## 5. Textbook Guide (Phase 3)

### 5.1 Common

| Book | Chapters to Read | When | How to Use |
|---|---|---|---|
| Network Learning Roadmap for Online Game Developers | Ch.3 (Game Network Architecture & Authority), Ch.4 (Synchronization & State Management, Tick Rate, Lag Compensation), Ch.5 (Performance Optimization) | Days 36-47 | The Ch.3 Server Authoritative section is the textbook for §4.1 "Server Authority." **After reading, the assignment is to build a per-item judgment table of "needed for the Omok server or not"** (training in applying judgment). Ch.4 is applied in 3-3 |
| C# Design Patterns for Game Developers | Ch.7 (Command), Ch.8 (Observer), Ch.9 (State), Ch.10 (Strategy), Ch.11 (Game Loop), Ch.12 (Update Method), Ch.16 (Anti-Patterns) | Days 36-39 | Command maps to Job, State maps to the state machine, Game Loop/Update maps to tick processing. The C++ track also reads it as a conceptual text and translates the ideas. **Note in `DESIGN.md` where each pattern is used in your own server** |
| Building an FPS Game Matching System | Ch.1 (Basic Concepts), Ch.3 (Matching Algorithm Design) | Day 42 | "2-player first-come" is enough for Omok, but after reading Ch.3, add 🟡 rating-range matching as an optional feature. Ch.4-7 are for Phase 6 |
| Network Knowledge for Online Game Client Developers | Ch.4 (Key Issues and Solutions), Ch.5 (Real-Time Gameplay) | Days 46-47 | Reflect "what the client does when the server sends this" into the design of the reconnection snapshot and spectator notifications |
| ECS-Based Online Game Server 🟡 | Ch.1 (Introduction to ECS), Ch.3 (Basic Server Architecture), `omok_ecs.md` | Day 54 | Compare against an architecture different from the room actor/OOP approach. Look at an example of the same Omok game built with ECS and list "5 differences from my design" |

### 5.2 C# Track

| Book | Chapters to Read | When | How to Use |
|---|---|---|---|
| Building a C# async/await Library | Ch.16 (Channel-Based Actor Model), Ch.17 (Room/Session Async Patterns), Ch.18 (Backpressure and Load Isolation) | Day 37 | **The direct textbook for the room actor.** Build `samples/src/Chapter16~18` and compare against your own actor. Ch.18's backpressure is the basis for the room queue's cap policy |
| C# Socket Programming for Game Server Development | Ch.11 (Omok Game Server) | Days 41-43 | A reference design. **Don't copy it** — write an ADR on the difference between "the book's thread model vs. my room actor" |
| C# Game Server Programming with SuperSocketLite | Ch.7 (Online Omok Server/Client) | Days 43-49 | Port `codes/OnlineOmok` to .NET 10, build it, and play. It is also acceptable to modify the WinForms client for use instead of 3-3 |
| Mastering C# Async/Await | Ch.10 (Practical Patterns: Tick Loop, Backpressure, Graceful Shutdown) | Day 48 | Checklist for implementing the shutdown sequence |
| C# Network Library Analysis for Study | The receive/send flow and strategy sections of the three libraries | Day 54 | Give the `*.txt` files to the AI and ask "does this library have anything equivalent to a room actor?" |

### 5.3 C++ Track

| Book | Chapters to Read | When | How to Use |
|---|---|---|---|
| Modern Windows Multithreading | Ch.6 (Thread Pool API), Ch.11 (IOCP + Thread Pool Architecture), Ch.12 (C++23 Synergy) 🟡 | Days 36-38 | **Ch.11 is the core textbook for this Phase.** Don't use the book's architecture as-is — decide "where to place the room actor" and write an ADR |
| TCP/IP Windows Socket Programming Every Game Server Developer Should Know | Ch.11 (Reread IOCP Chat), Ch.14 (Delayed Processing, Batch Processing) | Days 37-47 | Ch.14's delayed processing is the basis for timer Jobs and batch broadcasting |
| Modern Win32 API Programming for Game Server Developers | Ch.2 (Reread Memory Pools), Ch.4 (Thread Priority, Affinity) | Days 51-52 | Experiment with logic worker affinity/priority and measure the effect (if there's no effect, that's also a conclusion) |
| Building an Online Game Server with C++ Boost.Asio 🟡 | Ch.11 (Chat), Ch.12 (Strand), Ch.13 (Timer), Ch.17 (Extension Design) | Day 54 | Strand corresponds to the room actor's serialization guarantee. For comparison only — do not switch tracks |

### 5.4 Secondary-Track Reading

- C# main track: Modern Windows Multithreading Ch.11 → 1 page on "the difference in thread composition between a native IOCP server and a .NET server"
- C++ main track: Building a C# async/await Library Ch.16 → 1 page on "what is needed to port the Channel actor to C++"

---

## 6. AI Collaboration Guide (Phase 3)

### 6.1 Prompts

**Design Review (After Writing the Document First)**
```
Here is my Omok server design document. (paste DESIGN.md)
Please review it from the perspective of a senior game server developer.
1) 2 weak points from a failure perspective (session disconnects, room destruction, and timeouts occurring simultaneously)
2) 2 blocking points from a scaling perspective (10,000 rooms, 2 servers)
3) 2 things missing from an operations perspective (logging, configuration, shutdown)
For each point, give me a chance to push back. If my pushback is valid, acknowledge it; if not, provide justification.
```

**"First-Time Developer" Document Test**
```
You are a developer seeing this project for the first time. Reading only the attached design document,
explain in your own words: (1) the server's thread composition (2) who changes room state (3) the path through which a move request is processed.
Mark any part you had to guess because it wasn't in the document as a "guess."
```
→ Whatever the AI had to guess is exactly the document's defect. Fill that part into the document.

**Bottleneck Prediction → Verification**
```
If this server (design document attached) is run with 100 rooms and 300 connections, predict the top 3 places where bottlenecks would occur.
Attach a verification method (counter, log, profiler view) to each prediction.
Do not draw a conclusion yet. I will bring back the actual measurements afterward and compare them against your predictions.
```

**Defect Injection (Room Logic)**
```
Based on my room actor code (attached), create a version that hides 2 race conditions, 1 timer leak, 1 reconnection state bug, and 1 server-authority bypass.
The defects must compile and behave correctly in most runs.
Keep the answers hidden until I say "reveal the answers."
```

**Extracting Rule Test Cases**
```
Extract a list of test cases for the Omok win-check function (boundaries/corners, 6-or-more in a row, diagonals, draws, moving on a finished board, etc.).
I will add at least 5 cases missing from this list. Evaluate whether the cases I add are valid.
```

### 6.2 What to Delegate / Not to Delegate

| Delegate | Do Not Delegate |
|---|---|
| Handler dispatch boilerplate, configuration loading | Thread model selection, the room actor's core loop |
| Draft list of rule test cases | The body of the design document (only get it reviewed) |
| Log field standardization code, snapshot serialization boilerplate | Deciding the shutdown sequence order, reconnection policy |
| Defect injection code | The final conclusion on bottleneck diagnosis |

### 6.3 Points Where AI Frequently Gets It Wrong

- Suggests code that **directly modifies room state from a timer callback**, saying "it's simpler" → violates the actor principle; it must always go through a Job
- Suggests `Channel<T>` **Unbounded** by default → no backpressure. Demand a cap and an overflow policy
- In C++, makes the room referenced everywhere via `shared_ptr`, leaving destruction timing unclear → use **ID references + an explicit owner** instead
- Gives an example that talks about "server authority" while still storing the win/loss result sent by the client
- Implements reconnection by reusing the session (treating session and user as the same object) → board state is lost
- Serializes once per spectator in the spectator broadcast

---

## 7. Assignment Specifications

### 7.1 Common Assignment 3-C. Design Document + ADR (Required, 16h, Days 36-40, 54)

**Goal**: fix the design in writing before implementation, and update the differences after implementation. **This assignment is the prototype for the Phase 6 capstone document.**

**Required `DESIGN.md` Items** (template: `08-templates.md` T4)

1. **Goals and non-goals**: what is done now / not done (e.g., authentication is Phase 4, multi-server is Phase 6)
2. **Constraints**: single Windows process, target of 300 connections, 100 rooms, move p99 50ms, memory cap
3. **Component diagram**: network, session manager, lobby, matching, room, rules, timer, logging, configuration
4. **Thread model diagram**: which thread executes what, with arrows for the queues between threads. Rationale for the thread count
5. **Data ownership table**

   | Data | Owning Component | Thread Allowed to Modify | Read Access | Notes |
   |---|---|---|---|---|
   | Session list | SessionManager | I/O thread | Logic (snapshot) | Concurrent modification prohibited |
   | Room state (board, turn) | Room | The single worker executing that room | None (via Job) | No locks |
   | Matching queue | Matcher | A single matching thread | None | |
   | User list | UserManager | Logic thread | | For reconnection |

6. **3 state transition tables** (session, room, matching): state × event → next state + action + illegal-case handling
7. **Packet list**: an extension of 2-C, 30+ game packets (see the table below)
8. **3 sequence diagrams**: (1) match success (2) move → judgment → notification (3) disconnect → reconnect
9. **Risks and mitigations**: packets arriving during room destruction, timer-vs-shutdown races, spectator surges, queue saturation
10. **(Added after implementation) Deviations from the design**: a table of item / design / actual / reason — 5+ rows

**Game Packet List (5000 Range, Key Excerpt of at Least 30 Types)**

| ID | Name | Direction | Fields | Allowed State |
|---|---|---|---|---|
| 5000 | MatchReq | C→S | mode:uint8 | Lobby |
| 5001 | MatchRes | S→C | result:uint16 | - |
| 5002 | MatchCancelReq | C→S | - | Matching |
| 5003 | MatchCancelRes | S→C | result:uint16 | - |
| 5004 | MatchTimeoutNotify | S→C | waitedMs:int32 | - |
| 5010 | GameStartNotify | S→C | roomId:int32, opponent:{userId,name}, myStone:uint8, turnTimeoutMs:int32 | - |
| 5020 | PlaceReq | C→S | x:uint8, y:uint8, turnSeq:int32 | InGame |
| 5021 | PlaceRes | S→C | result:uint16 | - |
| 5022 | PlaceNotify | S→C | userId:int32, x:uint8, y:uint8, nextTurnUserId:int32, turnSeq:int32, remainMs:int32 | - |
| 5030 | TurnTimeoutNotify | S→C | loserUserId:int32 | - |
| 5031 | ResignReq | C→S | - | InGame |
| 5040 | GameEndNotify | S→C | winnerUserId:int32, reason:uint8(five-in-a-row/timeout/resignation/draw/disconnect), boardHash:int64 | - |
| 5050 | ReconnectReq | C→S | name:string, roomId:int32 | Connected |
| 5051 | ReconnectRes | S→C | result:uint16, snapshot:{board:bytes(225), turnUserId, remainMs, myStone, moves:[{x,y,seq}]} | - |
| 5052 | OpponentDisconnectNotify | S→C | userId:int32, graceMs:int32 | - |
| 5053 | OpponentReconnectNotify | S→C | userId:int32 | - |
| 5060 | SpectateListReq | C→S | page:uint16 | Lobby |
| 5061 | SpectateListRes | S→C | rooms:[{roomId, players, spectators, movesCount}] | - |
| 5062 | SpectateEnterReq | C→S | roomId:int32 | Lobby |
| 5063 | SpectateEnterRes | S→C | result:uint16, snapshot:{...} | - |
| 5064 | SpectateLeaveReq | C→S | - | Spectating |
| 5070 | RematchReq | C→S | - | Result |
| 5080 | GameChatReq | C→S | text:string(≤100) | InGame, Spectating |
| 5081 | GameChatNotify | S→C | userId:int32, name:string, text:string, isSpectator:bool | - |

**3+ ADRs**

- `0002-thread-model.md`: choosing the room actor — alternatives (single logic thread / worker pool + locks), reasons for rejection, cost of reverting
- `0003-room-session-ref.md`: the room-session reference approach — ID reference vs. pointer/shared_ptr, the destruction-timing problem
- `0004-matching.md`: the matching approach — first-come vs. rating range, matching cadence (immediate vs. batch)

**Grading**

| Item | Points | Criteria |
|---|---|---|
| Completeness | 30 | All 10 required items, 30 packet types, 3 diagrams |
| Clarity | 30 | Accurately re-explains (1)(2)(3) in the AI "first-time developer" test (0-1 guessed items) |
| ADR rationale | 20 | Each ADR has 2+ alternatives, reasons for rejection, and the cost of reverting |
| Update history | 20 | "Deviations from the design" table with 5+ rows + a diagram-update commit |

### 7.2 Track Assignment 3-1. Multi-Room Real-Time Versus Game Server (Required, 80h, Days 41-53)

**Game specification**: Omok. 15×15 board, black moves first, win with 5 in a row. **Pick and document one policy for handling 6-or-more in a row** (recommended: also count 6-in-a-row as a win — simpler). Turn limit of 30 seconds.

**Functional Requirements**

1. **Lobby**: nickname login (integrated with authentication in Phase 4), lobby user list, lobby chat (reuse of 2-1)
2. **Matching**: request/cancel, on a successful pairing of 2 create a room and notify both sides, cancellation notice after 60 seconds, handling departures right before a match completes
3. **Game**: move request → server judgment (6 checks) → notify both sides → switch turns; automatic loss on a 30-second turn timeout, resignation, win/loss/draw determination, room destruction after the result notification
4. **Reconnection**: 60-second grace period on disconnect, recover via a **board-state snapshot** on reconnection with the same nickname, loss if the grace period is exceeded
5. **Spectating**: list of in-progress rooms, spectator entry (10 per room), receiving move notifications, spectator chat, rejecting spectator moves
6. **Rematching**: return to the lobby after the result, immediate rematching
7. **Cheat defense**: moving out of turn / packets targeting a different room / more than 10 per second / out-of-range coordinates / an already-occupied cell → reject + log, terminate the session after 3 cumulative violations

**Structural Requirements**

- Room logic runs **only on a single thread** (zero locks). Packets, timers, and inter-room messages are all Jobs
- 3-layer separation: `Rules` (zero external dependencies) / `Logic` (room, matching, user) / `Net` (session, dispatch). Logic knows nothing about sockets
- Reuse the 2-2 packet library and the 1-2 object pool
- Structured logging (sessionId, userId, roomId, packetId, durationMs), a configuration file, graceful shutdown

**Non-Functional Requirements**

| Condition | Criteria |
|---|---|
| 100 rooms (200 playing + 100 spectating = 300 connections), 10 minutes | Move request→notification p99 **under 50ms**, zero crashes |
| Memory | Growth from the 1-minute mark to the 10-minute mark within 10% |
| Reconnection | 100 sessions disconnect simultaneously → reconnect after 30 seconds, 100% board-state recovery rate |
| Room-destruction race | 100 packets arrive during destruction, zero crashes |
| C++ | Pass the above scenario once with an ASan build |

**30 Rule Tests (§7.2 Required List)**

| # | Case | Expected |
|---|---|---|
| 1-5 | 5-in-a-row horizontal/vertical/down-right diagonal/up-right diagonal, exactly 5 | Win |
| 6-8 | 5-in-a-row at the board's top-left, bottom-right, and corners | Win (boundary handling) |
| 9-10 | 6-in-a-row (6 consecutive stones) | Per the documented policy |
| 11-12 | 4-in-a-row with both ends blocked | Not a win |
| 13 | A run of 5 broken by an opponent's stone | Not a win |
| 14-15 | Draw (board full) | Draw |
| 16 | Placing a stone on an already-finished board | GameOver error |
| 17 | An already-occupied cell | Occupied |
| 18-19 | Out-of-range coordinates (-1, 15) | OutOfRange |
| 20 | Placing a stone on the opponent's turn | NotYourTurn |
| 21-24 | Start/end index boundaries of the 5-in-a-row check in each direction | Correct |
| 25 | The first move is always black | Enforced |
| 26 | Recording move order (for replay) | Order preserved |
| 27 | Board hash is identical for the same state | Matches |
| 28-29 | Attempting another move right after a win | Rejected |
| 30 | Playing up to move 224 and winning on the last move | Judged correctly |

**25 Logic Tests (Using Fake Sessions)**: 10 state transitions, 5 illegal transitions, 5 cheats, 5 reconnections

**Load Report `LOAD-3-1.md`**

```markdown
## 1. Environment / 2. Scenario (100-room composition, ramp-up, duration) / 3. Results Table
| Run | Rooms | Connections | Games Completed | Move p50 | p95 | p99 | Max Room Queue | CPU | Memory (1 min / 10 min) |
## 4. Bottleneck Hypothesis Table (hypothesis / verification method / result / accepted or rejected) ← at least 1 rejection required
## 5. Fix Details and Before/After Comparison
## 6. Reconnection Scenario Results (recovery rate, time taken)
## 7. Room-Destruction Race Results
## 8. Remaining Bottlenecks and Phase 5 Tasks
```

**Grading**

| Item | Points | Criteria |
|---|---|---|
| Functionality | 25 | All of requirements 1-7 |
| Structure | 20 | Zero locks in room logic (proven by search), 3-layer separation (proven by build), rule tests run independently |
| Stability & Performance | 25 | 100-room p99 50ms, 10 minutes without incident, 100% reconnection, memory within 10% |
| Tests | 15 | 30 rule tests + 25 logic tests + integration scenario |
| Design-Code Consistency | 15 | 0-1 discrepancies when comparing the updated `DESIGN.md` against the code |

**Common Mistakes**
- A Job remaining in the queue executes after room destruction and crashes → check room state + use a generation number
- Creating a new user on reconnection loses board state → separate session and user
- A timer holds a strong reference to the room, delaying destruction → use an ID reference
- Serializing N times in the spectator broadcast → serialize once and share it
- A match success and a cancellation occurring at the same time → handle it singly on the matching thread (or use CAS)

### 7.3 Common Assignment 3-2. Extending the Scenario Test Client (Required, 16h, Day 49)

**Requirements**

1. Add 4 scenarios to the 2-3 client
   - `omok`: connect → lobby → matching → game (random **legal** moves, 0.5-2s per turn) → result → repeat rematching
   - `spectate`: enter an in-progress room as a spectator → count received notifications → leave
   - `reconnect`: forced disconnect after N seconds during the game → reconnect after 30 seconds → **verify board-state consistency** (compare board hashes)
   - `cheat`: moving out of turn / out-of-range coordinates / an already-occupied cell / 50 requests per second → count rejection responses
2. The bot references the rules library and places only legal moves (reusing the same code as the server)
3. Statistics: number of rooms, completed games, move→notification p50/p95/p99, reconnection success rate, cheat rejection rate, average game length (moves)
4. Save results as JSON + a Markdown table; compute variance across 3 runs under the same conditions

**Deliverables**: code, results from 3 runs of a 100-room execution, a variance table

**Grading**: 4 scenarios working 50 / statistics accuracy 30 / reproducibility (documented 3-run variance within 10%) 20

### 7.4 Advanced Assignment 3-3. Mini Client or 2D Action Game (Optional 🟡, 24h)

- **3-3a GUI client**: an Omok client in Unity/Godot (C#), or modify the WinForms client from the SuperSocketLite Ch.7 book to match your protocol. Required: login, matching, move, result, and reconnection UI
- **3-3b 2D action prototype**: 4 players, movement and attacks, 20Hz tick, position synchronization + interpolation. Apply Ch.4 of the roadmap book. Required: a tick-based room actor (compared against Omok's event-based one), bandwidth measurement
- **Deliverables**: a 1-minute run video, a 1-page explanation of the synchronization approach applied, a comparison table of the room actor differences between Omok (event-based) and the action game (tick-based)

---

## 8. Learning Completion Assessment (Day 55, Friday)

### 8.1 Checklist

**Assignments**
- [ ] 3-C: `DESIGN.md` (10 items + deviations table) + 3 ADRs
- [ ] 3-1: 7 features, 30 rule tests, 25 logic tests passing
- [ ] 3-2: 4 scenarios working, a 3-run variance table
- [ ] 🟡 3-3, or a reason for not doing it

**Measurements**
- [ ] `LOAD-3-1.md`: 100 rooms, 300 connections, 10 minutes, move p99 under 50ms (or root-cause analysis if not met)
- [ ] 100% recovery rate for 100 reconnecting sessions
- [ ] Zero crashes across 100 room-destruction races, (C++) ASan passes

**Structure**
- [ ] Zero lock/mutex in room logic — screenshot of the code search result
- [ ] Zero dependencies in the Rules project, Logic does not reference Net (proven by build)

**Records**
- [ ] 20 days of learning notes, 4 weekly retrospectives

### 8.2 Reimplementation Exam (No AI, 120 Minutes)

**Problem**: implement a room actor in an empty project and pass the 4 provided tests.

Required API
```
Room.Post(job) -> bool        // false when the queue cap is exceeded
Room.RunLoop(token)           // single consumer loop
Scheduler.PostTimer(roomId, delayMs, job)   // timer → room queue
Room.SendTo(otherRoomId, job) // inter-room message
```

Provided Tests
1. `OrderGuarantee`: 4 threads post 10,000 Jobs to one room → execution order matches posting order (per posting thread)
2. `NoConcurrentExecution`: 1,000 Jobs on 1 room, 8 workers → zero instances of two workers executing at once (verified by a counter)
3. `TimerFiring`: a 100ms delayed Job executes on the room queue, and a timer that fires after room destruction is ignored
4. `InterRoomMessage`: room A sends a message to room B → B processes it and responds to A, the round trip succeeds

**Pass criteria**: pass all 4 within 120 minutes.

### 8.3 Oral Exam Question Bank (10 Random Questions, Average 4.0)

1. The pros and cons of the 4 thread models. What would you choose for an Omok server versus an MMO zone server?
2. Why process a room on a single thread. What do you do when one room becomes heavy?
3. Why shouldn't a timer callback directly change room state? How did you work around it?
4. The order for cleaning up room state when a session disconnects, and the problems that arise if the order is changed
5. Why server-authoritative judgment is needed. How does it coexist with client prediction?
6. Why separate session and user when implementing reconnection grace periods?
7. What policy do you use when the room queue is full? What is the rationale for that choice?
8. How are Jobs remaining in the queue handled after a room is destroyed?
9. What happens to the room thread with 100 spectators? How did you mitigate it?
10. 3 causes of move p99 spikes and how to verify each one
11. How is it handled when a match success and a cancellation arrive at the same time?
12. What bugs arise if the state machine isn't documented (a real example)?
13. The reason and benefit of separating the rules into their own project
14. One point where the design and implementation diverged, and why
15. (C#) What options exist when a `Channel<T>` Bounded is full? Which did you choose?
16. (C#) The benefit of `SingleReader=true`
17. (C++) The approach of handling the room's running flag with atomic CAS, and the possibility of starvation
18. (C++) Why reference rooms by ID? What goes wrong if you use `shared_ptr`?

### 8.4 Defect Hunting (45 Minutes)

Room logic: 2 race conditions / 1 timer leak / 1 reconnection state bug / 1 authority bypass — find 4 of 5 and propose fixes.

### 8.5 Response When Falling Short

| Shortfall Item | Response |
|---|---|
| Move p99 not under 50ms | Submitting a root-cause analysis (including profiling results) allows proceeding to Phase 4; remeasure in the Phase 5 performance assignment |
| Locks remain in room logic | **Cannot pass.** Extend by up to 1 week to remove them (the actor principle is the core of this course) |
| Reconnection recovery rate below 90% | Redesign the session-user separation, then retest |
| Design document AI test fails | Fill every guessed item into the document, then retest |
| Reimplementation exceeds 120 minutes | Practice the room actor alone, twice, in the first week of Phase 4 |

---

## 9. Common Sticking Points

| Symptom | Cause | Remedy |
|---|---|---|
| Room has ended but Jobs keep executing | Leftover queue after destruction, timers not cancelled | Check room state `Destroyed` and ignore, add generation numbers to timers |
| Board state is off on reconnection | Session and user managed as the same object | Separate user (game state) from session (connection), replace the session pointer |
| Move p99 occasionally spikes | GC (C#)/allocation bursts, synchronous log writes, timer resolution, room queue wait | Allocation profiling, async log sink, log room queue length |
| Only rooms with many spectators are slow | N-times serialization on the room thread | Serialize once and share the buffer (reference counting) |
| Bots don't get matched with each other | Matching queue timing, bots don't request simultaneously | Send simultaneous match requests after ramp-up, check matching-cadence logs |
| C++ crashes on room destruction | The session holds a room pointer | Reference the room by ID + look it up via the manager; destruction is the room thread's last Job |
| Two workers execute the same room | CAS ordering error (order of queue posting and flag setting) | Fix the order to Post → CAS → schedule, verify with an assert |
| The room queue keeps staying full | Production outpaces consumption (broadcast bursts) | Queue-cap policy, batch processing, remove unnecessary notifications |
| Timers pile up in large numbers | A new timer is registered on every move without cancelling the old one | Cancel the existing timer before registering a new one, invalidate via generation numbers |
| Logic tests require the network | Layer separation failed | Introduce the `ISessionSender` interface, use fake sessions |
| The game result is occasionally processed twice | A timeout and a winning move occur at the same time | Only allow the `Ended` room-state transition at a single point |
| Can't trace a game through logs | Missing fields (no roomId) | Enforce the log field standard, use structured logging |

---

## 10. Preparing for Phase 4

- [ ] SQLite requires no separate installation (the default DB). 🟡 If you chose MySQL, confirm that MySQL 8 starts as a Windows service
- [ ] Confirm Redis (redis-windows) starts and `redis-cli ping` → `PONG`
- [ ] C++ track: decide the Phase 4 path (pure C++ vs. mixing in C#). If mixing, allocate 20h to fully read "Essential C# Guide for ASP.NET Core Web API" in the first week of week 12
- [ ] Set up a **"game result" event hook** (interface) in the 3-1 server — it will be wired to an API server call in Phase 4
  ```
  interface IGameResultSink { void OnGameEnded(GameResult r); }   // just logging for now, becomes an HTTP call in Phase 4
  ```
- [ ] Prepare to handle the user identifier as a **userId (integer)** instead of a nickname (in preparation for Phase 4 authentication integration)
- [ ] Phase 3 retrospective: the biggest reason design and implementation diverged, a moment you were tempted to violate the actor principle, one case where AI was wrong

---

## 11. 2026-09-05 Revisions (this section wins on conflicts)

### 11.1 Required architecture rules

#### L3-C-03b Fixed-Tick Loop and Drift (60 min) 🔴
- **Steps**: implement monotonic clock, accumulator, catch-up cap, and over-budget counter; measure ten-minute drift
- **Acceptance criteria**: injected stalls do not spiral; report ticks, drift, and over-budget count

- Day 38 adds a required 60-minute fixed-timestep loop with accumulator, catch-up cap, spiral-of-death counter, and drift measurement even if Omok gameplay stays event-driven
- Room A→B never waits synchronously; B posts a continuation Job back to A. Implement game-end→lobby notification as a real cross-room message
- Login returns a 16-byte `reconnectToken`; `ReconnectReq{name,roomId,reconnectToken}` rejects another user's name or a bad token. Duplicate names reject the new login and stay separate from reconnect
- Pause the turn timer while disconnected; the player loses when the 60-second grace expires. Add `Spectating` and `Lobby→Spectating→Lobby`
- Define `Ping/Pong(clientTs)` and `Heartbeat/HeartbeatAck` in the 5000 range. Timeout decisions use only the server monotonic clock; record RTT p50/p99
- Reject stale `turnSeq`; fix `boardHash` as FNV-1a 64 over 225 board bytes, next turn, and turnSeq. Count one rate violation per one-second window and close after three windows

### 11.2 Schedule and evaluation

- Limit Day 39 to two diagrams and Day 40 to six core design items; split gameplay flow from Day 44 onward and move overload from Days 43/46/48/55 to half-day buffers
- Rebudget 3-C/3-1/3-2 as 12h/58h/12h. Require 70/100 for 3-C and 3-1 with full structure points
- Fix the logic suite at 41 tests: transitions 10, illegal 5, matching 8, cheats 6, reconnect 7, spectating 5. Add `PlaceResult` enum↔wire `uint16` mapping
- The 100-room/300-connection target assumes 8 cores/16GB; otherwise use 50 rooms and record the environment. ASan detects UAF; lifecycle counters judge C++ leaks
- Day 41 DoD includes committed `CLAUDE.md` T3/T3'. The no-AI hour reimplements that day's room/timer/state code from an empty file

### 11.3 Additional validation

- Add per-packet token buckets, UTF-8/control-character and byte/character chat validation, plus a `replay` scenario that replays moves+seed and matches boardHash
- Add questions on fixed timestep, server clocks, cross-room deadlock, token reconnect, and queue saturation. Require failing tests before accepting `DropWrite+TryWrite`, nickname-only reconnect, or synchronous cross-room calls
