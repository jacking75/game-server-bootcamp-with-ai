# 작업 로그

## 2026-09-05 22:34 KST — REVIEW-2026-09-05 전체 반영

- 리뷰 작업 패키지 A~G를 반영했다. 패킷 길이·Channel 포화 계약·SQLite 공유 캐시·비밀번호 해시·로그인 성능 기준·C++ 풀 소유권·ASan 한계·캡스톤 프로세스·알람 전달 경로를 직접 수정했다.
- Phase 1~6 한·영문에 일정 재배치, 완충 슬롯, 신규 필수 실습, DoD 수치, 질문 은행·AI 오답·막힘·선수 조건·트랙 균형 보강 절을 동기화했다.
- `08-templates`에 T13~T18(AI-USAGE, SECURITY-CHECK, 저장소 README, ARCHITECTURE/계약, 측정·메트릭·장애 주입, 테스트·면접·발표)을 추가하고 T1/T2/T3/T4/T6/T7/T9를 갱신했다.
- 기본 `README.md`를 영어 총론으로 전환하고 기존 한국어판을 `README_kr.md`로 보존했다. 교재 읽기 예산·모드, 포트·프로세스 표, ADR 번호표, 한영 동기화 정책을 추가했다.
- `.gitattributes`를 추가해 Markdown LF를 고정하고, 리뷰 보고서 자체에 완료 상태를 기록했다.

## 2026-09-04 22:19 KST — Phase 문서 6종 전면 상세화 (3배 이상 확장)

- docs-working의 Phase 문서 6개(01~06)를 "그냥 따라 하면 되는" 수준으로 전면 재작성했다. 각 문서에 ① 문서 사용법·선수 조건 자가 진단 ② **일 단위 학습 계획**(Phase별 15~30일, 오전 개념/오후 실습/AI 없는 1시간/DoD 체크박스) ③ **실습 카탈로그**(목표·단계·기대 결과·확인 기준·흔한 오류 형식, 총 180여 개) ④ 상세 과제 명세(API 시그니처·테이블 DDL·패킷 목록·테스트 케이스 목록·채점표·리포트 템플릿) ⑤ 설명 시험 질문 은행(문서당 15~22문항) ⑥ 확장된 막힘 포인트 표를 추가했다.
- 분량: 01(348→1,450줄), 02(338→1,221), 03(330→1,072), 04(311→1,160), 05(297→1,170), 06(300→1,151). 전 문서 3배 이상.
- 과정 전체를 1~130일차 연속 번호로 이어 붙여 어느 날 무엇을 하는지 바로 찾을 수 있게 했고, 과제·실습·교재·평가가 서로 번호로 참조되도록 정리했다.
- 기존 정책(VS 2026/.NET 10, CI 미사용→로컬 스크립트, DB는 Repository+DI로 SQLite 기본·MySQL 선택, redis-windows, Serilog, 패킷 관찰 제외, 배포 제외→로컬 다중 프로세스, AI 에이전트 비강제)을 전 문서에 일관되게 유지했다.

## 2026-09-04 21:20 KST — 전체 검증 및 최종 정합성 수정

- README.md, docs-working 8개 문서 전체를 직접 재검토(git diff·grep·전체 파일 정독)해 VS2026/.NET10, CI 미사용, DB(SQLite 기본/MySQL 선택+DI), redis-windows, Serilog, 패킷 관찰 제외, 배포 제외, AI 에이전트 유연화 9개 정책이 모두 일관되게 반영됐는지 확인.
- README.md §3 전체 로드맵 표의 Phase 4 행이 "MySQL" 중심 서술로 남아 있던 것을 "DB(SQLite 기본/MySQL 선택)+DI" 서술로 수정.
- Phase 4/5/6 문서의 시간·배점 합계(140h/60h/60h, 채점표 100점 등)가 재구성 후에도 정확히 맞는지 검산 완료.
- 각 파일별 잔여 stale 표현(Wireshark, VS2022, net9.0, Memurai/WSL2, ZLogger, CI/배포 관련) 저장소 전체 grep 재검색 결과 정책 위반 없음 확인.

## 2026-09-04 21:16 KST — 정책 반영 수정 보완

- 07-books-guide.md: "게임 서버 개발자를 위한 최신 Win32 API 프로그래밍" 카드가 여전히 Phase 5 5-3(Win32 서비스 구현)을 언급하던 것을 05-phase5-operations.md의 실제 내용(배포·서비스 주제 삭제)에 맞게 정정.
- 04-phase4-data-api.md: 심화 과제 4-3에 "4-3c MySQL 전환"(SQLite→MySQL 구현체 교체 후 EXPLAIN·인덱스·100만 건 성능 동일 조건 재측정)을 추가해 MySQL 선택 심화 학습 경로를 명시.

## 2026-09-04 21:10 KST — README.md 및 docs-working 문서 정책 반영 수정

- IDE/런타임을 Visual Studio 2026·.NET 10 기준으로 전체 문서(README, 01~08)에서 갱신(각 책의 실제 기준 버전 같은 사실 서술은 유지).
- CI(GitHub Actions 등) 관련 요구사항·산출물·채점 항목을 전부 제거하고 로컬 빌드·테스트 실행으로 대체.
- DB 아키텍처를 "Repository 인터페이스 + DI로 추상화, 기본은 SQLite(설치 불필요), 선택 심화로 MySQL 전환 가능"으로 재정의하고 Phase 4를 중심으로 전 문서에 반영(스키마·트랜잭션·EXPLAIN 등도 SQLite 기준으로 먼저 통과시키고 MySQL은 선택 시 확장).
- Redis 대체 수단(Memurai/WSL2)을 redis-windows(https://github.com/redis-windows/redis-windows)로 일원화.
- C# 로깅은 ZLogger 언급을 제거하고 Serilog로 통일(C++ spdlog는 유지).
- 패킷 관찰(Wireshark) 학습·과제·확인 방법을 제거하고 서버 로그·`netstat`/`Get-NetTCPConnection`·코드 기반 확인으로 대체(Phase 2가 가장 큰 영향).
- 배포(Windows 서비스·CI/CD·배포 대상 머신·롤백·원격 배포)를 전면 제외하고, Phase 5 제목을 "성능·관측·보안"으로 변경, Phase 5/6의 배포 과제를 "로컬 다중 프로세스 실행 자동화(PowerShell 스크립트)"로 대체.
- AI 코딩 에이전트 표기를 "Claude Code를 기본으로 하되 Codex CLI 등도 사용 가능"으로 완화(특정 도구 필수화 해제).
