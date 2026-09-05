# Textbook Usage Guide — `jacking75/programming-books-with-ai`

> 🇰🇷 Korean version: [07-books-guide_kr.md](07-books-guide_kr.md)

> Repository: https://github.com/jacking75/programming-books-with-ai
> This document selects, from the books in the repository, the ones used in this curriculum (C#/C++ track, Windows, no Docker), and guides **when, what, how, and how much** to read from them. It pairs with the "Textbook Usage" section of each Phase document.
> Environment for this curriculum: Visual Studio 2026 / .NET 10 SDK, building and testing with local scripts instead of CI, the DB abstracted via the Repository pattern + DI (SQLite by default, MySQL 8.0 optional), and Redis via redis-windows (a native Windows binary). Packet capture (Wireshark) exercises and deployment (Windows services / CI/CD) are not covered, and the default AI coding agent is Claude Code, though other agents such as Codex CLI may also be used.

---

## 1. The Nature of This Repository and How to Approach Reading It

### 1.1 What Kind of Books Are These
- This is a collection of books created by the author asking AI "what kind of book to make and what content should go into it," then editing the text the AI wrote. The repository README explicitly states, **"There may be errors in the example code, so please get help from AI."**
- Because they are written in Markdown, **they can be fed directly into an AI's context.** This is the biggest difference from paper books, and the reason this curriculum uses this repository as its textbook source.
- Each book differs in completeness, length, and baseline version (.NET 8/9/10, C++20/23/26). This is noted on each book's card (§4).
- Only some books have an example code folder (TCP/IP sockets, C# Socket, SuperSocketLite, the network library analysis, the two Async books, 2D MMORPG, ECS, the ASP.NET WebAPI game server, Boost.Asio). The rest only have code embedded in the body text.

### 1.2 Reading Attitude (Important)
1. **Do not read a book cover to cover.** Read only the chapters each Phase document specifies, at that point in time. This curriculum does not require reading an entire book.
2. **Always build the examples.** If a build error occurs, that is your first debugging exercise. Record what you fixed (the diff) and the root cause in your learning notes. The books in this repository are "books you read by fixing."
3. **A book's explanation is not necessarily correct.** Cross-check it against official documentation (learn.microsoft.com, cppreference, the MySQL/Redis docs, RFCs). If they differ, follow the official documentation and note the discrepancy. The "Where AI got it wrong" column in your learning notes should also include errors found in the books.
4. **Do not paste a book's code directly into your assignment.** Most books use "gacha-collection RPG," "chat," or "FPS matching" examples, while the assignment is an "Omok (Gomoku) lobby/game." The process of rewriting the code for a different domain is where the learning happens.
5. **You can send a PR.** If you fixed an error, it is recommended that you submit a PR to the repository. It also becomes part of your portfolio.
6. **Version differences are expected.** Most books in this repository were written around VS 2022 / .NET 9 (some around .NET 8 or 10). Since this curriculum uses VS 2026 / .NET 10, the default approach when opening a book's project is to migrate it to VS 2026 and bump `TargetFramework` to `net10.0` before building. Since this curriculum uses SQLite as the default DB and MySQL as an optional choice (switchable via DI), when a book's code assumes MySQL, the default approach is to take only the SQL and design concepts and move the connection code over to SQLite for practice (when MySQL is chosen instead, use the book's code as-is). For installing Redis, even if a book guides you through Docker/WSL2/Memurai, this curriculum uses redis-windows (https://github.com/redis-windows/redis-windows).

### 1.3 Three Reading Modes
| Mode | When | Method | Time |
|---|---|---|---|
| Full read | Core chapters designated at the start of a Phase | Read during the day's concept-study time (2h), with 5 AI quiz questions at the end of each chapter | 1-2h per chapter |
| Reference | When stuck while implementing an assignment | Find and read only the relevant section, and compare the book's code with your own | Within 30 minutes |
| Code analysis | Books with a code folder | Build → run → trace the flow with breakpoints → reconstruct the structure as a diagram → comparison table against your own design | 2-4h |

---

## 2. Protocol for Reading Books with AI

Place the book's Markdown file in your Claude Code working folder (or attach it to a chat AI) and proceed in the order below.

### 2.1 Standard Procedure for Reading One Chapter (about 90 minutes)
1. **Skim (10 min)**: Look only at the headings and write down "3 questions this chapter answers" yourself.
2. **Read (40 min)**: Read the body text, marking unfamiliar terms and claims.
3. **Explain (15 min)**: Tell the AI, "I will explain the core of this chapter: ... point out what's wrong and what's missing."
4. **Verify (15 min)**: Check 2 of the marked claims against official documentation or by running code.
5. **Quiz (10 min)**: Ask the AI to write and grade 5 questions covering this chapter, and record the score in your notes.

### 2.2 Example Prompts

Generating questions instead of a chapter summary
```
The attached Markdown is Chapter <N> of "<book title>." Do not summarize it.
Create 5 questions that can verify whether I properly understood this chapter (2 recall-type, 2 principle-type, 1 application-type).
When I answer, grade them, and for wrong answers only, quote the relevant part of the text and explain.
```

Cross-checking the book against official documentation
```
This chapter (attached) explains that "<claim sentence>." Please verify whether this is accurate according to official documentation, and whether it varies by version. Also provide the URL of the documentation you used to verify it.
```

Example build error
```
I built the Chapter <N> example from "<book title>" on .NET 10 (or VS 2026 C++20) and got the following error: (paste error)
Don't fix it right away. Tell me 1) 2 candidate causes, 2) what I should check, and 3) the direction to fix it after checking.
```

Analyzing a code folder
```
The attached source (e.g., FastSocketLite.txt) is an entire C# network library.
1) Summarize, in function-call order, the path that incoming data takes from the socket to the application handler.
2) Summarize this library's threading model in one sentence, and tell me the code locations that support it.
3) 3 differences from my server (design document attached).
```

Porting the book's code to my domain
```
The "shop purchase" code in this chapter is based on a gacha-collection RPG. My project is an Omok lobby (currency: coins, products: profile borders, spectator passes).
Instead of porting the code for me, make a table of what needs to change when porting it (schema, validation, transaction boundaries) versus what stays the same. I will do the implementation myself.
```

### 2.3 Things Not to Do
- Having the AI summarize the whole chapter and only reading that (that is not the same as reading it)
- Just telling the AI to "fix the book's example so it works" and moving on without understanding the cause
- Having multiple books open at the same time (outside the chapters designated by the Phase document, use only reference mode)

---

## 3. Phase × Textbook Matrix

◎ Primary textbook (full read of designated chapters) / ○ Supplementary (full read of designated chapters, or reference) / △ Optional/reference / — Not used

| Textbook | P1 | P2 | P3 | P4 | P5 | P6 | Track |
|---|---|---|---|---|---|---|---|
| Understanding Game Server Development, Starting from Networking | | ◎ | | | | ○ (review) | Common |
| Network Theory Every Game Server Developer Should Know | | ◎ | | | | ○ (review) | Common |
| Network Learning Roadmap for Online Game Developers | | ○ | ○ | △ | ○ | ○ | Common |
| Network Knowledge for Online Game Client Developers | | | ○ | | | △ | Common |
| TCP/IP Windows Socket Programming Every Game Server Developer Should Know | | ◎ | ○ | | ○ | | C++ |
| C# Socket Programming for Game Server Development | | ◎ | ○ | | ○ | | C# |
| C# Game Server Programming Using SuperSocketLite | | ○ | ○ | | | | C# |
| An Analysis of C# Network Libraries for Study | | | ○ | | | | C# |
| Mastering C# Async/Await | ◎ | | ○ | | ○ | | C# |
| Building a C# async/await Library | ○ | ○ | ◎ | | ○ | △ | C# |
| C# Design Patterns for Game Developers | ○ | | ◎ | | △ | △ | Common (concepts) |
| Essential C# Guide for ASP.NET Core Web API | ○ | | | ◎ (mixed) | | | C#/Mixed |
| Building an API Game Server with ASP.NET Core Web API | | | | ◎ | ○ | | C#/Mixed |
| Building a Game Server with ASP.NET Core Web API | | | | ○ | ○ | | C#/Mixed |
| Learn MySQL C# Programming in a Week | | | | ○ (if MySQL chosen) | | | Common (SQL) + C# |
| Learn Redis Programming in a Week | | | | ◎ | | ○ | Common + C# |
| Practical Guide to Implementing Gacha Probability for New Game Programmers | | | | ○ | △ | △ | Common |
| Building a 2D MMORPG Game Server | | | | ○ | | ◎ | C# |
| An ECS-Based Online Game Server | | | △ | | | △ | C# |
| Building an FPS Game Matching System | | | ○ | | | ◎ | Common |
| Modern Windows Multithreading | ◎ | ○ | ◎ | | ◎ | ○ | C++ |
| Modern Win32 API Programming for Game Server Developers | ◎ | | ○ | | ◎ | | C++ |
| The Complete Guide to the C++23 Memory Model | ○ | | | | ○ | | C++ |
| Thread-Local Storage in Practice | | | | | ○ | | C++ |
| Safe and Elegant Programming with Modern C++ | ◎ | | | | ○ | △ | C++ |
| Safe and Easy C++ Programming, Starting with Modern C++ | △ (basic reinforcement) | | | | | | C++ |
| Building an Online Game Server with C++ Boost.Asio | | △ | △ | | | △ | C++ |
| Modern C++ Programming as Safe as Rust | | | | | | △ | C++ |
| WinRT in Practice: Networking and Multithreaded Programming | | △ (design chapters) | | | | | C++ |
| Game Server Monitoring Using Prometheus | | | | | ◎ | ○ | Common |
| Log Collection and Forwarding Using fluentd | | | | | ○ | | Common |
| Getting Started with AI Coding Agents Using OpenCode | ○ | | | | | | Common |
| Building an Online Game Client with MonoGame | | | △ | | | △ | C# |
| Behavior Tree AI for Online Game Servers | | | | | | △ | C# |
| Building an Online Game Using C# and P2P Communication | | | | | | △ | C# |
| Docker for Game Server Developers | — | — | — | — | — | — | (Not used) |
| Books on Rust, Go, TypeScript, Python, DuckDB, TimescaleDB, eBPF, OpenSiv3D | — | — | — | — | — | — | (Out of scope) |

---

## 4. Book Cards (When, What, How, Notes)

### 4.1 Network Theory (Common)

**Understanding Game Server Development, Starting from Networking** — `게임_서버_개발_네트워크부터_이해하기/README.md`
- When: The first two days of Phase 2 (Week 4, Mon-Tue), Phase 6 interview review
- What: All 5 chapters (terminology, OSI/TCP-IP, the journey of a packet, key concepts, basic programming concepts). About 2,800 lines, no examples
- How: Use it like a glossary. Turn Chapter 1's terminology into 5 AI quiz questions each day. Chapter 5 (blocking/non-blocking, I/O multiplexing, sync/async) is the starting point for the Phase 2 "server I/O model" study
- Note: Since this is an introductory book, the depth is shallow. Where the "why" is lacking, supplement it with the roadmap book or RFCs

**Network Theory Every Game Server Developer Should Know** — `게임_서버_개발자가_알아야할_네트워크_이론/README.md`
- When: Phase 2, Week 4, Wed-Thu
- What: The entire book (about 2,000 lines). OSI/TCP-IP, the 3-way/4-way handshake, reliability/flow/congestion control, Nagle, MSS/MTU, UDP
- How: **Verify each item by reproducing it through log/netstat observation or a self-made test client**, and attach your own observation results next to the book's diagrams. The bar for "having read" this book is the 3 kinds of observation/reproduction (handshake, termination, Nagle comparison)
- Note: The author explicitly states it was "mostly created with AI." Re-verify congestion control algorithm names and behavior against official sources

**Network Learning Roadmap for Online Game Developers** — `온라인게임-개발자를_위한_네트워크_학습_로드맵/READEME.md`
- When: Phase 2 (Ch. 1, 2, 7), Phase 3 (Ch. 3, 4, 5), Phase 5 (Ch. 5, 6), Phase 6 (Ch. 8)
- What: Level 0 terminology + Chapters 1-8. At about 12,000 lines, this is the largest of the repository's network books
- How: Never read it cover to cover. Read only the chapters designated per Phase. After reading each chapter, make a judgment table of "needed / not needed for the Omok server" — this is how you use it (training your applicability judgment). Chapter 2's TCP vs UDP selection criteria, Chapter 3's Server Authoritative concept, and Chapter 6's authentication/DDoS are recurring topics in the explanation exam
- Note: The filename is `READEME.md` (a typo). Chapter 8's heading Markdown is broken. The cloud (GameLift/Agones/Kubernetes) section is out of scope for this curriculum, so file it under "post-employment reading list"

**Network Knowledge for Online Game Client Developers** — `온라인게임-클라이언트_개발자를_위한_네트워크_지식/README.md`
- When: Phase 3, Week 10 (Ch. 4, 5), Phase 6 interview (Ch. 6)
- What: Synchronization, lag compensation, cheating, and genre-specific characteristics from the client's perspective
- How: Read it to understand "if the server sends things this way, what should the client do." Useful for the 3-3 mini client design and for preparing the interview question "what if the genre were different?"
- Note: Server implementation is not covered

### 4.2 Socket Programming

**TCP/IP Windows Socket Programming Every Game Server Developer Should Know** (C++) — `게임_서버_개발자_알아야할_TCPIP_Windows_소켓_프로그래밍/Chapter01~14.md`
- When: Phase 2 primary textbook (Ch. 1-6, 8-11, 13), Phase 3 (Ch. 11, 14), Phase 5 (Ch. 12, 13)
- What: Winsock basics → data transfer → multithreading → socket options → I/O models → select-based chat → IOCP chat → zero-copy → buffer pooling → latency/batch processing. Based on C++23, Windows 11, VS 2022
- How: **Code analysis mode.** Build and run `codes/tcp_server_01~03`, `first_IOCP`, `SelectChatServer`, and `IocpNetLib` (Echo, IocpChatServer) in order, and compare against the puml sequence diagrams. Do all of the "hands-on" sections in Chapters 5-6. After understanding this book's IOCP structure, **rebuild your own IOCP server from an empty project** (do not copy)
- Note: README.md is a merged file of all chapters, so it's redundant — read the per-chapter files instead. Chapter 7 (UDP) is concept-only. The `infographic/` PNGs are reference material for presentations. SelectChatServer's test client is written in C#

**C# Socket Programming for Game Server Development** — `게임_서버_개발을_위한_CSharp_Socket_프로그래밍/01~12.md`, `FreeNetLite.md`
- When: Phase 2 primary textbook (Ch. 1-7, 9, 10, FreeNetLite), Phase 3 (Ch. 11 Omok), Phase 5 (Ch. 8 performance testing, appendix)
- What: Socket basics → synchronous → asynchronous (TAP) → high-performance techniques → data processing/framing → library design → performance testing → chat server → advanced SAEA → Omok server. Based on .NET 9. The target audience is "a CS student who knows C#," which matches this curriculum's learners
- How: Build `codes/7장`, `codes/9장`, `codes/10장` (NetServerLib, ChatServer) to grasp the structure, then connect `codes/FreeNetLiteRe`'s EchoServer/test_client to your own test client. Make a comparison table between the book's NetServerLib structure and your own 2-1 design. Chapter 11's Omok has no code folder, so read the text and write up the threading-model differences as an ADR
- Note: The README discloses the generation prompt (so you can see what intent the book was made with). Check the appendix's ".NET 9 new networking features" section against the actual version

**C# Game Server Programming Using SuperSocketLite** — `SuperSocketLite를_이용한_CSharp_게임_서버_프로그래밍/Chapter01~07.md`
- When: Phase 2, Week 7 (Ch. 1-6), Phase 3, Weeks 9-10 (Ch. 7)
- What: Builds echo, chat, a test client, MemoryPack, and an Omok server/client on top of a library (a port of SuperSocket 1.6). .NET 8 or later
- How: Use this to see the difference between "a server you built yourself" and "a server on top of a library." Build `codes/OnlineOmok` (server on net9.0 + WinForms client) and play it. You're also allowed to modify this WinForms client to fit your own protocol and use it in place of 3-3. Apply Chapter 6's MemoryPack packet-definition approach in 2-2
- Note: The server example depends on MySQL/NLog (a preview of Phase 4). Also check the external repository `jacking75/SuperSocketLite` and its DeepWiki documentation. The README is a merged file excluding Chapter05

**An Analysis of C# Network Libraries for Study** — `CSharp_학습을_위한_CSharp네트워크라이브러리_분석/`
- When: Phase 3, Week 11
- What: An analysis comparing the structure, receive/send flow, and strategy of three libraries — FastSocketLite, LiteNetwork, laster40Net — plus source (`edu_*`) and a merged single-file `*.txt`
- How: **The best material for the AI code-analysis protocol (§2.2).** Give the `*.txt` to the AI and extract "the receive-path function order," "the threading model in one sentence," and "3 differences from my server." Make a comparison table of the three libraries' threading models against your own room-actor server
- Note: This is more of an analysis document than a book. It is not on the completed list

### 4.3 C# Language, Async, and Patterns

**Mastering C# Async/Await (based on .NET 10)** — `CSharp-Async_Await_완전_정복/`
- When: Phase 1 primary textbook (Ch. 1-4, 6, 7), Phase 3 (Ch. 10), Phase 5 (Ch. 5, Appendix B)
- What: State machines, SynchronizationContext, ConfigureAwait, the Awaitable pattern, async in servers, 12 pitfalls, async synchronization objects, AsyncLocal, new .NET 10 features, practical patterns, a debugging cheat sheet. About 3,300 lines, densely packed
- How: Give the checklist at the end of each chapter to an AI interviewer for an oral exam. Build `samples/AsyncAwaitBook.sln` (.NET 10, C# 14). For Chapter 6's 12 pitfalls, **create a project that reproduces and fixes each one** (a Phase 1 assignment item)
- Note: Based on .NET 10. If the SDK is unavailable, lower `TargetFramework` to `net9.0` and flag any .NET 10-only APIs. The target audience is "1+ year of C# experience," so if your grammar/syntax is shaky, read the Essential C# Guide first

**Building a C# async/await Library — Learning .NET 10's Async Internals by Implementing Them Yourself** — `CSharp-Async_Await_라이브러리_만들기/README.md`
- When: Phase 1 (Ch. 1-6), Phase 2 (Ch. 7), Phase 3 primary textbook (Ch. 16-18), Phase 5 (Ch. 14, 19, 20), Phase 6 (Ch. 13, 15)
- What: Task/ValueTask internals, dissecting the state machine, custom Awaitables, the Socket async pattern, rate limiters/retries/circuit breakers, Channel actors, the Room/Session pattern, backpressure, leak hunting, benchmarking. 20 chapters, each with learning goals and exercises
- How: Build `samples/AsyncAwaitLab.sln` (`src/Chapter01~20`) and **treat the exercises as assignments.** Chapters 16-18 are the direct textbook for the Phase 3 room actor, and Chapter 14 is the textbook for implementing the Phase 5 resilience patterns yourself
- Note: Read the build FAQ in samples/README first (switching to net9.0 if the .NET 10 SDK isn't installed, BenchmarkDotNet NU1100, CS9057). This is intermediate-to-advanced, so in Phase 1 read only Chapters 1-6

**C# Design Patterns for Game Developers** — `게임_개발자를_위한_CSharp_디자인_패턴/`
- When: Phase 1 (Ch. 3), Phase 3 primary textbook (Ch. 7-12, 16), Phase 5 (Ch. 3, 16 reread), Phase 6 (Ch. 17)
- What: Creational, structural, and behavioral patterns + game-specific patterns (Game Loop, Update Method, Service Locator, MVC/MVP) + a mini RPG + anti-patterns + a selection guide. .NET 9. No code folder
- How: Map Command to Job, State to your state machine, and Game Loop/Update to your tick, and note in your design document where you used each in your own server. **The C++ track should also read this as a conceptual textbook and port the ideas to C++**
- Note: Some Unity-specific caveats are mixed in (irrelevant to servers). To avoid using patterns for their own sake, read Chapter 16 (anti-patterns) alongside

**Essential C# Guide for ASP.NET Core Web API** — `ASPNETCore-ASPNETCoreWebAPI를_위한_필수_CSharp_가이드/README.md`
- When: Phase 1 (a quick review read for the C# track), Phase 4, Week 12, first week (mandatory 20h for the C++ mixed path)
- What: 27 sections condensing the C# syntax needed for Web API (about 1,800 lines). Includes an accompanying pptx
- How: Mark only the sections you don't know and type out the examples yourself. You must build Sections 12 and 27 (the practical examples)
- Note: Since this is a syntax summary, it lacks depth. Get deeper content from the Async books or official documentation

### 4.4 C# Game Server in Practice

**Building a 2D MMORPG Game Server** — `2D_MMORPG_게임_서버_개발/01~08.md`, `code/`
- When: Phase 4, Week 14 (Week 2 of the book), Phase 6 primary textbook (Weeks 1-4, 8 of the book)
- What: A three-part solution — a SuperSocketLite game server + an ASP.NET Core API server + a MonoGame client. Uses MemoryPack, SqlKata, CloudStructures. Structured over 8 weeks, with a heavy emphasis on code and short explanations
- How: **A reference architecture.** Build `code/01~07` (all net10.0, includes .sql) week by week, and draw sequence diagrams of "how the game server calls the API server" and "how the client moves between the two servers." Make a comparison table against your capstone's structure
- Note: .NET 10 is required. There is no code folder for Week 8. This is the most recently added book in the repository, so it may be less polished

**An ECS-Based Online Game Server** — `ECS_기반_온라인_게임_서버/`
- When: Phase 3, Week 11 (optional), Phase 6 (optional)
- What: A C# ECS framework implementation, server structure, extensibility, optimization, an MMORPG mini-project, and an Omok ECS example. Built on SuperSocketLite, .NET 9
- How: Use it to compare an architecture different from room-actor/OOP. Look at `omok_ecs.md` and `codes/SimpleECSGameServer` and list "5 differences from my design"
- Note: The README is only 2 lines, with no table of contents (figure it out from each chapter file's H1). It is not on the completed list. Since the temptation to expand scope is high, treat it as optional only

**Building an API Game Server with ASP.NET Core Web API** — `ASPNETCore-API_게임서버_실습/01~24.md`, `Appendix.md`
- When: Phase 4 primary textbook (Ch. 1-14, 16-18, 20, 22, Appendices A-C, F, G), Phase 5 (Ch. 23, 24)
- What: POST-based APIs, MySQL/SqlKata, custom token authentication/middleware, Redis/CloudStructures, characters/inventory/mail/shop/dungeons/attendance/friends/rankings, logging/configuration. The appendices include a design document, an API spec guide, schemas, the token algorithm, and a security checklist. .NET 9
- How: This is a 24-chapter hands-on book but has no code folder, so **type out the code in the text yourself** as you go, and reimplement the gacha-collection RPG domain as an Omok lobby. Use Appendix B (the spec guide) to write 4-C, and Appendix G to check 4-2
- Note: Skip the Docker/WSL2/Memurai section of Chapter 12's Redis exercise — this curriculum uses redis-windows (a native Windows binary). Chapters 15, 19, and 21 are optional

**Building a Game Server with ASP.NET Core Web API** — `ASP_NET_WebAPI로_만드는_게임서버/01~17.md`, Appendix
- When: Phase 4 supplementary (Ch. 5-8, 13, 16), Phase 5 (Ch. 14, 16, 17)
- What: Web API basics, .http testing, MySQL/Redis, authentication, characters/inventory/gacha/combat, async/optimization/security/deployment. The appendix covers modularizing content/collection/social/shop APIs. About 48,000 lines — very large
- How: Since its topics overlap with the hands-on book, use it as supplementary material. Look at the Controllers/Repository/Services structure of `codes/GameAPIServer_Template_01` (net9.0) and write a project-structure ADR. Chapter 17 (Docker, deployment) is out of scope for this curriculum (per the no-CI/no-deployment policy), so skip it
- Note: Chapter 3 (environment setup) is only 22 lines — practically nonexistent. The appendix is longer than the main text. The README advises "fill in any gaps with AI"

**Building an FPS Game Matching System** — `매칭-FPS_게임_매칭_시스템_만들기/`
- When: Phase 3 (Ch. 1, 3), Phase 6 primary textbook (Ch. 4-7, Ch. 2 optional)
- What: Matching requirements, rating systems (ELO/Glicko/TrueSkill), matching algorithms, a distributed architecture (Redis, sharding), a C# implementation, monitoring, and case studies. .NET 9. 4 HTML visual guides
- How: Write up, as an ADR, your **judgment call in scaling down** the FPS large-scale baseline to your own scale (2 players, 2 servers). The C++ track should also read Chapter 4's architecture, which is language-agnostic
- Note: Chapter 1 is very short at 97 lines. No code folder

**Practical Guide to Implementing Gacha Probability for New Game Programmers** — `게임-신입_게임_프로그래머를_위한_가챠_확률_구현_실전_가이드/`
- When: Phase 4, Week 14 (Ch. 9-12, 16), Phase 5 (Ch. 15), Phase 6 (Ch. 4-6, 14 optional)
- What: Random number generation, roulette/tiered/pity(ceiling)/box gacha, principles of server-side drawing, data design, **currency-deduction atomicity, race conditions**, regulations, statistical testing, monitoring, and incident case studies. .NET 10, SqlKata, CloudStructures, xUnit
- How: More than the gacha mechanics themselves, **Chapters 11 and 12 are the best material on "currency-deduction atomicity."** Read them right before designing your shop-purchase transaction. Chapter 16's incident case studies are material for the 4-2 abuse scenarios. Ignore the Docker Compose section in Appendix A (the code collection)
- Note: The target audience is "comfortable with C# 11+ syntax," which fits this curriculum's learners

### 4.5 Windows / C++ Systems

**Modern Windows Multithreading: High-Performance Concurrent Programming for Game Server Developers** — `게임_서버-모던_Windows_멀티스레딩_.../01~12.md`
- When: Phase 1 primary textbook (Ch. 2-5, 8), Phase 2 (Ch. 6), Phase 3 primary textbook (Ch. 6, 11, 12), Phase 5 primary textbook (Ch. 10, Appendices B, C), Phase 6 (Ch. 11 reread)
- What: SRW Lock, condition variables, one-time initialization, the Windows thread pool, barriers, WaitOnAddress/lock-free techniques, UMS, WPT/ETW performance analysis, an IOCP + thread-pool game server architecture, and C++23 synergies. VS, C++23. Includes accompanying pdf/pptx
- How: Read it with a 1:1 mapping table between the standard library and the Win32 API. Chapter 11 is a core textbook for Phase 3, but decide for yourself "where to place the room actor" and write it up as an ADR. Chapter 10 is the direct textbook for the Phase 5 ETW/WPA exercises
- Note: Assumes you already know Win32 threading basics (intermediate-to-advanced). Chapter 9 (UMS) is out of scope for this curriculum. `12.md` starts with `## 12장` (Chapter 12) instead of an H1

**Modern Win32 API Programming for Game Server Developers** — `게임_서버_개발자를_위한_최신_Win32_API 프로그래밍/` (a space in the folder name)
- When: Phase 1 primary textbook (Ch. 1, 2, 4-6), Phase 3 (Ch. 2, 4 reread), Phase 5 (Ch. 7, Appendix C)
- What: Win32 basics, memory management (VirtualAlloc, heaps, memory pools), file I/O (overlapped), processes/threads, advanced synchronization, interlocked/lock-free techniques, performance counters/ETW, system information, security/permissions, service programming (not covered in this curriculum), COM/WinRT. VS 2022, C++20
- How: Refer to Chapter 2's memory pools for the 1-2 assignment. Use Chapter 7 to expose performance counters (PDH) through your own metrics (Phase 5). Chapter 10 (service programming) doesn't need to be read since this curriculum doesn't cover deployment. It's also good to read Appendix D's VS 2022 tips early on
- Note: Winsock is not covered (sockets are handled by the TCP/IP book). No code folder

**The Complete Guide to the C++23 Memory Model (Memory Order)** — `Cpp-MemoryModel/01~05.md`
- When: Phase 1, Week 3 (Parts 1-2), Phase 5 (Parts 3-4)
- What: Caches/reordering, the 6 memory_order values, spinlocks/lock-free/DCL/fences, happens-before, platform differences, and practical projects (a logger, a work queue, a memory pool)
- How: Read Parts 1-2 right before the 1-1 lock-free ring buffer. In Phase 5, measure the actual effect of relaxing seq_cst to acq/rel (if there's no measurable effect, that's a valid conclusion too)
- Note: The README's "Part 6: Debugging and Testing" has no corresponding body file. Ask the AI to explain, from an ARM perspective, the trap where `relaxed` appears to work fine on x86

**Thread-Local Storage in Practice** — `Cpp-Thread-Local_Storage/01~06.md`
- When: Phase 5, Week 18 (Ch. 4-6)
- What: The Win32 TLS API vs. `thread_local`, per-thread memory pools, avoiding false sharing, PRNGs, statistics collection, initialization order, DLLs, and leaks
- How: Use it as an optimization candidate for 5-2 (per-thread pools, statistics counters). Measurement before/after applying it is mandatory
- Note: No code folder; includes an accompanying pptx

**Safe and Elegant Programming with Modern C++** — `Cpp_Modern_Cpp로_안전하고_우아한_프로그래밍/01~20.md`
- When: Phase 1 primary textbook (Ch. 2-7, 10, 12, 13), Phase 5 (Ch. 14, 18), Phase 6 (Ch. 17)
- What: The evolution of C++, the Core Guidelines, RAII, smart pointers, containers, strong typing, templates, exception safety, `std::expected`, basic/advanced concurrency, parallel algorithms, coroutines, networking/game components/cross-platform projects, benchmarking, and a comparison with Rust. Based on VS 2022, C++20
- How: Since there's no code folder, the assignment is to move the code in the text into a project and build it. Grade your own code using the checklists in Chapters 3, 7, and 10
- Note: The development environment and authorship notes are in `01.md`, not the README. It overlaps with "Modern C++ Programming as Safe as Rust," but this one is based on MSVC, which fits this curriculum

**Safe and Easy C++ Programming, Starting with Modern C++** — `Cpp_모던_Cpp로_시작하는_안전하고_쉬운_Cpp_프로그래밍/`
- When: Phase 1, Week 1 (only for learners shaky on syntax)
- What: An introductory book starting from Hello World (Ch. 1-17 language, Ch. 18-28 Siv3D GUI/projects). About 48,000 lines — the largest volume
- How: Read only the weak parts among Chapters 1-17. This is for reinforcement if you fall short of this curriculum's prerequisite (knowing the syntax)
- Note: The repository README explicitly states "some Siv3D examples may have build errors — fix with AI." Chapter 18 onward is unrelated to this curriculum

**Building an Online Game Server with C++ Boost.Asio** — `Cpp_BoostAsio로_만드는_온라인_게임_서버/`, `Samples_VS2022/`
- When: Phase 2, Week 7 (optional: Ch. 1, 2, 4-7), Phase 3, Week 11 (optional: Ch. 11-14, 17), Phase 6 (optional: Ch. 17, 20, 21)
- What: vcpkg Boost, sync/async, io_context, handlers, UDP, chat, strand, timers, coroutines, optimization, scalable design, protocols, security, MMO/mobile servers. 25 VS 2022 projects
- How: You should read this **after building your own IOCP**, so you can see what Asio abstracts away. Build `Samples_VS2022`'s Async TCP, strand, and Timer examples and make a comparison table against your own IOCP server. This curriculum's required path is implementing IOCP yourself; Asio is for comparison only
- Note: The H1 in files from Chapter 15 onward reads `# workingBooks` (a leftover editing artifact). Do not switch over to the Asio path (that's an option for after you're employed)

**Modern C++ Programming as Safe as Rust** — `Cpp-Rust처럼_안전한_Modern_Cpp_프로그래밍/`
- When: Phase 6, Week 25 (optional: Ch. 1, 16)
- What: C++26 safety features, ownership, thread safety, std::execution, reflection, and verification tools. Based on GCC 14+/Clang 18+
- How: Read for concepts only, to prepare for the interview topic of "C++ safety." Do not do the hands-on exercises (many features are unsupported by MSVC)
- Note: Many examples use a Linux toolchain. It doesn't match this curriculum's environment (Windows/MSVC), so use it for reference only

**WinRT in Practice: Networking and Multithreaded Programming** — `Cpp-실전_WinRT_네트워크와_멀티스레드_프로그래밍/`
- When: Phase 2, Weeks 5-6 (reference: Ch. 17, 19, 20)
- What: C++/WinRT types, async, and threading; WinRT sockets (StreamSocket, etc.); protocols/serialization/framing/sessions; chat/lobby/real-time servers; optimization/testing. Windows 11, C++20
- How: The WinRT API itself isn't used in this curriculum, but the protocol/framing/session-design chapters are API-independent. Use them as a checklist for the 2-C spec
- Note: This is not Winsock/IOCP. Do not build your server with this book's API

### 4.6 DB, Infrastructure, and Operations

**Learn MySQL C# Programming in a Week** — `DB-1주일만에_배우는_MySQL_CSharp_프로그래밍/01~07.md`
- When: Phase 4 optional advanced textbook (when MySQL is chosen; Days 1, 2, 5 common, Days 3, 4, 6, 7 C#)
- What: DBMS/InnoDB, installation/Workbench, SQL basics/table design, MySqlConnector/SqlKata, JOINs/indexes, game content (login, inventory, rankings, friends, mail, dungeons, attendance), performance/backup. Appendix cheat sheet and per-game design examples. .NET 9, MySQL 8.0
- How: Since this curriculum defaults to SQLite, the default approach is to take the SQL syntax and table-design concepts as-is and move only the connection code over to a SQLite client for practice. Follow Day 1's MySQL installation/Workbench section only if you chose MySQL. Compare Day 2's table design against 4-C; for Day 5's indexing, practice with `EXPLAIN QUERY PLAN` on SQLite or `EXPLAIN` if you chose MySQL. Day 6 nearly matches the 4-1 requirements, so use it as a reference but design the transaction boundaries yourself. Print Appendix A's cheat sheet. The C++ track should also follow the SQL portion the same way
- Note: No code folder. Uses the same stack as the API server books (MySqlConnector + SqlKata)

**Learn Redis Programming in a Week** — `DB-1주일만에_배우는_Redis_프로그래밍/01~07.md`
- When: Phase 4 primary textbook (Ch. 1, 2 common, Ch. 3-5, 7 C#), Phase 6 (Ch. 6 Pub/Sub)
- What: Installation (including a WSL2/Memurai section), internal structure/data types, CloudStructures, sessions/duplicate-login handling, profile/inventory caching, ZSet-based rankings, Pub/Sub, events/pipelining/monitoring. .NET 9, Redis 6+
- How: Skip all of Chapter 1's Docker/WSL2/Memurai sections entirely and replace them with installing redis-windows (https://github.com/redis-windows/redis-windows). Extend Chapter 2's data-structure table into a judgment table of "is this data suited to the DB (SQLite/MySQL) or to Redis." Chapter 6's Pub/Sub is a candidate for inter-server communication in the capstone (note its loss characteristics in an ADR)
- Note: No code folder

**Game Server Monitoring Using Prometheus** — `게임_서버-프로메테우스를_이용한_게임_서버_모니터링/README.md`
- When: Phase 5 primary textbook (the entire book)
- What: Monitoring concepts, the Prometheus architecture, **Windows installation** (Prometheus, windows_exporter, Grafana), instrumenting the .NET 9 API/socket server examples, custom exporters, alerting, dashboards, and operational scenarios. About 2,500 lines
- How: Instrument your own 3-1/4-1 instead of the example server, following the order Ch. 3 → 4 → 6 → 8 → 9 → 10 → 11 → 12. Keep Appendix B (Grafana query examples) handy while working on dashboards. The C++ track should translate the instrumentation code to prometheus-cpp
- Note: Since this is a short hands-on guide, supplement deeper topics like PromQL and histogram bucket design with official documentation

**Log Collection and Forwarding Using fluentd** — `게임_서버-fluentd를_이용한_로그_수집과_전송/README.md`
- When: Phase 5, Week 17 (reading Ch. 1-7, 13 is mandatory; hands-on is optional)
- What: Log pipeline concepts, installing the Windows fluent-package, integrating Serilog, socket server logs, output plugins, storing to MySQL/MongoDB, pipeline patterns, and incident response. .NET 8
- How: Follow only the Windows fluent-package section, without Docker. The hands-on portion is optional (file logging + PowerShell analysis is also acceptable). "What is a log pipeline" is within the scope of the explanation exam
- Note: Bump the .NET 8 code to 9 to build. The first line of the file has a BOM

**Docker for Game Server Developers** — `게임_서버_개발자_위한_Docker/`
- This curriculum **does not use it** (per the no-Docker policy). If you need Docker after getting a job, you can read Chapters 1-7 then. Based on Windows 11, WSL2, .NET 8

### 4.7 Client, Miscellaneous, and AI Tools

**Building an Online Game Client with MonoGame** — Optional. Read Ch. 3-4, 7-8 when you need a demo GUI client in Phase 3/6. Keep the client to a minimum in a server developer's portfolio

**Behavior Tree AI for Online Game Servers** — Optional. Read Part 1 only when expanding Phase 6 content into "action with NPCs." Requires VS 2026, .NET 10; a single 25,000-line file. Not recommended by default due to the high risk of scope creep

**Building an Online Game Using C# and P2P Communication** — Optional. For Phase 6 interview prep, read only Ch. 1-2 (an introduction to NAT and hole punching). Implementation is out of scope for this curriculum

**Getting Started with AI Coding Agents Using OpenCode** — `ai-agent-coding-with-opencode.md`
- When: Phase 1, Week 1, Wednesday (Guide 1, Parts 2, 5)
- What: Installing and understanding the terminal AI coding agent OpenCode (sessions, Build/Plan mode, AGENTS.md), C# integration, automation, documentation writing, and multi-agent setups
- How: This curriculum uses Claude Code as the default coding agent, but other agents such as Codex CLI may also be used. Regardless of which agent you use, the concepts of "an agent instructions file (CLAUDE.md for Claude Code, AGENTS.md for Codex CLI)," "Plan then Build," and a "pre-commit automated review hook" apply as-is. After reading, identify 5 items to reflect in your own instructions file (CLAUDE.md or AGENTS.md)
- Note: Tool versions change quickly. Verify commands and settings against the current tool documentation

---

## 5. Reading Order by Track (At a Glance)

### 5.1 C# Track
1. (P1) Quick read of the Essential C# Guide → Mastering Async/Await Ch. 1-4, 6, 7 → Building the async Library Ch. 1-6 → Design Patterns Ch. 3 → OpenCode guide Part 2, 5
2. (P2) Starting from Networking → Network Theory (+ log/netstat observation) → C# Socket Ch. 1-7, 9, 10 → Building the async Library Ch. 7 → the Roadmap Ch. 1, 2, 7 → FreeNetLite → SuperSocketLite Ch. 1-6
3. (P3) Building the async Library Ch. 16-18 → Design Patterns Ch. 7-12, 16 → the Roadmap Ch. 3-5 → C# Socket Ch. 11 → SuperSocketLite Ch. 7 → Matching Ch. 1, 3 → Client Networking Ch. 4, 5 → Mastering Async/Await Ch. 10 → Network Library Analysis → (optional) ECS
4. (P4) the API Game Server Lab Appendices A-C, F, G → MySQL Day 1, 2 → the API Game Server Lab Ch. 1-7 → MySQL Day 3, 4 → the API Game Server Lab Ch. 8-11 → Redis Ch. 1-5 → the API Game Server Lab Ch. 12-14, 16-18, 20, 22 → MySQL Day 5-7 → Gacha Ch. 9-12, 16 → the WebAPI Game Server book Ch. 5-8, 13, 16 → 2D MMORPG, Week 2 (of the book)
5. (P5) C# Socket Ch. 8, Appendix → Mastering Async/Await Ch. 5, Appendix B → Prometheus (entire book) → fluentd Ch. 1-7, 13 → the API Game Server Lab Ch. 23, 24 → Building the async Library Ch. 14, 19, 20 → the WebAPI Game Server book Ch. 14, 16, 17 → the Roadmap Ch. 5, 6 → Gacha Ch. 15
6. (P6) Matching Ch. 4-7 → 2D MMORPG Ch. 1-4, 8 + code → Redis Ch. 6 → the Roadmap Ch. 8 → Building the async Library Ch. 13, 15 → Design Patterns Ch. 17 → review the 2 network theory books → (optional) MonoGame, Gacha, ECS, P2P Ch. 1-2

### 5.2 C++ Track
1. (P1) (if needed) Starting with Modern C++ Ch. 1-17 → Modern C++ (Safe and Elegant) Ch. 2-7, 10, 12, 13 → Modern Windows Multithreading Ch. 2-5, 8 → Win32 API Ch. 1, 2, 4-6 → Memory Model Parts 1-2 → Design Patterns Ch. 3 (concepts) → OpenCode guide Part 2, 5
2. (P2) Starting from Networking → Network Theory (+ log/netstat observation) → TCP/IP Windows Sockets Ch. 1-6, 8-11, 13 + codes → the Roadmap Ch. 1, 2, 7 → Modern Windows Multithreading Ch. 6 → (reference) WinRT Ch. 17, 19, 20 → (optional) Boost.Asio Ch. 1, 2, 4-7
3. (P3) Modern Windows Multithreading Ch. 6, 11, 12 → TCP/IP Sockets Ch. 11, 14 → Design Patterns Ch. 7-12, 16 (concepts) → the Roadmap Ch. 3-5 → Win32 API Ch. 2, 4 → Matching Ch. 1, 3 → Client Networking Ch. 4, 5 → (optional) Boost.Asio Ch. 11-14, 17; ECS
4. (P4) Path (a): MySQL Day 1, 2, 5 (SQL) → Redis Ch. 1, 2 → the API Game Server Lab Appendices A-C, F, G → Gacha Ch. 9-12, 16 + official documentation (cpp-httplib, MySQL Connector/C++, redis-plus-plus). Path (b): the entire Essential C# Guide → follow the C# track's P4 order
5. (P5) Modern C++ (Safe and Elegant) Ch. 18, 14 → Modern Windows Multithreading Ch. 10, Appendices B, C → Win32 API Ch. 7, 10, Appendix C → Prometheus (entire book, instrumenting with prometheus-cpp) → fluentd Ch. 1-7, 13 → TLS Ch. 4-6 → Memory Model Parts 3-4 → TCP/IP Sockets Ch. 12, 13, 14 → the Roadmap Ch. 5, 6
6. (P6) Matching Ch. 4-7 (architecture) → 2D MMORPG (as a reference structure) → Redis Ch. 6 → the Roadmap Ch. 8 → Modern C++ (Safe and Elegant) Ch. 17 → Modern Windows Multithreading Ch. 11 reread → (optional) Boost.Asio Ch. 17, 20, 21; Modern C++ Programming as Safe as Rust Ch. 1, 16 → review the 2 network theory books

---

## 6. Books Not Used in This Curriculum, and Why
- **Docker for Game Server Developers**: Per the curriculum's policy (no Docker). For reference after you're employed
- **The 5 Rust books, 2 Go books, TypeScript, Python**: Out of scope for this language track. Recommended as a "second language" after the capstone, and in particular "The Essential Guide to Rust Vibe Coding" is a good example of how to learn a new language together with AI
- **DuckDB, TimescaleDB, eBPF**: These are analytics/time-series/Linux-observability topics that are out of scope for this curriculum (eBPF is Linux-only)
- **OpenSiv3D, Game Programming with MonoGame**: Client/graphics study books
- **Building an Online Game Client with MonoGame, NPC Behavior Trees, P2P**: Optional only (§4.7)

---

## 7. Tips for Managing the Textbooks
- Clone the repository locally, and connect it as a `books/` submodule or symbolic link in the curriculum repository so Claude Code can read the needed chapters directly
- Create a `NOTES.md` in each book's folder and accumulate "chapters read / examples built / errors fixed / differences from official documentation." This serves as evidence of "where AI (the book) got it wrong" for the end-of-Phase evaluation
- Since the repository is updated over time (most recently added, as of 2026-08: 2D MMORPG), run `git pull` at the start of each Phase and update the book cards if any books have changed
