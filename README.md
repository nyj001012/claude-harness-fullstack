# claude-harness-fullstack

클로드 코드를 이용할 때 사용할 하네스 (풀스택 웹 개발)

12개의 에이전트 페르소나와 13개의 스킬로 구성된, **애자일 SDLC 전체를 자동으로 굴리는 하네스**다. 요구사항 분석 ➔ 아키텍처 설계 ➔ 티켓팅 ➔ TDD 병렬 개발 ➔ E2E 검증 ➔ PR 생성 ➔ 문서화까지를 페이즈별 에이전트 팀으로 나눠 수행한다.

## 핵심 원리: 기술 스택을 전제하지 않는다

이 하네스는 특정 프레임워크에 묶여 있지 않다. 스택은 **`system-architect`가 요구사항에서 역산해 확정**하고, 하위 에이전트 전원이 그 결정을 읽고 따른다.

```
사용자 요구사항
      │
      ▼
┌─────────────────────────────────────────────┐
│ system-architect                            │
│   → .claude/_workspace/01_architecture/     │
│         design.md  (SSOT)                   │
│                                             │
│   ① 기술 스택 (선정 근거 + 탈락 대안)          │
│   ② 디렉터리 구조 및 역할별 쓰기 소유권         │
│   ③ 표준 명령어 (린트/타입검사/테스트/빌드)     │
│   ④ 계약 산출 형식                            │
│   ⑤ 아키텍처 규약                             │
│   ⑥ 도메인 모델 경계 + 배포 제약               │
└─────────────────────────────────────────────┘
      │  (착수 전 반드시 읽는다)
      ▼
 FE 개발 · BE 개발 · QA · 리뷰어 · E2E · DevOps · 문서화
```

덕분에 같은 하네스로 Next.js 프로젝트도, Spring 프로젝트도, FastAPI 프로젝트도 진행할 수 있다. 에이전트는 자기가 아는 스택이 아니라 `design.md`가 정한 스택의 관용대로 구현하며, **스택 정보가 없으면 추측하지 않고 질의를 띄우고 멈춘다.**

## 파이프라인

`run_pipeline` 스킬이 마스터 오케스트레이터로서 페이즈를 동적 라우팅한다.

| Phase | 하는 일 | 투입 에이전트 |
|---|---|---|
| **0** | 컨텍스트 분석 · 라우팅 · **스택 확보 선행 검사** | (오케스트레이터) |
| **1** | 아키텍처 및 기술 스택 확정 | `system-architect` |
| **2** | 이슈 생성 · 작업 브랜치 파생 · 인터페이스 계약 설계 | `issue-pm`, `tech-leader` |
| **3** | Track A: TDD 병렬 개발 (테스트 선행 → 구현 → 리뷰 핑퐁)<br>Track B: 인프라 · CI/CD | `backend-qa`, `backend-developer`, `frontend-qa`, `frontend-developer`, `code-reviewer`, `devops-engineer` |
| **4** | 실행 환경에서 사용자 시나리오 통합 검증 | `e2e-tester` |
| **5** | 원격 Push · PR/MR 생성 · 위키 문서화 | `release-manager`, `tech-writer` |

라우팅은 요청에 따라 필요한 페이즈만 고른다. (전체 구축 / FE 단독 / BE 단독 / 인프라 단독 / 문서 단독)

각 페이즈 종료 시 오케스트레이터가 마이크로 커밋을 남기고, `.claude/_workspace/log/orchestrator-log.jsonl`에 감사 로그를 append한다.

## 에이전트

| 에이전트 | 역할 | 모델 |
|---|---|---|
| `system-architect` | 기술 스택·구조·규약·소유권 확정, 도메인 경계 설계 | opus |
| `issue-pm` | 마이크로 태스크 분할, GitHub/GitLab 이슈 생성, 작업 브랜치 파생 | sonnet |
| `tech-leader` | FE/BE/QA가 병렬 개발할 수 있는 인터페이스 계약 설계 | sonnet |
| `frontend-qa` | UI 렌더링·이벤트·폴백에 대한 실패하는(Red) 테스트 선행 작성 | sonnet |
| `frontend-developer` | 계약과 테스트를 만족하는 UI·클라이언트 상태 구현 | sonnet |
| `backend-qa` | API·비즈니스 로직의 블랙박스 테스트 선행 작성 | sonnet |
| `backend-developer` | 계층 분리·트랜잭션·구조화 로깅을 지킨 서버 로직 구현 | sonnet |
| `code-reviewer` | 계약·규약·보안·성능 검수, 승인/반려 게이트키퍼 | sonnet |
| `devops-engineer` | 실행 환경·설치 스크립트·CI/CD·관측성 구축 | sonnet |
| `e2e-tester` | 실제 실행 환경에서 사용자 시나리오 통합 검증 | sonnet |
| `release-manager` | 원격 Push 및 PR/MR 생성 | haiku |
| `tech-writer` | API 명세·아키텍처 개요(ADR)·운영 가이드 문서화 | haiku |

## 스킬

| 스킬 | 용도 |
|---|---|
| `run_pipeline` | 마스터 오케스트레이터 (페이즈 라우팅·팀 스폰·커밋) |
| `design_system_architecture` | 기술 스택 선정 및 시스템 설계 |
| `create_agile_issues` | 이슈 생성 및 작업 브랜치 파생 |
| `design_interface_contracts` | 풀스택 인터페이스·데이터 계약 설계 |
| `design_frontend_tdd_cases` | UI TDD 케이스 설계 |
| `design_backend_tdd_cases` | 서버 블랙박스 TDD 케이스 설계 |
| `implement_frontend_ui` | UI·클라이언트 상태 구현 |
| `implement_backend_api` | 서버 API·비즈니스 로직 구현 |
| `perform_code_review` | 코드 리뷰 및 보안·성능 감사 |
| `perform_e2e_testing` | E2E 시나리오 테스트 |
| `setup_infra_cicd` | 인프라·CI/CD·관측성 구축 |
| `create_pr_mr` | Push 및 PR/MR 생성 |
| `write_technical_wiki` | 위키·API 명세 문서화 |

## 산출물 구조

에이전트 간 인수인계는 모두 파일로 이뤄진다.

```
.claude/_workspace/
├── 01_architecture/design.md      # 기술 스택·규약·소유권 (SSOT)
├── 02_issues/issue_report.md      # 생성된 티켓과 작업 브랜치
├── 03_contracts/                  # 인터페이스 계약 (형식은 design.md가 정함)
├── 04_infrastructure/             # 설치·배포 스크립트
└── log/orchestrator-log.jsonl     # 페이즈 감사 로그
```

## 설계 원칙

- **클린 룸 TDD** — QA는 구현 코드를 열람하지 않고 계약만 보고 실패하는 테스트를 먼저 짠다. 개발자는 테스트를 **단 한 줄도 수정할 수 없고** 프로덕션 코드로만 통과시킨다.
- **역할별 쓰기 소유권** — 각 에이전트는 `design.md`의 소유권 표에서 자기에게 배정된 경로만 수정한다. 리뷰어는 경계 위반을 검수 항목으로 확인한다.
- **읽기 전용 게이트키퍼** — `code-reviewer`는 어떤 파일도 직접 고치지 않고, 구체적인 대안 스니펫을 담아 반려한다.
- **객관적 룰북** — 리뷰 기준은 리뷰어의 취향이 아니라 `design.md`다. 규약에 없는 지적은 반려가 아닌 "규약 공백" 제안으로 처리한다.
- **3회 재시도 후 `[PASS WITH WARNING]`** — 무한 핑퐁을 막되 산출물은 보존하고, 인간 개입이 필요한 지점을 명시적으로 남긴다.
- **P2P 통신** — 팀원은 리더를 거치지 않고 서로 직접 `SendMessage`로 피드백 루프를 돈다. 브로드캐스트(`to: "all"`)는 쓰지 않는다.
- **메인 브랜치 보호** — `issue-pm`이 이슈 번호 기반 `<타입>/<이슈번호>-<슬러그>` 브랜치를 먼저 따고, 그 위에서만 개발이 진행된다.

## 사용법

1. 대상 프로젝트 루트에 이 저장소의 `.claude/` 디렉터리를 복사한다.
2. Claude Code에서 하고 싶은 작업을 요청한다. (예: "센서 관제 대시보드를 만들어줘", "로그인 API만 구현해줘")
3. `run_pipeline`이 요청 성격에 맞는 페이즈만 골라 실행한다.

기존 코드베이스가 있는 프로젝트라면 Phase 0에서 현행 스택을 조사해 `design.md`에 기록한 뒤 개발에 들어간다. 신규 프로젝트라면 Phase 1에서 스택을 새로 확정한다. 어느 쪽도 불가능하면 파이프라인은 추측하지 않고 멈춰서 사용자에게 스택 결정을 묻는다.
