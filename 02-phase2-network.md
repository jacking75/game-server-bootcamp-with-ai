# Phase 2. Network Programming Fundamentals (Weeks 4-7, 160h)

> 🇰🇷 Korean version: [02-phase2-network_kr.md](02-phase2-network_kr.md)

> Common 70h / Track 90h. By the end of this Phase, learners will handle TCP at the socket level, design and implement a packet protocol on their own, and build a server that keeps hundreds of connections stable on Windows.

## 0. How to Use This Document

Same rules as Phase 1. Each day, keep the corresponding day block from §2 open and work through it; for the exercises that the day block calls (`L2-C-xx`, `L2-CS-xx`, `L2-CPP-xx`), look them up in §3 and carry them out as written. Detailed specifications for the assignments (2-C, 2-1, 2-2, 2-3, 2-4) are in §7.

| Notation | Meaning |
|---|---|
| `L2-C-xx` | Phase 2 common exercise |
| `L2-CS-xx` / `L2-CPP-xx` | C# / C++ track exercise |
| 🔴 | Mandatory — skipping it blocks later progress |
| 🟡 | Item that can be postponed if time is short |
| **DoD** | Definition of Done for that day |

Three core principles of this Phase

1. **Do not use packet capture tools.** Verify TCP behavior via (a) server application logs (bytes received, timestamps), (b) connection state from `netstat -an` / `Get-NetTCPConnection`, and (c) code-based tests. This approach actually forces you to focus on "what does my code see."
2. **Follow the order echo → framing → chat.** If you build the chat feature first, framing bugs will disguise themselves as feature bugs.
3. **Prove every condition with numbers.** Instead of "it runs fine," leave a log that shows "500 connections, 10 minutes, 0 loss, memory growth within 10%."

---

## 1. Overview

### 1.1 Goals

1. **Explain TCP from a code perspective.** Be able to connect the 3-way handshake, 4-way termination, TIME_WAIT/CLOSE_WAIT, flow control, and Nagle to the concrete symptoms they cause in your own server code.
2. **Implement framing yourself.** Build a length-prefix parser that defends against split, merged, and corrupted input, and prove it with 20 or more tests.
3. **Build an asynchronous server with your track's API.** Work directly with `SocketAsyncEventArgs` in C# and IOCP in C++ to sustain hundreds to a thousand connections.
4. **Manage session lifetime safely.** Achieve zero crashes and zero leaks under termination-while-receiving, termination-while-sending, and a termination storm.
5. **Generate load yourself.** Build a test client that reproduces a 500-connection, 10-minute scenario and produces statistics.

### 1.2 Prerequisite Self-Check (20 minutes before starting Phase 2)

| # | Question | Pass Criterion |
|---|---|---|
| 1 | Can you reimplement Phase 1's log queue within 60 minutes | Actually did it, and it passed |
| 2 | Is the object pool (1-2) separated out as an independent library | Can be referenced from another solution |
| 3 | Can you find a deadlock using the debugger's Parallel Stacks | Have a screenshot |
| 4 | Can you run a benchmark with a Release build and report the variance | Have one report |
| 5 | Can you tell LISTENING/ESTABLISHED/TIME_WAIT apart in `netstat -an` output | Immediate answer |

If fewer than 3 pass, handle Phase 1's unfinished assignments first. In particular, if #2 is not in place, all of this Phase's buffer management work will be blocked.

### 1.3 What You'll Be Able to Do After This Phase (in verifiable form)

- Write an **echo server in 30 minutes** starting from an empty project (synchronous version)
- **Reimplement the framing parser in 90 minutes** and pass the 10 provided tests
- Reproduce "the packet arrives truncated" using server logs, and explain through the code path how the parser handles it
- Explain, in chronological order, why a session-termination race (a close happening during the receive-completion callback) leads to a crash, and point out the defenses in your own code
- Present a 500-connection, 10-minute load test results table (loss, p50/p99 latency, memory trend)

### 1.4 Deliverables of This Phase

```
phase2/
├─ PROTOCOL.md                  Assignment 2-C: packet protocol specification
├─ adr/0001-serialization.md    ADR for the serialization choice
├─ PacketLib/                   Assignment 2-2: framing/serialization library (independent project)
│  ├─ src/  tests/(20+)
├─ EchoServer/                  Assignment 2-1a (sync) / 2-1b (async)
├─ ChatServer/                  Assignment 2-1c: chat server
│  ├─ Net/                      network layer (sessions, send/receive)
│  ├─ Logic/                    chat logic (rooms, users)  ← does not reference Net
│  ├─ tests/
│  └─ appsettings.json / config.toml
├─ LoadClient/                  Assignment 2-3: test client
├─ LOAD-2-1.md                  load results report
├─ COMPARE-2-1.md               sync vs async comparison table
└─ labs/                        lab exercise outputs
```

### 1.5 4-Week Roadmap at a Glance

| Week | Main Topic | Common | C# Track | C++ Track | Assignment |
|---|---|---|---|---|---|
| Week 4 | TCP and synchronous sockets | Layers, TCP behavior, state observation, socket API | `Socket` sync echo, thread-per-connection | Winsock sync echo, thread-per-connection | 2-C draft, 2-1a |
| Week 5 | Framing and asynchronous I/O | Stream boundaries, framing design, serialization comparison | TAP → SAEA, send queue | Non-blocking → select → Overlapped → IOCP | 2-2, 2-1b |
| Week 6 | Sessions and chat server | Session state machine, timeout, heartbeat, shutdown | MemoryPack packets, room logic | Reference counting, AcceptEx, room logic | 2-1c |
| Week 7 | Load and stabilization | Load design, statistics, interpreting results | 500-connection tuning, reading library comparisons | 500-connection tuning, ASan cleanup | 2-3, 2-4🟡, evaluation |

---

## 2. Weekly Detailed Plan (by day)

Day numbers continue across the whole program (Phase 1 is days 1-15, Phase 2 is days 16-35).

### 2.1 Week 4 — TCP and Synchronous Sockets

#### Day 16 (Mon) — Network Layers and Terminology 🔴

**Morning (2.5h)**
- Textbook "Game Server Development, Starting with Understanding Networks" chapters 1-3: terminology, OSI/TCP-IP layers, the journey of a packet
- Key points to nail down: encapsulation, IP/port/NAT, MTU/MSS, the relationship between Ethernet frames and TCP segments
- Come up with 3 questions yourself and answer them: e.g., "At which layer, and how, is the 100 bytes my server received assembled?"

**Afternoon (2.5h)**
- `L2-C-01` Prepare to observe connection state: bring up a simple listener and confirm LISTENING with `netstat -an | findstr 9000`
- `L2-C-02` Port/address concept exercise: bind the same port twice → check the error code, verify `SO_REUSEADDR` behavior
- Start Assignment 2-C: create the skeleton of `PROTOCOL.md` (placeholder for the header table, placeholder for the packet ID scheme)

**1 Hour Without AI**
- For each layer, draw a picture of "what gets attached as my chat message `hello` travels down to the LAN card"

**DoD**
- [ ] One layer diagram (in your notes)
- [ ] Listener is up and LISTENING is confirmed in `netstat`
- [ ] `PROTOCOL.md` skeleton committed

#### Day 17 (Tue) — TCP Behavior and State Transitions 🔴

**Morning (2.5h)**
- Skim the entire textbook "Network Theory Every Game Server Developer Should Know" and then summarize: 3-way handshake, 4-way termination, TIME_WAIT/CLOSE_WAIT, sequence/ACK, flow control (window), congestion control overview, Nagle and delayed ACK, keep-alive, RST
- Cross-check against official documentation (an RFC 9293 summary or learn.microsoft.com) and find at least one place where it differs from the book

**Afternoon (2.5h)**
- `L2-C-03` Observe state transitions: while going through connect → data → close, poll `Get-NetTCPConnection` every second and build a table of state changes
- `L2-C-04` Who closes first experiment: compare where TIME_WAIT ends up between (a) server closes first and (b) client closes first
- `L2-C-05` Producing an RST: call `closesocket` (or `Socket.Close(0)`) while data still remains in the receive buffer → confirm the error the peer receives

**1 Hour Without AI**
- In 5 sentences: "Why does TIME_WAIT pile up on the server when the server closes first, and why is that a problem?"

**DoD**
- [ ] State transition table (at least 6 rows: SYN_SENT/ESTABLISHED/FIN_WAIT_1/FIN_WAIT_2/TIME_WAIT/CLOSE_WAIT)
- [ ] Comparison table of server-closes-first vs. client-closes-first
- [ ] RST reproduction log

#### Day 18 (Wed) — Synchronous Socket Echo Server (Assignment 2-1a) 🔴

**Morning (2.5h)**
- Socket API concepts: blocking/non-blocking, `bind/listen/accept/connect/send/recv/shutdown/close`
- The existence of **partial sends**: `send` can send less than requested → a loop is required
- Track textbook: C# reads "C# Socket Programming" chapters 1-3 / C++ reads "TCP/IP Windows Socket Programming" chapters 1-4

**Afternoon (2.5h)**
- `L2-CS-01` or `L2-CPP-01`: synchronous echo server + client (thread-per-connection)
- Requirements: one thread per connection, treat a 0-byte `recv` as a normal close, a partial-send loop, log exceptions/error codes

**1 Hour Without AI**
- Rewrite the echo server **from an empty file in 30 minutes** (the baseline skill for this Phase)

**DoD**
- [ ] Synchronous echo server/client works, stable at 100 connections
- [ ] 0-byte receive handling and a partial-send loop are present in the code

#### Day 19 (Thu) — Measuring the Limits of a Synchronous Server 🔴

**Morning (2.5h)**
- The cost of thread-per-connection: thread stacks (1MB by default), context switching, kernel objects
- Begin the server I/O model overview: readiness notification vs. completion notification

**Afternoon (2.5h)**
- `L2-C-06` Measure the limits of a synchronous server: try 100 / 500 / 1,000 connections and record **thread count, working set (memory), CPU, and connection success rate**
  - Thread count: `Get-Process -Id <pid> | Select Threads` or Task Manager
  - Find and record the failure point (connections refused, response latency spiking)
- Fill in the "sync" column of `COMPARE-2-1.md` with the results

**1 Hour Without AI**
- Write 3 paragraphs, backed by your measurements, on "why 1,000 threads can't handle 1,000 connections"

**DoD**
- [ ] Measurement table across 3 connection-count tiers (threads, memory, CPU, success rate)
- [ ] "Sync" column of `COMPARE-2-1.md` completed

#### Day 20 (Fri) — Protocol Specification Draft + Weekly Checkpoint

**Morning (3h)**
- Get seriously started on Assignment 2-C (§7.1): header byte table, packet ID scheme, list of 20 chat packet types, session state machine, heartbeat rules
- Compare serialization candidates (manual/MemoryPack/MessagePack/Protobuf/FlatBuffers) → ADR draft

**Afternoon (4h) — Weekly Checkpoint**
1. Reimplement without AI (60 min): synchronous echo server
2. Explanation exam (45 min): the 4 stages of the handshake, TIME_WAIT, partial sends
3. Code review (45 min): focused on rubric item 4 (error handling)
4. Weekly retrospective (30 min): `notes/weekly/W04.md`

**DoD**
- [ ] `PROTOCOL.md` draft (header, ID scheme, 10+ packet types)
- [ ] ADR draft, weekly retrospective

### 2.2 Week 5 — Framing and Asynchronous I/O

#### Day 21 (Mon) — Reproducing the Stream Boundary Problem 🔴

**Morning (2.5h)**
- TCP is a byte stream: there are no message boundaries. Length-prefix vs. delimiter vs. fixed length
- Header design principles: position and size of the length field, maximum packet size, endianness (network byte order), alignment, version/flags
- The ring buffer concept

**Afternoon (2.5h)**
- `L2-C-07` Reproduce split receives: `send` 64KB in one call → in the server's receive callback, log "bytes received, cumulative total" each time → confirm with logs that **it does not all arrive at once**
- `L2-C-08` Reproduce merged receives: `send` 100 small messages back to back (Nagle on) → confirm in the log that multiple messages arrive attached to a single `recv`
- `L2-C-09` Nagle on/off comparison: with `TCP_NODELAY` on and off, compare the send-to-receive timestamp difference for 100 small messages in a table

**1 Hour Without AI**
- Write, by hand, the **pseudocode** for a parser that handles both phenomena above (states: waiting for header / waiting for body)

**DoD**
- [ ] Split and merge logs saved separately (`labs/L2-C-07`, `08`)
- [ ] Nagle on/off comparison table
- [ ] Parser pseudocode

#### Day 22 (Tue) — Implementing the Framing Library (Assignment 2-2) 🔴

**Morning (2.5h)** — Design first
- Finalize the `PacketReader`/`PacketWriter` API (use the signatures in §7.3)
- Error-handling policy: **return an error code, not an exception** → the caller decides whether to close the session
- Buffer strategy: use the 1-2 object pool, cap the maximum packet size (e.g., 64KB)

**Afternoon (2.5h)** — Implementation and testing
- `L2-C-10` Implement `PacketReader`: accumulation buffer + header parsing + waiting on a partial packet
- Write 8 tests first (feed 1 byte at a time, land exactly on a boundary, two merged packets, length 0, exceeding the maximum, incomplete header, incomplete body, consecutive calls)

**1 Hour Without AI**
- Reimplement `PacketReader` from an empty file (reuse the same tests)

**DoD**
- [ ] 8 `PacketReader` tests pass
- [ ] Error code scheme defined (`Ok/NeedMore/TooLarge/BadLength/BadVersion`)

#### Day 23 (Wed) — Serialization and Packet Definitions 🔴

**Morning (2.5h)**
- Serialization comparison: manual binary (`BinaryPrimitives`/`memcpy`), MemoryPack, MessagePack, Protobuf, FlatBuffers — size, speed, schema evolution, language support
- Textbook "Network Learning Roadmap" Chapter 7 (application protocols, serialization)

**Afternoon (2.5h)**
- `L2-C-11` Serialization comparison experiment: serialize the same 5 packet types with 2 methods → table of **byte size, serialization time, deserialization time**
- Continue Assignment 2-2: implement `PacketWriter` + define 20 chat packet types (C#: MemoryPack / C++: manual or FlatBuffers)
- Finish the ADR: why this serialization was chosen (including alternatives and reasons for rejection)

**1 Hour Without AI**
- Draw the byte layout of your own 5 packet types by hand (marking offsets and sizes)

**DoD**
- [ ] Serialization comparison table
- [ ] All 20 packet types defined, round-trip tests pass
- [ ] `adr/0001-serialization.md` completed

#### Day 24 (Thu) — Introduction to Asynchronous I/O (Start Assignment 2-1b) 🔴

**C# Track (5h)**
- `L2-CS-02` TAP async echo: `AcceptAsync/ReceiveAsync/SendAsync` (Task-based), the `Memory<byte>` overloads, cancellation tokens, handling `SocketException` codes
- `L2-CS-03` Intro to SAEA: echo using a single `SocketAsyncEventArgs` → confirm with logs that **when `ReceiveAsync` returns `false`, it completed synchronously** (this is the core point of this exercise)
- Textbook "C# Socket Programming" chapters 4 and 10

**C++ Track (5h)**
- `L2-CPP-02` Non-blocking + `select` echo: `ioctlsocket(FIONBIO)`, handling `WSAEWOULDBLOCK`, confirm the 64-socket limit of `fd_set`
- `L2-CPP-03` Overlapped I/O echo: `WSARecv/WSASend` + `OVERLAPPED` + event-based approach, **buffer lifetime rules** (never free before completion)
- Textbook "TCP/IP Windows Socket Programming" chapter 9 (I/O models), read the `first_IOCP` code

**1 Hour Without AI**
- C#: explain in code what happens if you don't handle SAEA's synchronous-completion path
- C++: in 3 sentences, why is it dangerous to put an OVERLAPPED on the stack

**DoD**
- [ ] Async echo works (C#: TAP+SAEA / C++: select+Overlapped)
- [ ] Synchronous (or immediate) completion path confirmed via logs

#### Day 25 (Fri) — Finishing the Asynchronous Server + Weekly Checkpoint

**Morning (3h)**
- C#: `L2-CS-04` SAEA pooling + session class (1 receive SAEA, 1 send SAEA) → sustain 1,000 connections
- C++: `L2-CPP-04` Basic IOCP: `CreateIoCompletionPort`, N worker threads, `GetQueuedCompletionStatus`, Per-Handle/Per-I/O data → sustain 1,000 connections
- After measuring, fill in the "async" column of `COMPARE-2-1.md` (thread count, memory, CPU, success rate)

**Afternoon (4h) — Weekly Checkpoint**
1. Reimplement without AI (90 min): the framing parser (same format as the §8.2 exam)
2. Explanation exam (45 min): stream boundaries, the pitfalls of length-prefixing, synchronous/immediate completion
3. Code review (45 min): focused on rubric item 3 (resources) — lifetime of buffers, SAEA, OVERLAPPED
4. Retrospective (30 min): `W05.md`

**DoD**
- [ ] 1,000 connections sustained, `COMPARE-2-1.md` completed (sync vs. async)
- [ ] 15+ tests pass for Assignment 2-2

### 2.3 Week 6 — Session Management and the Chat Server

#### Day 26 (Mon) — Session State Machine and Lifetime 🔴

**Morning (2.5h)**
- Session states: `Connected → Named (before/after auth) → Active → Closing → Closed`
- The termination race: another thread calls close while the receive-completion callback is running → double-free / use-after-free
- Defenses: in C#, hold a reference + `Interlocked` state transitions; in C++, **reference counting** (count of in-flight I/O) + `shared_ptr`
- `shutdown` vs. `closesocket`, `SO_LINGER`

**Afternoon (2.5h)**
- `L2-C-12` Implement a session state machine: state enum + allowed-transition table + logging illegal transitions
- `L2-C-13` Reproduce the termination race: force-close 100 times while receive processing is in progress → observe crashes/leaks → reapply the defense and rerun
- C++ additionally: `L2-CPP-05` Reference-counted session (in-flight I/O counter, free only when it reaches 0)

**1 Hour Without AI**
- Draw a diagram of "the timing where recv completion and close happen simultaneously," and write down the defense needed at each point

**DoD**
- [ ] State transition table and 5 illegal-transition tests
- [ ] Zero crashes across 100 repetitions of the termination race

#### Day 27 (Tue) — Send Queue and Backpressure 🔴

**Morning (2.5h)**
- **No concurrent Send**: if two threads send on the same socket at the same time, data gets interleaved
- Per-session send queue + an "1 send in flight" flag, send the next item from the send-completion callback
- Send coalescing (batching several small packets into one), backpressure (policy when the queue cap is exceeded: disconnect vs. drop)

**Afternoon (2.5h)**
- `L2-C-14` Implement and test the send queue: 100 threads calling `Send` on the same session simultaneously → verify on the receiving side that **message boundaries are not broken**
- `L2-C-15` Backpressure test: send a large volume to a slow client that isn't reading → confirm the policy kicks in once the queue cap is reached (no memory blow-up)

**1 Hour Without AI**
- Write down 3 scenarios where the send queue grows unbounded, and the defense for each

**DoD**
- [ ] Concurrent-Send test passes (0 data interleaving)
- [ ] Confirmed memory stays capped with a slow client

#### Day 28 (Wed) — Chat Logic (Assignment 2-1c) 🔴

**Morning (2.5h)**
- Layer-separation design: `Net` (sessions, send/receive) ↔ `Logic` (rooms, users). **Logic knows nothing about sockets** (sends via an interface)
- Room management data structures, broadcast cost (don't serialize N times for N recipients — serialize once and share)

**Afternoon (2.5h)**
- `L2-C-16` Implement chat logic: room create/join/leave/list, broadcast, whisper, a 100-person room cap
- Unit tests for the logic (no network): 10 tests verifying room join/leave and broadcast using a fake session

**1 Hour Without AI**
- Find the part of room-list lookup that's O(N) and write an improvement plan

**DoD**
- [ ] All 5 room features work, 10 logic tests pass
- [ ] The Logic project does not reference the Net project (proven by the build)

#### Day 29 (Thu) — Timeout, Heartbeat, Shutdown 🔴

**Morning (2.5h)**
- Heartbeat design: client sends every 10 seconds, server closes after 30 seconds of no response. Timer period vs. check cost
- Force-close a session 30 seconds after connecting if no nickname is set (cleaning up unauthenticated sessions = the basis of flood defense)
- Graceful shutdown order: block new connections → broadcast a shutdown notice to all sessions → wait for pending sends to finish (with a cap) → clean up sockets → join threads

**Afternoon (2.5h)**
- `L2-C-17` Implement a timeout-check timer (compare full session iteration vs. an expiration-time heap)
- `L2-C-18` Implement and test graceful shutdown: all threads exit within 5 seconds of the shutdown signal, with the order recorded in the log
- `L2-C-19` Defend against corrupted packets: receive a packet with a corrupted header/oversized length/state mismatch → **only that session** is closed, the server survives

**1 Hour Without AI**
- Draw the shutdown sequence as a flowchart and write, for each step, "the problem that occurs if this order is changed"

**DoD**
- [ ] Heartbeat timeout test (an unresponsive session is cleaned up within 30 seconds)
- [ ] Log showing graceful shutdown completes within 5 seconds
- [ ] 3 corrupted-packet tests pass

#### Day 30 (Fri) — Chat Server Integration + Weekly Checkpoint

**Morning (3h)**
- Integrate 2-1c: framing (2-2) + sessions + send queue + chat logic + config file (port, worker count, buffer size, timeouts)
- Use the config file to change the port and run two instances at once (a preview of Phase 6)

**Afternoon (4h) — Weekly Checkpoint**
1. Reimplement without AI (90 min): a minimal session class that includes defense against the termination race
2. Explanation exam (45 min): termination race, send queue, rationale for the heartbeat design
3. Code review (45 min): focused on rubric items 2 and 3 → fix the top 3 issues
4. Retrospective (30 min): `W06.md`

**DoD**
- [ ] Chat server boots from a config file, all 6 features work
- [ ] Retrospective written

### 2.4 Week 7 — Load Testing and Stabilization

#### Day 31 (Mon) — Test Client (Assignment 2-3) 🔴

**Morning (2.5h)**
- Load test design: ramp-up (N connections per second), steady-state hold, teardown
- Metrics: connection success/failure, messages sent/received, **round-trip latency p50/p95/p99**, throughput per second
- Limits of the local loopback: dynamic port exhaustion, the client itself becoming the bottleneck (CPU)

**Afternoon (2.5h)**
- `L2-C-20` Client skeleton: argument parsing (`--host --port --conn --rampup --scenario --duration`), a connection pool, a statistics collector
- Implement the `chat` scenario: connect → set nickname → join a room (spread across N/50) → send a message every second → disconnect
- Latency measurement: the time for your own message to come back via broadcast (matched by sequence number)

**1 Hour Without AI**
- Compare methods for computing p99 accurately (sorting vs. histogram) and justify the one you chose

**DoD**
- [ ] `chat` scenario runs for 5 minutes with 100 connections, statistics printed
- [ ] p99 computation verified (unit test against known data)

#### Day 32 (Tue) — 500-Connection Load and First-Pass Tuning 🔴

**Morning (2.5h)**
- Run the first 500-connection, 10-minute pass → record failures/bottlenecks (connection failures, latency spikes, memory growth, 100% CPU)
- Check the dynamic port range: `netsh int ipv4 show dynamicport tcp`, widen it if needed

**Afternoon (2.5h)**
- Come up with 3 bottleneck hypotheses and decide how to verify each (counters, logs)
- Fix and re-measure. Common causes: allocating on every receive (pool unused), serializing N times on broadcast, synchronous log writes, send-queue lock contention
- Record the results in `LOAD-2-1.md` (comparing pass 1 vs. pass 2)

**1 Hour Without AI**
- List 5 candidate causes for "p99 latency spikes" and how to observe each

**DoD**
- [ ] Completed the 500-connection, 10-minute run at least once (record it even if it fails)
- [ ] At least one hypothesize-fix-re-measure cycle recorded

#### Day 33 (Wed) — Termination Storm and Resource Leaks 🔴

**Morning (2.5h)**
- Termination storm: problems that occur when 500 connections drop at once (reference-count errors, handle leaks, dangling dictionary entries)
- Resource tracking: log handle count (Task Manager), session count, and unreturned pool count every 5 seconds

**Afternoon (2.5h)**
- `L2-C-21` Implement the `storm` scenario (client) + run it: 500 connections, then simultaneous disconnect × 5 repetitions
- Pass criteria: 0 crashes, handle count returns to the starting level, session count 0, unreturned pool count 0
- C++: pass once with an ASan build (measure performance separately in Release)

**1 Hour Without AI**
- Write a design for where to place leak-tracking counters (session creation/destruction, buffer rent/return)

**DoD**
- [ ] Handle/session/pool counters return to normal after 5 repetitions of the termination storm
- [ ] C++: ASan pass log

#### Day 34 (Thu) — Stabilization and Reading Comparisons

**Morning (2.5h)**
- **Final measurement** at 500 connections, 10 minutes: 0 loss, memory growth within 10% (1-minute mark vs. 10-minute mark), record p99
- Complete `LOAD-2-1.md` (report format in §7.2)

**Afternoon (2.5h)**
- C#: `L2-CS-05` Read the structure of SuperSocketLite / FreeNetLite → build a comparison table of **threading model, buffer management, and session lifetime** against your own server
- C++: `L2-CPP-06` Read the structure of `IocpNetLib` → the same comparison table. 🟡 Build the Boost.Asio sample and compare lines of code
- 🟡 Do Assignment 2-4 (cross-language communication)

**1 Hour Without AI**
- Looking at the comparison table, pick "3 things to fix in my server right now" and register them as issues

**DoD**
- [ ] `LOAD-2-1.md` completed (500 connections, 10 minutes, termination storm, memory trend)
- [ ] One library comparison table

#### Day 35 (Fri) — Phase 2 Evaluation 🔴

**Morning (3h)**
1. **Reimplementation exam (90 min, no AI)**: §8.2 — the framing parser + pass the 10 provided tests
2. **Explanation exam (60 min)**: 10 questions from the §8.3 question bank
3. **Bug hunt (45 min)**: §8.4 — find 4 of 5 planted defects in framing/session termination/send queue

**Afternoon (3h)**
- Go through the checklist (§8.1), fill in anything incomplete
- `W07.md` + Phase 2 retrospective
- §10 preparation for Phase 3 (organize shared libraries)

**DoD**
- [ ] Every item in §8.1 complete, explanation exam average of 4.0 or higher
- [ ] `PacketLib` and `LoadClient` are referenceable from other solutions

---

## 3. Exercise Catalog

Save results under `phase2/labs/<exercise-number>/`. Logs, tables, and screenshots must be kept as files (they're the evidence for the explanation exam and reports).

### 3.1 Common Exercises (L2-C)

#### L2-C-01 Listener and Connection State Check (30 min) 🔴
- **Goal**: view the state of a server socket from the OS's perspective
- **Steps**: listener on port 9000 → `netstat -an | findstr 9000` and `Get-NetTCPConnection -LocalPort 9000` → connect 1 client and check again
- **Expected**: 1 `LISTENING` row before the connection, 2 `ESTABLISHED` rows after (server side and client side)
- **Pass criterion**: save the output at both points to a file, with a comment on what each row means
- **Note**: why are there 2 ESTABLISHED rows (loopback on the same machine)

#### L2-C-02 Bind Conflicts and SO_REUSEADDR (30 min)
- **Steps**: 2 listeners on the same port → check the error code (`WSAEADDRINUSE`/`SocketException 10048`) → set `SO_REUSEADDR` and retry → attempt to rebind right after the server closes (TIME_WAIT effect)
- **Pass criterion**: a table of error codes and behavior with/without the option
- **Caution**: Windows' `SO_REUSEADDR` has different semantics from Linux's (hijacking is possible). Note the difference

#### L2-C-03 Observing State Transitions (45 min) 🔴
- **Steps**: while performing connect → send/receive data → normal close, record `Get-NetTCPConnection -LocalPort 9000 | Select State,LocalPort,RemotePort` in the background every second
- **Expected**: `LISTEN → ESTABLISHED → FIN_WAIT_1/2 or CLOSE_WAIT → TIME_WAIT → gone`
- **Pass criterion**: a chronological table containing at least 6 states
- **Note**: TIME_WAIT's default duration and the reason for it

#### L2-C-04 Who Closes First (40 min) 🔴
- **Steps**: observe **which side** ends up with TIME_WAIT in two scenarios: (a) the server closes first, (b) the client closes first. Repeat each 100 times and compare counts with `netstat -an | findstr TIME_WAIT | Measure-Object`
- **Pass criterion**: a table of TIME_WAIT counts for both cases
- **Note**: why the risk of port exhaustion grows when the server closes first

#### L2-C-05 Producing an RST (30 min)
- **Steps**: close the socket while data remains in the receive buffer, or set `SO_LINGER(0)` and close → check the error on the peer's side (`WSAECONNRESET` / `SocketException 10054`)
- **Pass criterion**: log the difference in what the peer receives between a normal close (FIN) and a forced close (RST)

#### L2-C-06 Measuring the Limits of a Synchronous Server (90 min) 🔴
- **Steps**: increase the connection count through 100 / 500 / 1,000 and measure thread count, working set, CPU, connection success rate, and average response time (3 times each)
- **Pass criterion**: a 4-column table + 3 lines on "at what point did what break first"
- **Common mistake**: the client dies first → spread the client across 2 processes

#### L2-C-07 Reproducing Split Receives (40 min) 🔴
- **Steps**: `send` 64KB in one call → the server logs `[t, bytes, total]` on every receive callback
- **Pass criterion**: the log shows 2 or more `recv` calls, totaling 65,536
- **Note**: explain "why it doesn't all arrive at once" in terms of MSS, window, and kernel buffers

#### L2-C-08 Reproducing Merged Receives (30 min) 🔴
- **Steps**: `send` 100 20-byte messages back to back (Nagle on) → confirm in the server log that a single recv contains multiple messages
- **Pass criterion**: a logged case where a single recv contains 2 or more messages
- **Note**: what bug results if you code "1 recv = 1 message" without framing

#### L2-C-09 Nagle On/Off Comparison (40 min)
- **Steps**: with `TCP_NODELAY` on and off, send 100 20-byte messages each; the client records the send time and the server records the receive time for each message (clock is shared since it's the same machine)
- **Pass criterion**: a table comparing average/max latency
- **Note**: why game servers turn Nagle off, and when turning it off is actually a loss

#### L2-C-10 Implementing PacketReader (120 min) 🔴
- **Goal**: the core of Assignment 2-2. Reconstruct packets correctly no matter how they arrive fragmented
- **Steps**: ① design an accumulation buffer (ring buffer or growable array) ② header parsing ③ waiting for the body ④ return a slice once complete ⑤ return error codes
- **Pass criterion**: pass 12 of the 20 framing-related tests in §7.3
- **Common mistake**: offset calculation errors → catch these with boundary tests (exactly the header size, header size + 1 byte)

#### L2-C-11 Serialization Comparison Experiment (60 min)
- **Steps**: serialize 5 packet types (login request/response, chat, room list, heartbeat) using two methods → measure byte size and serialization/deserialization time (10,000 iterations)
- **Pass criterion**: a table + 3 lines justifying the choice → feed into the ADR

#### L2-C-12 Session State Machine (60 min) 🔴
- **Steps**: state enum, transition table (current state × event → next state), log + close the session on an illegal transition
- **Pass criterion**: 5 illegal-transition tests pass (e.g., chatting before authentication, joining while closing)

#### L2-C-13 Reproducing and Defending Against the Termination Race (90 min) 🔴
- **Steps**: repeat 100 times a scenario where another thread closes the session while receive processing is in progress → observe crashes/double-free → apply a defense (reference counting or a state CAS) → rerun 1,000 times
- **Pass criterion**: 1,000 clean runs after the fix; C++ passes under ASan

#### L2-C-14 Send Queue and Preventing Concurrent Send (90 min) 🔴
- **Steps**: have 100 threads call `Send` with different messages on the same session simultaneously → verify every message arrives **intact** on the receiving side (0 broken boundaries)
- **Pass criterion**: 100% integrity across 10 repetitions

#### L2-C-15 Backpressure Test (60 min)
- **Steps**: create a client that doesn't read, and have the server send a large volume → confirm the policy (disconnect or drop) kicks in once the send-queue cap is reached, and server memory stays capped
- **Pass criterion**: memory growth stays within the cap, policy is logged

#### L2-C-16 Implementing Chat Logic (120 min) 🔴
- **Steps**: room create/join/leave/list/broadcast/whisper, a member cap. Structure it so it's testable **without a network**
- **Pass criterion**: 10 logic tests pass, the Logic project does not reference the socket API

#### L2-C-17 Timeout Checking (60 min)
- **Steps**: implement both (a) full session iteration and (b) an expiration-time heap, and compare the check cost at 5,000 sessions
- **Pass criterion**: a cost comparison table for both approaches + the reasoning for your choice

#### L2-C-18 Graceful Shutdown (60 min) 🔴
- **Steps**: shutdown signal (Ctrl+C handler) → block new connections → broadcast a shutdown notice → wait for pending sends (up to 2 seconds) → clean up sockets → join threads. Log each step
- **Pass criterion**: fully shuts down within 5 seconds, with the step order recorded in the log

#### L2-C-19 Defending Against Corrupted Packets (45 min) 🔴
- **Steps**: have the client send (a) a length field of 0xFFFFFFFF, (b) length 0, (c) an undefined packet ID, (d) a packet that doesn't match the current state
- **Pass criterion**: in each case, only that session is closed, the server survives, and the reason is logged

#### L2-C-20 Test Client Skeleton (120 min) 🔴
- **Steps**: argument parsing → ramp-up connections → run the scenario → print statistics every 5 seconds → save to a file on exit
- **Pass criterion**: runs for 5 minutes at 100 connections, statistics file is produced

#### L2-C-21 Termination Storm Scenario (60 min) 🔴
- **Steps**: acquire N connections, then disconnect all simultaneously, repeated 5 times. Record the server's handle count, session count, and unreturned pool count before/after each round
- **Pass criterion**: counters return to their starting values after 5 rounds

### 3.2 C# Track Exercises (L2-CS)

#### L2-CS-01 Synchronous Echo Server (90 min) 🔴
- **Skeleton**
  ```csharp
  var listener = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
  listener.Bind(new IPEndPoint(IPAddress.Any, 9000));
  listener.Listen(backlog: 512);
  while (true) {
      var client = listener.Accept();
      new Thread(() => HandleClient(client)) { IsBackground = true }.Start();
  }
  // HandleClient: Receive loop; on 0 bytes, close; Send via a partial-send loop
  ```
- **Pass criterion**: works normally at 100 connections, handles 0 bytes, a partial-send loop exists
- **Note**: what `Listen(backlog)` means and what happens if the value is too small

#### L2-CS-02 TAP Asynchronous Echo (90 min)
- **Steps**: `AcceptAsync/ReceiveAsync/SendAsync` (Task-based) + `Memory<byte>` overloads + `CancellationToken`
- **Pass criterion**: at 1,000 connections, thread count is noticeably lower than the sync version (compare in a table)

#### L2-CS-03 SAEA Synchronous-Completion Path (60 min) 🔴
- **Steps**: receive using `SocketAsyncEventArgs` and confirm via logging that a `ReceiveAsync(e)` **return value of false means it completed immediately (synchronously)** → in that case, the `Completed` event never fires, so you must run the processing loop yourself
- **Pass criterion**: log the number of synchronous completions, 0 missed handling
- **Note**: why handling synchronous completion via recursion risks a stack overflow (why to convert it to a loop)

#### L2-CS-04 SAEA Pooled Sessions (150 min) 🔴
- **Steps**: one receive SAEA + one send SAEA per session, rent/return from a SAEA pool, use the 1-2 pool for buffers, keep a session reference in `UserToken`
- **Pass criterion**: sustains 1,000 connections; 0 unreturned SAEA/buffers after repeated connect/disconnect

#### L2-CS-05 Reading Library Comparisons (90 min)
- **Steps**: lay out the receive flow, send flow, and session lifetime of `FreeNetLiteRe` and SuperSocketLite as function call sequences
- **Pass criterion**: a 4-row comparison table against your own server (threading model / buffer management / session lifetime / backpressure)

### 3.3 C++ Track Exercises (L2-CPP)

#### L2-CPP-01 Winsock Synchronous Echo (90 min) 🔴
- **Skeleton**: `WSAStartup` → `socket/bind/listen` → `accept` loop → a `std::thread` per connection → `recv/send` loop → `shutdown/closesocket` → `WSACleanup`
- **Pass criterion**: works normally at 100 connections, handles `recv` of 0, a `send` partial-send loop, `WSAGetLastError()` logged on every error
- **Note**: the difference between `sockaddr_in` and `getaddrinfo`, and which fields need byte-order conversion

#### L2-CPP-02 Non-Blocking + select (90 min)
- **Steps**: `ioctlsocket(FIONBIO)`, handle multiple sockets with `select`, handle `WSAEWOULDBLOCK`, confirm the `fd_set` limit of 64 (behavior when exceeded)
- **Pass criterion**: works normally at 60 connections, the limit is logged when exceeding 64
- **Note**: the O(N) scan cost of select

#### L2-CPP-03 Overlapped I/O Echo (90 min) 🔴
- **Steps**: `WSARecv/WSASend` + `OVERLAPPED` + event-based approach. A structure that **keeps buffers and OVERLAPPED alive until the completion notification** (heap allocation with a clear owner)
- **Pass criterion**: normal echo; deliberately break it once using a stack-allocated OVERLAPPED, then fix it
- **Note**: the window during which the kernel owns the buffer

#### L2-CPP-04 Basic IOCP (180 min) 🔴
- **Steps**: `CreateIoCompletionPort` → N worker threads (based on core count) → `GetQueuedCompletionStatus` loop → distinguish Per-Handle (session) / Per-I/O (OVERLAPPED extension) data → asynchronous accept via `AcceptEx`
- **Pass criterion**: sustains 1,000 connections, worker thread count is fixed, 0 handle leaks across repeated connect/disconnect
- **Note**: `lpOverlapped` can still be valid even when `GetQueuedCompletionStatus` fails → a cleanup code path is needed

#### L2-CPP-05 Reference-Counted Session (120 min) 🔴
- **Steps**: an in-flight I/O counter (atomic) on the session → increment when I/O starts, decrement on completion → free only once it reaches 0 and the close flag is set. Be careful with `enable_shared_from_this` when using `shared_ptr`
- **Pass criterion**: 0 crashes and 0 ASan errors in a 500-connection termination storm

#### L2-CPP-06 Reading the IocpNetLib Structure (90 min)
- **Steps**: lay out the textbook code's receive path as a function call sequence, and compare with your own server
- **Pass criterion**: a 4-row comparison table (threading model / buffers / session lifetime / send queue)

---

## 4. Learning Items in Detail

### 4.1 Common (70h)

**Network Fundamentals Theory (12h)**
- What: OSI/TCP-IP layers, encapsulation, IP/port/NAT, MTU/MSS, an overview of ARP/ICMP
- Why: to explain "why packets get split" and "why something that works locally doesn't work externally" in terms of layers
- How: read the textbook straight through → daily terminology quiz (5 questions) → map it onto real connection state with `L2-C-01`
- Verify: observe your own server's connections with `netstat -an`/`Get-NetTCPConnection` and write notes on how each state maps to the layer concepts

**TCP Behavior (14h)**
- What: 3-way handshake, 4-way termination, TIME_WAIT/CLOSE_WAIT, sequence/ACK, flow control (window), congestion control overview, Nagle and delayed ACK, keep-alive, RST
- Why: "detecting a dropped connection," "sources of latency," and "port exhaustion" all trace back to this
- How: textbook → cross-check with official docs → reproduce with `L2-C-03/04/05/09`
- Verify: ① a state transition table ② a comparison table of server/client closing first ③ a Nagle on/off latency comparison table

**Stream Characteristics and Framing (12h)**
- What: no message boundaries, length-prefix/delimiter/fixed-length, header design, maximum size limits, ring buffers, endianness
- Why: getting framing wrong makes the server crash at arbitrary moments or blow up memory
- How: order is `L2-C-07` (split) → `L2-C-08` (merge) → `L2-C-10` (implement the parser). **See the phenomenon first, then build the parser**
- Verify: split/merge logs and 12 parser tests

**Serialization (8h)**
- What: size, speed, schema evolution, and language support of manual binary, MemoryPack, MessagePack, Protobuf, FlatBuffers
- Why: the protocol is shared with the client, so you must be able to explain your reasoning for the choice
- How: `L2-C-11` comparison experiment → write the ADR
- Verify: a comparison table of 5 packet types across 2 methods, plus the ADR

**Session Management (10h)**
- What: state machine, timeout, heartbeat, duplicate connections, forced close, receive/send races during shutdown, reconnection (a preview of Phase 3)
- Why: Phase 3's game server user management is built on top of this
- How: `L2-C-12` (state machine) → `L2-C-13` (termination race) → `L2-C-17` (timeout)
- Verify: 5 illegal-transition tests + 1,000 clean runs of the termination race

**Server I/O Model Overview (6h)**
- What: thread-per-connection → select → completion notification (IOCP) → epoll/io_uring (concept), C10K, readiness notification vs. completion notification
- Why: you need to know which model your track's API is, and be able to compare it, so you can explain it in an interview
- How: `L2-C-06` (measure sync limits) → track exercises → write the comparison table
- Verify: a table comparing the 4 models in terms of "thread count, syscall count, memory at 10,000 connections"

**Load Testing Fundamentals (8h)**
- What: ramp-up, concurrent connections vs. connections per second, message rate, p50/p95/p99, loopback limits (port exhaustion, client-side bottleneck)
- Why: you must prove the "500 connections, 10 minutes" condition of 2-1 yourself
- How: `L2-C-20` client → `L2-C-21` termination storm → results table
- Verify: `LOAD-2-1.md`

### 4.2 C# Track (90h)

| Item | Time | What | Learning Order | Verification |
|---|---|---|---|---|
| Synchronous sockets | 8h | `Socket`, `TcpListener/TcpClient`, `NetworkStream`, blocking Send/Receive, thread-per-connection | L2-CS-01 → L2-C-06 | Observe thread count at 100 connections |
| Asynchronous sockets (TAP) | 12h | `AcceptAsync/ReceiveAsync/SendAsync`, `Memory<byte>`, cancellation, `SocketException` codes | L2-CS-02 | Compare thread count at 1,000 connections |
| SocketAsyncEventArgs | 16h | SAEA pooling, **handling synchronous completion**, pinned buffers, the `Completed` event, per-session receive/send SAEA | L2-CS-03 → L2-CS-04 | Test the synchronous-completion path, 0 unreturned |
| Send queue | 8h | No concurrent Send, per-session queue + in-progress flag, send coalescing, backpressure | L2-C-14 → L2-C-15 | Integrity under 100-thread concurrent Send |
| Implementing framing | 10h | Accumulation buffer, header parsing, partial packets, defense against oversized packets, using the 1-2 pool | L2-C-10 | 20 parser tests |
| System.IO.Pipelines 🟡 | 8h | `PipeReader/Writer`, `SequenceReader`, backpressure options | after 2-2 is done | Compare code size/performance against the Pipelines version |
| MemoryPack packets | 8h | `[MemoryPackable]`, source generators, versioning, splitting into a shared project | after L2-C-11 | Round-trip all 20 packet types |
| Chat server logic | 12h | Room management, broadcast (serialize once and share), whisper, timeout timer | L2-C-16 → L2-C-17 | 10 logic tests |
| Library comparison | 8h | Reading the structure of SuperSocketLite / FreeNetLite | L2-CS-05 | 4-row comparison table |

### 4.3 C++ Track (90h)

| Item | Time | What | Learning Order | Verification |
|---|---|---|---|---|
| Winsock basics | 8h | `WSAStartup`, `socket/bind/listen/accept/connect`, `sockaddr_in`, `getaddrinfo`, `WSAGetLastError` | L2-CPP-01 | Sync echo + error logging |
| Data transfer | 8h | Partial-send loop, byte order, socket options (`SO_RCVBUF`, `TCP_NODELAY`, `SO_LINGER`) | L2-C-07 → L2-C-09 | Handling the send return value for a 64KB transfer |
| Non-blocking and select | 10h | `FIONBIO`, `select`, the `fd_set` limit of 64, `WSAEWOULDBLOCK` | L2-CPP-02 | select server at 60 connections |
| Overlapped I/O | 8h | `WSARecv/WSASend` + `OVERLAPPED`, buffer lifetime rules | L2-CPP-03 | Break it with a stack OVERLAPPED, then fix it |
| IOCP | 20h | `CreateIoCompletionPort`, `GetQueuedCompletionStatus`, worker count, Per-Handle/Per-I/O, `AcceptEx`, zero-byte recv | L2-CPP-04 | Sustain 1,000 connections, 0 handle leaks |
| Session lifetime management | 12h | Reference counting, termination race, `shutdown` vs. `closesocket`, session pool | L2-CPP-05 → L2-C-13 | Pass a termination storm under ASan |
| Send buffer chaining | 8h | Per-session send queue, guaranteeing only 1 in flight, `WSABUF` arrays, backpressure | L2-C-14 | Integrity under concurrent sends |
| Protocol and serialization | 8h | Header parsing, ring buffer, FlatBuffers/Protobuf (vcpkg) or manual, handler dispatch | L2-C-10 → L2-C-11 | 20 parser tests |
| Comparative reading 🟡 | 8h | Mapping Boost.Asio's io_context/strand onto IOCP | after L2-CPP-06 | "IOCP by hand vs. Asio" comparison table |

---

## 5. Textbook Usage Guide (Phase 2)

### 5.1 Common

| Textbook | Chapters to Read | When | How to Use |
|---|---|---|---|
| Game Server Development, Starting with Understanding Networks | All (5 chapters) | Days 16-17 | Use it like a glossary. Check chapter 1's terms with a daily 5-question AI quiz. Chapter 5 (blocking/non-blocking, I/O multiplexing, sync/async) is the foundation for the "Server I/O Model Overview" |
| Network Theory Every Game Server Developer Should Know | All | Day 17 | Cross-check each item against official documentation (RFCs, etc.), and wherever possible reproduce it with server logs/`netstat`/`Get-NetTCPConnection` and write it in your notes. **Find at least one place that differs from the book** is this book's assignment |
| Network Learning Roadmap for Online Game Developers | Ch.1 (networking fundamentals), Ch.2 (transport protocols in depth — TCP/UDP selection, reliable UDP), Ch.7 (application protocols, serialization) | Days 21-23 | Only the assigned chapters. Ch.2's "TCP vs. UDP selection criteria" is a favorite in explanation exams. Use Ch.7's serialization comparison as grounding for the 2-C ADR. Chapters 3-6 and 8 belong to Phases 3, 5, and 6 |

### 5.2 C# Track

| Textbook | Chapters to Read | When | How to Use |
|---|---|---|---|
| C# Socket Programming for Game Server Development | Ch.1-4 (overview, basics, sync, async), Ch.5 (high performance), Ch.6 (data handling, framing), Ch.7 (library design), Ch.9 (chat server), Ch.10 (SAEA in depth) | Days 18-29 | **Main textbook.** Upgrade `codes/Chapter7`, `codes/Chapter9`, `codes/Chapter10` to .NET 10, build them, and compare the NetServerLib structure against your own 2-1 design in a table. Chapter 8 belongs to Phase 5, Chapter 11 (Omok) to Phase 3 |
| C# Socket Programming — FreeNetLite.md | All | Days 32-34 | Build the EchoServer/test_client from `codes/FreeNetLiteRe` and connect it to your own 2-3 client |
| C# Game Server Programming Using SuperSocketLite | Ch.1-6 | Day 34 | For comparing "a server you built yourself vs. a server built on a library." Chapter 5's test client is reference material for the 2-3 design; apply Chapter 6's MemoryPack packet definitions to 2-2 |
| Building a C# async/await Library | Ch.7 (Socket async patterns) | Day 24 | Read how it wraps SAEA in `ValueTask`, and write one ADR on why you chose TAP or EAP |

### 5.3 C++ Track

| Textbook | Chapters to Read | When | How to Use |
|---|---|---|---|
| TCP/IP Windows Socket Programming Every Game Server Developer Should Know | Ch.1-5 (sockets, addresses, server/client, data transfer), Ch.6 (multithreading), Ch.8 (socket options), Ch.9 (I/O models), Ch.10 (select-based chat), Ch.11 (IOCP-based chat), Ch.13 (buffer pooling) | Days 18-30 | **Main textbook.** Build and run `codes/tcp_server_01~03`, `first_IOCP`, `SelectChatServer`, `IocpNetLib` in order, and cross-check the code against the PlantUML sequence diagrams. Do every "exercise" section in chapters 5-6. **After understanding the book's IOCP structure, rebuild your own server from an empty project** (no copying) |
| Modern Windows Multithreading | Ch.6 (Windows Thread Pool API) | Day 29 | Compare building your own IOCP worker versus delegating to a thread pool. Chapter 11 belongs to Phase 3 |
| Building Online Game Servers with C++ Boost.Asio 🟡 | Ch.1-2, 4-6, 7 | Day 34 | Read this after building IOCP yourself, so you can see what Asio abstracts away. The required path is implementing IOCP directly |
| Practical WinRT: Networking and Multithreaded Programming (reference) | Ch.17 (protocol design), Ch.19 (framing), Ch.20 (session management) | Reference for days 21-26 | The API (WinRT) isn't used, but the design chapters are API-independent. Use as a checklist for the 2-C spec |

### 5.4 Cross-Track Reading Assignment

- C# main track: TCP/IP Windows Socket Programming chapter 9 (I/O models) → **write 1 page on "how does SAEA operate on top of IOCP"**
- C++ main track: C# Socket Programming chapter 10 (SAEA) → **write 1 page on "how did .NET wrap IOCP"**

---

## 6. AI Collaboration Guide (Phase 2-specific)

### 6.1 Prompts

**Interpreting server logs**
```
Here is my server's receive-callback log when receiving a 64KB packet: (bytes per recv, timestamps)
1) Explain why the application couldn't receive it all in a single recv from this log.
2) My handling approach is this: "(my explanation)." Point out the holes in this approach.
3) Suggest 3 test cases that reproduce this phenomenon (case descriptions only, no code).
```

**Step-by-step implementation (C++ IOCP)**
```
I want to build an IOCP echo server. Don't give me the whole thing at once — let's split it into steps.
Step 1: listening socket + IOCP creation + N worker thread skeletons (accept is still synchronous for now)
Keep each step's code under 60 lines, with a one-line comment on "why it's needed" for every new Win32 API.
Don't move on to step 2 until I say "next."
If I try to explain step 1's code myself first, only point out what's wrong.
```

**Session-termination-race review**
```
Here is my session-termination code. (paste it in)
Simulate the scenario "another thread closes this session while the receive-completion callback is running" in chronological order (T1, T2, ...),
and if there's an ordering that causes a crash, double-free, or leak, show that ordering in a table. Don't give me a fix yet.
```

**Bottleneck hypotheses**
```
If I scale connections from 100 to 1,000, what do you think will break first in my server?
Give me 3 hypotheses and, for each, a way to check it (counters, logs, tools).
Don't jump to a conclusion yet. I'll come back with results after measuring.
```

**Protocol spec review**
```
Here is my chat protocol specification. (paste PROTOCOL.md)
You are a developer on a different team who has to implement a client using only this spec.
1) Point out every part where you'd have to guess (anything ambiguous).
2) Flag any fields a malicious client could abuse.
```

### 6.2 Fault-Injection Topics (This Phase)

| Area | Fault to Plant |
|---|---|
| Framing | missing length-field validation (negative/huge values), missing partial-packet handling, offset calculation errors (off-by-one) |
| Session | send-after-close, reference-count mismatch, socket handle leaks, missing state checks |
| Send | allowing concurrent Send, unbounded send-queue growth, failing to send the next item from the send-completion callback |
| C# | unhandled SAEA synchronous completion, missing `ArrayPool` returns, duplicate event handler registration |
| C++ | freeing OVERLAPPED/buffers before completion, `WSABUF` lifetime issues, missing worker-thread shutdown signal |

### 6.3 Where AI Often Gets It Wrong

- If `Socket.ReceiveAsync(SAEA)` returns `false`, it completed **synchronously** and the `Completed` event never fires — code that misses this handling is common
- In IOCP, `lpOverlapped` can still be valid even when `GetQueuedCompletionStatus` fails (cleanup is required)
- The mistaken claim that "TCP delivers data packet by packet" → refute it with the `L2-C-07/08` logs
- Misdiagnosing a loopback 500-connection failure as a server problem → check client dynamic port exhaustion first
- Suggesting an unbounded `Channel`/queue capacity → demand a backpressure policy
- Code that serializes N times for N broadcast recipients → demand serializing once and sharing the buffer (with reference counting)

---

## 7. Assignment Specifications

### 7.1 Common Assignment 2-C. Packet Protocol Specification (required, 8h, days 16, 20, 23)

**Goal**: produce a protocol document that will be extended throughout every later Phase. 30 game packet types get added to it in Phase 3.

**Requirement 1 — Header structure** (fill in the table below and put it in `PROTOCOL.md`. You may change the values to your own design, but justify them)

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0 | 2 | `TotalLength` | uint16 | Total length including the header. Max 65,535 |
| 2 | 2 | `PacketId` | uint16 | Packet type |
| 4 | 1 | `Version` | uint8 | Protocol version. Reject on mismatch |
| 5 | 1 | `Flags` | uint8 | bit0 compression, bit1 encryption (unused this Phase) |
| 6 | 2 | `Reserved` | uint16 | For alignment, fixed at 0 |
| 8 | N | `Body` | bytes | Serialized payload |

- Endianness: **fixed little-endian** (x86-oriented, avoids conversion cost) or network byte order — pick one and justify it
- Maximum packet size: 64KB (close the session immediately if exceeded)
- Minimum packet size: the header size (8). Error if `TotalLength < 8`

**Requirement 2 — Packet ID scheme**

| Range | Purpose | Examples |
|---|---|---|
| 1000-1999 | System | heartbeat, version check, error notification |
| 2000-2999 | Auth/session | set nickname, login (extended in Phase 4) |
| 3000-3999 | Lobby/rooms | room create/join/leave/list |
| 4000-4999 | Chat | room chat, whisper, announcements |
| 5000-5999 | Game | used starting in Phase 3 |

- A rule like "requests are even, responses are request+1" makes debugging easier (optional)

**Requirement 3 — 20+ packet types** (the list below is the minimum. Attach a field table to each packet)

| ID | Name | Direction | Fields | Allowed State |
|---|---|---|---|---|
| 1000 | HeartbeatReq | C→S | (none) | Named, Active |
| 1001 | HeartbeatRes | S→C | serverTimeMs:int64 | - |
| 1002 | VersionCheckReq | C→S | clientVersion:uint8 | Connected |
| 1003 | VersionCheckRes | S→C | result:uint16 | - |
| 1010 | ErrorNotify | S→C | errorCode:uint16, message:string | - |
| 2000 | SetNameReq | C→S | name:string(2-16) | Connected |
| 2001 | SetNameRes | S→C | result:uint16, userId:int32 | - |
| 2010 | DisconnectNotify | S→C | reason:uint16 | - |
| 3000 | RoomListReq | C→S | page:uint16 | Named |
| 3001 | RoomListRes | S→C | rooms:[{id,name,count,max}] | - |
| 3002 | RoomCreateReq | C→S | name:string, maxCount:uint16 | Named |
| 3003 | RoomCreateRes | S→C | result:uint16, roomId:int32 | - |
| 3004 | RoomEnterReq | C→S | roomId:int32 | Named |
| 3005 | RoomEnterRes | S→C | result:uint16, members:[{userId,name}] | - |
| 3006 | RoomEnterNotify | S→C | userId:int32, name:string | - |
| 3007 | RoomLeaveReq | C→S | (none) | InRoom |
| 3008 | RoomLeaveRes | S→C | result:uint16 | - |
| 3009 | RoomLeaveNotify | S→C | userId:int32 | - |
| 4000 | ChatReq | C→S | text:string(≤200) | InRoom |
| 4001 | ChatRes | S→C | result:uint16 | - |
| 4002 | ChatNotify | S→C | userId:int32, name:string, text:string, seq:int64 | - |
| 4010 | WhisperReq | C→S | targetName:string, text:string | Named |
| 4011 | WhisperRes | S→C | result:uint16 | - |
| 4012 | WhisperNotify | S→C | fromName:string, text:string | - |

**Requirement 4 — Session state machine** (draw it in Mermaid)

```
Connected --SetNameReq(success)--> Named
Named --RoomEnterReq(success)--> InRoom
InRoom --RoomLeaveReq--> Named
any --heartbeat timeout/error/disconnect request--> Closing --> Closed
```
- For each state, write a table of the **allowed packet IDs**. On receiving a disallowed packet: log it + send `ErrorNotify` + close the session (state the policy explicitly)

**Requirement 5 — Rules**
- Heartbeat: client sends every 10 seconds; server closes after 30 seconds with no response since the last receipt
- Version mismatch: send `VersionCheckRes(result=VersionMismatch)` then close immediately
- Error code table: `0=Success, 1=InvalidPacket, 2=InvalidState, 3=NameDuplicated, 4=RoomFull, 5=RoomNotFound, 6=VersionMismatch, 7=RateLimited, ...`

**Requirement 6 — ADR**: `adr/0001-serialization.md` — choice of serialization method (context, decision, 3 alternatives, reasons for rejection, consequences). Template is T5 in `08-templates.md`

**Deliverables**: `PROTOCOL.md`, `adr/0001-serialization.md`

**Grading**

| Item | Points | Criterion |
|---|---|---|
| Completeness | 40 | header table, ID scheme, 20 packet types, state machine, error codes, heartbeat rules all present |
| No ambiguity | 40 | every ambiguous item flagged in the AI "developer from another team" review (§6.1) has been resolved |
| State machine | 20 | per-state allowed-packet table and the illegal-transition handling policy are explicit |

### 7.2 Track Assignment 2-1. Async Echo Server → Chat Server (required, 48h)

**Stage breakdown**

| Stage | Timing | Content | Deliverable |
|---|---|---|---|
| 2-1a | Days 18-19 | Synchronous echo (thread-per-connection), measuring limits at 100/500/1,000 connections | `EchoServer/` (sync), measurement table |
| 2-1b | Days 24-25 | Async echo (C# SAEA / C++ IOCP), sustaining 1,000 connections | `EchoServer/` (async), `COMPARE-2-1.md` |
| 2-1c | Days 26-30 | Chat server (rooms, whisper, heartbeat, graceful shutdown) | `ChatServer/` |

**2-1c Functional Requirements**

1. **Nickname**: force-close if `SetNameReq` isn't received within 30 seconds of connecting. Reject duplicate nicknames (`NameDuplicated`)
2. **Rooms**: create/join/leave/list (paginated), 100-member room cap, no room owner (kept simple), empty rooms auto-destroy
3. **Chat**: room broadcast (including the sender), whisper (by nickname), 200-character message limit, **`RateLimited` if more than 5 per second**
4. **Heartbeat**: server closes a session after 30 seconds with no response. Check interval 5 seconds
5. **Error handling**: on a corrupted packet (length/ID/state mismatch), log it and close only that session. The server survives
6. **Graceful shutdown**: on Ctrl+C, ① block new connections ② broadcast `DisconnectNotify` ③ wait for sends to finish (up to 2 seconds) ④ clean up sockets ⑤ join threads. Complete within 5 seconds
7. **Config file**: port, worker thread count, receive buffer size, max connections, timeouts, log level. C# uses `appsettings.json`, C++ uses TOML/JSON

**Non-Functional Requirements (proven by measurement)**

| Condition | Criterion |
|---|---|
| 500 concurrent connections, 1 message/sec per connection (50B), 10 minutes | 0 message loss, 0 crashes |
| Memory | growth from the 1-minute mark to the 10-minute mark within 10% |
| Latency | p99 chat round-trip (time for your own message to come back via broadcast) ≤ 100ms |
| Termination storm | 500 connections disconnecting simultaneously × 5 rounds, 0 crashes, handle/session/pool counters return to normal |
| C++ | pass the above scenario once with an ASan build (measure performance in Release) |

**Structural Requirements**

```
ChatServer/
├─ Net/      sessions, send/receive, framing integration, session manager   ← references PacketLib
├─ Logic/    users, rooms, chat rules                                      ← does not reference Net (sends via an interface)
├─ Host/     entry point, config loading, shutdown handling
└─ tests/    Logic unit tests + Net integration tests
```
- `Logic` only knows about an interface like `ISessionSender { void Send(int sessionId, ReadOnlySpan<byte>); void Close(int sessionId); }` → substitute a fake in tests
- Use the **1-2 object pool** for receive/send buffers
- Prevent concurrent sends with a per-session send queue

**Testing Requirements**

- Unit (Logic, no network): 5 for room join/leave, 3 for broadcast, 3 for whisper, 5 for state transitions, 2 for rate limiting — 18 minimum
- Integration (using the 2-3 client): 100-connection, 5-minute chat scenario, storm scenario, 3 corrupted-packet types

**LOAD-2-1.md Template**

```markdown
# 2-1 Load Report
## 1. Environment
CPU/RAM/OS, whether server and client are on the same machine, build configuration, commit hash, dynamic port range setting
## 2. Scenario
connection count / ramp-up / message interval / duration / repetitions
## 3. Results
| Round | Connections Succeeded | Loss | p50 | p95 | p99 | CPU% | Memory (1min/10min) |
## 4. Termination Storm
| Round | Crash | Handle Count (before/after) | Session Count (after) | Unreturned Pool |
## 5. Bottleneck Analysis
Hypothesis → observation method → result → adopted/rejected (at least 3, including at least 1 rejected)
## 6. Fix History and Re-Measurement Results (before/after table)
## 7. Remaining Limitations and Next-Phase Tasks
```

**Grading**

| Item | Points | Criterion |
|---|---|---|
| Functionality | 25 | all of requirements 1-7, including corrupted-packet defense |
| Stability | 30 | 500 connections, 10 minutes, no loss, memory within 10%, 5 termination-storm rounds |
| Concurrency/Resources | 20 | AI review score of 4+ on rubric items 2-3, C++ passes ASan, 0 unreturned |
| Structure | 15 | Logic does not reference Net (proven by build), uses pools, has a send queue |
| Documentation | 10 | `LOAD-2-1.md`, `COMPARE-2-1.md`, config explanation |

**Common Mistakes**
- Serializing once per recipient on broadcast → serialize once and share the buffer (reference counting/copy)
- Doing a full scan of the room list every time → paginate + cache
- Writing logs synchronously to a file → reuse Phase 1's log queue
- Removing a session from the dictionary without closing its socket → handle leak

### 7.3 Track Assignment 2-2. Packet Framing/Serialization Library (required, 24h, days 22-23)

**Goal**: implement the 2-C spec as a library, to be reused in 2-1 and Phases 3-6. Must be an **independent project**.

**API (C#)**
```csharp
public enum ParseResult { Ok, NeedMore, TooLarge, BadLength, BadVersion }

public sealed class PacketReader
{
    public PacketReader(int maxPacketSize = 65535, BufferPool? pool = null);
    // Accumulate a chunk read from the socket
    public void Feed(ReadOnlySpan<byte> chunk);
    // Pop one completed packet. If Ok, packet holds a header+body slice
    public ParseResult TryRead(out PacketView packet);
    public int BufferedBytes { get; }
    public void Reset();
}

public readonly ref struct PacketView
{
    public ushort Id { get; }
    public byte Version { get; }
    public ReadOnlySpan<byte> Body { get; }
}

public static class PacketWriter
{
    // Fill in the header and return a pooled buffer holding the serialized body (caller is responsible for returning it)
    public static PooledBuffer Write<T>(ushort id, in T body) where T : IPacketBody;
}
```

**API (C++)**
```cpp
enum class ParseResult { Ok, NeedMore, TooLarge, BadLength, BadVersion };

struct PacketView {
    std::uint16_t id;
    std::uint8_t  version;
    std::span<const std::byte> body;
};

class PacketReader {
public:
    explicit PacketReader(std::size_t max_packet_size = 65535);
    void Feed(std::span<const std::byte> chunk);
    ParseResult TryRead(PacketView& out);
    std::size_t BufferedBytes() const noexcept;
    void Reset();
};
```

**Requirements**
1. Report errors via **return values, not exceptions** (the caller decides whether to close the session)
2. `NeedMore` is normal flow (a partial packet). Do not log it
3. Exceeding the maximum size or an invalid length field must immediately return `TooLarge`/`BadLength` → the caller closes the session
4. Rent buffers from the 1-2 pool, and document who's responsible for returning them
5. Serialization: C# MemoryPack (or MessagePack) / C++ manual or FlatBuffers. Define **all 20 packet types from 2-C**
6. Minimize allocation: aim for 0 heap allocations per packet on the normal path (prove it in C# with `MemoryDiagnoser`)

**Test List (20 tests, all required)**

| # | Name | Input | Expected |
|---|---|---|---|
| 1 | CompletePacketOnce | exactly 1 packet | Ok, 0 remaining |
| 2 | SplitByteByByte | Feed 1 byte at a time | Ok on the last byte |
| 3 | HeaderOnlyArrived | 8 bytes | NeedMore |
| 4 | HeaderMinusOneByte | 7 bytes | NeedMore |
| 5 | BodyIncomplete | header + half the body | NeedMore |
| 6 | TwoPacketsMerged | 2 packets back to back | Ok twice, 0 remaining |
| 7 | ThreeAndAHalfPackets | 3.5 packets | Ok 3 times, then NeedMore |
| 8 | ExactBoundary | exactly the buffer size | Ok |
| 9 | ZeroLength | TotalLength=0 | BadLength |
| 10 | LengthLessThanHeader | TotalLength=5 | BadLength |
| 11 | LengthExceeded | TotalLength=70000 | TooLarge |
| 12 | VersionMismatch | Version=99 | BadVersion |
| 13 | UndefinedId | Id=9999 | Ok (the parser passes it through; the caller decides) |
| 14 | MaxSizePacket | exactly 64KB | Ok |
| 15 | Consecutive1000Packets | 1,000 in a row | all Ok, order preserved |
| 16 | ReuseAfterReset | Reset after an error | parsing resumes normally |
| 17 | RoundTripChat | serialize→deserialize ChatNotify | fields match |
| 18 | RoundTripRoomList | includes a variable-length array | fields match |
| 19 | Utf8String | 200 Korean characters | no corruption, length validated |
| 20 | ZeroAllocationHappyPath | parse 10,000 packets | (C#) 0 allocations or constant |

**Deliverables**: the library project, 20 tests, a usage example (10 lines), README (usage, buffer-return rules)

**Grading**: correctness (20 tests) 50 / defensive logic (error codes, caps) 20 / API design (ease of use, allocation) 30

### 7.4 Common Assignment 2-3. Test Client (required, 16h, days 31, 33)

**Goal**: a tool that reproduces load and scenarios. Extended with game scenarios in Phase 3, and into a production-grade tool in Phase 5.

**CLI Interface**
```
LoadClient --host 127.0.0.1 --port 9000 --conn 500 --rampup 50 \
           --scenario chat --duration 600 --out results/run1.json
```

| Argument | Meaning | Default |
|---|---|---|
| `--conn` | target concurrent connection count | 100 |
| `--rampup` | connections created per second | 20 |
| `--scenario` | `chat` \| `storm` \| `garbage` | chat |
| `--duration` | steady-state hold time (seconds) | 300 |
| `--msg-interval` | message interval (ms) | 1000 |
| `--rooms` | number of rooms to spread across | conn/50 |
| `--out` | results file path (JSON) | stdout |

**Scenarios**
- `chat`: connect → `SetNameReq` → join a room (spread out) → chat periodically → leave and disconnect cleanly on exit
- `storm`: acquire the target connection count, then disconnect **all at once** (with an option for the normal-close/forced-close ratio)
- `garbage`: send corrupted packets (invalid length, undefined ID, state violation) and confirm the server survives

**Statistics Requirements**
- Connections: attempted/succeeded/failed (by error code)
- Messages: sent/received/lost (sequence-based), throughput per second
- Latency: time for your own message to come back via broadcast, p50/p95/p99/max — **histogram-based computation recommended**
- Count of detected server disconnects, count by exception type
- Console summary every 5 seconds, save a JSON + markdown table file on exit

**Accuracy Requirement**: a unit test verifying the p99 computation against a known dataset (1-1000) is required

**Deliverables**: code, 2 result files from a 500-connection run (chat/storm), usage README

**Grading**: scenarios work 40 / statistics accuracy (including the p99 verification test) 30 / usability (arguments, output, reproducibility) 30

### 7.5 Cross-Cutting Assignment 2-4. Cross-Language Interoperability (advanced 🟡, 8h, day 34)

- Connect your own 2-3 client to a reference echo server in the other language (C# track uses the `IocpNetLib` Echo, C++ track uses the `FreeNetLiteRe` EchoServer)
- If the header formats differ, build a **protocol adapter** in the client (converting between your protocol and the other one)
- Deliverables: communication logs (request/response timestamps), a 1-page adapter description, "3 differences in header design between the two implementations"

---

## 8. Learning Completion Assessment (Friday, Day 35)

### 8.1 Checklist

**Assignments**
- [ ] 2-C: `PROTOCOL.md` (20 packet types, state machine, error codes) + 1 ADR
- [ ] 2-1a/b/c complete, all 7 chat features working
- [ ] 2-2: 20 tests pass, split out as an independent library
- [ ] 2-3: chat/storm/garbage scenarios all work, p99 verification test passes
- [ ] 🟡 2-4 done, or the reason it wasn't recorded

**Measurements**
- [ ] `LOAD-2-1.md`: 500 connections, 10 minutes, no loss, memory within 10%, p99 recorded
- [ ] Results of 5 termination-storm rounds (handle/session/pool counters returned to normal)
- [ ] `COMPARE-2-1.md`: sync vs. async (thread count, memory, success rate)

**Observation Records**
- [ ] TCP state transition table, comparison table of server/client closing first
- [ ] Split-receive log, merge-receive log, Nagle on/off comparison table

**Records**
- [ ] 20 days of learning notes, 4 weekly retrospectives

### 8.2 Reimplementation Exam (No AI, 90 minutes)

**Problem**: implement a framing parser from an empty project and pass the 10 provided tests below.
The header is `[TotalLength:uint16][PacketId:uint16][Version:uint8][Flags:uint8][Reserved:uint16]` = 8 bytes, little-endian, max 64KB.

Provided tests: 10 of #1-#12 from the §7.3 test list (1, 2, 3, 5, 6, 7, 9, 10, 11, 12)

**Pass criterion**: pass all 10 within 90 minutes.

### 8.3 Explanation Exam Question Bank (10 questions at random, average 4.0+)

**Common**
1. What is the server socket's state at each stage of the 3-way handshake? Which stage does SYN flooding attack?
2. Who holds TIME_WAIT and why? What problem arises if the server closes first?
3. How do you handle, in code, the fact that TCP has no message boundaries? What are the pitfalls of the length-prefix approach?
4. Why turn off Nagle, and when should you not turn it off?
5. Explain the difference between IOCP (completion notification) and epoll (readiness notification) in terms of the recv flow
6. Explain, using your own measurements, why thread-per-connection breaks down at 1,000 connections
7. What happens without backpressure? What's your own server's policy?
8. What must you be careful of when broadcasting to 100 people?
9. What are the steps of graceful shutdown, and what problems arise if the order changes?
10. Why close only the affected session, rather than the whole server, when a corrupted packet is received?
11. How did you decide on the heartbeat interval and timeout value?
12. Name 3 limitations of loopback load testing

**C#**
13. Why reuse SAEA? What must you do when `ReceiveAsync` returns false?
14. Why is a per-session send queue necessary, and how do you prevent unbounded queue growth?
15. What to watch out for when using `ArrayPool` for socket buffers (when to return, sizing)
16. Which did you choose between TAP and EAP (SAEA), and why?
17. How do you split the use of `Memory<byte>` and `Span<byte>` in asynchronous code?

**C++**
18. How do you decide the number of IOCP worker threads? What happens if it's more than the core count?
19. The lifetime rules for OVERLAPPED and buffers. What happens if you free them before completion?
20. Why is session reference counting necessary? Explain the sequence when recv completion and close happen at the same time
21. Why call `closesocket` after `shutdown(SD_SEND)`?
22. What are the advantages of `AcceptEx` over `accept`, and what should you watch out for?

### 8.4 Bug Hunt (45 minutes)

In code with 5 planted defects on the topics of framing, session termination, and the send queue, find 4 or more and propose fixes (using the topic list from §6.2).

### 8.5 Response to Falling Short

| Shortfall | Response |
|---|---|
| Failed to meet the 500-connection condition | 8h of remediation in the first week of Phase 3, then re-measure. If still short, lower the bar to 300 connections, but **a root-cause analysis document is mandatory** |
| Reimplementation exceeded 90 minutes | practice the parser 3 more times in Phase 3 week 1 (60 minutes each) |
| Explanation exam score below 3.5 | 8h of remediation on the 2 weakest topics in Phase 3 week 1, then retest |
| Failed 2-2 tests | **cannot pass**. Since the Phase 3 server depends on this library, complete it as top priority |
| Termination-storm crash | **cannot pass**. Redesign session lifetime (introduce reference counting) |

---

## 9. Common Sticking Points

| Symptom | Cause | Countermeasure |
|---|---|---|
| Client connection failures at 500 loopback connections | dynamic port exhaustion, TIME_WAIT buildup | `netsh int ipv4 set dynamicport tcp start=10000 num=55000`, spread the client across 2 processes, wait 30 seconds between test runs |
| Firewall popups / cannot connect externally | Windows Defender Firewall | add an inbound rule (specifying the port), test on the private network profile |
| C# server at 100% CPU | handling SAEA synchronous completion via recursion instead of a loop, or not treating a 0-byte receive as a close | treat 0 bytes as a normal close, handle synchronous completion with a while loop |
| Intermittent C++ IOCP crashes | freeing the session/buffer before completion, reference-count errors | add reference-count logging, reproduce with an ASan build, unify the shutdown path |
| Memory slowly growing | unreturned SAEA/buffers, lingering session dictionary entries, send-queue buildup | log unreturned-pool count, session count, and queue length every 5 seconds |
| Messages arriving interleaved | concurrent Send | send queue + a single-in-flight flag |
| One specific client stops responding | the send queue is stuck (slow receiver), backpressure not implemented | apply a queue cap and policy, close the connection when the cap is exceeded |
| Only chat p99 latency spikes | synchronous log writes, GC, repeated broadcast serialization | use the log queue, serialize once and share, profile allocations |
| Process won't die on shutdown | worker threads are blocked waiting | for IOCP, send a shutdown signal per worker via `PostQueuedCompletionStatus`; for C#, use a cancellation token + Complete |
| Parser occasionally emits garbage packets | offset calculation errors, missing reset on buffer reuse | add boundary tests (#4, #8), validate slice ranges |
| Test client dies before the server | client CPU/port limits | spread across 2 machines (or processes), adjust the message interval |
| `Get-NetTCPConnection` is slow | thousands of connections | use the `-LocalPort` filter, increase the sampling interval |

---

## 10. Preparing to Move on to Phase 3

- [ ] Organize `PacketLib` (2-2) and `LoadClient` (2-3) into shared projects and tag a version (`v0.2`)
- [ ] Mark the 5000-range (game) area reserved for Phase 3 in `PROTOCOL.md`
- [ ] Clone chapters 7-12 of the textbook "C# Design Patterns for Game Developers" (Command, Observer, State, Strategy, Game Loop, Update Method)
- [ ] Read ahead through templates T4 (design document) and T5 (ADR) in `08-templates.md`, and create an empty `DESIGN.md` to use in Phase 3
- [ ] Write a 3-line note (from a thread-ownership perspective) on what needs to change when turning the chat server's "room" code into Phase 3's "room actor"
- [ ] Phase 2 retrospective: the longest-stuck bug and its cause, one case where AI was wrong, one habit to carry into Phase 3
