# Template Collection

> 🇰🇷 Korean version: [08-templates_kr.md](08-templates_kr.md)

> These are document templates reused throughout the course. Copy each template as-is and use it. Feel free to trim any item that takes more than 5 minutes to fill in, but never leave the "why" field blank.

---

## T1. Daily Study Note — `notes/daily/YYYY-MM-DD.md`

```markdown
# 2026-MM-DD (Phase N, Week W, day of week)

## Today's Goals (write in the morning)
- [ ] 
- [ ] 

## Morning Quiz (AI-generated, covering previous day's scope)
- Score: /5
- Wrong answers and reasons:

## What I Learned (up to 3 key points, in my own words)
1. 
2. 
3. 

## What I Didn't Know / What's Confusing
- 

## What the AI (or textbook) Got Wrong and How I Found It
- Example: "The AI said ReceiveAsync always completes asynchronously, but per the official docs it completes synchronously when it returns false" — verified on learn.microsoft.com

## What I Did in 1 Hour Without AI
- What I reimplemented and the result (success/partial/failure):

## Assignment Progress
- Assignment number: 
- What I did / where I got stuck / what's next:

## Tomorrow's Plan (write in the evening)
- 
- Morning quiz scope:
```

---

## T2. Weekly Retrospective — `notes/weekly/W##.md`

```markdown
# W## Retrospective (Phase N)

## Weekly Check Results
| Item | Result |
|---|---|
| Reimplementation without AI (60 min) | Component: / Tests passed: /N |
| Explanation test (3 topics) | Average score: / Weakest topic: |
| Top 3 issues from code review | 1. 2. 3. (fix commit: ) |

## What Went Well
- 

## What I Got Stuck On and Why
- 

## What I Learned About Using AI
- Cases where I used it well:
- Cases where I used it poorly (over-reliance on AI / missed verification):
- Adjustments to AI usage rules for next week:

## Time Usage
- Actual h out of planned 40h. Unfinished items and catch-up plan:

## Adjustments for Next Week
- 
```

---

## T3. CLAUDE.md for Learning (C# version)

> This template is an example based on Claude Code. If you use another coding agent such as Codex CLI, write the same content in that tool's instruction file (e.g., `AGENTS.md`).

```markdown
# Project: <name> (for learning)

## Purpose
This project is a learning exercise. The learner (me) understanding all the code takes priority over completing the code.

## Environment
- Windows 11, .NET 10 SDK, Visual Studio 2026 / VS Code
- Tests: xUnit. Benchmarks: BenchmarkDotNet (Release only)
- No Docker. No CI (build/test via local scripts). Default DB is SQLite (file-based); optionally MySQL 8 (swap implementation via Repository interface + DI). Redis runs locally via redis-windows(https://github.com/redis-windows/redis-windows)

## Structure
- src/<Project>.Net : Network layer (sessions, send/receive)
- src/<Project>.Logic : Game logic (rooms, users, matching) — must not reference sockets
- src/<Project>.Rules : Game rules — no external dependencies
- tests/ : Unit and integration tests
- docs/ : DESIGN.md, PROTOCOL.md, adr/

## Coding Rules
1. Enable nullable; treat warnings as errors
2. async methods use the Async suffix; async void is forbidden
3. Document the owner of shared state in a comment. No locks in room logic (use a job queue instead)
4. Socket buffers must always be borrowed from and returned to the pool
5. Catch exceptions only at boundaries (handler entry points) and log with Serilog

## Agent Behavior Rules (for learning purposes)
- When generating code, add a one-line comment to each class/major function explaining "why it was designed this way"
- Create only one file at a time. Stop before moving on to the next file
- If I give the next instruction without explaining the previous work, confirm the gist of the previous work with a one-sentence question before proceeding
- When adding a new NuGet package, include the reason for choosing it and one alternative
- Do not report functional code as complete without tests
- If I request "defect injection" mode, never reveal the location of the answer until I say "reveal the answer"
- If I attach a design document, point out any discrepancy with the document before proposing a different structure
```

## T3'. CLAUDE.md for Learning (C++ version) — differences only

```markdown
## Environment
- Windows 11, Visual Studio 2026 (MSVC, C++20/23), CMake 3.28+, vcpkg manifest mode
- Tests: GoogleTest. Benchmarks: Google Benchmark (Release only). ASan always on in Debug configuration
- Networking: Winsock2 + IOCP implemented directly (no Boost.Asio — except in a separate project for comparison)

## Coding Rules
1. /W4 /permissive-, treat warnings as errors
2. Do not call new/delete directly. Default to unique_ptr; use shared_ptr only when ownership sharing is required, with a comment
3. The lifetime of every buffer pointed to by OVERLAPPED/WSABUF must be guaranteed until after the completion notification (state the owner)
4. No mutex in room logic (use a job queue instead). When using atomic, comment the reasoning behind the chosen memory_order
5. UB candidates (uninitialized reads, sign overflow, dangling references) are top priority in review

## Additional Agent Behavior Rules
- When adding a new vcpkg dependency, note the reason, alternatives, and impact on build time
- When using Win32 APIs, don't forget to handle GetLastError/WSAGetLastError on failure
```

---

## T4. Design Document — `docs/DESIGN.md`

```markdown
# <Server Name> Design Document

## 1. Goals and Non-Goals
- Goals:
- Non-goals (not doing this time):

## 2. Constraints
- Environment (Windows, single-process/multi-process), number of connections, number of rooms, latency target, restrictions on libraries used

## 3. Components
(Diagram: network, session, lobby, matching, room, rules, timer, log, config)
| Component | Responsibility | Dependencies |
|---|---|---|

## 4. Threading Model
(Diagram: which thread runs what, queues between threads)
- Chosen model and rationale (ADR link)
- Each thread's role, count, and when it is created/terminated

## 5. Data Ownership
| Data | Owning Component | Thread Allowed to Modify | Read Permitted | Notes |
|---|---|---|---|---|

## 6. State Machines
- Session / room / matching state transition tables (state, event, next state, action, handling of illegal transitions)

## 7. Packet List
(Link to PROTOCOL.md + table of packets this server handles: ID, direction, allowed states, handler)

## 8. Key Sequences
- Sequence 1: 
- Sequence 2: 
- Sequence 3: 

## 9. Risks and Mitigations
| Risk | Trigger Condition | Mitigation |
|---|---|---|

## 10. What Changed After Implementation (write once implementation is complete)
| Item | Design | Actual | Reason |
|---|---|---|---|
```

---

## T5. ADR — `docs/adr/NNNN-<title>.md`

```markdown
# ADR-NNNN: <Decision Title>

- Date: 
- Status: Proposed / Accepted / Deprecated / Superseded (ADR-XXXX)

## Context
What problem or constraint makes this decision necessary.

## Decision
What was decided to do (one paragraph).

## Alternatives Considered
| Alternative | Pros | Cons | Reason for Rejection |
|---|---|---|---|

## Consequences
- What this decision makes easier:
- What it makes harder / risks accepted:
- What would need to change to reverse it:

## AI Review Log
- Weaknesses the AI pointed out and my judgment (accepted/rebutted):
```

---

## T6. Load Test / Performance Improvement Report — `PERF-REPORT.md`

```markdown
# Performance Report: <Target Server> (<Date>)

## 1. Environment
- Server machine (CPU/RAM/OS), client machine, network (loopback/LAN), build configuration (Release), version/commit

## 2. Scenario
- Tool: (5-1 dummy client) / scenario name / number of connections / ramp-up / hold duration / repetition count

## 3. Baseline
| Metric | Run 1 | Run 2 | Run 3 | Median | Deviation |
|---|---|---|---|---|---|
| TPS | | | | | |
| p50 / p99 (ms) | | | | | |
| Error rate | | | | | |
| CPU / Memory | | | | | |

## 4. Hypotheses and Experiments
| # | Hypothesis | Verification Method | Result | Accepted/Rejected |
|---|---|---|---|---|
| 1 | | | | |
| 2 | | | | |
| 3 | | | | |

## 5. Changes Applied
- Change 1: what / why / commit
- Change 2:

## 6. Final Before/After
| Metric | Before | After | Change |
|---|---|---|---|

## 7. Side Effects / Costs
- Changes in memory/CPU/code complexity:

## 8. Remaining Bottlenecks and Next Candidates
- 
```

---

## T7. Runbook — `RUNBOOK.md`

```markdown
# Runbook: <Service Name>

## Service Configuration
| Service | Machine | Port | Run Account | Log Path | Health Check |
|---|---|---|---|---|---|

## Common Commands
- Check status: `Get-Service <name>`; start/stop: `Start-Service`/`Stop-Service`
- Recent logs: `Get-Content <path> -Tail 200`
- Dashboard: <URL>, Alert list: <URL>

## Incident 1: <Name> (e.g., MySQL not responding, if MySQL was chosen)
- Detection: Alert <name> / Symptoms (spike in login failures, dashboard panel X)
- Diagnosis (within 5 minutes):
  1. `Get-Service MySQL80` status (if MySQL was chosen)
  2. `DbTimeout` count in API server logs
  3. DB latency panel on the dashboard
- Response:
  1. Restart the service: `Restart-Service MySQL80`
  2. Check circuit breaker status (wait for automatic half-open)
- Recovery confirmation: login success rate back to normal, alert cleared
- Post-incident: write a postmortem, list items to prevent recurrence

## Incident 2: ...

## Rollback Procedure
- 

## Contacts/Escalation (if studying solo, "a note to my future self")
```

---

## T8. Postmortem — `docs/postmortem/YYYY-MM-DD-<title>.md`

```markdown
# Postmortem: <Title>

- Occurred: / Detected: / Resolved: / Duration of impact:
- Impact: (number of connections, number of failed requests)

## Timeline
| Time | What Happened | Action Taken |
|---|---|---|

## Root Cause
## Why Detection Was Slow/Fast
## What Went Well / What Didn't
## Improvement Items (owner, deadline)
```

---

## T9. AI Interviewer Prompt (End-of-Phase Explanation Test)

```
You are a technical interviewer conducting a job interview for an entry-level online game server developer position. Over 30 minutes, ask in-depth questions within the following topic scope.
Scope: <paste the list of example explanation-test questions for Phase N>
Rules:
- One question at a time. Follow up on my answer with up to 3 follow-up questions.
- If an answer is vague, demand that I "explain concretely with code/numbers."
- If I say I don't know, move to the next question without giving hints.
- At the end, score each question on a 5-point scale (5: principle, rationale, alternatives, and limitations all correct / 4: principle and rationale correct, alternatives or limitations weak / 3: concept correct but insufficient rationale / 2: partially incorrect / 1: unable to explain), then tell me the average and the 2 weakest topics.
- Scoring table format: | Question | Score | Rationale |
```

## T10. AI Quiz-Setter Prompt (Daily Morning Quiz)

```
Yesterday's study scope: <3 "What I Learned" items from notes + textbook chapter>
Create 5 questions from this scope: 2 multiple choice (4 options, with plausible wrong answers), 1 short answer, 1 essay (asking "why"), 1 code-defect-finding question (a code snippet under 10 lines with 1 defect).
Show all of them at once, and when I send my answers, grade and explain them. For wrong answers, tell me the relevant textbook chapter or official documentation keywords.
```

## T11. AI Defect Injection Prompt

```
Based on the following requirements/code (attached), create a version with 5 hidden defects.
Defect types (one each): <e.g., race condition, resource leak, boundary condition error, missing error handling, protocol rule violation>
Rules:
- Defects should compile and work correctly in most cases, only revealing themselves under specific conditions.
- Never mention the answer (location/type) until I say "reveal the answer." If I ask for a hint, give only the "type."
- When I report a defect I found, only tell me whether I'm right or wrong; if wrong, just say "no" without giving any reasoning.
```

## T12. AI "Developer Seeing This for the First Time" README Reproduction Test Prompt

```
You are a developer seeing this project for the first time today. Using only the attached README, describe step by step the procedure to run it locally.
Do not actually run the commands — judge based on the document alone. Point out the following:
1) Things you have to guess because they're not in the document
2) Steps that seem out of order or missing
3) Points that seem like they won't finish within 10 minutes
```
