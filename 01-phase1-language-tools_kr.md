# Phase 1. 서버 개발자를 위한 언어·도구 심화 (1~3주, 120h)

> 공통 40h / 트랙 80h. 이 Phase가 끝나면 학습자는 "문법을 아는 사람"에서 "서버 코드를 읽고 쓰고 측정할 수 있는 사람"이 된다.

## 0. 이 문서 사용법

이 문서는 **그대로 따라 하면 3주가 끝나도록** 쓰였다. 읽는 순서는 다음과 같다.

1. §1 개요를 읽고 §1.2 자가 진단을 먼저 푼다. 미달 항목이 있으면 §1.2의 보강 경로를 1주차 전에 끝낸다.
2. 매일 아침 §2의 해당 일자 블록을 펼쳐 놓고 시작한다. 각 일자는 **오전(개념) → 오후(실습·과제) → AI 없는 1시간 → 저녁 정리** 순서이며 마지막에 **완료 조건(DoD)** 체크박스가 있다. DoD를 못 채우면 다음 날 오전 30분을 보충에 쓴다.
3. §3 실습 카탈로그는 일자 블록에서 `L-C-03` 처럼 번호로 호출된다. 실습마다 목표·단계·기대 결과·확인 기준이 있으므로 그대로 수행한다.
4. §7 과제 명세는 이 Phase의 제출물이다. 과제는 일자 블록에 분산 배치되어 있으므로 §2를 따라가면 자연히 완성된다.
5. §8 학습 완료 판정은 3주차 금요일에 자기 자신에게 실시한다.

표기 규칙

| 표기 | 뜻 |
|---|---|
| `L-C-xx` | 공통 실습(Lab, Common). 트랙과 무관하게 전원 수행 |
| `L-CS-xx` | C# 트랙 실습 |
| `L-CPP-xx` | C++ 트랙 실습 |
| `1-C`, `1-1`, `1-2`, `1-3` | 과제(Assignment). 1-C는 공통, 나머지는 트랙 과제 |
| **DoD** | Definition of Done. 그날의 완료 조건. 체크박스를 실제로 채운다 |
| 🔴 | 건너뛰면 이후 Phase가 막히는 필수 항목 |
| 🟡 | 시간이 부족하면 주말이나 다음 주로 미뤄도 되는 항목 |

환경 전제(총론 §1과 동일)

- Windows 11, **Visual Studio 2026**, **.NET 10 SDK**, PowerShell 7, Git for Windows
- C++ 트랙은 MSVC + CMake 3.28+ + vcpkg(매니페스트 모드)
- DB는 이후 Phase 4에서 쓰지만 **설치 확인만 이번 주에** 한다: SQLite(기본, 설치 불필요) / (선택) MySQL 8 / Redis는 redis-windows
- CI는 쓰지 않는다. 대신 `scripts/build-and-test.ps1`을 만들어 **커밋 전에 직접 실행**한다
- 코딩 에이전트는 Claude Code를 기본 예시로 쓰되 Codex CLI 등 다른 도구도 무방하다. 지침 파일은 도구에 맞게 `CLAUDE.md` 또는 `AGENTS.md`를 쓴다

---

## 1. 개요

### 1.1 목표

이 Phase의 목표는 "언어를 더 배우는 것"이 아니라 **서버 개발자의 기본 위생을 몸에 붙이는 것**이다. 구체적으로 다음 네 가지다.

1. **동시성을 코드로 재현하고 고칠 수 있다.** 경쟁 조건·데드락·메모리 가시성 문제를 각각 30줄 이내 코드로 만들어 내고, 최소 두 가지 방법으로 고칠 수 있다.
2. **메모리를 수치로 볼 수 있다.** C#은 GC 세대·할당 바이트·LOH를, C++은 소유권·수명·누수를 도구로 확인할 수 있다.
3. **측정할 수 있다.** BenchmarkDotNet / Google Benchmark로 Release 빌드에서 워밍업·반복·분산을 갖춘 측정을 하고, 그 수치로 "A가 B보다 빠르다"를 주장할 수 있다.
4. **도구 위생.** Git 브랜치·충돌 해결, 단위 테스트, 디버거(병렬 스택·조건부 중단점), 로컬 빌드·테스트 스크립트를 손에 익힌다.

여기에 **AI 협업 규칙**(총론 §2)을 습관화한다. 특히 "설명 못 하면 커밋 금지", "매일 AI 없는 1시간"을 이 Phase에서 몸에 붙인다. 코딩 에이전트는 Claude Code든 Codex CLI든 상관없다. 중요한 것은 **지침 파일을 직접 쓰고, 생성된 코드를 검증하는 절차**다.

### 1.2 선수 조건 자가 진단 (1주차 시작 전 30분)

아래 10문항을 **AI 없이** 풀어 본다. 답을 코드로 쓰거나 종이에 적는다. 6개 미만이면 보강 경로를 먼저 수행한다.

| # | 질문 | 통과 기준 |
|---|---|---|
| 1 | 클래스와 구조체(struct)의 차이를 메모리 관점에서 설명하라 | 스택/힙, 값/참조 복사 언급 |
| 2 | `List<T>`(C#) 또는 `std::vector<T>`(C++)에 1만 개를 넣을 때 내부에서 무슨 일이 일어나는가 | 용량 증가·재할당·복사 언급 |
| 3 | 예외가 발생하면 스택은 어떻게 되는가 | 스택 되감기(unwinding), 소멸자/finally 실행 |
| 4 | 파일에 한 줄씩 1만 번 쓰는 코드를 작성하라 | 리소스 해제(using / RAII) 포함 |
| 5 | 딕셔너리(해시맵)의 평균 조회 복잡도와 최악의 경우는 | O(1) / O(n), 해시 충돌 |
| 6 | 인터페이스(추상 클래스)를 쓰는 이유 2개 | 교체 가능성, 테스트 용이성 |
| 7 | 재귀 함수를 반복문으로 바꾸는 예를 하나 쓰라 | 스택 사용량 차이 언급 |
| 8 | 문자열을 1만 번 이어 붙이면 왜 느린가 | 불변 문자열/재할당, StringBuilder 또는 reserve |
| 9 | 널 참조(nullptr) 접근이 왜 위험한가 | 정의되지 않은 동작 / 예외 |
| 10 | 단위 테스트를 한 번이라도 써 본 적이 있는가 | 있으면 통과, 없으면 §3 `L-C-04`를 먼저 |

보강 경로

- **C# 부족**: 교재 "ASP.NET Core Web API를 위한 필수 C# 가이드" 전체 속독(1일). 모르는 절만 표시해 예제를 직접 타이핑
- **C++ 부족**: 교재 "모던 C++로 시작하는 안전하고 쉬운 C++ 프로그래밍" 1~17장 중 약한 부분만(1~2일). 18장 이후(Siv3D)는 이 과정과 무관
- **테스트 경험 없음**: `L-C-04`(xUnit/GoogleTest 첫 테스트)를 1주차 화요일 오전으로 앞당긴다

### 1.3 이 Phase가 끝나면 할 수 있는 것 (검증 가능한 형태)

- 빈 프로젝트에서 **스레드 세이프 로그 큐를 60분 안에** 구현하고 제공 테스트 3개를 통과시킨다
- 데드락을 일부러 만들고 **디버거 병렬 스택 창에서 두 스레드가 서로 기다리는 지점**을 짚는다
- `Debug`와 `Release` 측정치를 비교해 "왜 Debug 측정은 의미가 없는가"를 수치와 함께 설명한다
- C#: `await` 한 줄이 컴파일러 상태 머신으로 어떻게 바뀌는지 디컴파일 결과를 보며 설명한다
- C++: `unique_ptr`/`shared_ptr`의 비용 차이를 설명하고, ASan이 잡아 준 use-after-free를 고친 이력이 있다

### 1.4 이 Phase의 산출물 목록

3주 뒤 저장소에 다음이 있어야 한다. 이것이 곧 §8 체크리스트다.

```
gameserver-course-<이름>/
├─ README.md                     과정 소개, 트랙 선택 이유, 진행 표
├─ CLAUDE.md (또는 AGENTS.md)    에이전트 지침 파일
├─ .editorconfig
├─ .gitignore
├─ scripts/
│  └─ build-and-test.ps1         빌드 + 테스트 한 번에 실행
├─ notes/
│  ├─ daily/2026-mm-dd.md        15일치
│  └─ weekly/W01.md ~ W03.md     3개
└─ phase1/
   ├─ LogQueue/                  과제 1-1 (트랙 언어)
   │  ├─ src/  tests/  bench/
   │  └─ REPORT-1-1.md
   ├─ ObjectPool/                과제 1-2
   │  ├─ src/  tests/  bench/
   │  └─ REPORT-1-2.md
   ├─ FileTool/                  과제 1-3 (심화, 선택)
   └─ labs/                      실습 결과물(실습 번호별 폴더)
      ├─ L-C-05-deadlock/        데드락 재현 + 병렬 스택 스크린샷
      ├─ L-C-06-race/
      └─ ...
```

### 1.5 3주 로드맵 한눈에 보기

| 주차 | 큰 주제 | 공통 | C# 트랙 | C++ 트랙 | 과제 진도 |
|---|---|---|---|---|---|
| 1주 | 환경·도구·메모리 | 환경 구축, Git, 단위 테스트, 지침 파일, OS 기초 | .NET 런타임·GC·값/참조·Span·ArrayPool | RAII·스마트 포인터·이동·vcpkg/CMake·ASan | 1-C 완료, 1-2 착수 |
| 2주 | 동시성 | 경쟁 조건·데드락·가시성, 디버거 심화 | async/await 내부, 동기화 도구, Channel | thread/mutex/CV/atomic, Win32 SRWLock | 1-1 두 버전 구현 |
| 3주 | 측정·심화 | 벤치마크 방법론, 리포트 작성, 평가 | ThreadPool 기아, 비동기 함정 12가지 | lock-free SPSC, false sharing, UB | 1-1·1-2 완료, 1-3 선택, 평가 |

주당 시간 배분(40h) 기준: 개념 학습 10h, 실습 8h, 과제 구현 14h, AI 없는 1시간 5h, 주간 점검 3h.

---

## 2. 주차별 상세 계획 (일 단위)

각 일자는 8시간 기준이다. 시간이 부족한 날은 🟡 항목부터 줄인다. 🔴 항목은 반드시 그날 안에 끝낸다.

### 2.1 1주차 — 환경·도구·메모리 모델

이번 주가 끝나면: 저장소가 서고, 테스트가 돌고, 지침 파일이 있고, 메모리 모델(GC 또는 소유권)을 실험으로 확인한 상태가 된다.

#### 1일차 (월) — 개발 환경 구축과 저장소 골격 🔴

**오전 (2.5h) — 설치와 확인**

1. Visual Studio 2026 설치·확인
   - C# 트랙: ".NET 데스크톱 개발" + "ASP.NET 및 웹 개발" 워크로드
   - C++ 트랙: "C++를 사용한 데스크톱 개발" 워크로드(MSVC, Windows SDK, CMake 도구 포함)
2. .NET 10 SDK 확인: PowerShell에서 `dotnet --list-sdks` → `10.x` 가 보여야 한다
3. Git 설정
   ```powershell
   git config --global user.name  "<이름>"
   git config --global user.email "<메일>"
   git config --global init.defaultBranch main
   git config --global core.autocrlf true
   ```
4. Windows Terminal + PowerShell 7 설치, 프로필 기본 셸을 PowerShell 7로
5. C++ 트랙만: vcpkg 클론 후 부트스트랩
   ```powershell
   git clone https://github.com/microsoft/vcpkg C:\dev\vcpkg
   C:\dev\vcpkg\bootstrap-vcpkg.bat
   ```
6. 🟡 이후 Phase 대비 확인만: SQLite CLI(`sqlite3 --version`), redis-windows 다운로드 후 `redis-server` 실행 → `redis-cli ping` → `PONG`

**오후 (2.5h) — 과제 1-C 착수**

- `L-C-01` 저장소 골격 만들기(§3)
- `L-C-02` `.gitignore` / `.editorconfig` 작성
- 첫 커밋: `chore: 저장소 골격과 개발 환경 설정`

**AI 없는 1시간**

- Git 명령 20개를 도움말 없이 사용해 본다: `init add commit status log diff branch checkout switch merge rebase reset restore stash tag remote push pull fetch show`
- 브랜치 2개를 만들어 같은 줄을 다르게 고치고 **일부러 충돌**을 낸 뒤 병합으로 해결한다. 해결 과정을 `notes/daily`에 적는다

**저녁 정리 (20분)**

- 학습 노트 작성(총론 템플릿 T1), 내일 아침 퀴즈 범위 지정: "프로세스와 스레드의 차이"

**DoD**

- [ ] `dotnet --list-sdks`에 10.x 표시 (C++ 트랙은 `cl.exe` 버전 확인)
- [ ] GitHub 저장소 생성, 첫 커밋 push 완료
- [ ] `.gitignore`, `.editorconfig`, `README.md`, `notes/` 폴더 존재
- [ ] 충돌을 일부러 만들고 해결한 이력이 `git log`에 있다

#### 2일차 (화) — 단위 테스트와 로컬 빌드·테스트 스크립트 🔴

**오전 (2.5h) — 개념**

- 단위 테스트의 목적: 서버는 GUI가 없다. **테스트가 곧 실행 방법**이다
- AAA 패턴(Arrange–Act–Assert), 테스트 이름 규칙(`대상_상황_기대결과`)
- 테스트 더블: 스텁 / 페이크 / 목의 차이, 이 과정에서는 **페이크 우선**(진짜에 가까운 가짜)
- 테스트 가능한 코드: 의존성을 생성자 인자로 받는다(파일 경로, 시계, 난수 시드)
- 파라미터화 테스트(`[Theory]` / `TEST_P`)

**오후 (2.5h) — 실습·과제**

- `L-C-03` 테스트 프로젝트 만들기(C#: xUnit / C++: GoogleTest + vcpkg)
- `L-C-04` 첫 테스트 5개 작성: 문자열 파서(`"k=v;k2=v2"` → 딕셔너리)에 대해 정상·빈 문자열·중복 키·구분자 없음·공백 처리
- `L-C-05` `scripts/build-and-test.ps1` 작성(§3에 스크립트 전문)

**AI 없는 1시간**

- 위 파서를 **빈 파일에서 다시** 구현하고 테스트 5개를 통과시킨다(구현 40분, 테스트 20분)

**저녁 정리**

- 노트에 "테스트를 먼저 쓰면 설계가 어떻게 달라지는가" 3줄

**DoD**

- [ ] `pwsh scripts/build-and-test.ps1` 한 줄로 빌드+테스트가 돌고 초록색으로 끝난다
- [ ] 테스트 5개 이상 통과, 일부러 하나를 깨뜨려 빨간색을 확인한 뒤 되돌린 이력
- [ ] 커밋 2개 이상(테스트 추가 / 스크립트 추가)

#### 3일차 (수) — 에이전트 지침 파일과 AI 협업 루틴 🔴

**오전 (2.5h) — 개념**

- 교재 "OpenCode로 시작하는 AI 코딩 에이전트" 가이드 1 Part 2(세션·Plan/Build 모드·AGENTS.md), Part 5(자동화)
- 지침 파일이 하는 일: 프로젝트 구조·규칙·"학습용"이라는 목적을 에이전트에게 고정시킨다
- AI 4역할(총론 §2.1) 실습: 튜터 / 페어 / 리뷰어 / 출제자를 각각 한 번씩 써 본다

**오후 (2.5h) — 과제 1-C 마무리**

- `L-C-06` 지침 파일(`CLAUDE.md` 또는 `AGENTS.md`) 초안 작성 — 템플릿은 `08-templates.md` T3
- 에이전트에게 "이 지침대로 **파일 하나만**" 생성시켜 보고, 지침을 어기면 어디가 모호했는지 지침을 고친다(2회 반복)
- README에 트랙 선택 이유 3문장 + 진행 표 작성 → **과제 1-C 제출 상태로 만든다**

**AI 없는 1시간**

- 오늘 쓴 프롬프트 5개를 노트에 손으로 다시 적고, 각각 "무엇을 시켰고 무엇을 검증했는가"를 한 줄씩 붙인다

**저녁 정리**

- 지침 파일 v1을 커밋. 이후 매주 금요일에 갱신한다

**DoD**

- [ ] 과제 1-C 완료(§7.1 채점표로 자기 채점 80점 이상)
- [ ] 지침 파일에 "한 번에 파일 하나", "설계 의도 주석", "테스트 동반" 규칙이 문장으로 들어 있다
- [ ] 에이전트가 지침을 어긴 사례 1개와 지침 수정 내역이 노트에 있다

#### 4일차 (목) — OS 기초와 메모리 모델 실험 🔴

**오전 (2.5h) — 개념**

- 프로세스 / 스레드 / 핸들, 컨텍스트 스위칭 비용(수 µs), 사용자↔커널 모드 전환
- 가상 메모리와 페이지, 워킹셋, 페이지 폴트
- CPU 캐시 계층과 캐시 라인(64B), false sharing이 왜 생기는가
- 스레드를 많이 만들면 왜 느려지는가(스위칭 + 캐시 오염 + 메모리)

**오후 (2.5h) — 트랙 실습**

- 공통: `L-C-07` 스레드 수 1→2→4→64→1,000으로 늘리며 같은 작업 처리 시간 측정 표 만들기
- C#: `L-CS-01` GC 세대 관찰, `L-CS-02` LOH 확인(85KB 경계)
- C++: `L-CPP-01` unique_ptr/shared_ptr 수명 추적, `L-CPP-02` 순환 참조 누수 재현 → weak_ptr로 수정

**AI 없는 1시간**

- 스레드 2개로 공유 카운터를 100만 번 증가시켜 **결과가 틀리는 것**을 재현한다. 왜 틀리는지 어셈블리 수준(읽기-수정-쓰기)으로 노트에 적는다

**저녁 정리**

- 측정 표를 `notes/daily`에 붙이고 "스레드가 몇 개일 때 가장 빨랐는가, 왜인가" 3줄

**DoD**

- [ ] 스레드 수별 처리 시간 표(5개 구간) 완성
- [ ] 트랙 실습 2개 완료, 결과 스크린샷 또는 출력 로그 저장
- [ ] 카운터 경쟁 조건 재현 코드가 `phase1/labs/`에 있다

#### 5일차 (금) — 과제 1-2 착수 + 주간 점검

**오전 (2.5h) — 과제 1-2 설계**

- 교재 "게임 개발자를 위한 C# 디자인 패턴" 3장(Object Pool) 읽기(C++ 트랙도 개념용으로 읽고 C++로 옮김)
- 과제 1-2(§7.3) 요구사항 정독 후 **직접 설계 먼저**: API 시그니처, 상태, 통계 필드를 종이나 마크다운에 적는다
- 그 다음에야 에이전트에게 골격을 요청한다(설계는 내 것, 보일러플레이트만 위임)

**오후 (4h) — 주간 점검** (총론 §5.1)

1. **AI 없이 재구현(60분)**: 2일차 파서를 빈 파일에서 다시 구현 + 테스트 5개 통과
2. **설명 시험(45분)**: AI 면접관에게 "프로세스/스레드", "캐시 라인", "GC 세대 또는 RAII" 3주제 설명, 5점 척도 채점
3. **코드 리뷰(45분)**: 이번 주 커밋 전체를 루브릭(총론 §5.3)으로 리뷰 요청 → 상위 3개 문제 수정
4. **주간 회고(30분)**: 템플릿 T2로 `notes/weekly/W01.md` 작성

**DoD**

- [ ] 과제 1-2 설계 문서(API·상태·통계) 초안이 `phase1/ObjectPool/DESIGN-note.md`에 있다
- [ ] 주간 점검 4항목 완료, 설명 시험 평균 점수 기록
- [ ] `notes/weekly/W01.md` 작성, 학습 노트 5일치 존재

### 2.2 2주차 — 동시성

이번 주가 끝나면: 경쟁 조건·데드락·가시성 문제를 스스로 만들고 고칠 수 있으며, 과제 1-1의 두 버전이 동작한다.

#### 6일차 (월) — 동시성의 세 가지 병 🔴

**오전 (2.5h) — 개념**

- 경쟁 조건(race condition)과 임계 구역
- 데드락 4조건(상호 배제, 점유 대기, 비선점, 순환 대기)과 각 조건을 깨는 방법
- 메모리 가시성: 컴파일러 재배치, CPU 재배치, 캐시 일관성
- 락 종류: 뮤텍스 / 스핀락 / 읽기-쓰기 락, 각각의 적합한 상황

**오후 (2.5h) — 실습**

- `L-C-08` 경쟁 조건 최소 재현 + 3가지 수정(락 / 원자 연산 / 스레드 로컬 합산 후 병합)
- `L-C-09` 데드락 최소 재현(두 락을 반대 순서로) + 2가지 수정(락 순서 통일 / try-lock + 백오프)
- `L-C-10` 가시성 문제 재현(플래그를 통한 종료 신호가 안 보이는 경우) + 수정(volatile/atomic)

**AI 없는 1시간**

- 위 3개를 **주석 없이** 다시 작성하고, 각각 무엇이 문제인지 자기 말로 주석을 단다

**저녁 정리**

- "내 서버 코드에서 이 세 가지가 나올 만한 자리"를 3개 예상해 노트에 적는다

**DoD**

- [ ] 세 가지 문제 재현 코드와 수정본이 각각 `phase1/labs/`에 있다
- [ ] 각 문제마다 수정 방법 2가지 이상을 노트에 기록

#### 7일차 (화) — 디버거 심화 🔴

**오전 (2.5h) — 개념·도구**

- 조건부 중단점, 적중 횟수 중단점, 데이터 중단점(C++)
- 병렬 스택 창 / 스레드 창 / 작업(Task) 창 읽는 법
- 메모리 창, 조사식, `Debugger.Break()` / `__debugbreak()`
- 크래시 덤프: 작업 관리자 또는 `procdump`로 저장 → VS로 열기

**오후 (2.5h) — 실습**

- `L-C-11` 어제 만든 데드락을 디버거로 잡기: 병렬 스택 창에서 순환 대기 지점 **스크린샷** 저장(§8 체크리스트 항목)
- `L-C-12` 조건부 중단점으로 "1만 번째 반복에서만 멈추기"
- `L-C-13` 크래시 덤프 만들고 열어서 크래시 라인 특정

**AI 없는 1시간**

- 에이전트에게 결함 3개를 숨긴 코드를 만들게 한 뒤(총론 T11), **디버거만으로** 찾는다

**저녁 정리**

- 노트에 "로그로 못 잡고 디버거로만 잡히는 버그의 특징" 3줄

**DoD**

- [ ] 병렬 스택 창 데드락 스크린샷이 `phase1/labs/L-C-05-deadlock/`에 있다
- [ ] 덤프 파일 1개를 열어 크래시 라인을 특정한 기록

#### 8일차 (수) — 트랙 동시성 도구 ①

**C# 트랙 오전·오후 (5h)**

- 개념: `lock`(Monitor), `Interlocked`, `SemaphoreSlim`, `ReaderWriterLockSlim`, `ConcurrentQueue/Dictionary`
- `L-CS-03` 같은 카운터를 lock / Interlocked / Concurrent 컬렉션으로 각각 구현하고 처리량 비교
- `L-CS-04` `Monitor.Wait/Pulse`로 생산자-소비자 큐 만들기 → **과제 1-1 (A) 버전의 뼈대**

**C++ 트랙 오전·오후 (5h)**

- 개념: `std::thread/jthread`, `mutex/lock_guard/scoped_lock`, `condition_variable`, `atomic`
- `L-CPP-03` 같은 카운터를 mutex / atomic / 스레드 로컬 합산으로 구현하고 처리량 비교
- `L-CPP-04` `condition_variable`로 생산자-소비자 큐 만들기(반드시 `wait(lock, pred)` 형태) → **과제 1-1 (A) 버전의 뼈대**

**AI 없는 1시간**

- 생산자-소비자 큐를 **빈 파일에서** 다시 구현(30분) → 테스트 2개(순서, 종료) 작성(30분)

**DoD**

- [ ] 카운터 3방식 처리량 비교 표
- [ ] 생산자-소비자 큐 초안이 컴파일되고 테스트 2개 통과

#### 9일차 (목) — 트랙 동시성 도구 ②

**C# 트랙 (5h)**

- 개념: async/await 상태 머신, `Task` vs `ValueTask`, `ConfigureAwait`, `CancellationToken`, `async void` 금지
- `L-CS-05` ILSpy로 자기 `async` 메서드를 디컴파일해 상태 머신 클래스와 `MoveNext` 흐름 확인
- `L-CS-06` `Channel<T>`(Bounded) 생산자-소비자 → **과제 1-1 (B) 버전의 뼈대**
- 교재 "C# Async/Await 완전 정복" 1~4장

**C++ 트랙 (5h)**

- 개념: `atomic`과 메모리 오더(relaxed/acquire/release/seq_cst), Win32 `SRWLOCK`/`CONDITION_VARIABLE`
- `L-CPP-05` 표준 `mutex` 버전과 `SRWLOCK` 버전 성능 비교
- `L-CPP-06` SPSC 링 버퍼 뼈대(3주차에 lock-free로 완성) → **과제 1-1 (B) 버전의 뼈대**
- 교재 "모던 Windows 멀티스레딩" 3~5·8장

**AI 없는 1시간**

- C#: `await` 앞뒤로 스레드 ID를 찍어 어디서 스레드가 바뀌는지 확인하고 이유를 적는다
- C++: `acquire/release` 쌍이 왜 필요한지 SPSC 큐로 설명하는 글 10줄

**DoD**

- [ ] 트랙별 (B) 버전 뼈대가 컴파일된다
- [ ] C#: 상태 머신 디컴파일 캡처 / C++: SRWLock 비교 표

#### 10일차 (금) — 과제 1-1 구현 집중 + 주간 점검

**오전 (3h)** — 과제 1-1(§7.2)의 (A)(B) 두 버전을 기능 완성 수준까지: `Enqueue/Start/Stop`, 통계, 용량 정책(Drop/Block)

**오후 (4h) — 주간 점검**

1. AI 없이 재구현(60분): 생산자-소비자 큐 + 종료 처리
2. 설명 시험(45분): "데드락 4조건", "경쟁 조건 수정 3가지", 트랙 주제 1개
3. 코드 리뷰(45분): 루브릭 2번(동시성) 중심으로 AI 리뷰 → 상위 3개 수정
4. 주간 회고(30분): `notes/weekly/W02.md`

**DoD**

- [ ] 1-1의 두 버전이 테스트(순서·flush·용량 정책)를 통과
- [ ] 주간 회고 작성, 학습 노트 10일치

### 2.3 3주차 — 측정과 심화

이번 주가 끝나면: 신뢰할 수 있는 벤치마크 결과와 리포트가 있고, Phase 1 평가를 통과한 상태가 된다.

#### 11일차 (월) — 벤치마크 방법론 🔴

**오전 (2.5h) — 개념**

- 워밍업(JIT/캐시), 반복 횟수, 평균 vs 중앙값 vs p99, 표준편차 표기
- Debug/Release 차이, 최적화로 코드가 사라지는 문제(결과를 소비해야 한다)
- 측정 대상 격리(파일 I/O를 빼고 큐만 재기), 같은 조건 유지(전원 옵션·백그라운드 앱)

**오후 (2.5h) — 실습**

- C#: `L-CS-07` BenchmarkDotNet 프로젝트 만들기 + `MemoryDiagnoser` + `[Params]`로 생산자 수 1/4/16
- C++: `L-CPP-07` Google Benchmark 프로젝트 만들기 + `benchmark::DoNotOptimize`
- 공통: `L-C-14` 같은 코드를 Debug/Release로 각각 측정해 표로 비교

**AI 없는 1시간**

- "왜 Debug 측정은 의미가 없는가"를 자기 수치를 근거로 한 문단 쓴다

**DoD**

- [ ] 벤치마크 프로젝트가 Release로 돌고 결과 표가 나온다
- [ ] Debug/Release 비교 표 완성

#### 12일차 (화) — 과제 1-1 측정과 리포트

**오전 (2.5h)** — 1-1 두 버전 벤치마크: 생산자 1/4/16, 처리량(msg/s), 항목당 할당 바이트, p99 지연

**오후 (2.5h)** — `REPORT-1-1.md` 작성(§7.2 리포트 템플릿 그대로): 환경 → 측정 방법 → 표 → 왜 차이가 나는지 3문단 → 선택 기준 결론

**AI 없는 1시간**

- 리포트 결론 문단을 AI 없이 다시 쓴다. 그 뒤 AI에게 "내 해석에서 놓친 요인"만 지적받는다

**DoD**

- [ ] `REPORT-1-1.md` 완성(표·분산·결론·"AI가 틀린 것" 1개 이상)
- [ ] 과제 1-1 자기 채점(§7.2) 80점 이상

#### 13일차 (수) — 과제 1-2 완성과 측정

**오전 (2.5h)** — 1-2 오브젝트 풀 완성: `Rent/Return`, 최대 보유 수, 통계, 이중 반환 감지

**오후 (2.5h)** — 측정
- C#: `new byte[4096]` vs 자작 풀 vs `ArrayPool<byte>.Shared` 3자 비교(`MemoryDiagnoser`), `dotnet-counters`로 GC 횟수 관찰
- C++: `new/delete` vs 풀 비교, ASan 빌드에서 누수 0 확인

**AI 없는 1시간**

- 풀의 "이중 반환 감지"를 다른 방식으로 한 번 더 구현(예: 대여 토큰 방식)

**DoD**

- [ ] 1-2 테스트 4종(대여/반환/최대 보유/이중 반환) 통과
- [ ] 3자 비교 표 완성, `REPORT-1-2.md` 작성

#### 14일차 (목) — 트랙 심화

**C# 트랙 (5h)**

- ThreadPool 기아 재현·진단(`dotnet-counters`의 큐 길이), `.Result` 동기 대기의 위험
- 교재 "C# Async/Await 완전 정복" 6장 함정 12가지 → `L-CS-08` 12개를 **각각 재현하고 수정**한 프로젝트
- 🟡 `L-CS-09` Span/ArrayPool로 문자열 파싱 할당 줄이기 전후 비교

**C++ 트랙 (5h)**

- lock-free SPSC 링 버퍼 완성(`L-CPP-08`), acquire/release 근거를 주석으로
- `L-CPP-09` false sharing 유무 두 버전 벤치마크(`alignas(64)`)
- 🟡 `L-CPP-10` UB 목록 실습: 부호 오버플로·초기화 안 된 읽기·댕글링을 각각 재현하고 컴파일러 최적화 영향 관찰

**AI 없는 1시간**

- C#: 함정 12가지 중 3개를 빈 파일에서 재현
- C++: SPSC 링 버퍼를 메모리 오더 주석 없이 다시 쓰고, 나중에 주석을 채운다

**DoD**

- [ ] C#: 함정 재현 프로젝트 12개 케이스 / C++: lock-free 큐 ASan 통과 + false sharing 비교 표

#### 15일차 (금) — Phase 1 평가 🔴

**오전 (3h) — 평가 1~2**

1. **재구현 시험(60분, AI 없음)**: §8.2 — 빈 프로젝트에서 로그 큐 단일 방식 구현 + 제공 테스트 3개 통과
2. **설명 시험(60분)**: §8.3 질문 은행에서 무작위 10문항, 5점 척도 채점
3. **결함 찾기(45분)**: §8.4 — AI가 결함 5개를 숨긴 큐/풀 코드에서 4개 이상 발견

**오후 (3h) — 마무리**

- 체크리스트(§8.1) 전 항목 점검, 미완 항목 보충
- 주간 회고 `W03.md` + Phase 1 회고(무엇이 가장 어려웠나, AI를 어디서 잘못 썼나)
- §10 Phase 2 준비 항목 수행(버퍼 풀을 라이브러리 프로젝트로 분리)

**DoD**

- [ ] §8.1 체크리스트 전 항목 완료
- [ ] 설명 시험 평균 4.0 이상(미달 시 §8.5 보강 계획 수립)
- [ ] 1-2 풀이 별도 라이브러리 프로젝트로 분리되어 Phase 2에서 참조 가능

---

## 3. 실습 카탈로그

각 실습은 **목표 / 준비 / 단계 / 기대 결과 / 확인 기준 / 자주 나는 오류 / 노트에 남길 것** 형식이다. 결과물은 `phase1/labs/<실습번호>/`에 저장한다.

### 3.1 공통 실습 (L-C)

#### L-C-01 저장소 골격 만들기 (40분) 🔴

- **목표**: 26주 동안 쓸 저장소 구조를 만든다
- **준비**: GitHub 계정, Git 설치 완료
- **단계**
  1. GitHub에서 `gameserver-course-<이름>` 저장소 생성(README 체크 해제, 빈 저장소)
  2. 로컬에서:
     ```powershell
     mkdir C:\dev\gameserver-course-<이름>; cd C:\dev\gameserver-course-<이름>
     git init
     mkdir notes\daily, notes\weekly, phase1\labs, scripts
     "# 게임 서버 부트캠프 학습 저장소" | Out-File README.md -Encoding utf8
     git remote add origin https://github.com/<계정>/gameserver-course-<이름>.git
     ```
  3. README에 다음 절 만들기: 과정 소개 / 트랙 선택(1주 수요일까지 확정) / 진행 표(Phase별 과제 체크박스) / 실행 방법
  4. 첫 커밋 후 `git push -u origin main`
- **기대 결과**: GitHub에 §1.4 구조의 뼈대가 보인다
- **확인 기준**: 다른 PC에서 clone 했을 때 폴더 구조가 그대로 재현된다(빈 폴더는 `.gitkeep`으로 유지)
- **자주 나는 오류**: 빈 폴더가 Git에 안 올라감 → 각 폴더에 `.gitkeep` 파일 추가
- **노트에 남길 것**: 왜 `notes/`를 코드와 같은 저장소에 두는가(1줄)

#### L-C-02 `.gitignore` / `.editorconfig` (25분)

- **목표**: 빌드 산출물이 커밋되지 않게 하고 코드 스타일을 고정한다
- **단계**
  1. `.gitignore`: C#은 `bin/ obj/ *.user`, C++은 `build/ out/ .vs/ CMakeUserPresets.json`, 공통으로 `BenchmarkDotNet.Artifacts/ *.dmp *.etl`
  2. `.editorconfig`: 들여쓰기(C# 4칸, C++ 4칸), `end_of_line = crlf`, `charset = utf-8`, C#은 `dotnet_diagnostic.CA2007.severity` 등 분석기 규칙 3개 이상 명시
  3. 일부러 빌드해 `bin/`이 생기게 한 뒤 `git status`에 안 뜨는지 확인
- **확인 기준**: `git status`가 깨끗하다
- **노트에 남길 것**: 분석기 경고를 오류로 올리는 이유(1줄)

#### L-C-03 테스트 프로젝트 만들기 (40분) 🔴

- **목표**: 이후 모든 과제가 얹힐 테스트 기반을 만든다
- **단계 (C#)**
  ```powershell
  cd phase1
  dotnet new sln -n Phase1
  dotnet new classlib -n Phase1.Core   -f net10.0
  dotnet new xunit    -n Phase1.Tests  -f net10.0
  dotnet sln add Phase1.Core Phase1.Tests
  dotnet add Phase1.Tests reference Phase1.Core
  dotnet test
  ```
- **단계 (C++)**
  1. `phase1/CMakeLists.txt`에 `cmake_minimum_required(VERSION 3.28)`, `project(Phase1 CXX)`, `set(CMAKE_CXX_STANDARD 23)`, `enable_testing()`
  2. `vcpkg.json` 매니페스트에 `gtest`, `benchmark` 추가
  3. `add_subdirectory(src)`, `add_subdirectory(tests)` 구성 후
     ```powershell
     cmake -B build -S . -DCMAKE_TOOLCHAIN_FILE=C:/dev/vcpkg/scripts/buildsystems/vcpkg.cmake
     cmake --build build --config Debug
     ctest --test-dir build -C Debug
     ```
- **확인 기준**: 테스트 0개라도 러너가 "성공"으로 끝난다
- **자주 나는 오류**: vcpkg 툴체인 경로 누락 → `-DCMAKE_TOOLCHAIN_FILE` 확인

#### L-C-04 첫 테스트 5개 — 설정 문자열 파서 (60분) 🔴

- **목표**: AAA 패턴과 테스트 이름 규칙을 손에 익힌다
- **대상**: `"port=9000;workers=4;name=svr"` → `{port:9000, workers:4, name:svr}` 로 바꾸는 `ConfigParser.Parse(string)`
- **테스트 5개(반드시 이 5개)**
  | # | 이름 | 입력 | 기대 |
  |---|---|---|---|
  | 1 | `Parse_정상입력_키값3개반환` | `"a=1;b=2;c=3"` | 3개 항목 |
  | 2 | `Parse_빈문자열_빈결과` | `""` | 0개, 예외 없음 |
  | 3 | `Parse_중복키_마지막값우선` | `"a=1;a=2"` | `a=2` |
  | 4 | `Parse_구분자없음_예외` | `"a1;b=2"` | `FormatException`(C++는 `std::invalid_argument` 또는 오류 코드) |
  | 5 | `Parse_공백포함_트림` | `" a = 1 ; b = 2 "` | `a=1,b=2` |
- **단계**: 테스트 5개를 **먼저** 쓰고 빨간불 확인 → 구현 → 초록불
- **확인 기준**: 5개 통과, 테스트 이름만 읽어도 무엇을 검증하는지 안다
- **노트에 남길 것**: 테스트를 먼저 썼을 때 API 설계가 달라진 점 1개

#### L-C-05 로컬 빌드·테스트 스크립트 (45분) 🔴

- **목표**: CI 없이도 "한 줄로 전부 검증"되는 습관을 만든다
- **단계**: `scripts/build-and-test.ps1` 작성. 아래를 그대로 쓰고 자기 경로에 맞게 고친다
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
- **확인 기준**: 테스트를 하나 일부러 깨뜨리면 `exit 1`과 빨간 메시지, 되돌리면 `ALL GREEN`
- **노트에 남길 것**: CI 없이 이 습관을 유지하려면 무엇이 필요한가(1줄)

#### L-C-06 에이전트 지침 파일 작성 (60분) 🔴

- **목표**: 에이전트를 "학습을 돕는 페어"로 고정한다
- **단계**
  1. `08-templates.md`의 T3(C#) 또는 T3'(C++)을 복사해 `CLAUDE.md`(또는 `AGENTS.md`) 작성
  2. 최소 포함: 프로젝트 구조 / 언어·버전(.NET 10, C++23) / 코딩 규칙 5개 / "설계 의도 주석" / "한 번에 파일 하나" / "테스트 동반" / "라이브러리 추가 시 이유·대안"
  3. 에이전트에게 시험 지시: "이 지침에 따라 `ConfigParser` 파일 **하나만** 만들어라"
  4. 지침 위반이 나오면(예: 여러 파일 생성, 주석 없음) 어느 문장이 모호했는지 찾아 고친다 → 다시 시험(2회 반복)
- **확인 기준**: 3번 지시에서 지침대로 파일 하나만, 주석과 테스트가 함께 나온다
- **노트에 남길 것**: 모호했던 문장 → 고친 문장(before/after)

#### L-C-07 스레드 수와 처리 시간 (60분)

- **목표**: "스레드를 많이 만들면 빨라진다"는 착각을 수치로 깬다
- **작업 내용**: 1억 번의 정수 덧셈을 스레드 N개로 분할 처리
- **단계**: N = 1, 2, 4, 논리코어수, 64, 1000 각각 5회 측정, 중앙값 기록
- **기대 결과**: 논리 코어 수 근처에서 최고, 이후 하락
- **확인 기준**: 표에 N / 중앙값 / 최고 대비 비율 / 관찰 메모가 있다
- **노트에 남길 것**: 1,000 스레드에서 느려진 이유 3가지(스위칭·캐시·메모리)

#### L-C-08 경쟁 조건 재현과 3가지 수정 (60분) 🔴

- **단계**
  1. 스레드 4개가 공유 카운터를 각 100만 번 증가 → 결과가 400만보다 작음을 확인(10회 반복해 편차 기록)
  2. 수정 A: 락으로 감싸기
  3. 수정 B: 원자적 증가(`Interlocked.Increment` / `std::atomic::fetch_add`)
  4. 수정 C: 스레드별 지역 카운터 합산 후 마지막에 병합
  5. 세 방식의 처리 시간 비교
- **확인 기준**: 세 방식 모두 정확히 400만, 처리 시간 표 존재
- **노트에 남길 것**: C가 가장 빠른 이유(캐시 라인 경합)

#### L-C-09 데드락 재현과 2가지 수정 (45분) 🔴

- **단계**: 락 A→B 순서로 잠그는 스레드와 B→A 순서로 잠그는 스레드 → 멈춤 확인 → (수정1) 락 순서 통일 (수정2) `try_lock` + 실패 시 전부 해제 후 백오프
- **확인 기준**: 재현 코드가 5초 내 100% 멈추고, 수정본은 10만 회 반복에도 통과
- **노트에 남길 것**: 데드락 4조건 중 각 수정이 어떤 조건을 깼는가

#### L-C-10 메모리 가시성 문제 (45분)

- **단계**: 일반 `bool` 플래그로 워커 종료 신호 → Release 빌드에서 종료되지 않는 현상 재현 시도 → `volatile`(C#) / `std::atomic<bool>`(C++)로 수정
- **주의**: x86은 강한 메모리 순서라 재현이 안 될 수 있다. 이때는 "왜 x86에서는 잘 되는데 ARM에서는 깨지는가"를 조사해 노트에 적는다(이것이 이 실습의 진짜 목적)
- **확인 기준**: 재현 여부와 무관하게 "가시성 ≠ 원자성" 설명을 자기 말로 쓴다

#### L-C-11 디버거로 데드락 잡기 (45분) 🔴

- **단계**: L-C-09 재현 코드 실행 → VS에서 "프로세스에 연결" 또는 F5 → 모두 중단 → **병렬 스택 창**에서 두 스레드가 서로 대기하는 지점 확인 → 스크린샷 저장
- **확인 기준**: 스크린샷에 두 스레드의 대기 지점이 보이고, 노트에 "누가 무엇을 기다리는지" 문장으로 적혀 있다

#### L-C-12 조건부 중단점 (30분)

- **단계**: 10만 회 반복 루프에서 "i == 99999일 때만" 멈추는 조건부 중단점, "10회마다" 적중 횟수 중단점, C++은 특정 변수 변경 시 멈추는 데이터 중단점
- **확인 기준**: 세 종류를 각각 사용한 스크린샷 또는 절차 메모

#### L-C-13 크래시 덤프 수집·분석 (45분)

- **단계**: 널 참조로 크래시하는 콘솔 앱 → 작업 관리자에서 "덤프 파일 만들기"(또는 `procdump -ma <pid>`) → VS로 `.dmp` 열기 → 크래시 스택에서 원인 라인 특정
- **확인 기준**: 덤프에서 특정한 라인 번호가 실제 코드와 일치

#### L-C-14 Debug vs Release 측정 비교 (40분) 🔴

- **단계**: 동일 루프(문자열 파싱 또는 정수 연산)를 Debug/Release로 각각 측정, 배수 계산
- **확인 기준**: 표에 두 구성의 수치와 배수, "Debug 수치를 리포트에 쓰면 안 되는 이유" 한 문단

### 3.2 C# 트랙 실습 (L-CS)

#### L-CS-01 GC 세대 관찰 (45분)

- **단계**: 작은 객체 100만 개 생성 → `GC.CollectionCount(0/1/2)`와 `GC.GetTotalAllocatedBytes()` 출력 → 일부를 정적 리스트에 살려 두고 다시 측정 → 세대 승격 관찰
- **확인 기준**: "살아남은 객체가 많을수록 Gen1/Gen2 수집이 늘어난다"를 자기 수치로 설명

#### L-CS-02 LOH 경계 확인 (30분)

- **단계**: `new byte[84_000]`과 `new byte[86_000]`을 각각 10만 번 → `GC.GetGeneration`, LOH 크기 관찰(`dotnet-counters monitor --counters System.Runtime`)
- **확인 기준**: 85KB 경계 전후의 동작 차이를 수치로 확인

#### L-CS-03 카운터 3방식 비교 (45분)

- **단계**: `lock` / `Interlocked.Increment` / `ConcurrentQueue`에 넣고 세기 → 처리량 비교
- **확인 기준**: 표 + "왜 Interlocked가 lock보다 빠른가" 설명

#### L-CS-04 Monitor 기반 생산자-소비자 큐 (90분) 🔴

- **골격**
  ```csharp
  public sealed class MonitorQueue<T>
  {
      private readonly Queue<T> _q = new();
      private readonly int _capacity;
      private bool _completed;

      public bool TryEnqueue(T item);        // 용량 초과 시 정책에 따라 Drop/Block
      public bool TryDequeue(out T item);    // 비었으면 Wait, Complete면 false
      public void Complete();                // 소비자를 깨워 종료시킴
  }
  ```
- **확인 기준**: 생산자 4 / 소비자 1로 10만 건 왕복, 유실 0, `Complete()` 후 소비자 스레드가 5초 내 종료

#### L-CS-05 async 상태 머신 디컴파일 (45분)

- **단계**: 간단한 `async Task<int> FooAsync()` 작성 → Release 빌드 → ILSpy(또는 `dotnet-ildasm`)로 열어 `<FooAsync>d__0` 상태 머신 클래스와 `MoveNext` 확인
- **확인 기준**: 상태(`<>1__state`) 전이와 `AwaitUnsafeOnCompleted` 호출 지점을 자기 말로 설명

#### L-CS-06 Channel 기반 큐 (90분) 🔴

- **단계**: `FullMode=Wait`로 구성한다. Drop 정책은 `TryWrite`의 `false`를 직접 집계하고, Block 정책은 `WriteAsync`를 사용한다. `ReadAllAsync`로 소비하고 `Writer.Complete()`로 종료한다
- **확인 기준**: L-CS-04와 동일 시나리오 통과 + 코드 줄 수 비교(대체로 절반 이하)

#### L-CS-07 BenchmarkDotNet 설정 (60분) 🔴

- **단계**
  ```powershell
  dotnet new console -n Phase1.Bench -f net10.0
  dotnet add Phase1.Bench package BenchmarkDotNet
  ```
  `[MemoryDiagnoser]`, `[SimpleJob(warmupCount:3, iterationCount:10)]`, `[Params(1,4,16)]` 적용, `dotnet run -c Release`
- **확인 기준**: 결과에 Mean/Error/StdDev/Allocated가 나오고 Debug로 실행하면 경고가 뜬다

#### L-CS-08 비동기 함정 12가지 재현 (180분)

- **단계**: 교재 "C# Async/Await 완전 정복" 6장의 12개 함정을 각각 **재현 → 증상 확인 → 수정** 순으로 별도 폴더에 만든다(`Trap01_Deadlock`, `Trap02_LostException`, …)
- **확인 기준**: 12개 폴더 각각에 재현 코드·수정 코드·1줄 설명

#### L-CS-09 Span/ArrayPool로 할당 줄이기 (60분) 🟡

- **단계**: `L-C-04` 파서를 (a) `Substring` 버전 (b) `ReadOnlySpan<char>` 버전으로 만들고 `MemoryDiagnoser` 비교
- **확인 기준**: 할당 바이트가 유의미하게 감소(대체로 0에 근접)

#### L-CS-10 ThreadPool 기아 재현 (60분)

- **단계**: `Task.Run` 안에서 `.Result`로 동기 대기하는 작업을 코어 수보다 많이 던진다 → 응답 없음 재현 → `dotnet-counters monitor --counters System.Runtime`로 `threadpool-queue-length` 증가 확인 → `await`로 수정
- **확인 기준**: 큐 길이 그래프 캡처 + 수정 후 정상화

#### L-CS-11 Task vs ValueTask (40분) 🟡

- **단계**: 동기 완료가 잦은 메서드를 두 버전으로 만들고 할당 비교
- **확인 기준**: "언제 ValueTask가 이득인가/위험한가(두 번 await 금지)" 정리

#### L-CS-12 CancellationToken 전파 (40분)

- **단계**: 3단계 중첩 async 호출에 토큰을 전파하고 취소 시 `OperationCanceledException` 처리, 토큰을 빠뜨린 버전과 비교
- **확인 기준**: 취소가 1초 내 전파되고, 빠뜨린 버전은 계속 도는 것을 확인

### 3.3 C++ 트랙 실습 (L-CPP)

#### L-CPP-01 스마트 포인터 수명 추적 (45분)

- **단계**: 소멸자에서 로그를 찍는 `Tracked` 클래스로 `unique_ptr` 이동, `shared_ptr` 복사 시 `use_count()` 변화 관찰
- **확인 기준**: 언제 소멸자가 불리는지 예측한 뒤 실행 결과와 비교(예측 5개 중 4개 이상 적중)

#### L-CPP-02 순환 참조 누수와 weak_ptr (45분) 🔴

- **단계**: 서로 `shared_ptr`로 참조하는 Node 두 개 → 소멸자 미호출 확인 → 한쪽을 `weak_ptr`로 교체 → 소멸 확인
- **확인 기준**: 수정 전 "소멸자 로그 없음", 수정 후 "둘 다 소멸"

#### L-CPP-03 카운터 3방식 비교 (45분)

- **단계**: `mutex` / `atomic<int>::fetch_add(relaxed)` / 스레드 로컬 합산 → 처리량 비교
- **확인 기준**: 표 + "relaxed로 충분한 이유"(합계만 필요, 순서 불필요) 설명

#### L-CPP-04 condition_variable 생산자-소비자 큐 (90분) 🔴

- **골격**
  ```cpp
  template <class T>
  class BlockingQueue {
  public:
      explicit BlockingQueue(size_t capacity);
      bool TryPush(T v);          // 용량 초과 시 정책(Drop/Block)
      bool Pop(T& out);           // 비었으면 대기, Complete면 false
      void Complete();            // 소비자 깨워 종료
  private:
      std::mutex m_;
      std::condition_variable cv_;
      std::deque<T> q_;
      size_t cap_;
      bool completed_ = false;
  };
  ```
- **주의**: `cv_.wait(lock, [&]{ return !q_.empty() || completed_; })` 형태만 사용(술어 없는 wait 금지)
- **확인 기준**: 생산자 4 / 소비자 1로 10만 건, 유실 0, 종료 5초 이내

#### L-CPP-05 SRWLock vs std::mutex (45분)

- **단계**: 동일 임계 구역을 `std::mutex`와 `SRWLOCK`(`AcquireSRWLockExclusive`)으로 각각 구현해 처리량 비교, 읽기 위주 워크로드에서 `AcquireSRWLockShared`도 측정
- **확인 기준**: 3가지 수치 표 + "읽기 공유 락이 언제 이득인가" 설명

#### L-CPP-06 SPSC 링 버퍼 뼈대 (60분)

- **단계**: 고정 크기 배열 + head/tail 인덱스로 단일 생산자·단일 소비자 큐 뼈대 작성(먼저 mutex 버전으로 정확성 확보)
- **확인 기준**: 100만 건 왕복에 유실·중복 0

#### L-CPP-07 Google Benchmark 설정 (60분) 🔴

- **단계**: vcpkg `benchmark` 추가 → `BENCHMARK(BM_Foo)->Arg(1)->Arg(4)->Arg(16);` → `benchmark::DoNotOptimize()`로 최적화 제거 방지 → Release 빌드로 실행
- **확인 기준**: 결과에 반복 횟수·시간·표준편차가 나온다

#### L-CPP-08 lock-free SPSC 완성 (120분) 🔴

- **단계**: L-CPP-06의 인덱스를 `std::atomic<size_t>`로 바꾸고 생산자는 `store(release)`, 소비자는 `load(acquire)` 사용. 각 메모리 오더 선택 근거를 주석으로 남긴다
- **확인 기준**: mutex 버전과 결과 동일(유실·중복 0), 처리량 비교 표, ASan 통과
- **노트에 남길 것**: "왜 seq_cst가 아니어도 되는가"를 happens-before로 설명

#### L-CPP-09 false sharing 측정 (60분)

- **단계**: 스레드 4개가 각각 증가시키는 카운터를 (a) 인접 배열 (b) `alignas(64)` 정렬 배열로 두고 비교
- **확인 기준**: 두 버전의 처리량 차이가 표에 있고, 캐시 라인으로 설명한다

#### L-CPP-10 UB 실습 (60분) 🟡

- **단계**: 부호 있는 정수 오버플로 / 초기화 안 된 변수 읽기 / 댕글링 참조를 각각 재현하고 `-O0`과 `-O2`(`/Od`, `/O2`)에서 동작 차이 관찰
- **확인 기준**: "최적화가 바꾼 동작" 사례 1개 이상 기록

#### L-CPP-11 ASan으로 use-after-free 잡기 (45분) 🔴

- **단계**: VS 프로젝트 속성 → C/C++ → 일반 → "AddressSanitizer 사용" 활성(`/RTC` 끄기) → 해제된 메모리 접근 코드 실행 → 리포트 확인
- **확인 기준**: ASan 리포트에서 할당·해제·접근 위치 3개를 짚는다

#### L-CPP-12 이동 의미론 복사 카운터 (45분)

- **단계**: 복사 생성자/이동 생성자에 카운터를 심은 `Buffer` 클래스 → `vector`에 push_back, `std::move` 유무 비교, `reserve` 유무 비교
- **확인 기준**: 복사·이동 횟수 표 + "언제 복사가 사라지는가(복사 생략)" 설명

---

## 4. 학습 항목 상세

각 항목은 **무엇을 / 왜 / 어떻게(순서) / 확인** 으로 구성된다. "확인"이 곧 학습 완료 판단 근거다. 괄호 안 실습 번호는 §3을 가리킨다.

### 4.1 공통 (40h)

**Git 워크플로 (4h)**
- 무엇: 브랜치 전략(main + feature), 셀프 리뷰, 리베이스, 충돌 해결, 커밋 메시지 규칙(제목 50자, 본문에 "왜")
- 왜: 모든 과제가 저장소 단위로 평가되고, 취업 후 첫날부터 쓰는 도구다
- 어떻게: ① 기능마다 `feat/<이름>` 브랜치 ② 커밋은 "동작하는 최소 단위" ③ 병합 전 `git rebase main`으로 정리 ④ 충돌은 손으로 해결하고 이유를 커밋 본문에
- 확인: 충돌 2건을 일부러 만들고 각각 병합/리베이스로 해결한 이력이 `git log --graph`에 보인다 (L-C-01)

**단위 테스트 (6h)**
- 무엇: AAA, 테스트 이름 규칙, 페이크, 의존성 주입, 파라미터화 테스트
- 왜: 서버는 GUI가 없어 테스트가 곧 실행 방법이며, 이후 평가 기준이 "제공 테스트 통과"다
- 어떻게: ① 실패하는 테스트 먼저 ② 최소 구현 ③ 리팩터 ④ 경계값(빈 입력·최대값·중복)을 반드시 케이스로
- 확인: 1-1 로그 큐의 순서 보장·flush·용량 정책이 각각 독립된 테스트로 증명된다 (L-C-04)

**디버깅 (6h)**
- 무엇: 조건부/적중/데이터 중단점, 병렬 스택·스레드 창, 메모리 창, 덤프 저장과 분석
- 왜: 서버 버그 다수가 멀티스레드이고 재현이 어렵다. 로그만으로는 못 잡는다
- 어떻게: ① 재현 조건을 스크립트로 고정 ② 조건부 중단점으로 문제 반복 구간만 멈춤 ③ 병렬 스택으로 스레드 간 관계 파악 ④ 필요하면 덤프를 떠서 사후 분석
- 확인: 데드락 병렬 스택 스크린샷 + 덤프 1건 분석 기록 (L-C-11, L-C-13)

**OS 기초 (6h)**
- 무엇: 프로세스/스레드/핸들, 컨텍스트 스위칭, 사용자·커널 모드, 가상 메모리, 캐시 라인(64B), false sharing
- 왜: "스레드를 늘리면 왜 느려지는가", "왜 구조체 배열이 빠른가"의 근거
- 어떻게: 개념 → L-C-07(스레드 수 실험) → L-CPP-09/L-CS-03(캐시 경합) 순서로 **수치를 먼저 보고 이론을 붙인다**
- 확인: 스레드 수 1→1,000 처리 시간 표와 해석 3줄 (L-C-07)

**동시성 개념 (10h)**
- 무엇: 경쟁 조건, 임계 구역, 데드락 4조건, 락 종류, 원자 연산, 가시성·재배치, lock-free의 의미와 한계
- 왜: 게임 서버 버그 중 가장 비싸고 재현이 어렵다
- 어떻게: 세 가지 병(경쟁·데드락·가시성)을 **각각 최소 코드로 재현 → 2가지 이상 수정 → 처리량 비교** 순서
- 확인: L-C-08, L-C-09, L-C-10 결과물 3세트

**Windows 개발 환경 (4h)**
- 무엇: VS 2026 워크로드, .NET SDK 버전 고정(`global.json`), vcpkg 매니페스트, PowerShell 7 스크립트, SQLite 확인(설치 불필요), (선택) MySQL 8, redis-windows
- 왜: Docker를 쓰지 않으므로 로컬 설치·프로세스 관리 자체가 실무 능력이다. DB는 개발 편의를 위해 SQLite가 기본, MySQL은 선택이다
- 어떻게: 설치 → 버전 확인 명령 → 서비스 시작/중지를 PowerShell로 → 확인 결과를 노트에 표로
- 확인: `sqlite3 --version` 응답, `redis-cli ping` → `PONG`, (선택) `mysql -u root -p` 접속. PowerShell로 redis 프로세스 시작/중지 가능

**벤치마크 방법론 (4h)**
- 무엇: 워밍업, 반복, 평균/중앙값/p99, 분산 표기, Debug/Release, 최적화 제거 방지, 조건 통제
- 왜: Phase 5에서 "수치로 개선 증명"을 하려면 측정 자체가 신뢰 가능해야 한다
- 어떻게: ① 항상 Release ② 워밍업 3회 이상 ③ 10회 이상 반복해 중앙값·표준편차 ④ 결과 소비(`DoNotOptimize`) ⑤ 환경(CPU·전원 옵션·백그라운드)을 리포트에 기록
- 확인: Debug/Release 비교 표와 "Debug 측정이 무의미한 이유" 한 문단 (L-C-14)

### 4.2 C# 트랙 (80h)

| 항목 | 시간 | 무엇을 | 학습 순서 | 확인 방법 |
|---|---|---|---|---|
| .NET 런타임과 GC | 10h | JIT, 세대별 GC, LOH, 워크스테이션/서버 GC, 할당 비용, `GC.GetTotalAllocatedBytes`, `dotnet-counters` | 개념 → L-CS-01 → L-CS-02 → 1-2 측정에 적용 | LOH 경계 실험 결과와 `dotnet-counters` 캡처 |
| 값/참조 타입, 구조체 | 6h | struct vs class, `readonly struct`, `ref struct`, 박싱 비용, `in`/`ref` 매개변수 | 개념 → 박싱 발생/회피 두 버전 벤치 | 박싱 유무 할당량 비교 표 |
| Span/Memory/ArrayPool | 8h | `Span<T>`, `Memory<T>`, `stackalloc`, `ArrayPool<T>.Shared`, 파싱 시 할당 제거 | L-CS-09 → 1-2에서 풀과 비교 | Substring vs Span 할당 비교 |
| async/await 내부 | 14h | 상태 머신, SynchronizationContext, `ConfigureAwait`, Task vs ValueTask, `async void` 금지, 예외 전파, 취소 | 교재 1~4장 → L-CS-05 → L-CS-11 → L-CS-12 | 디컴파일한 상태 머신 흐름 설명 |
| ThreadPool | 6h | 워크 아이템, 기아, `.Result` 동기 대기 위험, `Task.Run` 남용 | 개념 → L-CS-10 | 기아 재현 + 카운터 캡처 + 수정 |
| 동기화 도구 | 12h | `lock`, `Interlocked`, `SemaphoreSlim`, `ReaderWriterLockSlim`, `ConcurrentQueue/Dictionary`, `Channel<T>` | L-CS-03 → L-CS-04 → L-CS-06 | 과제 1-1 두 버전 |
| BenchmarkDotNet | 6h | 벤치마크 작성, `MemoryDiagnoser`, `Baseline`, `Params`, 결과 해석 | L-CS-07 → 1-1·1-2 측정 | 리포트 2건 |
| 프로젝트 구성 | 6h | .NET CLI, 솔루션 구조, NuGet, 분석기, `TreatWarningsAsErrors`, nullable | L-C-03 → 1-C | 경고 0으로 빌드되는 솔루션 |
| 비동기 함정 | 12h | 데드락, 잃어버린 예외, 취소 무시, 파이어앤포겟, 컨텍스트 캡처 | 교재 6장 → L-CS-08 | 12개 재현·수정 프로젝트 |

### 4.3 C++ 트랙 (80h)

| 항목 | 시간 | 무엇을 | 학습 순서 | 확인 방법 |
|---|---|---|---|---|
| 소유권과 RAII | 10h | 객체 수명, 스택/힙, `unique_ptr/shared_ptr/weak_ptr`, 순환 참조, 커스텀 deleter | 교재 4·5장 → L-CPP-01 → L-CPP-02 | 순환 참조 누수 재현·수정 |
| 이동 의미론 | 8h | 값 카테고리, 이동 생성자/대입, `std::move`/`forward`, 복사 생략, Rule of 0/5 | 개념 → L-CPP-12 | 복사·이동 횟수 카운터 표 |
| 표준 동시성 | 14h | `thread/jthread`, `mutex/lock_guard/scoped_lock`, `condition_variable`, `atomic`, 메모리 오더 | L-CPP-03 → L-CPP-04 → L-CPP-06 → L-CPP-08 | 과제 1-1 두 버전 |
| Win32 동기화 | 8h | `SRWLOCK`, `CONDITION_VARIABLE`, `InitOnce`, `WaitOnAddress`, 스레드 풀 API 개요 | 교재 3~5·8장 → L-CPP-05 | mutex vs SRWLock 비교 표 |
| 빌드 시스템 | 8h | CMake 타깃/프로퍼티, vcpkg 매니페스트, Debug/Release, `/W4 /permissive-`, `/analyze` | L-C-03 → L-C-05 | 로컬 스크립트로 빌드·테스트 통과 |
| 새니타이저·테스트 | 8h | ASan(MSVC), Application Verifier, GoogleTest, Google Benchmark | L-CPP-11 → L-CPP-07 | ASan 리포트 캡처 |
| 템플릿·유틸리티 | 8h | 템플릿 기초, `std::function`/람다, `optional/variant/span/expected` | 개념 → 1-2를 템플릿으로 | 템플릿 풀 구현 |
| 캐시와 UB | 8h | false sharing, `alignas`, UB 목록, 최적화 영향 | L-CPP-09 → L-CPP-10 | false sharing 비교 벤치 |
| 언어 재정비(선택) | 8h | Core Guidelines 핵심, `constexpr`, 강타입, 예외 안전성 3단계 | 교재 3·7·10장 | 체크리스트 자기 채점 |

---

## 5. 교재 활용 가이드 (Phase 1)

교재 원문은 `programming-books-with-ai` 저장소에 있다. 지정된 장만, 지정된 시점에 읽는다. 읽는 방식은 총론 §2.3과 `07-books-guide.md` §2를 따른다.

### 5.1 공통

| 교재 | 읽을 장 | 언제 | 어떻게 활용 |
|---|---|---|---|
| OpenCode로 시작하는 AI 코딩 에이전트 | 가이드 1 Part 2(세션·Build/Plan 모드·AGENTS.md), Part 5(자동화) | 3일차 오전 | 이 과정은 Claude Code 또는 Codex CLI 등 코딩 에이전트를 자유롭게 쓰지만 "지침 파일", "Plan 후 Build" 개념은 동일하다. 읽은 뒤 자기 지침 파일에 반영할 항목 5개를 뽑아 L-C-06에 적용 |
| 게임 개발자를 위한 C# 디자인 패턴 | 3장 Object Pool | 5일차 오전 | 과제 1-2 설계 전에 읽는다. C++ 트랙도 개념 파악용으로 읽고 C++로 옮긴다. "풀이 오히려 손해인 경우"를 찾아 노트에 적는다 |

### 5.2 C# 트랙

| 교재 | 읽을 장 | 언제 | 어떻게 활용 |
|---|---|---|---|
| ASP.NET Core Web API를 위한 필수 C# 가이드 | 전체(짧음) | 1주차 초 속독 | 문법 구멍 점검용. 모르는 절만 표시해 예제를 직접 타이핑 |
| C# Async/Await 완전 정복 | 1~4장(기초·SynchronizationContext·ConfigureAwait·Awaitable), 6장(함정 12가지), 7장(동기화 객체) | 9·14일차 | 각 장 끝 체크리스트를 AI 면접관에게 주고 구두 시험. samples(.NET 10)를 그대로 빌드. 6장은 L-CS-08의 교재 |
| C# async/await 라이브러리 만들기 | 1~6장(Task/ValueTask 내부, 상태 머신 해부, Awaitable 설계, 커스텀 Awaitable, DelayAsync, 취소) | 3주차 | 2장을 읽은 뒤 L-CS-05 디컴파일 결과와 대조. 각 장 연습 문제를 과제로. 16~20장은 Phase 3·5 |

### 5.3 C++ 트랙

| 교재 | 읽을 장 | 언제 | 어떻게 활용 |
|---|---|---|---|
| Modern C++로 안전하고 우아한 프로그래밍 | 2장(개발 환경), 3장(Core Guidelines), 4장(RAII), 5장(스마트 포인터), 6장(컨테이너), 7장(강타입), 10장(예외 안전성), 12·13장(동시성) | 1~2주차 | VS 2022 기준으로 쓰였으나 VS 2026에서 그대로 빌드된다. 코드 폴더가 없으므로 본문 코드를 직접 프로젝트로 옮기는 것이 과제. 18장(벤치마킹)은 3주차 |
| 모던 Windows 멀티스레딩 | 2장(역사), 3장(SRW Lock), 4장(Condition Variable), 5장(One-Time Init), 8장(WaitOnAddress·Lock-Free) | 9일차 전후 | 표준 라이브러리와 Win32 API를 1:1 대응표로 정리하며 읽는다. L-CPP-05의 교재. 6장은 Phase 3, 10장은 Phase 5 |
| 게임 서버 개발자를 위한 최신 Win32 API 프로그래밍 | 1장(Win32 기초), 2장(메모리 관리·메모리 풀), 4장(프로세스·스레드), 5장(동기화 심화), 6장(인터락·무잠금) | 2~3주차 | 2장 메모리 풀은 과제 1-2의 직접 참고. 7장은 Phase 5 |
| C++23 메모리 모델 완벽 가이드 | 1부(기초 이론), 2부(메모리 순서 상세) | 14일차 직전 | L-CPP-08(lock-free SPSC) 직전에 읽는다. 3부는 Phase 5 |
| 모던 C++로 시작하는 안전하고 쉬운 C++ 프로그래밍 | 1~17장 중 약한 부분만 | 1주차(필요 시) | 문법이 불안한 학습자만. 18장 이후(Siv3D)는 무관 |

### 5.4 부 트랙 읽기 과제 (방식 A 병행자, 주당 4h 이내)

- C# 주 트랙: "Modern C++로 안전하고 우아한 프로그래밍" 4·5장 → **"C# GC vs C++ RAII" 비교표**(수명 결정 시점, 비용 위치, 누수 유형, 디버깅 도구 4행)
- C++ 주 트랙: "C# Async/Await 완전 정복" 1장 → **"C++ 스레드 vs C# Task" 비교표**(스케줄링 주체, 블로킹 비용, 취소 방법, 예외 전파 4행)

---

## 6. AI 협업 가이드 (Phase 1 특화)

### 6.1 그대로 써도 되는 프롬프트

**개념 확인 (튜터 모드)**
```
나는 [C# GC 세대 / C++ 이동 의미론]을 이렇게 이해했다:
"(내 설명 5~10문장)"
1) 틀린 부분과 빠진 부분을 지적해 달라.
2) 내 설명이 맞는지 직접 확인할 수 있는 10줄 이내 실험 코드를 제안해 달라.
3) 공식 문서에서 확인할 키워드를 알려 달라.
결론을 먼저 말하지 말고 내 설명의 오류부터 짚어라.
```

**결함 주입 (출제자 모드)**
```
다음 요구사항의 [스레드 세이프 로그 큐] 코드를 작성하되, 데드락 1개·경쟁 조건 1개·리소스 누수 1개를 의도적으로 숨겨 넣어라.
정답 위치는 내가 "정답 공개"라고 말하기 전까지 절대 알려주지 말라.
힌트를 요청하면 결함의 "종류"만 알려주고 위치는 말하지 말라.
결함은 컴파일되고 대부분의 실행에서는 정상 동작해야 한다.
```

**벤치마크 해석 (리뷰어 모드)**
```
BenchmarkDotNet(또는 Google Benchmark) 결과다: (표 붙여넣기)
빌드 구성: Release / 워밍업 3회 / 반복 10회 / CPU: (모델)
내 해석: "(2~3문장)"
내 해석에 동의하는지, 놓친 요인(워밍업·GC·캐시·측정 오차·전원 옵션)이 있는지 말해 달라.
```

**지침 파일 검토**
```
다음은 내 학습용 프로젝트의 에이전트 지침 파일(Claude Code는 CLAUDE.md, Codex CLI는 AGENTS.md) 초안이다. (붙여넣기)
학습 목적(코드를 내가 이해하며 진행)에 어긋나는 항목, 빠진 규칙, 너무 모호해서 지키기 어려운 문장을 지적해 달라.
각 지적마다 "이렇게 바꾸면 된다"는 구체적 문장을 제안해 달라.
```

**단계별 구현 요청 (페어 모드)**
```
[로그 큐]를 만들려 한다. 전체 코드를 한 번에 주지 말고 다음 단계로 나눠 진행하자.
단계 1: 공개 API 시그니처와 상태 필드만 (구현부는 TODO 주석)
내가 "다음"이라고 말하기 전에는 단계 2로 넘어가지 말라.
각 단계마다 내가 먼저 설계 의도를 설명하면 틀린 부분을 지적해 달라.
```

### 6.2 맡길 것 / 맡기지 않을 것

| 맡긴다 | 맡기지 않는다 |
|---|---|
| 로컬 빌드·테스트 스크립트 초안, `.editorconfig`, 프로젝트 골격 | 로그 큐·오브젝트 풀의 핵심 알고리즘 첫 구현 |
| 테스트 케이스 목록 뽑기(빠진 케이스는 내가 추가) | 벤치마크 결과 해석의 최종 결론 |
| 벤치마크 프로젝트 보일러플레이트 | 학습 노트·주간 회고 |
| 결함 주입 코드 생성 | 결함 찾기 |
| CMake/솔루션 설정 오류 해결 | 메모리 오더 선택 근거(직접 쓰고 검증만 받는다) |

### 6.3 AI가 자주 틀리는 지점 (검증 포인트)

- **.NET 버전별 API 차이**: 예) `ConfigureAwaitOptions`는 .NET 8+. 공식 문서 "적용 대상" 표로 확인한다. 이 과정은 .NET 10 기준
- **C++ 메모리 오더**: x86의 강한 순서 때문에 "relaxed도 잘 동작한다"는 예제가 자주 나온다. "ARM에서도 안전한가?"를 반드시 되묻는다
- **Debug 빌드 벤치마크**: 결과를 붙여 넣으면 AI가 구성까지는 못 본다. 항상 빌드 구성을 명시한다
- **`Channel<T>` Unbounded 기본 제안**: 백프레셔가 없다. 용량과 초과 정책을 명시하라고 요구한다
- **`lock` 안에서 `await`**: C#에서 불가능한데도 제안하는 경우가 있다. `SemaphoreSlim`으로 유도한다
- **C++ `condition_variable` 술어 없는 wait**: spurious wakeup을 놓친 코드. 항상 `wait(lock, pred)` 형태를 요구한다

---

## 7. 과제 명세

이 Phase의 제출물은 **1-C(공통), 1-1(트랙), 1-2(트랙)** 이며 1-3은 선택이다. 각 과제는 §2의 일자 계획에 배치되어 있다.

### 7.1 공통 과제 1-C. 학습 환경·저장소 구축 (필수, 8h, 1~3일차)

**목표**: 이후 26주 동안 쓸 저장소 구조와 검증 자동화를 만든다.

**산출물 트리**

```
gameserver-course-<이름>/
├─ README.md                  ① 과정 소개 ② 트랙 선택과 이유 3문장 ③ 진행 표 ④ 실행 방법
├─ CLAUDE.md 또는 AGENTS.md   에이전트 지침 파일
├─ .gitignore                 bin/ obj/ build/ .vs/ BenchmarkDotNet.Artifacts/ *.dmp
├─ .editorconfig              들여쓰기·인코딩·개행·분석기 규칙 3개 이상
├─ scripts/build-and-test.ps1 빌드+테스트 한 번에(§3 L-C-05 전문 사용)
├─ notes/daily/.gitkeep
├─ notes/weekly/.gitkeep
└─ phase1/                    Phase1 솔루션 또는 CMake 루트
```

**요구사항 상세**

1. **저장소**: GitHub에 `gameserver-course-<이름>` 생성. 위 트리대로 구성하고 빈 폴더는 `.gitkeep` 유지
2. **로컬 검증 자동화**(CI 대신): `scripts/build-and-test.ps1` 한 줄로 C#은 `dotnet build` + `dotnet test`, C++은 CMake configure/build + `ctest` 실행. 실패 시 `exit 1`
3. **커밋 전 실행 습관**: 최소 1회는 **일부러 테스트를 깨뜨려** 스크립트가 빨간불로 끝나는 것을 확인하고 되돌린 이력을 남긴다(커밋 2개)
4. **README**: 트랙 선택 이유 3문장(목표 회사군 스택 / 본인 선호 / 기존 숙련도 근거 각 1문장), Phase별 진행 표(체크박스), 실행 방법(`pwsh scripts/build-and-test.ps1`)
5. **지침 파일**: 프로젝트 구조 / 언어·버전(.NET 10 또는 C++23, VS 2026) / 코딩 규칙 5개 / "코드 생성 시 설계 의도 주석" / "한 번에 파일 하나" / "테스트 동반" / "라이브러리 추가 시 이유와 대안 1개"
6. **트랙 확정**: 3일차까지 주 트랙을 확정하고 README에 기록(이후 변경하려면 회고에 이유를 남긴다)

**제출물**: 저장소 URL / `build-and-test.ps1` 실행 로그(성공·실패 각 1회) / 지침 파일

**채점(자기 채점)**

| 항목 | 배점 | 기준 |
|---|---|---|
| 구조 | 25 | 위 트리 준수, 빈 폴더 유지, `.gitignore`로 산출물 제외 |
| 테스트 자동화 스크립트 | 35 | 1회 실행으로 빌드+테스트 통과, 실패 재현·복구 이력 1회, 실패 시 종료 코드 1 |
| 지침 파일 | 30 | 필수 항목 포함, 문장이 구체적(모호한 "잘 짜라" 금지), 위반 사례를 반영해 1회 이상 개정 |
| README | 10 | 트랙 선택 이유 3문장, 진행 표, 실행 방법 |

**자주 하는 실수**: 스크립트가 실패해도 종료 코드 0을 반환(→ `$LASTEXITCODE` 확인 누락) / 지침 파일이 일반론만 담아 실제 생성 결과가 달라지지 않음

### 7.2 트랙 과제 1-1. 스레드 세이프 로그 큐 (필수, 24h, 8~12일차)

**배경**: 게임 서버는 초당 수천 줄의 로그를 남긴다. 로그 쓰기가 게임 로직 스레드를 막으면 지연이 튄다. 그래서 "생산자는 큐에 넣기만 하고, 전용 소비자 스레드가 파일에 쓰는" 구조를 쓴다. 이 과제는 그 구조를 **두 가지 동기화 방식으로** 만들고 수치로 비교하는 것이다.

**산출물 트리**

```
phase1/LogQueue/
├─ src/    LogQueue 구현 (A 버전, B 버전, 공통 인터페이스)
├─ tests/  단위 테스트 15개 이상
├─ bench/  벤치마크 프로젝트
└─ REPORT-1-1.md
```

**기능 요구사항**

1. **공개 API** (C#)
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
       bool Enqueue(in LogEntry entry);   // Drop 정책이면 false 가능, Block이면 항상 true
       Task StopAsync();                  // 잔여 항목 전부 기록 후 종료
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
       virtual void Stop() = 0;            // 잔여 항목 전부 기록 후 스레드 조인
       virtual LogQueueStats Stats() const = 0;
   };
   ```
2. **동작 정의**
   - `Start()` 이전의 `Enqueue`는 예외(C#) 또는 `false` 반환(C++) — **택일해서 문서화**하고 테스트로 고정
   - `StopAsync()/Stop()`은 ① 신규 입력 차단 ② 잔여 항목 전부 파일 기록 ③ 파일 flush·close ④ 소비자 스레드 조인 순서
   - `Stop()` 이후 `Enqueue`는 항상 실패(예외 없이 false 권장)
   - 파일 한 줄 형식: `{seq}\t{producerId}\t{ISO8601 UTC}\t{message}`
   - 메시지의 `\t`, `\r`, `\n`은 역슬래시 이스케이프하며 파일은 UTF-8(LF)로 기록한다
   - `StopAsync()/Stop()`과 dispose는 멱등이다. Block 대기 중 Stop이 시작되면 대기 생산자를 깨워 `false`를 반환한다
   - `ILogSink`/동등한 C++ 콜백을 주입해 파일 쓰기 실패를 재현하고, 싱크 예외는 소비자 루프를 죽이지 않고 오류 통계에 반영한다
3. **정책**
   - `Drop`: 큐가 가득 차면 즉시 버리고 `Dropped++`, `Enqueue`는 false
   - `Block`: 자리가 날 때까지 생산자 대기(최대 대기 시간 옵션은 선택)
4. **불변식**(테스트로 증명해야 하는 것)
   - `Written == Enqueued - Dropped`
   - 같은 `ProducerId`의 항목은 파일에서 넣은 순서가 유지된다
   - 종료 후 파일 라인 수 == `Written`

**트랙별 구현 요구**

- **C#**: (A) `lock` + `Queue<T>` + `Monitor.Wait/Pulse` 버전, (B) `Channel<T>`(Bounded, `FullMode`로 정책 표현) 버전. 두 버전이 **같은 인터페이스**를 구현해야 한다
- **C++**: (A) `std::mutex` + `std::condition_variable` + `std::deque` 버전, (B) lock-free SPSC 링 버퍼 버전(생산자 1개 조건. 다중 생산자 테스트는 A만 수행). (B)는 `acquire/release` 선택 근거를 주석으로 남기고 ASan을 통과해야 한다

**테스트 목록(15개, 전부 필수)**

| # | 이름 | 시나리오 | 기대 |
|---|---|---|---|
| 1 | Enqueue_시작전_정의된동작 | Start 전 Enqueue | 문서화한 동작(예외 또는 false) |
| 2 | Enqueue_단일생산자_전부기록 | 1스레드 1,000건 | 파일 1,000줄 |
| 3 | Enqueue_다중생산자_유실없음 | 4스레드×10,000건, Block 정책 | 파일 40,000줄 |
| 4 | 순서보장_생산자별 | 4스레드가 각 1..10,000 | 생산자별 시퀀스 오름차순 |
| 5 | Drop정책_초과분버림 | 용량 100, 10,000건 폭주 | `Written == Enqueued - Dropped`, Dropped > 0 |
| 6 | Block정책_유실0 | 용량 100, 10,000건 | Dropped == 0 |
| 7 | Stop_잔여항목flush | 큐에 남긴 채 Stop | 라인 수 == Written |
| 8 | Stop_이후Enqueue_실패 | Stop 후 Enqueue | false(또는 정의한 예외) |
| 9 | Stop_두번호출_안전 | Stop 두 번 | 예외 없음, 파일 정상 |
| 10 | Stop_생산자대기중_교착없음 | Block 정책 가득 상태에서 Stop | 5초 내 종료 |
| 11 | 통계_정합성 | 임의 시나리오 | Enqueued/Dropped/Written 합이 맞음 |
| 12 | 빈큐_Stop_즉시종료 | 아무것도 안 넣고 Stop | 1초 내 종료, 빈 파일 |
| 13 | 큰메시지_처리 | 64KB 메시지 10건 | 손상 없이 기록 |
| 14 | 파일경로_없는폴더 | 존재하지 않는 폴더 | 명확한 오류(생성 또는 예외) |
| 15 | 장시간_메모리안정 | 30초간 초당 5,000건 | 큐 길이 상한 유지, 메모리 증가 추세 없음 |

**벤치마크 절차(반드시 이 순서)**

1. Release 빌드 확인, 전원 옵션 "고성능", 다른 앱 종료
2. 파라미터: 생산자 수 1 / 4 / 16, 메시지 100바이트, 각 조건 10회 반복(워밍업 3회)
3. 측정 지표: 처리량(msg/s), 항목당 할당 바이트(C# `MemoryDiagnoser`), `Enqueue` 지연 p50/p99
4. **파일 I/O를 뺀 큐 자체 처리량**도 따로 측정(소비자가 그냥 버리는 모드) — 병목이 큐인지 디스크인지 분리
5. p50/p99는 `Stopwatch.GetTimestamp()`/`steady_clock`으로 항목별 샘플을 수집해 정렬 계산한다. BenchmarkDotNet/Google Benchmark 요약만으로 대체하지 않는다
6. `bench/run.ps1`이 Release 빌드·실행·원시 결과 저장을 강제한다

**REPORT-1-1.md 템플릿**

```markdown
# 1-1 로그 큐 리포트
## 1. 환경
CPU / RAM / OS / 빌드 구성 / 런타임 버전 / 커밋 해시
## 2. 측정 방법
반복·워밍업·파라미터·측정 도구·제외한 요인
## 3. 결과 표
| 버전 | 생산자 | 처리량(msg/s) | p50(µs) | p99(µs) | 항목당 할당(B) | 표준편차 |
## 4. 왜 차이가 나는가 (3문단)
- 문단1: 동기화 비용(락 경합 vs 무락 구조)
- 문단2: 할당과 메모리 지역성
- 문단3: 정책(Drop/Block)이 처리량에 준 영향
## 5. 결론: 어느 상황에 어느 버전을 쓸 것인가
## 6. AI가 틀렸던 것 (최소 1건)
무엇을 잘못 말했고 어떻게 확인했는가
```

**진행 순서(권장)**

| 일자 | 할 일 |
|---|---|
| 8일차 | 인터페이스·옵션·통계 타입 확정, (A) 버전 뼈대 + 테스트 1~4 |
| 9일차 | (B) 버전 뼈대, 테스트 5~9 |
| 10일차 | 테스트 10~15, 종료 경로 정리, 두 버전 동일 인터페이스 확인 |
| 12일차 | 벤치마크 실행·표 작성·리포트 |

**채점**

| 항목 | 배점 | 기준 |
|---|---|---|
| 정확성 | 30 | 테스트 15개 전부 통과, 불변식 3개 성립 |
| 동시성 | 25 | 루브릭 2번 AI 리뷰 4점 이상, 1,000만 건 왕복 유실·중복 0을 10회 반복, 종료 경로에 교착 없음 |
| 측정 | 25 | Release·워밍업·반복·분산 표기, 큐 단독 측정 분리 |
| 리포트 | 20 | 수치 근거 결론, "AI가 틀린 것" 1건 이상 |

**자주 하는 실수**

- 소비자 루프가 `Complete()`/`Stop()` 신호를 못 받아 종료되지 않음 → 종료 신호와 큐 비움 조건을 함께 검사
- `Block` 정책에서 Stop 시 생산자가 영원히 대기 → Stop이 대기자를 깨우도록 설계
- 순서 보장을 "전역 순서"로 착각 → **생산자별** 순서만 보장하면 된다(전역 순서를 원하면 시퀀스 발급이 필요하고 비용이 는다)
- 파일 flush를 매 항목마다 수행해 처리량이 급락 → 주기적 flush 또는 버퍼링

### 7.3 트랙 과제 1-2. 오브젝트 풀 + 할당 측정 (필수, 16h, 5·13일차)

**배경**: Phase 2에서 패킷 수신 버퍼를 초당 수만 번 빌린다. 매번 새로 할당하면 GC(또는 malloc) 부담이 커진다. 여기서 만든 풀을 **Phase 2~6 내내 재사용**한다.

**산출물 트리**

```
phase1/ObjectPool/
├─ src/    BufferPool 구현(별도 라이브러리 프로젝트로!)
├─ tests/  테스트 8개
├─ bench/  3자 비교 벤치마크
└─ REPORT-1-2.md
```

**기능 요구사항**

1. **공개 API** (C#)
   ```csharp
   public sealed class BufferPool : IDisposable
   {
       public BufferPool(int bufferSize = 4096, int maxRetained = 1024);
       public byte[] Rent();                 // 없으면 새로 생성
       public void  Return(byte[] buffer);   // maxRetained 초과분은 폐기
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
       Handle Rent();                       // Handle 소멸 시 같은 풀로 자동 반환
       PoolStats Stats() const;
   };
   ```
2. **스레드 세이프**: 락 기반으로 시작. 여유가 있으면 스레드 로컬 캐시 + 전역 풀 2단계(🟡)
3. **잘못된 사용 방어**
   - C#은 같은 객체 2회 `Return` 감지 → Debug 빌드에서 assert 또는 예외. C++은 이동 전용 `Handle`만 노출해 명시적 이중 반환을 타입으로 차단
   - 풀 크기를 넘는 반환은 폐기하고 `Discarded++`
   - 소멸 시 미반환 개수를 경고 로그로(C++는 소멸자, C#은 `Dispose`)
4. **통계**: 대여/반환/생성/폐기/현재 보유 수

**테스트 목록(8개)**

| # | 이름 | 기대 |
|---|---|---|
| 1 | Rent_빈풀_새버퍼생성 | Created == 1 |
| 2 | Return후Rent_재사용 | Created 증가 없음 |
| 3 | 최대보유초과_폐기 | Discarded > 0, CurrentRetained ≤ maxRetained |
| 4 | 이중반환_감지 | C#은 Debug 예외/assert, C++은 복사·명시 반환 API가 컴파일되지 않음 |
| 5 | 멀티스레드_대여반환_정합성 | 8스레드×10,000회 후 통계 일치 |
| 6 | 반환된버퍼_내용초기화정책 | 문서화한 정책대로(초기화 여부 고정) |
| 7 | 소멸시_미반환경고 | 경고 로그 1건 |
| 8 | 잘못된크기버퍼_반환거부 | 거부 또는 예외 |

**측정 요구**

- **C#**: `new byte[4096]` 매번 / 자작 풀 / `ArrayPool<byte>.Shared` **3자 비교**를 `MemoryDiagnoser`로. 추가로 `dotnet-counters monitor --counters System.Runtime`로 Gen0 수집 횟수 관찰 캡처
- **C++**: `new/delete` vs 풀 비교(Google Benchmark). ASan은 UAF·범위 오류 검출에 사용하고, 누수 판정은 CRT 디버그 힙/VS 메모리 스냅숏과 `Rented == Returned`, `Created == destroyed+retained` 카운터로 증명
- 표에는 처리량·항목당 할당·GC 횟수(또는 할당 호출 수)를 함께 적는다

**REPORT-1-2.md 필수 내용**: 3자(또는 2자) 비교표 / 풀이 이득이 아닌 조건 1개 이상(예: 버퍼 수명이 매우 짧고 크기가 작을 때) / Phase 2에서 이 풀을 어떻게 쓸지 2문장

**채점**: 정확성 40(테스트 8개) / 측정 30(3자 비교·GC 관찰) / 코드 품질 30(루브릭 3·7번, 라이브러리로 분리되어 재사용 가능)

**중요**: 이 프로젝트는 **반드시 독립 라이브러리**로 만든다. Phase 2 패킷 라이브러리가 프로젝트 참조로 가져다 쓴다.

### 7.4 트랙 과제 1-3. 파일 대량 처리 CLI (심화 🟡, 12h)

**목표**: 순차 / 비동기 I/O / 병렬의 차이를 체감하고, "I/O 바운드에 스레드를 더 넣어도 안 빨라지는" 경험을 한다.

**요구사항**

1. 입력 생성 스크립트 직접 작성: 텍스트 파일 5,000개(각 10~100KB, 무작위 단어)
2. 단어 수 집계 CLI를 **세 버전**으로
   - `--mode seq`: 순차
   - `--mode async`: C# `File.ReadAllTextAsync` + `SemaphoreSlim`으로 동시성 제한 / C++ 스레드 풀 또는 Overlapped `ReadFile`
   - `--mode parallel`: C# `Parallel.ForEachAsync` / C++ `std::thread` N개 + 작업 분배
3. 공통 옵션: `--dir`, `--concurrency N`, `--repeat R`
4. 측정: 처리 시간, CPU 사용률, 최대 작업 집합(`Get-Process`), 동시성 N=1/4/8/32에서의 변화
5. 결론 1문단: 어떤 조건(SSD/HDD, 파일 크기, 코어 수)에서 어느 버전이 유리한가. **"동시성을 올려도 더 안 빨라지는 지점"을 수치로 제시**

**제출물**: 코드 / 생성 스크립트 / 비교표(3버전 × 4개 동시성) / 결론

### 7.5 부 트랙 읽기 과제 (방식 A 병행자)

- 상대 언어의 로그 큐 구현(교재 코드 또는 자기 트랙 코드를 AI에게 번역시킨 것)을 읽고 **차이 5개**를 표로: 소유권/수명 처리, 종료 처리, 예외 vs 오류 코드, 동기화 도구, 할당 위치
- 30분 이내로 끝낸다. 이 과제는 "다름을 인지"하는 것이 목적이지 습득이 목적이 아니다

---

## 8. 학습 완료 판정 (15일차 금요일)

### 8.1 체크리스트

**과제**
- [ ] 1-C 완료, `pwsh scripts/build-and-test.ps1` → `ALL GREEN`
- [ ] 1-1 테스트 15개 통과, `REPORT-1-1.md` 완성(표·분산·결론·AI 오류 1건)
- [ ] 1-2 테스트 8개 통과, 3자(또는 2자) 비교표, 독립 라이브러리로 분리
- [ ] 🟡 1-3(선택) 또는 미수행 사유를 회고에 기록

**실습 결과물**
- [ ] 경쟁 조건·데드락·가시성 재현/수정 코드 3세트 (L-C-08~10)
- [ ] 데드락 병렬 스택 스크린샷 (L-C-11)
- [ ] 스레드 수 1→1,000 처리 시간 표 (L-C-07)
- [ ] Debug/Release 비교 표 (L-C-14)
- [ ] C#: 상태 머신 디컴파일 캡처, 함정 12개 프로젝트 / C++: lock-free SPSC(ASan 통과), false sharing 비교

**환경**
- [ ] SQLite 확인, redis-windows `PONG` 확인, (선택) MySQL 접속 확인

**기록**
- [ ] 학습 노트 15일치, 주간 회고 3개
- [ ] 지침 파일을 1회 이상 개정한 이력

### 8.2 재구현 시험 (AI 없음, 60분)

**문제**: 빈 프로젝트에서 로그 큐(단일 방식, 파일 기록 포함)를 구현하고 아래 **제공 테스트 3개**를 통과시킨다. 테스트는 미리 공개되어 있으므로 시험 전에 읽어 두어도 된다.

1. `순서보장`: 생산자 2개가 각각 1..5,000을 넣고 종료. 파일에서 생산자별 순서가 오름차순
2. `flush`: 1,000건을 넣고 즉시 Stop. 파일 라인 수 == 1,000
3. `용량Drop`: 용량 100, 5,000건 폭주. `Written == Enqueued - Dropped`이고 Dropped > 0

**통과 기준**: 60분 내 3개 전부 통과. 실패 시 §8.5.

### 8.3 설명 시험 질문 은행 (AI 면접관, 30~60분)

무작위로 10문항을 뽑아 5점 척도로 채점(총론 §5.4). 평균 4.0 이상이면 통과.

**공통(10)**
1. 데드락 4조건을 말하고, 각 조건을 깨는 방법을 코드 예로 설명하라
2. 경쟁 조건을 세 가지 방법으로 고쳤다. 각각의 비용과 적합한 상황은
3. 캐시 라인과 false sharing이 처리량에 미친 영향을 자기 측정치로 설명하라
4. 평균 대신 p99를 보는 이유는. p99만 봐도 되는가
5. Debug 측정이 무의미한 이유를 자기 수치로 설명하라
6. 스레드 수를 코어 수 이상으로 올렸을 때 느려지는 이유 3가지
7. 워밍업이 필요한 이유(JIT, 캐시, 분기 예측)
8. 큐가 가득 찼을 때 Drop과 Block 중 무엇을 택할 것인가. 게임 서버 로그라면
9. 단위 테스트로 잡을 수 없는 버그의 예와 그때 쓰는 도구
10. 컨텍스트 스위칭이 비싼 이유를 커널/캐시 관점에서

**C#(10)**
11. `await`가 컴파일되면 어떤 구조가 되는가. `MoveNext`는 누가 언제 호출하는가
12. GC 세대가 3개인 이유, LOH가 별도인 이유와 85KB 경계의 의미
13. `lock` 기반 큐와 `Channel<T>`의 차이, 어느 상황에 어느 것을 쓰는가
14. ThreadPool 기아가 생기는 코드 패턴과 감지 방법, 해결 방법
15. `Task`와 `ValueTask`의 차이. `ValueTask`를 두 번 await 하면 왜 위험한가
16. `ConfigureAwait(false)`는 무엇을 바꾸는가. 서버 코드에서 필요한가
17. 박싱이 언제 일어나고 왜 비싼가. 피하는 방법 2가지
18. `Span<T>`를 필드에 저장할 수 없는 이유
19. `async void`가 금지인 이유와 예외
20. `ArrayPool<T>.Shared`와 직접 만든 풀의 차이, 언제 무엇을

**C++(10)**
21. `unique_ptr`과 `shared_ptr`의 비용 차이와 선택 기준, `weak_ptr`이 필요한 경우
22. 이동 생성자가 호출되는 조건과 복사 생략(copy elision)의 관계
23. acquire/release와 seq_cst의 차이를 SPSC 큐 예로 설명하라
24. `SRWLOCK`과 `std::mutex`의 차이, 읽기 공유 락이 이득인 조건
25. `condition_variable`에서 술어 없는 `wait`이 왜 위험한가
26. RAII가 예외 안전성을 어떻게 보장하는가. 예외 안전성 3단계는
27. false sharing을 코드에서 어떻게 없앴는가
28. UB 세 가지를 들고, 최적화가 동작을 바꾼 사례를 설명하라
29. ASan이 잡는 오류 종류와 못 잡는 오류 종류
30. Rule of 0/3/5는 무엇이며 왜 0이 최선인가

### 8.4 결함 찾기 (45분)

에이전트에게 자기 1-1/1-2 코드 기반으로 **결함 5개**(경쟁 조건, 리소스 누수, 경계 조건, 종료 처리 누락, 정책 위반)를 숨긴 버전을 만들게 하고, 4개 이상을 찾아 각각 수정안을 제시한다. 프롬프트는 §6.1 "결함 주입" 또는 총론 T11.

### 8.5 미달 시 대응

| 미달 항목 | 대응 |
|---|---|
| 재구현 60분 초과 | Phase 2 1주차에 매일 30분씩 5일간 재시도. 3회 실패 시 큐 구현을 처음부터 다시 학습 |
| 설명 시험 3.5 미만 | 약한 주제 2개를 Phase 2 첫 주에 8h 보강 후 재시험 |
| 1-1 리포트 미완 | Phase 2 2주차 금요일까지 완성(측정만 남았다면 우선 측정) |
| 1-2 미완 | **통과 불가**. Phase 2 패킷 버퍼가 이것을 쓰므로 최우선 완료 |
| 학습 노트 10일 미만 | 기억나는 범위에서 소급 작성하지 말고, 남은 기간 매일 작성으로 대체 |

---

## 9. 흔한 막힘 포인트와 대처

| 증상 | 원인 | 대처 |
|---|---|---|
| BenchmarkDotNet 결과가 매번 다르다 | Debug 빌드, 백그라운드 프로세스, 전원 옵션, 터보 부스트 | Release, 고성능 전원, 다른 앱 종료, 반복 후 중앙값·표준편차 표기 |
| BenchmarkDotNet이 "Debug 빌드" 경고 | `dotnet run`만 실행 | `dotnet run -c Release` |
| 벤치마크 대상이 통째로 사라진다 | 컴파일러가 결과 미사용 코드 제거 | C#은 결과를 필드에 저장, C++은 `benchmark::DoNotOptimize` |
| C++ ASan이 MSVC에서 안 켜진다 | 속성 미설정, `/RTC`와 충돌 | 속성 → C/C++ → 일반 → "AddressSanitizer 사용", `/RTC` 끄기, Debug 구성에서 실행 |
| `dotnet-counters`가 프로세스를 못 찾는다 | 도구 미설치, PID 오류 | `dotnet tool install -g dotnet-counters`, `dotnet-counters ps` |
| `Channel` 소비자가 종료되지 않는다 | `Writer.Complete()` 누락 | Stop에서 Complete 호출 후 `ReadAllAsync` 루프 종료 확인 |
| `condition_variable` 알림을 놓친다 | 술어 없는 wait, spurious wakeup | `wait(lock, pred)` 형태로만 사용 |
| Block 정책에서 Stop이 멈춘다 | 대기 중인 생산자를 안 깨움 | Stop에서 종료 플래그 설정 후 `notify_all`(C#은 Channel Complete) |
| 파일에 줄이 섞여 나온다 | 소비자가 2개 이상이거나 파일 스트림 공유 | 소비자는 반드시 1개, 스트림 소유자 명시 |
| 테스트가 가끔 실패한다(flaky) | `Thread.Sleep` 기반 동기화 | 이벤트/`TaskCompletionSource`/`future`로 결정적 대기 |
| CMake가 vcpkg 패키지를 못 찾는다 | 툴체인 파일 미지정 | `-DCMAKE_TOOLCHAIN_FILE=<vcpkg>/scripts/buildsystems/vcpkg.cmake` |
| `git status`에 빌드 산출물이 뜬다 | `.gitignore` 누락 | `bin/ obj/ build/ .vs/ BenchmarkDotNet.Artifacts/` 추가 후 `git rm -r --cached` |
| 에이전트가 만든 코드가 너무 커서 못 읽겠다 | 한 번에 전체 생성 | 지침 파일의 "한 번에 파일 하나" 재확인, 요청을 단계로 분할(§6.1 페어 모드 프롬프트) |
| 에이전트가 테스트 없이 "완료"라고 한다 | 지침이 모호 | 지침에 "테스트 없이 완료 보고 금지"를 명문화하고 위반 시 지적 |
| 데드락 재현이 안 된다 | 타이밍 문제 | 락 사이에 `Sleep(1)` 삽입, 반복 횟수 증가 |
| 가시성 문제가 x86에서 재현 안 됨 | x86의 강한 메모리 순서 | 정상이다. "왜 ARM에서는 깨지는가"를 조사해 노트에 남긴다 |

---

## 10. Phase 2로 넘어가기 전 준비

- [ ] 교재 "게임 서버 개발, 네트워크부터 이해하기"를 로컬에 클론(Phase 2 4일차에 통독)
- [ ] 과제 1-2 버퍼 풀을 **독립 라이브러리 프로젝트**로 분리하고, 다른 솔루션에서 참조되는지 확인
- [ ] `scripts/build-and-test.ps1`에 `phase2` 경로를 추가할 자리를 만들어 둔다(파라미터화)
- [ ] Phase 2에서 쓸 포트 대역(예: 9000~9100)이 방화벽에서 막히지 않는지 확인. 필요하면 인바운드 규칙 추가
- [ ] 로컬 루프백 테스트 시 동적 포트 고갈을 대비해 현재 설정 확인: `netsh int ipv4 show dynamicport tcp`
- [ ] Phase 1 회고를 회고 노트에 남긴다: 가장 오래 막힌 지점, AI를 잘못 쓴 사례 1개, 다음 Phase에 가져갈 습관 1개

---

## 11. 2026-09-05 보강 사항 (앞 절과 충돌하면 이 절 우선)

### 11.1 현실화한 일정과 필수 실습

- 주간 40시간은 개념 12h, 실습·과제 14h, AI 없는 재구현 5h, 리뷰·평가 4h, 완충 5h로 계산한다. 1-C/1-1/1-2의 명목 시간은 읽기·재구현·리뷰를 포함하며 실제 구현 슬롯은 각각 6h/14h/10h다
- 6일차 오전에 워커 예외·파일 쓰기 실패·디스크 풀 정책과 페이크 싱크 테스트를 2h 배치한다. 7일차에는 `appsettings.json`/`IOptions<T>` 또는 JSON/TOML 설정 로딩 30분을 배치한다
- 12일차에 Serilog `Sinks.Async(blockWhenFull)`와 spdlog `block/overrun_oldest`를 내 큐와 비교하고, `ILogSink` 테스트 더블·행(hang) 덤프·Release+PDB 디버깅을 실습한다
- 14일차 C++ 필수 블록은 `L-CPP-11`·`L-CPP-12` 90분이다. C#은 `L-CS-10`·`L-CS-11`·`L-CS-12` 중 2개를 수행한다. `L-CS-08`은 콘솔에서 재현 가능한 함정 5개만 필수로 줄인다
- 각 주에 반일 완충 슬롯을 두고 미완료 실습을 우선 소진한다. 9일차 교재 통독은 저녁/주말 참고 모드로, 10일차 장시간 테스트는 12일차로 이동한다

### 11.2 도구·메모리·테스트 보강

- MSVC ASan은 UAF·범위 오류는 잡지만 LeakSanitizer·데이터 레이스 검출은 제공하지 않는다. 누수는 `_CrtSetDbgFlag(_CRTDBG_ALLOC_MEM_DF | _CRTDBG_LEAK_CHECK_DF)`, VS 메모리 스냅숏, 대여/반환·생성/소멸 카운터로 판정한다
- C# false sharing(`[StructLayout(LayoutKind.Explicit)]`, `[FieldOffset]` 패딩), AoS/SoA, `Volatile.Read/Write`, `Interlocked`, .NET 9+ `System.Threading.Lock`을 비교한다. C++은 `InitOnce`, Event, `WaitForMultipleObjects`, `WaitOnAddress`/`atomic::wait`를 종료·Block 대기에 연결한다
- `Stopwatch`/`QueryPerformanceCounter`/`steady_clock`과 wall clock의 차이, 임시 디렉터리 fixture, 테스트별 timeout, slow-test 분리, `IClock` 주입을 공통 규율로 적용한다
- `git bisect`, `reflog`, `revert`, 태그, pre-commit 훅으로 `scripts/build-and-test.ps1` 실행, `.gitattributes`의 LF 정책을 실습한다. `global.json`을 1-C 산출물에 포함한다

### 11.3 평가·AI 검증

- 1-1에 싱크 예외 시 큐 생존·오류 통계 증가 테스트를 추가한다. C# lock 버전 300k msg/s, C++ SPSC 5M msg/s는 합격선이 아닌 참고 기준이며 환경 차이를 기록한다
- 질문 은행에 `volatile`과 원자성, false sharing, `steady_clock`, 테스트 더블, hang dump, ASan의 한계를 추가한다. AI가 lock-free를 무조건 빠르다고 하거나 ASan을 누수/레이스 검출기로 설명하면 공식 문서와 계측으로 반박한다
- Phase 2 IOCP 수신 버퍼는 OVERLAPPED 완료까지 RAII 핸들이 소유하고, 크기·슬라이스·워커 수·대여 빈도를 `REPORT-1-2.md`에 미리 명시한다
