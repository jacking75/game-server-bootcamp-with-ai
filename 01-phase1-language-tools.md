# Phase 1. Language and Tooling Deep Dive for Server Developers (Weeks 1-3, 120h)

> 🇰🇷 Korean version: [01-phase1-language-tools_kr.md](01-phase1-language-tools_kr.md)

> Common 40h / Track 80h. By the end of this Phase, learners will go from someone who "knows the syntax" to someone who can "read, write, and measure server code."

## 0. How to Use This Document

This document is written so that, if you follow it as written, three weeks will be enough to finish. Read it in the following order.

1. Read the §1 overview and work through the §1.2 self-check first. If you fall short on any item, finish the remediation path in §1.2 before Week 1 begins.
2. Each morning, open the corresponding day block in §2 and start from there. Each day follows the order **Morning (concepts) → Afternoon (exercises/assignment) → 1 Hour Without AI → Evening wrap-up**, ending with a **Definition of Done (DoD)** checklist. If you can't check off the DoD, spend 30 minutes the next morning catching up.
3. The §3 exercise catalog is referenced by number from the day blocks, like `L-C-03`. Each exercise has a Goal, Steps, Expected Result, and Pass Criterion — carry it out exactly as written.
4. The §7 assignment specifications are this Phase's deliverables. The assignments are distributed across the day blocks, so following §2 will complete them naturally.
5. Administer the §8 completion assessment to yourself on the Friday of Week 3.

Notation

| Notation | Meaning |
|---|---|
| `L-C-xx` | Common exercise (Lab, Common). Everyone does this regardless of track |
| `L-CS-xx` | C# track exercise |
| `L-CPP-xx` | C++ track exercise |
| `1-C`, `1-1`, `1-2`, `1-3` | Assignment. 1-C is common; the rest are track assignments |
| **DoD** | Definition of Done. That day's completion condition. Actually check the checkboxes |
| 🔴 | Mandatory item — skipping it blocks later Phases |
| 🟡 | Item that can be postponed to the weekend or the following week if time is short |

Environment assumptions (same as Overview §1)

- Windows 11, **Visual Studio 2026**, **.NET 10 SDK**, PowerShell 7, Git for Windows
- The C++ track uses MSVC + CMake 3.28+ + vcpkg (manifest mode)
- The DB will be used starting in Phase 4, but **just confirm the installs this week**: SQLite (default, no installation needed) / (optional) MySQL 8 / Redis via redis-windows
- CI is not used. Instead, create `scripts/build-and-test.ps1` and **run it yourself before every commit**
- Claude Code is used as the default example coding agent, but other tools such as Codex CLI are fine too. Write the instruction file for whichever tool you use — `CLAUDE.md` or `AGENTS.md`

---

## 1. Overview

### 1.1 Goals

The goal of this Phase is not "to learn more of the language" but to **build a server developer's basic hygiene into your body**. Concretely, that means the following four things.

1. **Being able to reproduce and fix concurrency issues in code.** Produce a race condition, a deadlock, and a memory-visibility problem each in 30 lines of code or fewer, and be able to fix each in at least two different ways.
2. **Being able to see memory as numbers.** For C#, be able to check GC generations, allocated bytes, and the LOH with tools; for C++, be able to check ownership, lifetime, and leaks with tools.
3. **Being able to measure.** Use BenchmarkDotNet / Google Benchmark to take a Release-build measurement with proper warmup, repetition, and variance, and use those numbers to claim "A is faster than B."
4. **Tool hygiene.** Get comfortable with Git branching and conflict resolution, unit testing, the debugger (Parallel Stacks, conditional breakpoints), and local build/test scripts.

On top of this, make the **AI collaboration rules** (Overview §2) into a habit. In particular, this Phase is where "no commit if you can't explain it" and "1 Hour Without AI every day" get built into your routine. It doesn't matter whether your coding agent is Claude Code or Codex CLI. What matters is the **procedure of writing the instruction file yourself and verifying the generated code**.

### 1.2 Prerequisite Self-Check (30 minutes before Week 1 starts)

Work through the 10 questions below **without AI**. Write the answers as code or on paper. If you get fewer than 6, do the remediation path first.

| # | Question | Pass Criterion |
|---|---|---|
| 1 | Explain the difference between a class and a struct from a memory perspective | Mentions stack/heap, value/reference copy |
| 2 | What happens internally when you put 10,000 items into a `List<T>` (C#) or `std::vector<T>` (C++) | Mentions capacity growth, reallocation, copying |
| 3 | What happens to the stack when an exception is thrown | Stack unwinding, destructor/finally execution |
| 4 | Write code that writes to a file one line at a time, 10,000 times | Includes resource cleanup (using / RAII) |
| 5 | What is the average and worst-case lookup complexity of a dictionary (hash map) | O(1) / O(n), hash collisions |
| 6 | Give two reasons to use an interface (abstract class) | Substitutability, testability |
| 7 | Give one example of turning a recursive function into a loop | Mentions the difference in stack usage |
| 8 | Why is concatenating a string 10,000 times slow | Immutable strings/reallocation, StringBuilder or reserve |
| 9 | Why is dereferencing a null reference (`nullptr`) dangerous | Undefined behavior / exception |
| 10 | Have you ever written a unit test | Pass if yes; if no, do §3 `L-C-04` first |

Remediation path

- **Weak on C#**: speed-read the entire textbook "The Essential C# Guide for ASP.NET Core Web API" (1 day). Mark only the sections you don't know and type out the examples by hand
- **Weak on C++**: read only the weak parts of chapters 1-17 of the textbook "Safe and Easy C++ Programming, Starting with Modern C++" (1-2 days). Chapter 18 onward (Siv3D) is unrelated to this course
- **No testing experience**: move `L-C-04` (your first xUnit/GoogleTest test) up to Tuesday morning of Week 1

### 1.3 What You'll Be Able to Do After This Phase (in Verifiable Form)

- Implement a **thread-safe log queue in 60 minutes** from an empty project and pass the 3 provided tests
- Deliberately create a deadlock and **point out, in the debugger's Parallel Stacks window, exactly where the two threads are waiting on each other**
- Compare `Debug` and `Release` measurements and explain, with numbers, "why a Debug measurement is meaningless"
- C#: explain, by looking at decompiled output, how a single `await` gets turned into a compiler-generated state machine
- C++: explain the cost difference between `unique_ptr`/`shared_ptr`, and have a track record of fixing a use-after-free that ASan caught

### 1.4 Deliverables of This Phase

```
gameserver-course-<name>/
├─ README.md                     Course intro, reason for track choice, progress table
├─ CLAUDE.md (or AGENTS.md)      agent instruction file
├─ .editorconfig
├─ .gitignore
├─ scripts/
│  └─ build-and-test.ps1         builds + tests in one shot
├─ notes/
│  ├─ daily/2026-mm-dd.md        15 files
│  └─ weekly/W01.md ~ W03.md     3 files
└─ phase1/
   ├─ LogQueue/                  Assignment 1-1 (track language)
   │  ├─ src/  tests/  bench/
   │  └─ REPORT-1-1.md
   ├─ ObjectPool/                Assignment 1-2
   │  ├─ src/  tests/  bench/
   │  └─ REPORT-1-2.md
   ├─ FileTool/                  Assignment 1-3 (advanced, optional)
   └─ labs/                      lab exercise outputs (one folder per exercise number)
      ├─ L-C-05-deadlock/        deadlock reproduction + Parallel Stacks screenshot
      ├─ L-C-06-race/
      └─ ...
```

### 1.5 3-Week Roadmap at a Glance

| Week | Main Topic | Common | C# Track | C++ Track | Assignment Progress |
|---|---|---|---|---|---|
| Week 1 | Environment, tools, memory | Environment setup, Git, unit testing, instruction file, OS basics | .NET runtime and GC, value/reference, Span, ArrayPool | RAII, smart pointers, move semantics, vcpkg/CMake, ASan | 1-C complete, 1-2 started |
| Week 2 | Concurrency | Race conditions, deadlocks, visibility, debugger deep dive | async/await internals, synchronization tools, Channel | thread/mutex/CV/atomic, Win32 SRWLock | Both versions of 1-1 implemented |
| Week 3 | Measurement and deep dives | Benchmark methodology, report writing, evaluation | ThreadPool starvation, 12 async pitfalls | lock-free SPSC, false sharing, UB | 1-1 and 1-2 complete, 1-3 optional, evaluation |

Weekly time budget (40h): 10h concept study, 8h exercises, 14h assignment implementation, 5h of 1-Hour-Without-AI, 3h weekly checkpoint.

---

## 2. Weekly Detailed Plan (Day by Day)

Each day is based on 8 hours. On days when time is short, cut 🟡 items first. 🔴 items must be finished that same day, without exception.

### 2.1 Week 1 — Environment, Tools, and the Memory Model

By the end of this week: your repository will be up, tests will run, the instruction file will exist, and you will have confirmed the memory model (GC or ownership) experimentally.

#### Day 1 (Mon) — Setting Up the Development Environment and the Repository Skeleton 🔴

**Morning (2.5h) — Installation and Verification**

1. Install and verify Visual Studio 2026
   - C# track: the ".NET desktop development" + "ASP.NET and web development" workloads
   - C++ track: the "Desktop development with C++" workload (including MSVC, Windows SDK, CMake tools)
2. Verify the .NET 10 SDK: in PowerShell, `dotnet --list-sdks` → you should see `10.x`
3. Git configuration
   ```powershell
   git config --global user.name  "<name>"
   git config --global user.email "<email>"
   git config --global init.defaultBranch main
   git config --global core.autocrlf true
   ```
4. Install Windows Terminal + PowerShell 7, set the profile's default shell to PowerShell 7
5. C++ track only: clone vcpkg and bootstrap it
   ```powershell
   git clone https://github.com/microsoft/vcpkg C:\dev\vcpkg
   C:\dev\vcpkg\bootstrap-vcpkg.bat
   ```
6. 🟡 Just confirm these for later Phases: the SQLite CLI (`sqlite3 --version`), and after downloading redis-windows, run `redis-server` → `redis-cli ping` → `PONG`

**Afternoon (2.5h) — Starting Assignment 1-C**

- `L-C-01` Build the repository skeleton (§3)
- `L-C-02` Write `.gitignore` / `.editorconfig`
- First commit: `chore: repository skeleton and dev environment setup`

**1 Hour Without AI**

- Use 20 Git commands without help: `init add commit status log diff branch checkout switch merge rebase reset restore stash tag remote push pull fetch show`
- Create 2 branches, edit the same line differently in each, **deliberately create a conflict**, then resolve it with a merge. Write up the resolution process in `notes/daily`

**Evening Wrap-Up (20 min)**

- Write a learning-note entry (Overview template T1), set tomorrow morning's quiz scope: "the difference between a process and a thread"

**DoD**

- [ ] `dotnet --list-sdks` shows 10.x (C++ track: verify the `cl.exe` version)
- [ ] GitHub repository created, first commit pushed
- [ ] `.gitignore`, `.editorconfig`, `README.md`, `notes/` folder all exist
- [ ] `git log` shows a history of a deliberately created and resolved conflict

#### Day 2 (Tue) — Unit Testing and a Local Build/Test Script 🔴

**Morning (2.5h) — Concepts**

- The purpose of unit testing: a server has no GUI. **Tests are how you run it.**
- The AAA pattern (Arrange-Act-Assert), test naming convention (`Subject_Scenario_ExpectedResult`)
- Test doubles: the difference between stub / fake / mock — this course **prefers fakes** (a fake close to the real thing)
- Testable code: take dependencies as constructor arguments (file path, clock, random seed)
- Parameterized tests (`[Theory]` / `TEST_P`)

**Afternoon (2.5h) — Exercises and Assignment**

- `L-C-03` Create a test project (C#: xUnit / C++: GoogleTest + vcpkg)
- `L-C-04` Write the first 5 tests: for a string parser (`"k=v;k2=v2"` → dictionary), cover normal, empty-string, duplicate-key, missing-delimiter, and whitespace cases
- `L-C-05` Write `scripts/build-and-test.ps1` (full script in §3)

**1 Hour Without AI**

- Reimplement the parser above **from an empty file** and pass the 5 tests (40 min implementation, 20 min tests)

**Evening Wrap-Up**

- Write 3 lines in your notes on "how writing tests first changes the design"

**DoD**

- [ ] `pwsh scripts/build-and-test.ps1` builds and tests in a single command and finishes green
- [ ] 5+ tests pass; a history of deliberately breaking one to see red and then reverting it
- [ ] 2+ commits (adding tests / adding the script)

#### Day 3 (Wed) — The Agent Instruction File and an AI Collaboration Routine 🔴

**Morning (2.5h) — Concepts**

- Textbook "Getting Started with AI Coding Agents Using OpenCode", Guide 1 Part 2 (sessions, Plan/Build mode, AGENTS.md), Part 5 (automation)
- What an instruction file does: it fixes the project structure, rules, and the "this is for learning" purpose in the agent's head
- Practice the AI's 4 roles (Overview §2.1): use each of tutor / pair / reviewer / quizmaster at least once

**Afternoon (2.5h) — Wrapping Up Assignment 1-C**

- `L-C-06` Draft the instruction file (`CLAUDE.md` or `AGENTS.md`) — the template is T3 in `08-templates.md`
- Have the agent generate **just one file** according to the instructions, and whenever it breaks a rule, revise the instructions to remove the ambiguity (repeat twice)
- In the README, write 3 sentences on why you chose your track plus a progress table → **bring Assignment 1-C to submission-ready state**

**1 Hour Without AI**

- Hand-copy the 5 prompts you used today into your notes, and add one line each on "what I asked for and what I verified"

**Evening Wrap-Up**

- Commit instruction file v1. Update it every Friday from now on

**DoD**

- [ ] Assignment 1-C complete (self-graded 80+ on the §7.1 rubric)
- [ ] The instruction file contains, in actual sentences, the rules "one file at a time," "design-intent comments," and "tests included"
- [ ] Notes contain 1 case of the agent violating the instructions and the resulting revision

#### Day 4 (Thu) — OS Fundamentals and Memory-Model Experiments 🔴

**Morning (2.5h) — Concepts**

- Processes / threads / handles, context-switch cost (a few µs), user-mode ↔ kernel-mode transitions
- Virtual memory and pages, working set, page faults
- The CPU cache hierarchy and cache lines (64B), why false sharing happens
- Why creating lots of threads makes things slower (switching + cache pollution + memory)

**Afternoon (2.5h) — Track Exercises**

- Common: `L-C-07` build a table of processing time for the same workload as the thread count grows 1→2→4→64→1,000
- C#: `L-CS-01` observe GC generations, `L-CS-02` confirm the LOH (85KB boundary)
- C++: `L-CPP-01` trace `unique_ptr`/`shared_ptr` lifetimes, `L-CPP-02` reproduce a reference-cycle leak → fix it with `weak_ptr`

**1 Hour Without AI**

- Increment a shared counter 1 million times each from 2 threads and reproduce **an incorrect result**. Write in your notes, at the assembly level (read-modify-write), why it's wrong

**Evening Wrap-Up**

- Attach the measurement table to `notes/daily` and write 3 lines on "at how many threads was it fastest, and why"

**DoD**

- [ ] Processing-time-by-thread-count table complete (5 data points)
- [ ] 2 track exercises complete, with screenshots or output logs of the results saved
- [ ] The counter race-condition reproduction code is in `phase1/labs/`

#### Day 5 (Fri) — Starting Assignment 1-2 + Weekly Checkpoint

**Morning (2.5h) — Designing Assignment 1-2**

- Read chapter 3 (Object Pool) of the textbook "C# Design Patterns for Game Developers" (the C++ track also reads it for the concept and ports it to C++)
- Read the requirements for Assignment 1-2 (§7.3) closely, then **design it yourself first**: write the API signature, state, and stats fields on paper or in Markdown
- Only after that, ask the agent for a skeleton (the design is yours; only the boilerplate is delegated)

**Afternoon (4h) — Weekly Checkpoint** (Overview §5.1)

1. **Reimplementation without AI (60 min)**: reimplement Day 2's parser from an empty file + pass the 5 tests
2. **Oral exam (45 min)**: explain "processes/threads," "cache lines," and "GC generations or RAII" (3 topics) to the AI interviewer, graded on a 5-point scale
3. **Code review (45 min)**: request a review of the week's full commit history against the rubric (Overview §5.3) → fix the top 3 issues
4. **Weekly retrospective (30 min)**: write `notes/weekly/W01.md` using template T2

**DoD**

- [ ] A draft design document for Assignment 1-2 (API, state, stats) is in `phase1/ObjectPool/DESIGN-note.md`
- [ ] All 4 weekly-checkpoint items complete, average oral-exam score recorded
- [ ] `notes/weekly/W01.md` written, 5 days of learning notes exist

### 2.2 Week 2 — Concurrency

By the end of this week: you can create and fix race conditions, deadlocks, and visibility problems yourself, and both versions of Assignment 1-1 work.

#### Day 6 (Mon) — The Three Concurrency Diseases 🔴

**Morning (2.5h) — Concepts**

- Race conditions and critical sections
- The 4 conditions for deadlock (mutual exclusion, hold-and-wait, no preemption, circular wait) and how to break each one
- Memory visibility: compiler reordering, CPU reordering, cache coherence
- Types of locks: mutex / spinlock / read-write lock, and when each fits

**Afternoon (2.5h) — Exercises**

- `L-C-08` Minimal race-condition reproduction + 3 fixes (lock / atomic operation / per-thread local sums merged at the end)
- `L-C-09` Minimal deadlock reproduction (locking two locks in opposite order) + 2 fixes (consistent lock ordering / try-lock + backoff)
- `L-C-10` Visibility-problem reproduction (a shutdown flag that isn't seen) + fix (volatile/atomic)

**1 Hour Without AI**

- Rewrite the 3 examples above **without any comments**, then annotate each in your own words explaining what the problem is

**Evening Wrap-Up**

- Write 3 predictions in your notes for "where these three problems are likely to show up in my server code"

**DoD**

- [ ] Reproduction code and fixes for all three problems are each in `phase1/labs/`
- [ ] Your notes record 2+ fixes for each problem

#### Day 7 (Tue) — Debugger Deep Dive 🔴

**Morning (2.5h) — Concepts and Tools**

- Conditional breakpoints, hit-count breakpoints, data breakpoints (C++)
- How to read the Parallel Stacks window / Threads window / Tasks window
- The Memory window, watch expressions, `Debugger.Break()` / `__debugbreak()`
- Crash dumps: save one via Task Manager or `procdump` → open it in VS

**Afternoon (2.5h) — Exercises**

- `L-C-11` Catch yesterday's deadlock with the debugger: save a **screenshot** of the circular-wait point in the Parallel Stacks window (§8 checklist item)
- `L-C-12` Use a conditional breakpoint to "stop only on the 10,000th iteration"
- `L-C-13` Produce a crash dump and open it to pinpoint the crashing line

**1 Hour Without AI**

- Have the agent generate code with 3 hidden defects (Overview T11), then find them **using only the debugger**

**Evening Wrap-Up**

- Write 3 lines in your notes on "what characterizes bugs that logging can't catch but the debugger can"

**DoD**

- [ ] The Parallel Stacks deadlock screenshot is in `phase1/labs/L-C-05-deadlock/`
- [ ] A record of opening 1 dump file and pinpointing the crash line

#### Day 8 (Wed) — Track Concurrency Tools ①

**C# Track, Morning + Afternoon (5h)**

- Concepts: `lock` (Monitor), `Interlocked`, `SemaphoreSlim`, `ReaderWriterLockSlim`, `ConcurrentQueue/Dictionary`
- `L-CS-03` implement the same counter with lock / Interlocked / a concurrent collection and compare throughput
- `L-CS-04` build a producer-consumer queue with `Monitor.Wait/Pulse` → **the skeleton for Assignment 1-1 version (A)**

**C++ Track, Morning + Afternoon (5h)**

- Concepts: `std::thread/jthread`, `mutex/lock_guard/scoped_lock`, `condition_variable`, `atomic`
- `L-CPP-03` implement the same counter with mutex / atomic / per-thread local sums and compare throughput
- `L-CPP-04` build a producer-consumer queue with `condition_variable` (must use the `wait(lock, pred)` form) → **the skeleton for Assignment 1-1 version (A)**

**1 Hour Without AI**

- Reimplement the producer-consumer queue **from an empty file** (30 min) → write 2 tests (ordering, shutdown) (30 min)

**DoD**

- [ ] A throughput-comparison table for the 3 counter approaches
- [ ] The producer-consumer queue draft compiles and both tests pass

#### Day 9 (Thu) — Track Concurrency Tools ②

**C# Track (5h)**

- Concepts: the async/await state machine, `Task` vs. `ValueTask`, `ConfigureAwait`, `CancellationToken`, why `async void` is forbidden
- `L-CS-05` decompile your own `async` method with ILSpy and check the state-machine class and the `MoveNext` flow
- `L-CS-06` build a `Channel<T>` (Bounded) producer-consumer → **the skeleton for Assignment 1-1 version (B)**
- Textbook "Mastering C# Async/Await" chapters 1-4

**C++ Track (5h)**

- Concepts: `atomic` and memory order (relaxed/acquire/release/seq_cst), Win32 `SRWLOCK`/`CONDITION_VARIABLE`
- `L-CPP-05` compare performance between the standard `mutex` version and an `SRWLOCK` version
- `L-CPP-06` build the SPSC ring buffer skeleton (to be finished lock-free in Week 3) → **the skeleton for Assignment 1-1 version (B)**
- Textbook "Modern Windows Multithreading" chapters 3-5, 8

**1 Hour Without AI**

- C#: print the thread ID before and after `await` to see where the thread changes, and write down why
- C++: write 10 lines explaining, using an SPSC queue, why an `acquire/release` pair is necessary

**DoD**

- [ ] The track-specific (B) version skeleton compiles
- [ ] C#: a capture of the decompiled state machine / C++: an SRWLock comparison table

#### Day 10 (Fri) — Focused Implementation of Assignment 1-1 + Weekly Checkpoint

**Morning (3h)** — bring both (A) and (B) versions of Assignment 1-1 (§7.2) to a feature-complete level: `Enqueue/Start/Stop`, stats, capacity policy (Drop/Block)

**Afternoon (4h) — Weekly Checkpoint**

1. Reimplementation without AI (60 min): the producer-consumer queue + shutdown handling
2. Oral exam (45 min): "the 4 conditions for deadlock," "3 ways to fix a race condition," 1 track-specific topic
3. Code review (45 min): request an AI review focused on rubric item 2 (concurrency) → fix the top 3 issues
4. Weekly retrospective (30 min): `notes/weekly/W02.md`

**DoD**

- [ ] Both versions of 1-1 pass their tests (ordering, flush, capacity policy)
- [ ] Weekly retrospective written, 10 days of learning notes

### 2.3 Week 3 — Measurement and Deep Dives

By the end of this week: you'll have trustworthy benchmark results and a report, and you'll have passed the Phase 1 evaluation.

#### Day 11 (Mon) — Benchmark Methodology 🔴

**Morning (2.5h) — Concepts**

- Warmup (JIT/cache), iteration count, mean vs. median vs. p99, reporting standard deviation
- The Debug/Release difference, code disappearing due to optimization (you must consume the result)
- Isolating what you measure (measure just the queue, without file I/O), keeping conditions constant (power settings, background apps)

**Afternoon (2.5h) — Exercises**

- C#: `L-CS-07` create a BenchmarkDotNet project + `MemoryDiagnoser` + `[Params]` for producer counts 1/4/16
- C++: `L-CPP-07` create a Google Benchmark project + `benchmark::DoNotOptimize`
- Common: `L-C-14` measure the same code in Debug/Release and compare in a table

**1 Hour Without AI**

- Write a paragraph on "why a Debug measurement is meaningless," grounded in your own numbers

**DoD**

- [ ] The benchmark project runs in Release and produces a results table
- [ ] Debug/Release comparison table complete

#### Day 12 (Tue) — Measuring Assignment 1-1 and Writing the Report

**Morning (2.5h)** — benchmark both versions of 1-1: producers 1/4/16, throughput (msg/s), bytes allocated per entry, p99 latency

**Afternoon (2.5h)** — write `REPORT-1-1.md` (following the §7.2 report template exactly): environment → measurement method → table → 3 paragraphs on why the difference occurs → conclusion on which to choose

**1 Hour Without AI**

- Rewrite the report's conclusion paragraph without AI. Afterward, ask AI to point out **only** factors your interpretation missed

**DoD**

- [ ] `REPORT-1-1.md` complete (table, variance, conclusion, 1+ "thing AI got wrong")
- [ ] Assignment 1-1 self-graded 80+ (§7.2)

#### Day 13 (Wed) — Finishing and Measuring Assignment 1-2

**Morning (2.5h)** — finish the object pool for 1-2: `Rent/Return`, max retained count, stats, double-return detection

**Afternoon (2.5h)** — measurement
- C#: 3-way comparison of `new byte[4096]` every time vs. your own pool vs. `ArrayPool<byte>.Shared` (with `MemoryDiagnoser`), and capture GC-count observations with `dotnet-counters`
- C++: `new/delete` vs. pool comparison, confirm 0 leaks under an ASan build

**1 Hour Without AI**

- Implement the pool's "double-return detection" a second time using a different approach (e.g., a rental-token scheme)

**DoD**

- [ ] All 4 tests for 1-2 pass (rent/return/max retained/double return)
- [ ] 3-way comparison table complete, `REPORT-1-2.md` written

#### Day 14 (Thu) — Track Deep Dive

**C# Track (5h)**

- Reproduce and diagnose ThreadPool starvation (queue length from `dotnet-counters`), the danger of synchronously waiting on `.Result`
- Textbook "Mastering C# Async/Await" chapter 6, the 12 pitfalls → `L-CS-08`, a project that **reproduces and fixes each of the 12**
- 🟡 `L-CS-09` compare allocations before/after reducing string-parsing allocations with Span/ArrayPool

**C++ Track (5h)**

- Finish the lock-free SPSC ring buffer (`L-CPP-08`), with the acquire/release rationale in comments
- `L-CPP-09` benchmark two versions with/without false sharing (`alignas(64)`)
- 🟡 `L-CPP-10` a UB checklist exercise: reproduce signed-integer overflow, reading an uninitialized value, and a dangling reference each, and observe the effect of compiler optimization

**1 Hour Without AI**

- C#: reproduce 3 of the 12 pitfalls from an empty file
- C++: rewrite the SPSC ring buffer without the memory-order comments, then fill the comments back in afterward

**DoD**

- [ ] C#: 12 pitfall-reproduction cases in the project / C++: lock-free queue passes ASan + a false-sharing comparison table

#### Day 15 (Fri) — Phase 1 Evaluation 🔴

**Morning (3h) — Evaluations 1-2**

1. **Reimplementation exam (60 min, no AI)**: §8.2 — implement one version of the log queue from an empty project + pass the 3 provided tests
2. **Oral exam (60 min)**: §8.3 — 10 random questions from the question bank, graded on a 5-point scale
3. **Defect hunting (45 min)**: §8.4 — find 4+ of the 5 defects AI hid in a queue/pool codebase

**Afternoon (3h) — Wrap-Up**

- Go through every item of the checklist (§8.1), fill in anything incomplete
- Weekly retrospective `W03.md` + a Phase 1 retrospective (what was hardest, where you misused AI)
- Complete the §10 Phase 2 preparation items (split the buffer pool out into a library project)

**DoD**

- [ ] Every item in the §8.1 checklist complete
- [ ] Oral-exam average of 4.0 or higher (if not, set up the §8.5 remediation plan)
- [ ] The 1-2 pool is split out as a standalone library project, referenceable from Phase 2

---

## 3. Lab Catalog

Each exercise follows the format **Goal / Prerequisites / Steps / Expected Result / Pass Criterion / Common Mistake / What to Note**. Save results under `phase1/labs/<exercise-number>/`.

### 3.1 Common Labs (L-C)

#### L-C-01 Building the Repository Skeleton (40 min) 🔴

- **Goal**: build the repository structure you'll use for the next 26 weeks
- **Prerequisites**: a GitHub account, Git installed
- **Steps**
  1. Create a `gameserver-course-<name>` repository on GitHub (uncheck README, empty repository)
  2. Locally:
     ```powershell
     mkdir C:\dev\gameserver-course-<name>; cd C:\dev\gameserver-course-<name>
     git init
     mkdir notes\daily, notes\weekly, phase1\labs, scripts
     "# Game Server Bootcamp Study Repository" | Out-File README.md -Encoding utf8
     git remote add origin https://github.com/<account>/gameserver-course-<name>.git
     ```
  3. Create the following sections in the README: course introduction / track choice (finalize by Wednesday of Week 1) / progress table (per-Phase assignment checkboxes) / how to run things
  4. First commit, then `git push -u origin main`
- **Expected Result**: the skeleton from §1.4 is visible on GitHub
- **Pass criterion**: cloning on another PC reproduces the same folder structure (empty folders kept alive with `.gitkeep`)
- **Common mistake**: empty folders don't get pushed to Git → add a `.gitkeep` file to each one
- **What to note**: why `notes/` lives in the same repository as the code (1 line)

#### L-C-02 `.gitignore` / `.editorconfig` (25 min)

- **Goal**: keep build artifacts out of commits and fix the code style
- **Steps**
  1. `.gitignore`: for C#, `bin/ obj/ *.user`; for C++, `build/ out/ .vs/ CMakeUserPresets.json`; common to both, `BenchmarkDotNet.Artifacts/ *.dmp *.etl`
  2. `.editorconfig`: indentation (C# 4 spaces, C++ 4 spaces), `end_of_line = crlf`, `charset = utf-8`; for C#, specify 3+ analyzer rules such as `dotnet_diagnostic.CA2007.severity`
  3. Deliberately build so `bin/` gets created, and confirm it doesn't show up in `git status`
- **Pass criterion**: `git status` is clean
- **What to note**: why analyzer warnings are promoted to errors (1 line)

#### L-C-03 Creating a Test Project (40 min) 🔴

- **Goal**: build the test foundation that every later assignment will sit on
- **Steps (C#)**
  ```powershell
  cd phase1
  dotnet new sln -n Phase1
  dotnet new classlib -n Phase1.Core   -f net10.0
  dotnet new xunit    -n Phase1.Tests  -f net10.0
  dotnet sln add Phase1.Core Phase1.Tests
  dotnet add Phase1.Tests reference Phase1.Core
  dotnet test
  ```
- **Steps (C++)**
  1. In `phase1/CMakeLists.txt`: `cmake_minimum_required(VERSION 3.28)`, `project(Phase1 CXX)`, `set(CMAKE_CXX_STANDARD 23)`, `enable_testing()`
  2. Add `gtest` and `benchmark` to the `vcpkg.json` manifest
  3. After configuring `add_subdirectory(src)`, `add_subdirectory(tests)`:
     ```powershell
     cmake -B build -S . -DCMAKE_TOOLCHAIN_FILE=C:/dev/vcpkg/scripts/buildsystems/vcpkg.cmake
     cmake --build build --config Debug
     ctest --test-dir build -C Debug
     ```
- **Pass criterion**: the runner finishes with "success" even with 0 tests
- **Common mistake**: missing the vcpkg toolchain path → check `-DCMAKE_TOOLCHAIN_FILE`

#### L-C-04 The First 5 Tests — a Config String Parser (60 min) 🔴

- **Goal**: get comfortable with the AAA pattern and the test-naming convention
- **Subject**: `ConfigParser.Parse(string)`, which turns `"port=9000;workers=4;name=svr"` into `{port:9000, workers:4, name:svr}`
- **The 5 tests (must be exactly these 5)**
  | # | Name | Input | Expected |
  |---|---|---|---|
  | 1 | `Parse_NormalInput_Returns3KeyValues` | `"a=1;b=2;c=3"` | 3 entries |
  | 2 | `Parse_EmptyString_EmptyResult` | `""` | 0 entries, no exception |
  | 3 | `Parse_DuplicateKey_LastValueWins` | `"a=1;a=2"` | `a=2` |
  | 4 | `Parse_NoDelimiter_Throws` | `"a1;b=2"` | `FormatException` (C++: `std::invalid_argument` or an error code) |
  | 5 | `Parse_WithWhitespace_Trims` | `" a = 1 ; b = 2 "` | `a=1,b=2` |
- **Steps**: write the 5 tests **first** and confirm red → implement → green
- **Pass criterion**: all 5 pass, and you can tell what each verifies just from the test name
- **What to note**: 1 way the API design changed because you wrote the tests first

#### L-C-05 Local Build/Test Script (45 min) 🔴

- **Goal**: build the habit of "verify everything with one command" even without CI
- **Steps**: write `scripts/build-and-test.ps1`. Use the following as-is and adapt the paths to your own
  ```powershell
  #requires -Version 7
  [CmdletBinding()]
  param(
      [ValidateSet('Debug','Release')] [string]$Configuration = 'Debug',
      [switch]$SkipCpp,
      [switch]$SkipCSharp
  )
  $ErrorActionPreference = 'Stop'
  $root = Split-Path $PSScriptRoot -Parent
  $failed = @()

  if (-not $SkipCSharp -and (Test-Path "$root\phase1\Phase1.sln")) {
      Write-Host "== C# build & test ($Configuration) ==" -ForegroundColor Cyan
      dotnet build   "$root\phase1\Phase1.sln" -c $Configuration --nologo
      if ($LASTEXITCODE -ne 0) { $failed += 'dotnet build' }
      dotnet test    "$root\phase1\Phase1.sln" -c $Configuration --nologo --no-build
      if ($LASTEXITCODE -ne 0) { $failed += 'dotnet test' }
  }

  if (-not $SkipCpp -and (Test-Path "$root\phase1\CMakeLists.txt")) {
      Write-Host "== C++ configure/build/test ($Configuration) ==" -ForegroundColor Cyan
      cmake -B "$root\phase1\build" -S "$root\phase1" `
            -DCMAKE_TOOLCHAIN_FILE=C:/dev/vcpkg/scripts/buildsystems/vcpkg.cmake
      if ($LASTEXITCODE -ne 0) { $failed += 'cmake configure' }
      cmake --build "$root\phase1\build" --config $Configuration
      if ($LASTEXITCODE -ne 0) { $failed += 'cmake build' }
      ctest --test-dir "$root\phase1\build" -C $Configuration --output-on-failure
      if ($LASTEXITCODE -ne 0) { $failed += 'ctest' }
  }

  if ($failed.Count -gt 0) {
      Write-Host "FAILED: $($failed -join ', ')" -ForegroundColor Red
      exit 1
  }
  Write-Host "ALL GREEN" -ForegroundColor Green
  ```
- **Pass criterion**: deliberately breaking a test produces `exit 1` and a red message; reverting it produces `ALL GREEN`
- **What to note**: what's needed to keep up this habit without CI (1 line)

#### L-C-06 Writing the Agent Instruction File (60 min) 🔴

- **Goal**: fix the agent's role as a "pair that helps you learn"
- **Steps**
  1. Copy T3 (C#) or T3' (C++) from `08-templates.md` to write `CLAUDE.md` (or `AGENTS.md`)
  2. Must include at minimum: project structure / language and version (.NET 10, C++23) / 5 coding rules / "design-intent comments" / "one file at a time" / "tests included" / "state a reason and an alternative when adding a library"
  3. Give the agent a test instruction: "following these instructions, create **only one** `ConfigParser` file"
  4. When it violates the instructions (e.g., creates multiple files, has no comments), find which sentence was ambiguous and fix it → test again (repeat twice)
- **Pass criterion**: on the 3rd instruction, only one file is created per the instructions, with both comments and tests
- **What to note**: the ambiguous sentence → the fixed sentence (before/after)

#### L-C-07 Thread Count and Processing Time (60 min)

- **Goal**: break the illusion that "more threads means faster" with numbers
- **Task**: split 100 million integer additions across N threads
- **Steps**: measure 5 times each for N = 1, 2, 4, logical core count, 64, 1000; record the median
- **Expected Result**: best near the logical core count, then it declines
- **Pass criterion**: a table with N / median / ratio to the best / observation notes
- **What to note**: 3 reasons it got slower at 1,000 threads (switching, cache, memory)

#### L-C-08 Reproducing a Race Condition and 3 Fixes (60 min) 🔴

- **Steps**
  1. Have 4 threads each increment a shared counter 1 million times → confirm the result is less than 4 million (repeat 10 times and record the deviation)
  2. Fix A: wrap it in a lock
  3. Fix B: atomic increment (`Interlocked.Increment` / `std::atomic::fetch_add`)
  4. Fix C: per-thread local counters merged at the end
  5. Compare the processing time of the three approaches
- **Pass criterion**: all 3 approaches give exactly 4 million, and a processing-time table exists
- **What to note**: why C is fastest (cache-line contention)

#### L-C-09 Reproducing a Deadlock and 2 Fixes (45 min) 🔴

- **Steps**: a thread that locks A→B and a thread that locks B→A → confirm it hangs → (fix 1) unify lock ordering (fix 2) `try_lock` + release everything and back off on failure
- **Pass criterion**: the reproduction code hangs 100% of the time within 5 seconds, and the fixed version passes 100,000 repetitions
- **What to note**: which of the 4 deadlock conditions each fix breaks

#### L-C-10 A Memory-Visibility Problem (45 min)

- **Steps**: use a plain `bool` flag to signal a worker to stop → try to reproduce it not stopping in a Release build → fix with `volatile` (C#) / `std::atomic<bool>` (C++)
- **Caution**: x86's strong memory ordering may prevent reproduction. If so, research "why does it work fine on x86 but break on ARM" and write it in your notes (this is the exercise's real purpose)
- **Pass criterion**: regardless of whether you reproduced it, write an explanation in your own words of "visibility ≠ atomicity"

#### L-C-11 Catching a Deadlock with the Debugger (45 min) 🔴

- **Steps**: run the L-C-09 reproduction code → "Attach to Process" or F5 in VS → Break All → check where the two threads are waiting on each other in the **Parallel Stacks window** → save a screenshot
- **Pass criterion**: the screenshot shows both threads' wait points, and your notes state in words "who is waiting on what"

#### L-C-12 Conditional Breakpoints (30 min)

- **Steps**: in a 100,000-iteration loop, set a conditional breakpoint that stops "only when i == 99999," a hit-count breakpoint that stops "every 10th time," and (C++) a data breakpoint that stops when a specific variable changes
- **Pass criterion**: a screenshot or a written procedure for each of the three types

#### L-C-13 Collecting and Analyzing a Crash Dump (45 min)

- **Steps**: a console app that crashes on a null reference → "Create Dump File" from Task Manager (or `procdump -ma <pid>`) → open the `.dmp` in VS → pinpoint the cause line from the crash stack
- **Pass criterion**: the line number pinpointed in the dump matches the actual code

#### L-C-14 Comparing Debug vs. Release Measurements (40 min) 🔴

- **Steps**: measure the same loop (string parsing or integer arithmetic) in Debug and Release separately, and compute the multiplier
- **Pass criterion**: a table with both configurations' numbers and the multiplier, plus a paragraph on "why you shouldn't put Debug numbers in a report"

### 3.2 C# Track Labs (L-CS)

#### L-CS-01 Observing GC Generations (45 min)

- **Steps**: create 1 million small objects → print `GC.CollectionCount(0/1/2)` and `GC.GetTotalAllocatedBytes()` → keep some alive in a static list and measure again → observe generational promotion
- **Pass criterion**: explain, with your own numbers, "the more objects survive, the more Gen1/Gen2 collections increase"

#### L-CS-02 Confirming the LOH Boundary (30 min)

- **Steps**: allocate `new byte[84_000]` and `new byte[86_000]` 100,000 times each → observe `GC.GetGeneration` and LOH size (`dotnet-counters monitor --counters System.Runtime`)
- **Pass criterion**: confirm the behavior difference across the 85KB boundary with numbers

#### L-CS-03 Comparing 3 Counter Approaches (45 min)

- **Steps**: count using `lock` / `Interlocked.Increment` / putting entries into a `ConcurrentQueue` → compare throughput
- **Pass criterion**: a table + an explanation of "why Interlocked is faster than lock"

#### L-CS-04 A Monitor-Based Producer-Consumer Queue (90 min) 🔴

- **Skeleton**
  ```csharp
  public sealed class MonitorQueue<T>
  {
      private readonly Queue<T> _q = new();
      private readonly int _capacity;
      private bool _completed;

      public bool TryEnqueue(T item);        // on overflow, behavior follows policy (Drop/Block)
      public bool TryDequeue(out T item);    // Wait if empty; false once Complete has been called
      public void Complete();                // wakes the consumer to shut it down
  }
  ```
- **Pass criterion**: 100,000 round trips with 4 producers / 1 consumer, 0 loss, and the consumer thread exits within 5 seconds of `Complete()`

#### L-CS-05 Decompiling the async State Machine (45 min)

- **Steps**: write a simple `async Task<int> FooAsync()` → build in Release → open it in ILSpy (or `dotnet-ildasm`) and check the `<FooAsync>d__0` state-machine class and the `MoveNext` flow
- **Pass criterion**: explain, in your own words, the state (`<>1__state`) transitions and the point where `AwaitUnsafeOnCompleted` is called

#### L-CS-06 A Channel-Based Queue (90 min) 🔴

- **Steps**: use `FullMode=Wait`; implement Drop with `TryWrite` and count `false`, and Block with `WriteAsync`. Consume via `ReadAllAsync` and shut down with `Writer.Complete()`
- **Pass criterion**: passes the same scenario as L-CS-04, plus a lines-of-code comparison (usually less than half)

#### L-CS-07 Setting Up BenchmarkDotNet (60 min) 🔴

- **Steps**
  ```powershell
  dotnet new console -n Phase1.Bench -f net10.0
  dotnet add Phase1.Bench package BenchmarkDotNet
  ```
  Apply `[MemoryDiagnoser]`, `[SimpleJob(warmupCount:3, iterationCount:10)]`, `[Params(1,4,16)]`, then `dotnet run -c Release`
- **Pass criterion**: results show Mean/Error/StdDev/Allocated, and running under Debug produces a warning

#### L-CS-08 Reproducing the 12 Async Pitfalls (180 min)

- **Steps**: for each of the 12 pitfalls in chapter 6 of "Mastering C# Async/Await," go **reproduce → observe the symptom → fix** in a separate folder each (`Trap01_Deadlock`, `Trap02_LostException`, ...)
- **Pass criterion**: each of the 12 folders has the reproduction code, the fixed code, and a 1-line explanation

#### L-CS-09 Reducing Allocations with Span/ArrayPool (60 min) 🟡

- **Steps**: build the `L-C-04` parser in (a) a `Substring` version and (b) a `ReadOnlySpan<char>` version, and compare with `MemoryDiagnoser`
- **Pass criterion**: allocated bytes drop meaningfully (usually close to 0)

#### L-CS-10 Reproducing ThreadPool Starvation (60 min)

- **Steps**: throw more work than the core count at `Task.Run` blocks that synchronously wait via `.Result` → reproduce the hang → confirm `threadpool-queue-length` rising with `dotnet-counters monitor --counters System.Runtime` → fix it with `await`
- **Pass criterion**: a capture of the queue-length graph + confirmation it returns to normal after the fix

#### L-CS-11 Task vs. ValueTask (40 min) 🟡

- **Steps**: build a method that frequently completes synchronously in two versions and compare allocations
- **Pass criterion**: a write-up of "when ValueTask pays off / when it's dangerous (never await it twice)"

#### L-CS-12 Propagating a CancellationToken (40 min)

- **Steps**: propagate a token through 3 levels of nested async calls, handle `OperationCanceledException` on cancellation, and compare against a version that omits the token
- **Pass criterion**: cancellation propagates within 1 second, and you confirm the version that omits the token keeps running

### 3.3 C++ Track Labs (L-CPP)

#### L-CPP-01 Tracing Smart Pointer Lifetimes (45 min)

- **Steps**: with a `Tracked` class that logs from its destructor, observe `unique_ptr` moves and how `use_count()` changes on `shared_ptr` copies
- **Pass criterion**: predict when the destructor will be called, then compare against the actual run (4+ of 5 predictions correct)

#### L-CPP-02 A Reference-Cycle Leak and weak_ptr (45 min) 🔴

- **Steps**: two Node objects referencing each other via `shared_ptr` → confirm the destructors are never called → replace one side with `weak_ptr` → confirm destruction
- **Pass criterion**: "no destructor log" before the fix, "both destroyed" after the fix

#### L-CPP-03 Comparing 3 Counter Approaches (45 min)

- **Steps**: `mutex` / `atomic<int>::fetch_add(relaxed)` / per-thread local sums → compare throughput
- **Pass criterion**: a table + an explanation of "why relaxed is enough" (only the sum matters, ordering doesn't)

#### L-CPP-04 A condition_variable Producer-Consumer Queue (90 min) 🔴

- **Skeleton**
  ```cpp
  template <class T>
  class BlockingQueue {
  public:
      explicit BlockingQueue(size_t capacity);
      bool TryPush(T v);          // policy on overflow (Drop/Block)
      bool Pop(T& out);           // waits if empty; false once Complete has been called
      void Complete();            // wakes the consumer to shut it down
  private:
      std::mutex m_;
      std::condition_variable cv_;
      std::deque<T> q_;
      size_t cap_;
      bool completed_ = false;
  };
  ```
- **Caution**: only use the form `cv_.wait(lock, [&]{ return !q_.empty() || completed_; })` (a predicate-less `wait` is forbidden)
- **Pass criterion**: 100,000 items with 4 producers / 1 consumer, 0 loss, shutdown within 5 seconds

#### L-CPP-05 SRWLock vs. std::mutex (45 min)

- **Steps**: implement the same critical section with `std::mutex` and with `SRWLOCK` (`AcquireSRWLockExclusive`) and compare throughput; for a read-heavy workload, also measure `AcquireSRWLockShared`
- **Pass criterion**: a table of the 3 numbers + an explanation of "when a shared read lock pays off"

#### L-CPP-06 SPSC Ring Buffer Skeleton (60 min)

- **Steps**: write a single-producer/single-consumer queue skeleton using a fixed-size array plus head/tail indices (first secure correctness with a mutex-based version)
- **Pass criterion**: 0 loss and 0 duplication over 1 million round trips

#### L-CPP-07 Setting Up Google Benchmark (60 min) 🔴

- **Steps**: add `benchmark` via vcpkg → `BENCHMARK(BM_Foo)->Arg(1)->Arg(4)->Arg(16);` → prevent optimization from eliminating the code with `benchmark::DoNotOptimize()` → run it as a Release build
- **Pass criterion**: the result shows iteration count, time, and standard deviation

#### L-CPP-08 Finishing the Lock-Free SPSC (120 min) 🔴

- **Steps**: change the indices from L-CPP-06 to `std::atomic<size_t>`, with the producer using `store(release)` and the consumer using `load(acquire)`. Leave a comment justifying each memory-order choice
- **Pass criterion**: identical results to the mutex version (0 loss, 0 duplication), a throughput comparison table, passes ASan
- **What to note**: explain "why seq_cst isn't needed here" in terms of happens-before

#### L-CPP-09 Measuring False Sharing (60 min)

- **Steps**: compare 4 threads each incrementing a counter laid out (a) in an adjacent array vs. (b) in an `alignas(64)`-aligned array
- **Pass criterion**: a table showing the throughput difference between the two versions, explained via cache lines

#### L-CPP-10 UB Exercise (60 min) 🟡

- **Steps**: reproduce signed-integer overflow / reading an uninitialized variable / a dangling reference each, and observe the behavior difference under `-O0` and `-O2` (`/Od`, `/O2`)
- **Pass criterion**: record at least 1 case where optimization changed the behavior

#### L-CPP-11 Catching a Use-After-Free with ASan (45 min) 🔴

- **Steps**: VS project properties → C/C++ → General → enable "Use AddressSanitizer" (turn off `/RTC`) → run code that accesses freed memory → check the report
- **Pass criterion**: point out the allocation, free, and access locations from the ASan report

#### L-CPP-12 Move-Semantics Copy Counter (45 min)

- **Steps**: a `Buffer` class with counters embedded in its copy and move constructors → push_back into a `vector`, compare with/without `std::move`, compare with/without `reserve`
- **Pass criterion**: a table of copy/move counts + an explanation of "when copying disappears (copy elision)"

---

## 4. Learning Items in Detail

Each item is structured as **What / Why / How (order) / Verify**. "Verify" is the basis for judging that the learning is complete. Exercise numbers in parentheses refer to §3.

### 4.1 Common (40h)

**Git Workflow (4h)**
- What: branching strategy (main + feature), self-review, rebasing, conflict resolution, commit message rules (50-char subject, "why" in the body)
- Why: every assignment is graded at the repository level, and this is a tool you'll use from day one on the job
- How: ① a `feat/<name>` branch per feature ② commits as "the smallest working unit" ③ clean up with `git rebase main` before merging ④ resolve conflicts by hand and put the reason in the commit body
- Verify: `git log --graph` shows a history of 2 deliberately created conflicts, each resolved via merge/rebase (L-C-01)

**Unit Testing (6h)**
- What: AAA, test-naming convention, fakes, dependency injection, parameterized tests
- Why: a server has no GUI so tests are how you run it, and later evaluation criteria are "pass the provided tests"
- How: ① a failing test first ② the minimal implementation ③ refactor ④ boundary values (empty input, max value, duplicates) must always be their own cases
- Verify: the 1-1 log queue's ordering guarantee, flush, and capacity policy are each proven by an independent test (L-C-04)

**Debugging (6h)**
- What: conditional/hit-count/data breakpoints, the Parallel Stacks/Threads windows, the Memory window, saving and analyzing dumps
- Why: most server bugs are multithreaded and hard to reproduce. Logs alone won't catch them
- How: ① pin down the reproduction condition in a script ② use a conditional breakpoint to stop only at the problem iteration ③ use Parallel Stacks to understand the relationship between threads ④ take a dump for post-mortem analysis when needed
- Verify: a deadlock Parallel Stacks screenshot + a record of analyzing 1 dump (L-C-11, L-C-13)

**OS Fundamentals (6h)**
- What: processes/threads/handles, context switching, user/kernel mode, virtual memory, cache lines (64B), false sharing
- Why: the basis for "why does adding threads make it slower" and "why is an array of structs fast"
- How: concept → L-C-07 (thread-count experiment) → L-CPP-09/L-CS-03 (cache contention), in that order — **see the numbers first, then attach the theory**
- Verify: a processing-time table from 1→1,000 threads plus a 3-line interpretation (L-C-07)

**Concurrency Concepts (10h)**
- What: race conditions, critical sections, the 4 conditions for deadlock, lock types, atomic operations, visibility/reordering, the meaning and limits of lock-free
- Why: among game-server bugs, these are the most expensive and the hardest to reproduce
- How: for each of the three diseases (race, deadlock, visibility): **reproduce with minimal code → apply 2+ fixes → compare throughput**
- Verify: 3 complete sets of results from L-C-08, L-C-09, L-C-10

**Windows Development Environment (4h)**
- What: VS 2026 workloads, pinning the .NET SDK version (`global.json`), the vcpkg manifest, PowerShell 7 scripts, confirming SQLite (no install needed), (optional) MySQL 8, redis-windows
- Why: since Docker isn't used, local installation and process management are themselves practical skills. SQLite is the default for development convenience; MySQL is optional
- How: install → a version-check command → start/stop services via PowerShell → tabulate the confirmation results in your notes
- Verify: a response from `sqlite3 --version`, `redis-cli ping` → `PONG`, (optional) connecting via `mysql -u root -p`. Able to start/stop the redis process via PowerShell

**Benchmark Methodology (4h)**
- What: warmup, repetition, mean/median/p99, reporting variance, Debug/Release, preventing optimization from eliminating the code, controlling conditions
- Why: to "prove an improvement with numbers" in Phase 5, the measurement itself must be trustworthy
- How: ① always Release ② 3+ warmup runs ③ 10+ repetitions for median/standard deviation ④ consume the result (`DoNotOptimize`) ⑤ record the environment (CPU, power settings, background apps) in the report
- Verify: a Debug/Release comparison table and a paragraph on "why a Debug measurement is meaningless" (L-C-14)

### 4.2 C# Track (80h)

| Item | Time | What | Learning Order | Verification |
|---|---|---|---|---|
| .NET runtime and GC | 10h | JIT, generational GC, LOH, workstation/server GC, allocation cost, `GC.GetTotalAllocatedBytes`, `dotnet-counters` | Concept → L-CS-01 → L-CS-02 → apply to 1-2 measurements | LOH-boundary experiment results and a `dotnet-counters` capture |
| Value/reference types, structs | 6h | struct vs. class, `readonly struct`, `ref struct`, boxing cost, `in`/`ref` parameters | Concept → benchmark with/without boxing | Allocation comparison table with/without boxing |
| Span/Memory/ArrayPool | 8h | `Span<T>`, `Memory<T>`, `stackalloc`, `ArrayPool<T>.Shared`, removing allocations during parsing | L-CS-09 → compared against the pool in 1-2 | Substring vs. Span allocation comparison |
| async/await internals | 14h | state machine, SynchronizationContext, `ConfigureAwait`, Task vs. ValueTask, why `async void` is forbidden, exception propagation, cancellation | Textbook ch.1-4 → L-CS-05 → L-CS-11 → L-CS-12 | An explanation of the decompiled state-machine flow |
| ThreadPool | 6h | work items, starvation, the danger of synchronously waiting on `.Result`, overusing `Task.Run` | Concept → L-CS-10 | Starvation reproduction + counter capture + fix |
| Synchronization tools | 12h | `lock`, `Interlocked`, `SemaphoreSlim`, `ReaderWriterLockSlim`, `ConcurrentQueue/Dictionary`, `Channel<T>` | L-CS-03 → L-CS-04 → L-CS-06 | Both versions of Assignment 1-1 |
| BenchmarkDotNet | 6h | writing benchmarks, `MemoryDiagnoser`, `Baseline`, `Params`, interpreting results | L-CS-07 → 1-1/1-2 measurements | 2 reports |
| Project configuration | 6h | .NET CLI, solution structure, NuGet, analyzers, `TreatWarningsAsErrors`, nullable | L-C-03 → 1-C | A solution that builds with 0 warnings |
| Async pitfalls | 12h | deadlocks, lost exceptions, ignored cancellation, fire-and-forget, context capture | Textbook ch.6 → L-CS-08 | A project reproducing and fixing all 12 |

### 4.3 C++ Track (80h)

| Item | Time | What | Learning Order | Verification |
|---|---|---|---|---|
| Ownership and RAII | 10h | object lifetime, stack/heap, `unique_ptr/shared_ptr/weak_ptr`, reference cycles, custom deleters | Textbook ch.4-5 → L-CPP-01 → L-CPP-02 | Reproducing and fixing a reference-cycle leak |
| Move semantics | 8h | value categories, move constructor/assignment, `std::move`/`forward`, copy elision, Rule of 0/5 | Concept → L-CPP-12 | A table of copy/move counts |
| Standard concurrency | 14h | `thread/jthread`, `mutex/lock_guard/scoped_lock`, `condition_variable`, `atomic`, memory order | L-CPP-03 → L-CPP-04 → L-CPP-06 → L-CPP-08 | Both versions of Assignment 1-1 |
| Win32 synchronization | 8h | `SRWLOCK`, `CONDITION_VARIABLE`, `InitOnce`, `WaitOnAddress`, an overview of the thread-pool API | Textbook ch.3-5, 8 → L-CPP-05 | A mutex vs. SRWLock comparison table |
| Build system | 8h | CMake targets/properties, the vcpkg manifest, Debug/Release, `/W4 /permissive-`, `/analyze` | L-C-03 → L-C-05 | Build and test pass via the local script |
| Sanitizers and testing | 8h | ASan (MSVC), Application Verifier, GoogleTest, Google Benchmark | L-CPP-11 → L-CPP-07 | ASan report capture |
| Templates and utilities | 8h | template basics, `std::function`/lambdas, `optional/variant/span/expected` | Concept → turn 1-2 into a template | A templated pool implementation |
| Cache and UB | 8h | false sharing, `alignas`, the list of UB, the effect of optimization | L-CPP-09 → L-CPP-10 | False-sharing comparison benchmark |
| Language refresher (optional) | 8h | Core Guidelines highlights, `constexpr`, strong typing, the 3 levels of exception safety | Textbook ch.3, 7, 10 | Self-graded checklist |

---

## 5. Textbook Guide (Phase 1)

The textbook source material lives in the `programming-books-with-ai` repository. Read only the assigned chapters, at the assigned time. Follow the reading approach in Overview §2.3 and `07-books-guide.md` §2.

### 5.1 Common

| Textbook | Chapters to Read | When | How to Use |
|---|---|---|---|
| Getting Started with AI Coding Agents Using OpenCode | Guide 1 Part 2 (sessions, Build/Plan mode, AGENTS.md), Part 5 (automation) | Day 3 morning | This course uses coding agents such as Claude Code or Codex CLI freely, but the concepts of an "instruction file" and "Plan then Build" are the same either way. After reading, pick 5 items to fold into your own instruction file and apply them in L-C-06 |
| C# Design Patterns for Game Developers | Ch.3 Object Pool | Day 5 morning | Read before designing Assignment 1-2. The C++ track also reads it to grasp the concept and ports it to C++. Find and note a case where "pooling is actually a net loss" |

### 5.2 C# Track

| Textbook | Chapters to Read | When | How to Use |
|---|---|---|---|
| The Essential C# Guide for ASP.NET Core Web API | All (short) | Early Week 1, speed-read | For checking gaps in your syntax. Mark only the sections you don't know and type out the examples by hand |
| Mastering C# Async/Await | Ch.1-4 (basics, SynchronizationContext, ConfigureAwait, Awaitable), ch.6 (the 12 pitfalls), ch.7 (synchronization objects) | Days 9, 14 | Hand the checklist at the end of each chapter to the AI interviewer for an oral exam. Build the samples (.NET 10) as-is. Chapter 6 is the textbook for L-CS-08 |
| Building a C# async/await Library | Ch.1-6 (Task/ValueTask internals, dissecting the state machine, designing an Awaitable, custom Awaitables, DelayAsync, cancellation) | Week 3 | After reading chapter 2, cross-check it against your L-CS-05 decompilation results. Turn each chapter's exercises into an assignment. Chapters 16-20 belong to Phases 3 and 5 |

### 5.3 C++ Track

| Textbook | Chapters to Read | When | How to Use |
|---|---|---|---|
| Safe and Elegant Programming with Modern C++ | Ch.2 (development environment), ch.3 (Core Guidelines), ch.4 (RAII), ch.5 (smart pointers), ch.6 (containers), ch.7 (strong types), ch.10 (exception safety), ch.12-13 (concurrency) | Weeks 1-2 | Written against VS 2022 but builds as-is on VS 2026. There's no code folder, so porting the in-text code into a project yourself is the assignment. Chapter 18 (benchmarking) is Week 3 |
| Modern Windows Multithreading | Ch.2 (history), ch.3 (SRW Lock), ch.4 (Condition Variable), ch.5 (One-Time Init), ch.8 (WaitOnAddress, Lock-Free) | Around Day 9 | Read while building a 1:1 mapping table between the standard library and the Win32 API. The textbook for L-CPP-05. Chapter 6 is Phase 3, chapter 10 is Phase 5 |
| Modern Win32 API Programming for Game Server Developers | Ch.1 (Win32 basics), ch.2 (memory management, memory pools), ch.4 (processes/threads), ch.5 (synchronization in depth), ch.6 (interlocked/lock-free) | Weeks 2-3 | Chapter 2's memory pool is a direct reference for Assignment 1-2. Chapter 7 is Phase 5 |
| The Complete Guide to the C++23 Memory Model | Part 1 (foundational theory), part 2 (memory order in depth) | Right before Day 14 | Read right before L-CPP-08 (lock-free SPSC). Part 3 is Phase 5 |
| Safe and Easy C++ Programming, Starting with Modern C++ | Only the weak parts of chapters 1-17 | Week 1 (if needed) | Only for learners shaky on syntax. Chapter 18 onward (Siv3D) is unrelated |

### 5.4 Secondary-Track Reading (Track A parallel learners, under 4h/week)

- C# main track: "Safe and Elegant Programming with Modern C++" ch.4-5 → **a "C# GC vs. C++ RAII" comparison table** (4 rows: when lifetime is decided, where the cost falls, leak types, debugging tools)
- C++ main track: "Mastering C# Async/Await" ch.1 → **a "C++ threads vs. C# Task" comparison table** (4 rows: who schedules, blocking cost, how to cancel, exception propagation)

---

## 6. AI Collaboration Guide (Phase 1)

### 6.1 Prompts

**Concept check (tutor mode)**
```
I understand [C# GC generations / C++ move semantics] as follows:
"(my 5-10 sentence explanation)"
1) Point out what's wrong and what's missing.
2) Suggest a 10-line-or-fewer experiment I can run myself to verify whether my explanation is right.
3) Give me keywords to look up in the official documentation.
Don't state the conclusion first — point out the errors in my explanation first.
```

**Defect injection (quizmaster mode)**
```
Write code for a [thread-safe log queue] meeting the following requirements, but deliberately hide 1 deadlock, 1 race condition, and 1 resource leak in it.
Do not reveal the locations under any circumstances until I say "reveal the answers."
If I ask for a hint, tell me only the "type" of defect, not its location.
The defects must compile and behave normally on most runs.
```

**Interpreting a benchmark (reviewer mode)**
```
Here are my BenchmarkDotNet (or Google Benchmark) results: (paste the table)
Build configuration: Release / 3 warmup runs / 10 iterations / CPU: (model)
My interpretation: "(2-3 sentences)"
Tell me whether you agree with my interpretation, and whether there are factors I missed (warmup, GC, cache, measurement error, power settings).
```

**Instruction-file review**
```
Here is a draft of my learning project's agent instruction file (Claude Code uses CLAUDE.md, Codex CLI uses AGENTS.md). (paste it in)
Point out anything that conflicts with the learning goal (that I understand the code as I go), any missing rules, and any sentences too vague to actually follow.
For each point, suggest a concrete replacement sentence.
```

**Step-by-step implementation request (pair mode)**
```
I want to build a [log queue]. Don't give me the whole thing at once — let's split it into steps.
Step 1: just the public API signature and state fields (mark the implementation with TODO comments)
Don't move on to step 2 until I say "next."
At each step, if I explain my design intent first, point out anything wrong with it.
```

### 6.2 What to Delegate vs. What Not to Delegate

| Delegate | Don't Delegate |
|---|---|
| A draft local build/test script, `.editorconfig`, project skeletons | The first implementation of the core algorithm for the log queue or object pool |
| Drawing up a list of test cases (I add any missing ones myself) | The final conclusion when interpreting benchmark results |
| Benchmark-project boilerplate | Learning notes, weekly retrospectives |
| Generating defect-injected code | Finding the defects |
| Fixing CMake/solution configuration errors | The rationale behind a memory-order choice (write it yourself, only get it verified) |

### 6.3 Where AI Often Gets It Wrong (Verification Points)

- **API differences across .NET versions**: e.g., `ConfigureAwaitOptions` is .NET 8+. Check the official docs' "applies to" table. This course targets .NET 10
- **C++ memory order**: because of x86's strong ordering, examples claiming "relaxed works fine too" show up often. Always ask back "is this safe on ARM too?"
- **Debug-build benchmarks**: if you just paste in results, AI can't see the build configuration. Always state the build configuration explicitly
- **Defaulting to an Unbounded `Channel<T>`**: it has no backpressure. Demand an explicit capacity and overflow policy
- **`await` inside a `lock`**: sometimes suggested even though it's impossible in C#. Steer it toward `SemaphoreSlim`
- **A predicate-less `wait` on a C++ `condition_variable`**: code that misses spurious wakeups. Always require the `wait(lock, pred)` form

---

## 7. Assignment Specifications

The deliverables of this Phase are **1-C (common), 1-1 (track), and 1-2 (track)**, with 1-3 optional. Each assignment is placed within the day plan in §2.

### 7.1 Common Assignment 1-C. Building the Learning Environment and Repository (required, 8h, Days 1-3)

**Goal**: build the repository structure and verification automation you'll use for the next 26 weeks.

**Deliverable tree**

```
gameserver-course-<name>/
├─ README.md                  ① course intro ② track choice and 3 reasons ③ progress table ④ how to run things
├─ CLAUDE.md or AGENTS.md     agent instruction file
├─ .gitignore                 bin/ obj/ build/ .vs/ BenchmarkDotNet.Artifacts/ *.dmp
├─ .editorconfig              indentation, encoding, line endings, 3+ analyzer rules
├─ scripts/build-and-test.ps1 build+test in one shot (use the §3 L-C-05 script in full)
├─ notes/daily/.gitkeep
├─ notes/weekly/.gitkeep
└─ phase1/                    the Phase 1 solution or CMake root
```

**Requirement Details**

1. **Repository**: create `gameserver-course-<name>` on GitHub. Build it out per the tree above, keeping empty folders alive with `.gitkeep`
2. **Local verification automation** (in place of CI): `scripts/build-and-test.ps1` runs, in one command, `dotnet build` + `dotnet test` for C# and CMake configure/build + `ctest` for C++. `exit 1` on failure
3. **Pre-commit habit**: at least once, **deliberately break a test** to confirm the script ends red, then revert it and leave a history of that (2 commits)
4. **README**: 3 sentences on why you chose your track (1 sentence each for target-company stack, personal preference, and existing proficiency), a per-Phase progress table (checkboxes), how to run things (`pwsh scripts/build-and-test.ps1`)
5. **Instruction file**: project structure / language and version (.NET 10 or C++23, VS 2026) / 5 coding rules / "design-intent comments when generating code" / "one file at a time" / "tests included" / "state a reason and one alternative when adding a library"
6. **Confirming the track**: finalize your main track by Day 3 and record it in the README (if you change it later, record the reason in a retrospective)

**Submission**: repository URL / `build-and-test.ps1` run logs (1 success, 1 failure) / the instruction file

**Grading (self-assessed)**

| Item | Points | Criterion |
|---|---|---|
| Structure | 25 | follows the tree above, empty folders kept, build artifacts excluded via `.gitignore` |
| Test-automation script | 35 | build+test pass in one run, 1 recorded failure-and-recovery, exit code 1 on failure |
| Instruction file | 30 | required items present, sentences are concrete (no vague "write good code"), revised at least once based on a violation |
| README | 10 | 3 sentences on track choice, progress table, how to run things |

**Common Mistakes**: the script returns exit code 0 even on failure (→ missing a check on `$LASTEXITCODE`) / the instruction file contains only generalities, so actual generated results don't change

### 7.2 Track Assignment 1-1. Thread-Safe Log Queue (required, 24h, Days 8-12)

**Background**: a game server produces thousands of log lines per second. If writing logs blocks the game-logic thread, latency spikes. That's why the "producers just enqueue, and a dedicated consumer thread writes to the file" structure is used. This assignment is to build that structure **in two different synchronization styles** and compare them with numbers.

**Deliverable tree**

```
phase1/LogQueue/
├─ src/    LogQueue implementation (version A, version B, common interface)
├─ tests/  15+ unit tests
├─ bench/  benchmark project
└─ REPORT-1-1.md
```

**Functional Requirements**

1. **Public API** (C#)
   ```csharp
   public enum OverflowPolicy { Drop, Block }

   public readonly record struct LogEntry(long Seq, int ProducerId, DateTime TimestampUtc, string Message);

   public readonly record struct LogQueueStats(long Enqueued, long Dropped, long Written, int CurrentCount);
   public interface ILogSink { void Write(in LogEntry entry); void Flush(); }

   public sealed class LogQueueOptions
   {
       public int Capacity { get; init; } = 10_000;
       public OverflowPolicy Policy { get; init; } = OverflowPolicy.Drop;
       public string FilePath { get; init; } = "log.txt";
       public int FlushIntervalMs { get; init; } = 100;
   }

   public interface ILogQueue : IAsyncDisposable
   {
       void Start();
       bool Enqueue(in LogEntry entry);   // may return false under the Drop policy; always true under Block
       Task StopAsync();                  // writes all remaining entries, then shuts down
       LogQueueStats Stats { get; }       // Enqueued, Dropped, Written, CurrentCount
   }
   ```
   (C++)
   ```cpp
   enum class OverflowPolicy { Drop, Block };

   struct LogEntry {
       std::uint64_t seq;
       int producer_id;
       std::chrono::system_clock::time_point ts;
       std::string message;
   };

   struct LogQueueOptions {
       std::size_t capacity = 10'000;
       OverflowPolicy policy = OverflowPolicy::Drop;
       std::filesystem::path file_path = "log.txt";
       int flush_interval_ms = 100;
   };

   class ILogQueue {
   public:
       virtual ~ILogQueue() = default;
       virtual void Start() = 0;
       virtual bool Enqueue(LogEntry entry) = 0;
       virtual void Stop() = 0;            // writes all remaining entries, then joins the thread
       virtual LogQueueStats Stats() const = 0;
   };
   ```
2. **Behavior definition**
   - `Enqueue` before `Start()` either throws (C#) or returns `false` (C++) — **pick one, document it**, and pin it down with a test
   - `StopAsync()/Stop()` proceeds in order: ① block new input ② write all remaining entries to the file ③ flush/close the file ④ join the consumer thread
   - After `Stop()`, `Enqueue` must always fail (returning false without an exception is recommended)
   - File line format: `{seq}\t{producerId}\t{ISO8601 UTC}\t{message}`
   - Escape `\t`, `\r`, and `\n` in messages; write UTF-8 with LF line endings
   - `StopAsync()/Stop()` and disposal are idempotent. If Stop begins while a producer is blocked, wake it and return `false`
   - Inject `ILogSink`/an equivalent C++ callback to reproduce write failures; sink exceptions must update error stats without killing the consumer loop
3. **Policies**
   - `Drop`: when the queue is full, discard immediately and increment `Dropped`; `Enqueue` returns false
   - `Block`: the producer waits until space frees up (a max-wait-time option is optional)
4. **Invariants** (must be proven by tests)
   - `Written == Enqueued - Dropped`
   - entries from the same `ProducerId` keep their insertion order in the file
   - after shutdown, the file's line count == `Written`

**Track-Specific Implementation Requirements**

- **C#**: (A) a `lock` + `Queue<T>` + `Monitor.Wait/Pulse` version, (B) a `Channel<T>` (Bounded, expressing the policy via `FullMode`) version. Both versions must implement **the same interface**
- **C++**: (A) a `std::mutex` + `std::condition_variable` + `std::deque` version, (B) a lock-free SPSC ring-buffer version (single-producer only; run the multi-producer test only against A). For (B), leave the `acquire/release` rationale in comments and it must pass ASan

**Test List (15 tests, all required)**

| # | Name | Scenario | Expected |
|---|---|---|---|
| 1 | Enqueue_BeforeStart_DefinedBehavior | Enqueue before Start | the documented behavior (exception or false) |
| 2 | Enqueue_SingleProducer_AllWritten | 1 thread, 1,000 items | 1,000 lines in the file |
| 3 | Enqueue_MultiProducer_NoLoss | 4 threads × 10,000 items, Block policy | 40,000 lines in the file |
| 4 | OrderGuarantee_PerProducer | 4 threads each sending 1..10,000 | ascending sequence per producer |
| 5 | DropPolicy_OverflowDiscarded | capacity 100, 10,000 items flooding in | `Written == Enqueued - Dropped`, Dropped > 0 |
| 6 | BlockPolicy_ZeroLoss | capacity 100, 10,000 items | Dropped == 0 |
| 7 | Stop_FlushesRemaining | Stop while items remain queued | line count == Written |
| 8 | Stop_ThenEnqueue_Fails | Enqueue after Stop | false (or the documented exception) |
| 9 | Stop_CalledTwice_Safe | Stop called twice | no exception, file is intact |
| 10 | Stop_WhileProducerBlocked_NoDeadlock | Stop while full under the Block policy | shuts down within 5 seconds |
| 11 | Stats_Consistency | an arbitrary scenario | Enqueued/Dropped/Written add up correctly |
| 12 | EmptyQueue_Stop_ImmediateShutdown | Stop without enqueueing anything | shuts down within 1 second, empty file |
| 13 | LargeMessage_Handled | 10 messages of 64KB | written without corruption |
| 14 | FilePath_MissingFolder | a folder that doesn't exist | a clear error (create it or throw) |
| 15 | LongRunning_MemoryStable | 5,000/sec for 30 seconds | queue length stays bounded, no memory-growth trend |

**Benchmark Procedure (follow this order exactly)**

1. Confirm a Release build, set power mode to "High performance," close other apps
2. Parameters: producer counts 1 / 4 / 16, 100-byte messages, 10 repetitions per condition (3 warmup runs)
3. Metrics: throughput (msg/s), bytes allocated per entry (C# `MemoryDiagnoser`), `Enqueue` latency p50/p99
4. Also measure **the queue's own throughput without file I/O** separately (a mode where the consumer just discards items) — to separate whether the bottleneck is the queue or the disk
5. Compute p50/p99 from sorted per-operation samples collected with `Stopwatch.GetTimestamp()`/`steady_clock`; BenchmarkDotNet/Google Benchmark summaries alone are insufficient
6. Require `bench/run.ps1` to build/run Release and save raw results

**REPORT-1-1.md Template**

```markdown
# 1-1 Log Queue Report
## 1. Environment
CPU / RAM / OS / build configuration / runtime version / commit hash
## 2. Measurement Method
repetitions, warmup, parameters, measurement tools, factors excluded
## 3. Results Table
| Version | Producers | Throughput (msg/s) | p50(µs) | p99(µs) | Allocation per entry (B) | Std. Dev |
## 4. Why the Difference Occurs (3 paragraphs)
- Paragraph 1: synchronization cost (lock contention vs. lock-free structure)
- Paragraph 2: allocation and memory locality
- Paragraph 3: the effect of the policy (Drop/Block) on throughput
## 5. Conclusion: Which Version to Use, and When
## 6. Where AI Was Wrong (at least 1)
what it said incorrectly and how you verified it
```

**Recommended Schedule**

| Day | Task |
|---|---|
| Day 8 | finalize the interface, options, and stats types; version (A) skeleton + tests 1-4 |
| Day 9 | version (B) skeleton, tests 5-9 |
| Day 10 | tests 10-15, clean up the shutdown path, confirm both versions share the same interface |
| Day 12 | run the benchmark, build the table, write the report |

**Grading**

| Item | Points | Criterion |
|---|---|---|
| Correctness | 30 | all 15 tests pass, all 3 invariants hold |
| Concurrency | 25 | 4+ on the rubric-item-2 AI review, zero loss/duplicates over 10 repetitions of 10 million round trips, no deadlock in the shutdown path |
| Measurement | 25 | Release, warmup, repetitions, variance reported; the queue-alone measurement is separated out |
| Report | 20 | numerically grounded conclusion, 1+ "thing AI got wrong" |

**Common Mistakes**

- The consumer loop never gets the `Complete()`/`Stop()` signal and never exits → check both the shutdown signal and the queue-empty condition together
- Under the `Block` policy, a producer waits forever on Stop → design Stop to wake waiting producers
- Mistaking "per-producer ordering" for "global ordering" → only **per-producer** ordering needs to hold (global ordering would require issuing sequence numbers, which costs more)
- Flushing the file on every single entry tanks throughput → flush periodically or buffer instead
### 7.3 Track Assignment 1-2. Object Pool + Allocation Measurement (required, 16h, Days 5, 13)

**Background**: in Phase 2, a packet receive buffer gets rented tens of thousands of times per second. Allocating fresh every time puts heavy pressure on the GC (or malloc). The pool you build here **gets reused throughout Phases 2-6**.

**Deliverable tree**

```
phase1/ObjectPool/
├─ src/    BufferPool implementation (as a separate library project!)
├─ tests/  8 tests
├─ bench/  a 3-way comparison benchmark
└─ REPORT-1-2.md
```

**Functional Requirements**

1. **Public API** (C#)
   ```csharp
   public sealed class BufferPool : IDisposable
   {
       public BufferPool(int bufferSize = 4096, int maxRetained = 1024);
       public byte[] Rent();                 // creates a new one if none available
       public void  Return(byte[] buffer);   // anything beyond maxRetained is discarded
       public PoolStats Stats { get; }       // Rented, Returned, Created, Discarded, CurrentRetained
   }
   ```
   (C++)
   ```cpp
   template <std::size_t BufferSize = 4096>
   class BufferPool {
   public:
       struct PoolDeleter;
       using Handle = std::unique_ptr<std::byte[], PoolDeleter>;
       explicit BufferPool(std::size_t max_retained = 1024);
       Handle Rent();                       // returns to the same pool when the handle is destroyed
       PoolStats Stats() const;
   };
   ```
2. **Thread safety**: start lock-based. If you have time to spare, do a 2-tier design of a per-thread local cache + a global pool (🟡)
3. **Guarding against misuse**
   - C# detects the same object being `Return`ed twice in Debug. C++ exposes only the move-only `Handle`, making an explicit double return impossible by type
   - Returns beyond the pool's size are discarded, incrementing `Discarded`
   - On destruction, warn-log the count of unreturned objects (C++: destructor, C#: `Dispose`)
4. **Stats**: rented / returned / created / discarded / currently retained counts

**Test List (8 tests)**

| # | Name | Expected |
|---|---|---|
| 1 | Rent_EmptyPool_CreatesNewBuffer | Created == 1 |
| 2 | ReturnThenRent_Reused | no increase in Created |
| 3 | ExceedsMaxRetained_Discarded | Discarded > 0, CurrentRetained ≤ maxRetained |
| 4 | DoubleReturn_Detected | C# throws/asserts in Debug; C++ copy/explicit-return misuse does not compile |
| 5 | MultiThreaded_RentReturn_Consistency | stats are consistent after 8 threads × 10,000 iterations |
| 6 | ReturnedBuffer_ContentResetPolicy | follows the documented policy (whether it's cleared is fixed) |
| 7 | OnDestruction_WarnsUnreturned | 1 warning log |
| 8 | WrongSizeBuffer_ReturnRejected | rejected or throws |

**Measurement Requirements**

- **C#**: a **3-way comparison** of `new byte[4096]` every time vs. your own pool vs. `ArrayPool<byte>.Shared`, using `MemoryDiagnoser`. Additionally, capture Gen0 collection-count observations via `dotnet-counters monitor --counters System.Runtime`
- **C++**: compare `new/delete` vs. pool with Google Benchmark. Use ASan for UAF/out-of-bounds; prove leak freedom with the CRT debug heap/VS memory snapshot and `Rented == Returned`, `Created == destroyed+retained` counters
- The table should include throughput, allocation per entry, and GC count (or allocation-call count) together

**Required Content of REPORT-1-2.md**: a 3-way (or 2-way) comparison table / at least 1 condition where pooling is not a net gain (e.g., when buffer lifetime is very short and size is small) / 2 sentences on how you'll use this pool in Phase 2

**Grading**: correctness 40 (8 tests) / measurement 30 (3-way comparison, GC observation) / code quality 30 (rubric items 3 and 7, split out as a reusable library)

**Important**: this project **must** be built as an independent library. Phase 2's packet library will pull it in as a project reference.

### 7.4 Track Assignment 1-3. Bulk File-Processing CLI (advanced 🟡, 12h)

**Goal**: feel the difference between sequential / async I/O / parallel processing firsthand, and experience that "throwing more threads at an I/O-bound workload doesn't make it faster."

**Requirements**

1. Write your own input-generation script: 5,000 text files (each 10-100KB, random words)
2. A word-count CLI in **three versions**
   - `--mode seq`: sequential
   - `--mode async`: C# `File.ReadAllTextAsync` + `SemaphoreSlim` to cap concurrency / C++ a thread pool or overlapped `ReadFile`
   - `--mode parallel`: C# `Parallel.ForEachAsync` / C++ N `std::thread`s + work distribution
3. Common options: `--dir`, `--concurrency N`, `--repeat R`
4. Measure: processing time, CPU usage, peak working set (`Get-Process`), the change across concurrency N=1/4/8/32
5. A 1-paragraph conclusion: under which conditions (SSD/HDD, file size, core count) which version wins. **State, with numbers, "the point past which raising concurrency stops helping"**

**Submission**: code / the generation script / a comparison table (3 versions × 4 concurrency levels) / the conclusion

### 7.5 Cross-Track Reading Assignment (for Track A parallel learners)

- Read the counterpart language's log-queue implementation (either the textbook's code, or your own track's code translated by AI) and tabulate **5 differences**: ownership/lifetime handling, shutdown handling, exceptions vs. error codes, synchronization tools, allocation placement
- Finish within 30 minutes. The goal of this assignment is "recognizing the differences," not mastering them

---

## 8. Learning Completion Assessment (Day 15, Friday)

### 8.1 Checklist

**Assignments**
- [ ] 1-C complete, `pwsh scripts/build-and-test.ps1` → `ALL GREEN`
- [ ] 1-1: 15 tests pass, `REPORT-1-1.md` complete (table, variance, conclusion, 1 AI-error case)
- [ ] 1-2: 8 tests pass, a 3-way (or 2-way) comparison table, split out as an independent library
- [ ] 🟡 1-3 (optional) done, or the reason it wasn't recorded in a retrospective

**Exercise Outputs**
- [ ] 3 sets of race-condition/deadlock/visibility reproduction-and-fix code (L-C-08~10)
- [ ] Deadlock Parallel Stacks screenshot (L-C-11)
- [ ] Processing-time table from 1→1,000 threads (L-C-07)
- [ ] Debug/Release comparison table (L-C-14)
- [ ] C#: state-machine decompilation capture, the 12-pitfall project / C++: lock-free SPSC (passes ASan), false-sharing comparison

**Environment**
- [ ] SQLite confirmed, redis-windows `PONG` confirmed, (optional) MySQL connection confirmed

**Records**
- [ ] 15 days of learning notes, 3 weekly retrospectives
- [ ] A history of revising the instruction file at least once

### 8.2 Reimplementation Exam (No AI, 60 minutes)

**Problem**: from an empty project, implement a log queue (one approach, including writing to a file) and pass the **3 provided tests** below. The tests are published in advance, so you may read them before the exam.

1. `OrderGuarantee`: 2 producers each send 1..5,000 and finish. Per-producer order in the file is ascending
2. `Flush`: enqueue 1,000 items and Stop immediately. File line count == 1,000
3. `CapacityDrop`: capacity 100, 5,000 items flooding in. `Written == Enqueued - Dropped` and Dropped > 0

**Pass criterion**: pass all 3 within 60 minutes. On failure, see §8.5.

### 8.3 Oral Exam Question Bank (AI Interviewer, 30-60 minutes)

Draw 10 random questions and grade them on a 5-point scale (Overview §5.4). An average of 4.0+ is a pass.

**Common (10)**
1. State the 4 conditions for deadlock, and explain with a code example how each is broken
2. You fixed a race condition three different ways. What's the cost and the right situation for each?
3. Explain, using your own measurements, the effect of cache lines and false sharing on throughput
4. Why look at p99 instead of the average? Is it enough to look only at p99?
5. Explain, with your own numbers, why a Debug measurement is meaningless
6. Give 3 reasons why raising the thread count above the core count slows things down
7. Why is warmup necessary (JIT, cache, branch prediction)
8. When the queue is full, would you choose Drop or Block? What about for game-server logs?
9. Give an example of a bug that a unit test can't catch, and the tool you'd use instead
10. Why is a context switch expensive, from the kernel/cache perspective?

**C# (10)**
11. What structure does `await` compile into? Who calls `MoveNext`, and when?
12. Why are there 3 GC generations, why is the LOH separate, and what does the 85KB boundary mean?
13. The difference between a `lock`-based queue and `Channel<T>`, and when you'd use which?
14. What code pattern causes ThreadPool starvation, how do you detect it, and how do you fix it?
15. The difference between `Task` and `ValueTask`. Why is awaiting a `ValueTask` twice dangerous?
16. What does `ConfigureAwait(false)` change? Is it needed in server code?
17. When does boxing happen and why is it expensive? Give 2 ways to avoid it
18. Why can't you store a `Span<T>` in a field?
19. Why is `async void` forbidden, and what's the exception?
20. The difference between `ArrayPool<T>.Shared` and a pool you build yourself — when to use which?

**C++ (10)**
21. The cost difference between `unique_ptr` and `shared_ptr`, how to choose, and when `weak_ptr` is needed
22. The condition under which the move constructor is called, and its relationship to copy elision
23. Explain the difference between acquire/release and seq_cst using an SPSC-queue example
24. The difference between `SRWLOCK` and `std::mutex`, and when a shared read lock pays off
25. Why is a predicate-less `wait` on a `condition_variable` dangerous?
26. How does RAII guarantee exception safety? What are the 3 levels of exception safety?
27. How did you eliminate false sharing in code?
28. Name 3 kinds of UB and explain a case where optimization changed the behavior
29. What kinds of errors does ASan catch, and what kinds does it miss?
30. What is the Rule of 0/3/5, and why is 0 the best choice?

### 8.4 Defect Hunting (45 minutes)

Have the agent generate a version of your own 1-1/1-2 code with **5 hidden defects** (race condition, resource leak, boundary condition, missing shutdown handling, policy violation), find 4 or more, and propose a fix for each. Use the prompt from §6.1 "Defect injection" or Overview T11.

### 8.5 Response If Standards Are Not Met

| Shortfall | Response |
|---|---|
| Reimplementation exceeded 60 minutes | retry for 30 minutes every morning for 5 days in Phase 2 Week 1. After 3 failures, relearn the queue implementation from scratch |
| Oral exam below 3.5 | 8h of remediation on 2 weak topics in Phase 2's first week, then retest |
| 1-1 report incomplete | finish by the Friday of Phase 2 Week 2 (if only the measurement is left, do that first) |
| 1-2 incomplete | **cannot pass**. Since Phase 2's packet buffer depends on this, finish it as top priority |
| Fewer than 10 days of learning notes | don't backfill from memory — instead, write every remaining day going forward |

---

## 9. Common Sticking Points

| Symptom | Cause | Countermeasure |
|---|---|---|
| BenchmarkDotNet results differ every run | Debug build, background processes, power settings, turbo boost | Release, High performance power mode, close other apps, report median/std. dev. after repeating |
| BenchmarkDotNet warns "Debug build" | ran with plain `dotnet run` | use `dotnet run -c Release` |
| The whole benchmark target disappears | the compiler eliminated the unused result | C#: store the result in a field; C++: `benchmark::DoNotOptimize` |
| C++ ASan won't turn on under MSVC | property not set, conflicts with `/RTC` | Properties → C/C++ → General → "Use AddressSanitizer," turn off `/RTC`, run in the Debug configuration |
| `dotnet-counters` can't find the process | tool not installed, wrong PID | `dotnet tool install -g dotnet-counters`, `dotnet-counters ps` |
| The `Channel` consumer never shuts down | missing `Writer.Complete()` | call Complete in Stop and confirm the `ReadAllAsync` loop exits |
| Missed `condition_variable` notifications | predicate-less wait, spurious wakeup | only ever use the `wait(lock, pred)` form |
| Stop hangs under the Block policy | waiting producers never get woken | set the shutdown flag then `notify_all` in Stop (for C#, Channel Complete) |
| Lines get interleaved in the file | 2+ consumers, or a shared file stream | there must be exactly 1 consumer, with an explicit stream owner |
| Tests fail occasionally (flaky) | `Thread.Sleep`-based synchronization | use an event/`TaskCompletionSource`/`future` for deterministic waiting |
| CMake can't find the vcpkg packages | toolchain file not specified | `-DCMAKE_TOOLCHAIN_FILE=<vcpkg>/scripts/buildsystems/vcpkg.cmake` |
| `git status` shows build artifacts | missing `.gitignore` entries | add `bin/ obj/ build/ .vs/ BenchmarkDotNet.Artifacts/`, then `git rm -r --cached` |
| The agent's generated code is too big to read | generated the whole thing at once | re-check the "one file at a time" rule in the instruction file, split the request into steps (§6.1 pair-mode prompt) |
| The agent reports "done" without tests | the instructions were vague | state explicitly in the instructions "no reporting done without tests," and call it out on violation |
| Can't reproduce the deadlock | timing issue | insert `Sleep(1)` between the locks, increase the repetition count |
| The visibility problem doesn't reproduce on x86 | x86's strong memory ordering | that's normal. Research "why does it break on ARM" and note it |

---

## 10. Preparing for Phase 2

- [ ] Clone the textbook "Game Server Development, Starting with Understanding Networks" locally (to read through on Phase 2 Day 4)
- [ ] Split Assignment 1-2's buffer pool out as a **standalone library project**, and confirm it can be referenced from another solution
- [ ] Leave room in `scripts/build-and-test.ps1` to add a `phase2` path later (parameterize it)
- [ ] Confirm the port range Phase 2 will use (e.g., 9000-9100) isn't blocked by the firewall. Add an inbound rule if needed
- [ ] Check your current dynamic-port settings to guard against port exhaustion during local loopback testing: `netsh int ipv4 show dynamicport tcp`
- [ ] Write up your Phase 1 retrospective in your retrospective notes: the point you got stuck on longest, 1 case where you misused AI, and 1 habit to carry into the next Phase

---

## 11. 2026-09-05 Revisions (this section wins on conflicts)

### 11.1 Realistic schedule and required labs

- Budget each 40h week as concepts 12h, labs/assignments 14h, no-AI reimplementation 5h, review/evaluation 4h, buffer 5h. Nominal assignment hours include reading, reimplementation, and review; implementation slots are 1-C 6h, 1-1 14h, and 1-2 10h
- Day 6 morning adds 2h on worker exceptions, sink failure, disk-full policy, and fake-sink tests. Day 7 adds 30 minutes for `appsettings.json`/`IOptions<T>` or JSON/TOML config loading
- Day 12 compares Serilog `Sinks.Async(blockWhenFull)` and spdlog `block/overrun_oldest` against the custom queue, and practices an `ILogSink` test double, hang dump, and Release+PDB debugging
- Day 14 C++ requires `L-CPP-11` and `L-CPP-12` (90 min). C# completes two of `L-CS-10/11/12`. `L-CS-08` requires only five traps reproducible in a console app
- Reserve a half-day buffer every week. Move Day 9 book read-through to evening/weekend reference mode and Day 10 long-running tests to Day 12

### 11.2 Tooling, memory, and testing

- MSVC ASan detects UAF/out-of-bounds but provides neither LeakSanitizer nor data-race detection. Judge leaks with `_CrtSetDbgFlag`, VS memory snapshots, and rent/return plus create/destroy counters
- Cover C# false sharing with explicit layout/padding, AoS vs SoA, `Volatile.Read/Write`, `Interlocked`, and .NET 9+ `System.Threading.Lock`. Connect C++ `InitOnce`, events, `WaitForMultipleObjects`, and `WaitOnAddress`/`atomic::wait` to shutdown and Block waits
- Apply monotonic clocks, temp-directory fixtures, per-test timeouts, slow-test categories, and injectable clocks. Practice `git bisect`, `reflog`, `revert`, tags, a pre-commit build/test hook, LF `.gitattributes`, and include `global.json` in 1-C
- Add a sink-exception survival/error-stat test. Treat 300k msg/s for C# lock and 5M msg/s for C++ SPSC only as environment-dependent reference values

### 11.3 Evaluation and AI checks

- Add questions on volatile vs atomicity, false sharing, monotonic clocks, test doubles, hang dumps, and ASan limitations. Challenge claims that lock-free is always faster or ASan detects leaks/races with official contracts and measurements
- `REPORT-1-2.md` must state how an IOCP receive buffer is owned until OVERLAPPED completion, including size, slices, worker count, and rental frequency
