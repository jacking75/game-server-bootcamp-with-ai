# Phase 1. 서버 개발자를 위한 언어·도구 심화 (1~3주, 120h)

> 공통 40h / 트랙 80h. 이 Phase가 끝나면 학습자는 "문법을 아는 사람"에서 "서버 코드를 읽고 쓰고 측정할 수 있는 사람"이 된다.

## 1. 개요

### 1.1 목표
- 동시성(스레드·락·원자 연산·메모리 가시성)과 메모리(할당·수명·GC 또는 소유권) 개념을 **코드로 재현하고 측정**할 수 있다
- Git·단위 테스트·디버거·벤치마크 도구를 손에 익혀 이후 모든 과제의 기본 위생으로 삼는다
- Windows 개발 환경(VS 2022, .NET SDK 또는 MSVC/vcpkg, MySQL, Redis 대체, PowerShell)을 구축한다
- AI 협업 규칙(§2 총론)을 몸에 익힌다. 특히 "설명 못 하면 커밋 금지", "AI 없는 1시간"을 이 Phase에서 습관화한다

### 1.2 선수 조건
- C# 또는 C++ 기본 문법(변수, 클래스, 컬렉션, 예외, 기본 제네릭/템플릿)을 안다
- 프로그래밍 경험이 얕아 문법이 불안하면 Phase 1 첫 주에 교재 "모던 C++로 시작하는 안전하고 쉬운 C++ 프로그래밍"(1~17장) 또는 "ASP.NET Core Web API를 위한 필수 C# 가이드"를 속독한다

### 1.3 이 Phase가 끝나면 할 수 있는 것
- 스레드 세이프 큐를 두 가지 방식으로 만들고 어느 쪽이 왜 빠른지 수치로 설명한다
- 데드락·경쟁 조건·리소스 누수를 디버거와 도구로 찾아낸다
- C#: async/await가 컴파일러 상태 머신으로 변환되는 과정, GC 세대와 할당 비용을 설명한다
- C++: 소유권·이동·RAII를 코드로 적용하고, atomic 메모리 오더의 차이를 설명한다

---

## 2. 주차별 계획

| 주차 | 공통 (8h/주 기준 + 금요일 점검) | C# 트랙 | C++ 트랙 | 과제 진도 |
|---|---|---|---|---|
| 1주 | 환경 구축, Git 워크플로, 테스트 프레임워크, CLAUDE.md 작성법, AI 학습 루틴 연습, OS 기초(프로세스/스레드/가상 메모리) | .NET 런타임 구조, GC 세대, 값/참조 타입·구조체, Span/Memory/ArrayPool, BenchmarkDotNet 첫 실행 | 소유권·RAII·스마트 포인터, 이동 의미론, vcpkg/CMake 프로젝트 구성, GoogleTest/Benchmark 첫 실행, ASan 켜기 | 1-C 완료, 1-2 오브젝트 풀 시작 |
| 2주 | 동시성 개념(경쟁 조건·데드락·메모리 가시성·락 vs 무락), 디버거 심화(조건부 BP, 스레드/병렬 스택 창, 덤프) | async/await 내부, Task/ValueTask, CancellationToken, lock/Interlocked/SemaphoreSlim, Channel<T>, ConcurrentQueue | std::thread/jthread, mutex/condition_variable, atomic과 메모리 오더, Win32 SRWLock/CV, 스레드 풀 개념 | 1-1 로그 큐 (두 버전) |
| 3주 | 벤치마크 방법론(워밍업·반복·분산·비교 함정), 성능 리포트 쓰는 법, Phase 말 평가 | ThreadPool 동작·기아, ConfigureAwait, 비동기 함정, 할당 줄이기 실전 | lock-free SPSC 링 버퍼, false sharing, 컴파일러 최적화와 UB, 새니타이저 상시화 | 1-2 완료, 1-3 심화(선택), Phase 말 평가 |

### 2.1 1주차 상세 (일 단위 예시)

| 요일 | 오전(개념) | 오후(구현) | AI 없는 1시간 |
|---|---|---|---|
| 월 | VS 2022/.NET 또는 MSVC+vcpkg 설치, Git 설정, GitHub 저장소 생성, Windows Terminal/PowerShell 7 | 1-C: 저장소 골격, `.editorconfig`, README, 학습 노트 폴더, 첫 커밋 | Git 명령 20개를 도움말 없이 사용해 브랜치 만들고 머지 |
| 화 | 단위 테스트(xUnit/GoogleTest) 구조, AAA, 테스트 이름 규칙 | 1-C: CI(GitHub Actions windows-latest) 빌드+테스트, 배지 | 계산기 클래스에 테스트 5개 작성 |
| 수 | CLAUDE.md 작성법, AI 4가지 역할 실습, 프롬프트 패턴 연습 | 1-C: CLAUDE.md 초안 작성, AI에게 "이 지침대로 파일 하나만 생성" 시켜보기 | 오늘 배운 프롬프트 패턴 5개를 노트에 손으로 정리 |
| 목 | OS 기초: 프로세스/스레드, 컨텍스트 스위칭, 가상 메모리, 캐시 라인 | 트랙: C# GC 세대 실험 / C++ 스마트 포인터 실험 | 스레드 2개로 카운터 증가 → 결과가 틀리는 것 재현 |
| 금 | 트랙 학습 마무리 | 주간 점검(재구현·설명 시험·리뷰·회고) | — |

---

## 3. 학습 항목 상세

각 항목은 **무엇을 / 왜 필요한가 / 어떻게 확인하는가** 세 줄로 적는다. "확인" 항목이 곧 학습 완료 판단 근거다.

### 3.1 공통 (40h)

**Git 워크플로 (4h)**
- 무엇: 브랜치 전략(main + feature), PR 셀프 리뷰, 리베이스, 충돌 해결, 커밋 메시지 규칙(제목 50자, 본문 "왜")
- 왜: 이후 모든 과제가 저장소 단위로 평가되며, 취업 후 첫날부터 쓰는 도구다
- 확인: 충돌이 나는 두 브랜치를 일부러 만들고 리베이스로 해결한 이력이 저장소에 있다

**단위 테스트 (6h)**
- 무엇: AAA 패턴, 테스트 더블(스텁/페이크), 테스트 가능한 코드 분리(의존성 주입), 파라미터화 테스트
- 왜: 서버는 GUI가 없어 테스트가 곧 실행 방법이다. 이후 "제공 테스트 통과"가 평가 기준이다
- 확인: 1-1 로그 큐의 순서 보장·flush·용량 정책을 각각 테스트로 증명한다

**디버깅 (6h)**
- 무엇: 조건부/데이터 브레이크포인트, 병렬 스택·스레드 창, 메모리 창, 크래시 덤프 저장(작업 관리자/procdump)과 열기
- 왜: 서버 버그의 대부분은 재현이 어렵고 멀티스레드다. 디버거 없이 로그만으로는 못 잡는다
- 확인: 일부러 데드락을 만든 뒤 병렬 스택 창에서 두 스레드가 서로 기다리는 지점 스크린샷을 남긴다

**OS 기초 (6h)**
- 무엇: 프로세스/스레드/핸들, 컨텍스트 스위칭 비용, 사용자/커널 모드 전환, 가상 메모리와 페이지, CPU 캐시와 캐시 라인(64바이트), false sharing
- 왜: "왜 스레드를 많이 만들면 느려지는가", "왜 구조체 배열이 빠른가"의 근거가 여기서 나온다
- 확인: 스레드 수를 1→2→4→64→1,000으로 늘리며 같은 작업의 처리 시간을 측정한 표를 만든다

**동시성 개념 (10h)**
- 무엇: 경쟁 조건, 임계 구역, 데드락 4조건, 락 종류(뮤텍스/스핀락/RW락), 원자 연산, 메모리 가시성과 재배치, 락 프리의 의미와 한계
- 왜: 게임 서버의 모든 버그 중 가장 비싸고 재현하기 어려운 것이 동시성 버그다
- 확인: 데드락·경쟁 조건·가시성 문제를 각각 최소 코드로 재현하고 각각의 수정 방법을 2가지 이상 적는다

**Windows 개발 환경 (4h)**
- 무엇: VS 2022 워크로드, .NET SDK 다중 버전 관리(global.json), vcpkg 매니페스트 모드, PowerShell 7 스크립트 기초, MySQL 8 설치(이후 Phase 대비), Redis 대체(Memurai 또는 WSL2 redis) 설치
- 왜: Docker를 쓰지 않으므로 로컬 설치·서비스 관리 자체를 익혀야 한다
- 확인: `mysql -u root -p`와 `redis-cli ping`(또는 Memurai)이 응답하고, 서비스 시작/중지를 PowerShell로 할 수 있다

**벤치마크 방법론 (4h)**
- 무엇: 워밍업, 반복 횟수, 평균 vs 중앙값 vs p99, 분산 표시, Debug/Release 차이, 측정 대상 격리, 비교 시 같은 조건 유지
- 왜: Phase 5에서 "수치로 개선을 증명"하려면 측정 자체가 신뢰 가능해야 한다
- 확인: 같은 코드를 Debug/Release로 각각 측정해 차이를 표로 남기고, "왜 Debug 측정은 의미가 없는가"를 한 문단 쓴다

### 3.2 C# 트랙 (80h)

| 항목 | 시간 | 무엇을 | 확인 방법 |
|---|---|---|---|
| .NET 런타임과 GC | 10h | JIT, 세대별 GC, LOH, 워크스테이션/서버 GC, 할당 비용, `GC.GetTotalAllocatedBytes`, `dotnet-counters` | 1MB 이상 객체를 반복 할당해 LOH 동작을 `dotnet-counters`로 관찰한 스크린샷 |
| 값/참조 타입, 구조체 | 6h | struct vs class, readonly struct, ref struct, 박싱 비용, in/ref 매개변수 | 박싱이 발생하는 코드와 안 하는 코드의 할당량 비교 |
| Span/Memory/ArrayPool | 8h | `Span<T>`, `Memory<T>`, `stackalloc`, `ArrayPool<T>.Shared`, 문자열 파싱 시 할당 줄이기 | 문자열 파싱을 Substring 버전과 Span 버전으로 만들어 MemoryDiagnoser 비교 |
| async/await 내부 | 14h | 상태 머신, SynchronizationContext, ConfigureAwait, Task vs ValueTask, `async void` 금지, 예외 전파, CancellationToken | ILSpy로 디컴파일해 상태 머신 클래스를 찾고 MoveNext 흐름을 설명 |
| ThreadPool | 6h | 워크 아이템, 기아(starvation), 동기 대기(`.Result`)의 위험, `Task.Run` 남용 | ThreadPool 기아를 재현하고 `dotnet-counters`의 큐 길이로 확인 |
| 동기화 도구 | 12h | `lock`, `Interlocked`, `SemaphoreSlim`, `ReaderWriterLockSlim`, `ConcurrentQueue/Dictionary`, `Channel<T>` | 1-1 로그 큐 두 버전 구현 |
| BenchmarkDotNet | 6h | 벤치마크 작성, MemoryDiagnoser, Baseline, 결과 해석 | 1-1, 1-2 리포트 |
| 프로젝트 구성 | 6h | .NET CLI, 솔루션/프로젝트 구조, NuGet, 분석기와 `TreatWarningsAsErrors`, nullable | 1-C 저장소 |
| 비동기 함정 | 12h | 데드락, 잃어버린 예외, 취소 무시, 파이어앤포겟, 동기화 컨텍스트 캡처 | 교재 6장의 함정 12가지를 각각 재현하고 수정한 예제 프로젝트 |

### 3.3 C++ 트랙 (80h)

| 항목 | 시간 | 무엇을 | 확인 방법 |
|---|---|---|---|
| 소유권과 RAII | 10h | 객체 수명, 스택/힙, `unique_ptr/shared_ptr/weak_ptr`, 순환 참조, 커스텀 deleter | 순환 참조로 누수가 나는 코드를 ASan 없이도 설명하고 weak_ptr로 고침 |
| 이동 의미론 | 8h | 값 카테고리, 이동 생성자/대입, `std::move`/`forward`, 복사 생략, Rule of 0/5 | 이동을 지원하는 버퍼 클래스를 만들고 복사 횟수를 카운터로 측정 |
| 표준 동시성 | 14h | `std::thread/jthread`, `mutex/lock_guard/scoped_lock`, `condition_variable`, `atomic`, 메모리 오더(relaxed/acquire/release/seq_cst) | 1-1 로그 큐 두 버전 |
| Win32 동기화 | 8h | `SRWLOCK`, `CONDITION_VARIABLE`, `InitOnce`, `WaitOnAddress`, Windows 스레드 풀 API 개요 | 표준 mutex 버전과 SRWLock 버전 성능 비교 표 |
| 빌드 시스템 | 8h | CMake 타깃/프로퍼티, vcpkg 매니페스트, Debug/Release, 경고 레벨(/W4 /permissive-), 정적 분석(/analyze) | 1-C 저장소 CMake로 빌드+테스트 CI 통과 |
| 새니타이저·테스트 | 8h | ASan(MSVC 지원), Application Verifier, GoogleTest, Google Benchmark | ASan이 잡는 use-after-free 예제를 만들고 리포트 캡처 |
| 템플릿·유틸리티 | 8h | 템플릿 기초, `std::function`/람다, `optional/variant/span/expected` | 1-2 오브젝트 풀을 템플릿으로 |
| 캐시와 UB | 8h | false sharing, alignas, 정의되지 않은 동작 목록(부호 오버플로, 초기화 안 된 읽기, 댕글링), 컴파일러 최적화 영향 | false sharing 유무 두 버전의 벤치마크 |
| 언어 재정비(선택) | 8h | Core Guidelines 핵심, `constexpr`, 강타입, 예외 안전성 3단계 | 교재 3·7·10장 체크리스트 자기 채점 |

---

## 4. 교재 활용 가이드 (Phase 1)

교재 원문은 `programming-books-with-ai` 저장소에 있다. 아래 순서로 읽되, 각 책에서 지정된 장만 읽는다. 읽는 방식은 총론 §2.3을 따른다.

### 4.1 공통
| 교재 | 읽을 장 | 언제 | 어떻게 활용 |
|---|---|---|---|
| OpenCode로 시작하는 AI 코딩 에이전트 | 가이드 1 Part 2(핵심 개념: 세션·Build/Plan 모드·AGENTS.md), Part 5(자동화) | 1주 수요일 | 도구는 Claude Code를 쓰지만 "에이전트 지침 파일(AGENTS.md ≒ CLAUDE.md)", "Plan 모드 후 Build" 개념은 그대로 적용된다. 읽은 뒤 자기 CLAUDE.md에 반영할 항목 5개를 뽑는다 |
| 게임 개발자를 위한 C# 디자인 패턴 | 3장 Object Pool | 1주 목요일 | 1-2 과제 설계 전에 읽는다. C++ 트랙도 개념 파악용으로 읽고 C++로 옮긴다 |

### 4.2 C# 트랙
| 교재 | 읽을 장 | 언제 | 어떻게 활용 |
|---|---|---|---|
| ASP.NET Core Web API를 위한 필수 C# 가이드 | 전체(1,839줄, 짧음) | 1주 초 속독 | 문법 구멍 점검용. 모르는 절만 표시해 두고 해당 절의 예제를 직접 쳐 본다 |
| C# Async/Await 완전 정복 | 1~4장(기초·SynchronizationContext·ConfigureAwait·Awaitable), 6장(함정 12가지), 7장(동기화 객체) | 2주 | 각 장 끝 체크리스트를 AI 면접관에게 주고 구두 시험을 본다. samples 폴더(.NET 10)를 빌드한다. .NET 10 SDK가 없으면 Directory.Build.props의 TargetFramework를 net9.0으로 낮춘다 |
| C# async/await 라이브러리 만들기 | 1~6장(Task/ValueTask 내부, 상태 머신 해부, Awaitable 설계, 커스텀 Awaitable, DelayAsync, 취소) | 3주 | 2장 상태 머신 해부를 읽은 뒤 ILSpy로 자기 코드를 디컴파일해 대조한다. 각 장의 연습 문제를 과제로 삼는다. 16~20장은 Phase 3·5에서 읽는다 |

### 4.3 C++ 트랙
| 교재 | 읽을 장 | 언제 | 어떻게 활용 |
|---|---|---|---|
| Modern C++로 안전하고 우아한 프로그래밍 | 2장(개발 환경·도구), 3장(Core Guidelines), 4장(RAII), 5장(스마트 포인터), 6장(컨테이너), 7장(강타입), 10장(예외 안전성), 12장(동시성 기초), 13장(고급 동시성) | 1~2주 | VS 2022/MSVC 기준이라 이 과정의 환경과 맞는다. 코드 폴더가 없으므로 본문 코드를 직접 프로젝트로 옮겨 빌드하는 것이 과제다. 18장(벤치마킹)은 3주에 읽는다 |
| 모던 Windows 멀티스레딩 | 2장(역사), 3장(SRW Lock), 4장(Condition Variable), 5장(One-Time Init), 8장(WaitOnAddress·Lock-Free) | 2주 | 표준 라이브러리와 Win32 API를 1:1로 대응시키며 읽는다. 6장(스레드 풀)·11장(아키텍처)은 Phase 3, 10장(성능 분석)은 Phase 5 |
| 게임 서버 개발자를 위한 최신 Win32 API 프로그래밍 | 1장(Win32 기초), 2장(메모리 관리·메모리 풀), 4장(프로세스·스레드), 5장(동기화 심화), 6장(인터락·무잠금) | 2~3주 | 2장의 메모리 풀은 1-2 과제 참고. 7장(성능 카운터)·10장(서비스)은 Phase 5 |
| C++23 메모리 모델 완벽 가이드 | 1부(기초 이론), 2부(메모리 순서 상세) | 3주 | 1-1의 lock-free 링 버퍼를 만들기 직전에 읽는다. 3부(스핀락·lock-free 자료구조)는 Phase 5 |
| 모던 C++로 시작하는 안전하고 쉬운 C++ 프로그래밍 | 1~17장 중 약한 부분만 | 1주(필요 시) | 문법이 불안한 학습자만. 18장 이후(Siv3D)는 이 과정과 무관하다 |

### 4.4 부 트랙 읽기 과제 (방식 A 병행자)
- C# 주 트랙: "Modern C++로 안전하고 우아한 프로그래밍" 4·5장을 읽고 "C# GC vs C++ RAII" 비교표 작성
- C++ 주 트랙: "C# Async/Await 완전 정복" 1장을 읽고 "C++ 스레드 vs C# Task" 비교표 작성

---

## 5. AI 협업 가이드 (Phase 1 특화)

### 5.1 이 Phase에서 연습할 프롬프트 (그대로 써도 된다)

개념 확인
```
나는 [C# GC 세대 / C++ 이동 의미론]을 이렇게 이해했다:
"(내 설명 5~10문장)"
1) 틀린 부분과 빠진 부분을 지적해 달라.
2) 내 설명이 맞는지 직접 확인할 수 있는 10줄 이내 실험 코드를 제안해 달라.
3) 공식 문서에서 확인할 키워드를 알려 달라.
```

결함 주입
```
다음 요구사항의 [스레드 세이프 로그 큐] C# 코드를 작성하되, 데드락 1개·경쟁 조건 1개·리소스 누수 1개를 의도적으로 숨겨 넣어라.
정답 위치는 내가 "정답 공개"라고 말하기 전까지 절대 알려주지 말라.
힌트를 요청하면 결함의 "종류"만 알려주고 위치는 말하지 말라.
```

벤치마크 해석
```
BenchmarkDotNet(또는 Google Benchmark) 결과다: (표 붙여넣기)
내 해석: "(2~3문장)"
내 해석에 동의하는지, 놓친 요인(워밍업·GC·캐시·측정 오차)이 있는지 말해 달라.
```

CLAUDE.md 초안 검토
```
다음은 내 학습용 프로젝트의 CLAUDE.md 초안이다. (붙여넣기)
학습 목적(코드를 내가 이해하며 진행)에 어긋나는 항목, 빠진 규칙, 너무 모호한 문장을 지적해 달라.
```

### 5.2 코딩 에이전트에게 맡길 것 / 맡기지 않을 것

| 맡긴다 | 맡기지 않는다 |
|---|---|
| CI 워크플로 YAML 초안, `.editorconfig`, 프로젝트 골격 | 로그 큐·오브젝트 풀의 핵심 알고리즘 첫 구현 |
| 테스트 케이스 목록 뽑기(빠진 케이스는 내가 추가) | 벤치마크 결과 해석의 최종 결론 |
| 벤치마크 프로젝트 보일러플레이트 | 학습 노트·주간 회고 |
| 결함 주입 코드 생성 | 결함 찾기 |

### 5.3 AI가 자주 틀리는 지점 (검증 포인트)
- .NET 버전별 API 차이(예: `ConfigureAwaitOptions`는 .NET 8+). 공식 문서의 "적용 대상" 표로 확인한다
- C++ 메모리 오더 설명에서 x86의 강한 순서 때문에 "relaxed도 잘 동작한다"고 착각하게 만드는 예제. ARM에서 깨지는지 물어본다
- BenchmarkDotNet 결과를 Debug 빌드로 낸 경우 AI가 눈치 못 챌 수 있다. 빌드 구성을 항상 명시한다

---

## 6. 과제 명세

### 6.1 공통 과제 1-C. 학습 환경·저장소 구축 (필수, 8h)

목표: 이후 26주 동안 쓸 저장소 구조와 자동화를 만든다.

요구사항
1. GitHub 저장소 `gameserver-course-<이름>` 생성. 구조:
   ```
   /README.md            과정 소개, 트랙 선택 기록, 진행 표
   /notes/daily/         일일 학습 노트 (YYYY-MM-DD.md)
   /notes/weekly/        주간 회고
   /phase1/              Phase별 과제 폴더
   /CLAUDE.md            에이전트 지침
   /.editorconfig
   /.github/workflows/ci.yml
   ```
2. CI: push/PR 시 `windows-latest` 러너에서 빌드 + 테스트. C#은 `dotnet test`, C++은 CMake configure/build + `ctest`
3. README에 CI 배지, 트랙 선택 이유 3문장
4. CLAUDE.md에 최소 포함: 프로젝트 구조, 언어/버전, 코딩 규칙 5개, "코드 생성 시 설계 의도 주석 + 테스트 동반", "한 번에 파일 하나", "라이브러리 추가 시 이유와 대안 명시"

제출물: 저장소 URL, CI 녹색 배지, CLAUDE.md

채점
| 항목 | 배점 | 기준 |
|---|---|---|
| 구조 | 20 | 위 구조 준수 |
| CI | 40 | 빌드+테스트 자동 실행, 실패 시 빨간 배지 확인 이력 1회 |
| CLAUDE.md | 30 | 필수 항목 포함, 문장이 구체적(모호한 "잘 짜라" 금지) |
| README | 10 | 트랙 선택 이유·진행 표 |

### 6.2 트랙 과제 1-1. 스레드 세이프 로그 큐 (필수, 24h)

목표: 다중 생산자 → 단일 소비자 구조를 두 가지 동기화 방식으로 구현하고 측정한다.

기능 요구사항
1. API: `Enqueue(LogEntry)`, `Start()`, `Stop()`(잔여 항목 모두 파일에 기록 후 종료), 통계(`Enqueued`, `Dropped`, `Written`)
2. 생산자 스레드 N개(테스트에서 1, 4, 16), 소비자 스레드 1개가 파일에 순서대로 기록
3. 큐 용량 상한(기본 10,000). 초과 시 정책 선택: `Drop`(버리고 Dropped 증가) 또는 `Block`(생산자가 대기)
4. 같은 생산자가 넣은 항목은 파일에서 순서가 유지된다
5. `Stop()` 후 파일 라인 수 = Enqueued − Dropped

트랙별 구현 요구
- C#: (A) `lock + Queue<T> + Monitor.Wait/Pulse` 버전, (B) `Channel<T>`(Bounded) 버전. BenchmarkDotNet으로 생산자 1/4/16에서 처리량 비교, MemoryDiagnoser로 항목당 할당 바이트 비교
- C++: (A) `std::mutex + std::condition_variable + std::deque` 버전, (B) lock-free SPSC 링 버퍼 버전(생산자 1개 조건; 다중 생산자는 (A)로). Google Benchmark로 비교, 두 버전 모두 ASan 통과, (B)는 acquire/release 메모리 오더 사용 근거를 주석으로

테스트(최소)
- 순서 보장: 생산자 4개가 각각 1..10,000을 넣고 파일에서 생산자별 순서 확인
- flush: Stop 직후 파일 라인 수 검증
- 용량 정책: Drop/Block 각각 동작 확인
- 종료 중 Enqueue 호출 시 예외 또는 무시(정의한 대로)

제출물: 코드, 테스트, `REPORT-1-1.md`(1페이지: 두 버전의 처리량·할당 표, 왜 차이가 나는지 3문단, 어느 상황에 어느 것을 쓸지 결론)

채점
| 항목 | 배점 | 기준 |
|---|---|---|
| 정확성 | 30 | 테스트 전부 통과, 라인 수 일치 |
| 동시성 | 25 | 리뷰 루브릭 2번 항목 AI 리뷰 4점 이상, C++는 ASan 통과 |
| 측정 | 25 | Release 빌드, 워밍업 포함, 표에 분산 표시 |
| 리포트 | 20 | 수치 근거로 결론, "AI가 틀린 것" 1개 이상 기록 |

### 6.3 트랙 과제 1-2. 오브젝트 풀 + 할당 측정 (필수, 16h)

목표: Phase 2 패킷 버퍼용 풀을 만들고 풀 사용 전/후 할당 차이를 측정한다.

요구사항
1. `Rent()`/`Return(obj)`, 최대 보유 수(초과 반환 시 폐기), 통계(대여 횟수, 미반환 개수, 현재 보유 수)
2. 스레드 세이프(락 기반으로 시작, 여유가 있으면 스레드별 풀 + 전역 풀 2단계)
3. 잘못된 사용 방어: 같은 객체 2회 Return 시 감지(디버그 빌드에서 assert 또는 예외)
4. C#: 4KB `byte[]` 풀. 풀 없이 매번 `new byte[4096]` vs 풀 vs `ArrayPool<byte>.Shared` 세 가지를 MemoryDiagnoser로 비교, `dotnet-counters`로 GC 횟수 관찰 스크린샷
5. C++: 4KB 버퍼 풀 템플릿. `new/delete` vs 풀 비교, ASan 누수 0, 소멸 시 미반환 객체 수를 로그로 경고

제출물: 코드, 테스트(대여/반환/최대 보유/이중 반환), 측정 전/후 표

채점: 정확성 40 / 측정 30 / 코드 품질(루브릭 3·7번) 30

### 6.4 트랙 과제 1-3. 파일 대량 처리 CLI 툴 (심화, 12h)

목표: 순차 / 비동기 I/O / 병렬 처리의 차이를 체감한다.

요구사항
1. 입력 디렉터리의 텍스트 파일 5,000개(생성 스크립트 제공은 내가 작성)의 단어 수 집계
2. 세 버전: 순차, 비동기 I/O(C#: `File.ReadAllTextAsync` + 동시성 제한 / C++: Overlapped `ReadFile` 또는 스레드 풀), 병렬(C#: `Parallel.ForEachAsync` / C++: `std::thread` N개 + 작업 분배)
3. 처리 시간·CPU 사용률·최대 메모리를 표로 비교(작업 관리자 또는 `Get-Process`)
4. 결론 1문단: 어떤 조건(SSD/HDD, 파일 크기, 코어 수)에서 어느 버전이 유리한가

제출물: 코드, 비교표, 결론

### 6.5 부 트랙 읽기 과제 (방식 A)
- 상대 언어로 작성된 로그 큐 샘플(제공 예정)을 읽고, 자기 트랙 코드와 다른 점 5개를 표로 정리한다(예: 소유권 처리, 종료 처리, 예외 vs 에러 코드)

---

## 7. 학습 완료 판정 (Phase 1 말, 3주 금요일)

### 7.1 체크리스트
- [ ] 1-C, 1-1, 1-2 완료, CI 녹색
- [ ] 1-1 리포트에 Release 측정·분산·결론 포함
- [ ] 학습 노트 15일치, 주간 회고 3개
- [ ] 데드락 스크린샷(병렬 스택 창) 제출
- [ ] MySQL과 Redis(대체) 로컬에서 응답 확인

### 7.2 재구현(AI 없음, 60분)
빈 프로젝트에서 로그 큐(단일 방식)를 구현하고 제공 테스트 3개(순서·flush·용량 Drop)를 통과시킨다. 제공 테스트는 사전에 공개된다.

### 7.3 설명 시험 (AI 면접관, 30분) — 예시 질문
공통
- 데드락의 4조건과, 그중 하나를 깨서 예방하는 방법을 코드 예로 설명하라
- 캐시 라인과 false sharing이 처리량에 미치는 영향을 측정한 결과를 설명하라
- 평균 대신 p99를 보는 이유는 무엇인가

C#
- `await`가 컴파일되면 어떤 구조가 되는가. `MoveNext`는 누가 언제 호출하는가
- GC 세대가 3개인 이유와 LOH가 별도인 이유
- `lock` 기반 큐와 `Channel<T>`의 차이, 어느 상황에 어느 것을 쓰는가
- ThreadPool 기아가 생기는 코드 패턴과 감지 방법

C++
- `unique_ptr`과 `shared_ptr`의 비용 차이와 선택 기준, `weak_ptr`이 필요한 경우
- 이동 생성자가 호출되는 조건과 복사 생략과의 관계
- acquire/release와 seq_cst의 차이를 SPSC 큐 예로 설명하라
- SRWLock과 std::mutex의 차이

채점: 5점 척도 평균 4.0 이상(총론 §5.4)

### 7.4 결함 찾기
AI가 만든 "결함 5개 주입" 로그 큐/풀 코드에서 4개 이상을 찾아 각각의 수정안을 제시한다.

### 7.5 미달 시
- 설명 시험 3.5 미만: Phase 2 첫 주에 해당 주제 8h 보강, 재시험
- 1-1 리포트 미완: Phase 2 진행 중 완성하되 2주차 금요일까지

---

## 8. 흔한 막힘 포인트와 대처

| 증상 | 원인 | 대처 |
|---|---|---|
| BenchmarkDotNet 결과가 매번 다르다 | Debug 빌드, 백그라운드 프로세스, 전원 옵션 | Release, 고성능 전원, 다른 앱 종료, 반복 실행 후 중앙값 |
| C++ ASan이 MSVC에서 안 켜진다 | 프로젝트 속성 미설정, 호환 안 되는 옵션(/RTC) | 속성 → C/C++ → 일반 → "AddressSanitizer 사용", /RTC 끄기 |
| `dotnet-counters`가 프로세스를 못 찾는다 | 도구 미설치, 프로세스 ID 확인 오류 | `dotnet tool install -g dotnet-counters`, `dotnet-counters ps` |
| Channel 소비자가 종료되지 않는다 | `Writer.Complete()` 누락 | Stop에서 Complete 호출 후 소비 루프의 `ReadAllAsync` 종료 확인 |
| condition_variable 알림을 놓친다 | 조건 없이 wait, spurious wakeup | `wait(lock, predicate)` 형태로만 사용 |
| AI가 만든 코드가 너무 커서 못 읽겠다 | 한 번에 전체 생성 | CLAUDE.md의 "한 번에 파일 하나" 규칙 재확인, 요청을 단계로 쪼갠다 |

---

## 9. Phase 2로 넘어가기 전 준비
- Phase 2 1주차에 읽을 교재 "게임 서버 개발 네트워크부터 이해하기"를 저장소에 클론해 둔다
- Wireshark를 설치하고 루프백 캡처가 가능한지(Npcap 옵션) 확인한다
- 1-2에서 만든 버퍼 풀을 Phase 2 패킷 라이브러리가 참조할 수 있도록 별도 프로젝트(라이브러리)로 분리해 둔다
