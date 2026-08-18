---
name: run_pipeline
description: 소프트웨어 개발 파이프라인(SDLC)을 지휘합니다. 브랜치 파생부터 마이크로 커밋, E2E 통합 테스트, MR 생성까지 애자일 사이클 전체를 통제합니다. 전체 시스템 개발, 하네스 가동, 특정 파트(프론트엔드 단독, 백엔드 단독, 인프라 단독 등) 작업 요청 시 반드시 이 스킬을 호출하십시오. 단, 코드의 단순 에러 디버깅 등 국소적인 작업에는 이 스킬을 트리거하지 마십시오.
allowed-tools:
  - TaskCreate
  - TaskUpdate
  - Agent
  - SendMessage
  - Read
  - Write
  - Bash
---

# Skill: Master Orchestrator Pipeline

이 스킬은 12개의 에이전트 페르소나를 페이즈(Phase)별로 동적 라우팅하여 스폰하고, 공유 작업 목록(Task)과 직접 통신(P2P)을 통해 작업을 조율한 뒤 안전하게 해체하는 마스터 지휘소다.

## 📌 Orchestration Rules (절대 준수 규칙)

1. **팀원 간 직접 통신 (P2P Communication)**
   - `Agent`로 teammate를 이름과 역할을 지정해 스폰한다. 첫 teammate가 스폰되면 현재 세션의 agent team이 자동 구성된다.
   - 단일 인스턴스 역할의 teammate 이름은 반드시 agent type과 동일하게 지정한다(예: `frontend-developer`). 같은 역할을 복제할 때는 `frontend-developer-1`처럼 고유 이름을 부여하고, spawn prompt에 함께 통신할 모든 실제 recipient 이름을 명시한다.
   - teammate들은 리더(오케스트레이터)를 거치지 않고, 반드시 `SendMessage(to: "정확한 에이전트명")`를 사용하여 서로 직접 소통하고 피드백 루프를 돌아야 한다.
   - 브로드캐스트 수신자 `"all"`은 사용하지 않는다. 모두에게 알려야 할 때는 활성 teammate별로 한 번씩 전송한다.
2. **명시적 작업 할당 (Task Board)**
   - 각 페이즈가 시작될 때 오케스트레이터는 구두로 지시하지 말고, 반드시 `TaskCreate`를 호출하여 에이전트들이 수행할 작업(Task)들을 명확한 티켓 형태로 보드에 등록해야 한다.
3. **마이크로 커밋 및 안전 종료 시퀀스 (Micro-commits & Graceful Shutdown)**
   - 각 Phase나 개발 트랙 종료 시 활성 teammate 각각에게 이름으로 `shutdown_request`를 전송하고, 파일 I/O 완료와 종료 승인을 모두 확인한다.
   - **그 다음, Rule 6의 페이즈 인계 파일(`.claude/_workspace/handoff/phase-<N>.md`)을 기록한다.** 다음 페이즈가 필요한 사실을 파일에 먼저 남긴 뒤 커밋해야, 컨텍스트가 요약되거나 세션이 끊겨도 인계가 끊기지 않는다.
   - **그 뒤에, 오케스트레이터가 직접 `Bash` 도구를 사용하여 해당 Phase의 변경만 스테이징하고 `git commit -m "feat(Phase N): [작업명] 완료"` 형식으로 스냅샷을 저장한다.**
   - 별도 팀 삭제 도구는 사용하지 않는다. 공유 팀 리소스는 세션 종료 시 자동 정리된다.
4. **감사 로그 기록 (Audit Logging)**
   - 각 페이즈가 시작하고 종료될 때마다 `.claude/_workspace/log/orchestrator-log.jsonl` 파일에 Append-only 방식으로 로그를 남긴다.
   - 포맷: `{"timestamp": "ISO8601", "phase": "Phase N", "status": "START|END", "task_batch": ["task1", "task2"]}`
5. **기술 스택 SSOT 강제 (Stack Binding) — 정적 주입 방식**
   - 이 하네스는 **어떤 기술 스택도 전제하지 않는다.** 스택·경로 소유권·표준 명령어·계약 형식·아키텍처 규약은 오직 `.claude/_workspace/01_architecture/design.md`가 정의한다.
   - ⛔ **에이전트에게 `design.md`를 도구로 읽으라고 지시하지 마라.** 스폰 프롬프트에 "착수 전 설계서를 읽어라", "`Read`로 design.md를 확인하라" 같은 문구를 넣는 것을 금지한다. 대신 아래 주입 절차를 따른다.
   - 🔧 **주입 절차 (하네스 단 정적 주입):** 에이전트를 스폰하기 **전에** 오케스트레이터가 직접 `Bash`로 실행한다.
     ```bash
     node .claude/tools/inject-design.mjs
     ```
     이 스크립트는 `fs.readFileSync`로 `design.md` 전문을 읽어 각 대상 에이전트 정의 파일(`.claude/agents/<name>.md`)의 **프론트매터 직후 = 시스템 프롬프트 최상단**에 `<!-- DESIGN_SPEC:BEGIN --> … <!-- DESIGN_SPEC:END -->` 블록으로 보간한다. 멱등이므로 몇 번을 실행해도 안전하다.
   - 📌 **주입이 캐싱에 유리한 이유:** 에이전트 정의 본문은 그대로 서브 에이전트의 시스템 프롬프트가 되며, 이는 대화의 **불변 접두사**다. 같은 에이전트 타입을 여러 번 스폰해도 이 영역은 캐시 히트한다. 반면 `Read` 호출은 에이전트마다 왕복을 1회씩 추가로 소모하고, 설계 전문이 접두사가 아닌 대화 중간에 실려 재사용되지 않는다.
   - 🕐 **재주입 시점 (필수):**
     - Phase 0 진입 직후 (1회)
     - `system-architect`가 `design.md`를 생성·수정한 직후 (Phase 1 종료 시점)
     - 파이프라인 도중 사용자가 설계를 수정한 것을 인지한 직후
   - ✅ **주입 최신성 검증:** 각 에이전트는 최종 보고 첫 줄에 `DESIGN_FINGERPRINT: <값>`을 반환한다. 오케스트레이터는 `node .claude/tools/inject-design.mjs --check`로 확인한 현재 지문과 대조한다.
     - **지문 불일치 또는 `none` 반환:** Claude Code가 세션 시작 시점의 에이전트 정의를 잡고 있다는 뜻이다. 파이프라인을 멈추고, 사용자에게 **세션 재시작**(또는 `/agents` 재로드) 후 재개를 요청한다. 스폰 프롬프트에 설계 전문을 붙여넣는 방식으로 우회하지 마라 — 그것이 바로 제거하려던 토큰 낭비다.
   - 🚫 하위 에이전트가 주입 블록에 없는 프레임워크·도구·명령어를 사용하려 하면 즉시 중단시키고, 아키텍처를 갱신(➔ 재주입)하거나 사용자에게 질의한다.
   - 🚫 `system-architect`와 `release-manager`는 주입 대상이 아니다. 전자는 `design.md`의 **생산자**이므로 낡은 사본을 주입받으면 안 되고(이 역할만 `design.md`를 직접 읽고 쓴다), 후자는 스택 의존성이 없다.
6. **페이즈 인계 계약 (Phase Handoff Contract)**
   - 오케스트레이터 컨텍스트는 길어지면 요약(auto-compact)되거나 세션 재시작으로 사라진다. 페이즈 경계에서 다음 페이즈가 필요한 사실을 **파일로 남겨** 컨텍스트를 잃어도 인계가 끊기지 않게 한다. 컨텍스트를 잘라내는 것이 아니라 **잃어도 무해한 구조**를 만드는 것이 목적이다.
   - **경로:** `.claude/_workspace/handoff/phase-<N>.md` (N은 페이즈 번호. 같은 페이즈를 재실행하면 덮어쓴다.)
   - **작성 시점:** 각 페이즈 종료 시퀀스에서 teammate 종료 승인을 확인한 **직후, 마이크로 커밋 직전** (Rule 3).
   - **필수 필드 9개:** `phase` / `status` / `design_fingerprint` / `branch` / `issue` / `commit` / `artifacts` / `open_items` / `next`
   - ⚠️ `commit`은 마이크로 커밋 **직후** 실제 해시로 그 한 줄만 갱신한다. 인계 파일을 커밋보다 먼저 쓰는 이유는 teammate의 파일 I/O가 끝난 상태를 먼저 고정하기 위함이므로, 해시를 모른다고 기록을 커밋 뒤로 미루지 않는다.
   - **상한: 40줄 또는 2KB.** 초과하면 `open_items`를 요약해 줄인다. 인계 파일이 길어지면 다음 페이즈가 그것을 읽는 비용이 절약분을 잡아먹는다.
   - ⛔ **설계서·계약·소스 코드의 본문을 복사해 넣지 마라. `artifacts`에는 파일 경로만 적는다.** 본문을 옮기면 컨텍스트를 절약하려는 목적 자체가 무너진다.
   - `_workspace/handoff/`는 런타임 산출물이며 **추적 대상이 아니다.** 스테이징하거나 커밋하지 않는다.
   - **템플릿:**
     ```markdown
     phase: 3
     status: DONE            # DONE | PAUSED | FAILED
     design_fingerprint: 4bd3eaa8e19e
     branch: feature/12-login-api
     issue: "#12 https://github.com/<org>/<repo>/issues/12"
     commit: a1b2c3d      # 마이크로 커밋 직후 채운다
     artifacts:
       - .claude/_workspace/03_contracts/auth.contract.ts
       - src/api/auth/
     open_items:
       - "code-reviewer 반려 1건: 토큰 만료 경계 테스트 누락 → Phase 4에서 재검증 필요"
     next: "Phase 4 (E2E). 진입 조건: 위 반려 1건 해소 확인"
     ```

---

## 🚀 Workflow (작업 순서)

### Phase 0: 컨텍스트 분석 및 동적 라우팅
- 사용자 요청과 `.claude/_workspace/`의 기존 산출물을 분석하여 필요한 페이즈만 선택한다.
- ⭐️ **재개 감지 (Resume Detection):** `.claude/_workspace/handoff/`에 인계 파일이 있으면 **가장 높은 번호 1개만** 읽어 그 지점부터 이어간다 (Rule 6).
  - ⛔ 이전 페이즈들의 인계 파일을 전부 읽거나 `orchestrator-log.jsonl` 전문을 다시 읽지 않는다. **최신 1개로 충분하며**, 여러 개를 읽으면 인계 파일을 둔 절약 효과가 사라진다.
  - 인계 파일의 `status`가 `PAUSED`나 `FAILED`면 그 `open_items`를 먼저 해소한 뒤 다음 페이즈로 진입한다.
- ⭐️ **설계 명세 주입 (최우선 실행):** 어떤 에이전트를 스폰하기 전에 반드시 먼저 실행한다.
  ```bash
  node .claude/tools/inject-design.mjs
  ```
  출력의 `fingerprint` 값을 이번 파이프라인의 기준 지문으로 기억하고, 감사 로그에 함께 남긴다. `design.md`가 없으면 스크립트가 `[NOT READY]` 블록을 주입하여 하위 에이전트가 스택 의존 작업에 착수하지 못하도록 차단한다.
- ⭐️ **스택 확보 선행 검사 (스크립트 위임):** 필수 섹션의 충족 여부를 오케스트레이터가 직접 읽어 판단하지 않고 아래 명령에 위임한다.
  ```bash
  node .claude/tools/inject-design.mjs --sections
  ```
  「기술 스택」·「디렉터리 구조 및 소유권」·「표준 명령어」·「계약 산출 형식」·「아키텍처 규약」 5개 섹션의 존재와 본문 유무를 검사해 미충족이 있으면 exit 1과 미충족 목록을 낸다. 표기 흔들림(헤딩 레벨, `「」`, 번호, 동의 표현)은 스크립트가 흡수한다.
  - ⛔ **오케스트레이터도 `design.md`를 전문 조회하지 않는다.** 판정에 필요한 것은 종료 코드와 섹션 요약뿐이며, 설계 본문은 이미 `inject-design.mjs`가 하위 에이전트의 시스템 프롬프트에 주입하고 있다. 전문을 읽으면 오케스트레이터 컨텍스트를 가장 크게 차지하는 항목이 되므로 `Read`·`cat`으로 열지 마라. (유일한 예외는 `design.md`의 **생산자**인 `system-architect`다. 이 역할만 직접 읽고 쓴다.)
  - **없거나 불완전한 경우:** 어떤 라우트든 구현 페이즈로 진입하기 전에 스택을 먼저 확정한다. 기존 코드베이스가 있으면 `system-architect`를 **스택 확정 목적으로 최소 범위 호출**하여 현행 스택(매니페스트·설정·디렉터리 구조 기반)을 `design.md`에 기록시킨다. 신규 프로젝트면 Phase 1을 정식 수행한다.
  - **추론조차 불가한 경우:** 파이프라인을 멈추고 사용자에게 스택 결정을 질의한다. 스택을 추측해 구현에 들어가지 않는다.
  - **전체 구축 (Full):** Phase 1 ➔ 2 ➔ 3(Track A+B) ➔ 4 ➔ 5
  - **프론트엔드 단독 (FE-only):** Phase 2(계약 필요 시) ➔ Phase 3(Frontend QA/Developer/Reviewer) ➔ 4 ➔ 5
  - **백엔드 단독 (BE-only):** Phase 2(계약 필요 시) ➔ Phase 3(Backend QA/Developer/Reviewer) ➔ 4 ➔ 5
  - **인프라 단독 (Infra-only):** Phase 3 Track B만 실행
  - **문서 단독 (Docs-only):** Phase 5의 `tech-writer`만 실행
- `orchestrator-log.jsonl`에 `INIT` 로그와 선택·생략한 페이즈, 근거, 그리고 `design_fingerprint`를 기록한다.

### Phase 1: 아키텍처 설계
- 전체 구축이거나 아키텍처 변경이 필요한 경우에만 `system-architect` agent type으로 teammate들을 이름을 지정해 스폰하고, `design.md`를 산출한다.
- ⭐️ **산출물 검수 (스크립트 위임):** `node .claude/tools/inject-design.mjs --sections`를 실행해 **기술 스택·디렉터리 구조 및 소유권·표준 명령어·계약 산출 형식·아키텍처 규약** 5개 섹션이 모두 채워졌는지 확인한다. exit 1이면 다음 페이즈로 진행하지 않고, 스크립트가 지목한 미충족 섹션만 아키텍트에게 보완 지시한다. 여기서도 `design.md` 전문을 열지 않는다.
- ⭐️ **[필수] 재주입:** 검수 통과 즉시 `node .claude/tools/inject-design.mjs`를 다시 실행하여 확정된 설계를 하위 에이전트의 시스템 프롬프트에 반영하고, 새 `fingerprint`를 기준 지문으로 갱신한다. **이 단계를 건너뛰면 Phase 2 이후 전원이 낡거나 비어 있는 명세로 작업하게 된다.**
- ⭐️ **[마이크로 커밋]** 완료 후 `git commit -m "docs(architecture): 시스템 설계 완료"` 실행. 주입으로 변경된 `.claude/agents/*.md`도 같은 커밋에 포함한다. 커밋 직전에 Rule 6의 인계 파일 `handoff/phase-1.md`를 기록한다.

### Phase 2: 티켓팅 및 브랜치 파생 (Sub-agent)
- 필요한 역할만 `Agent`로 호출한다. 신규 티켓이 필요하면 `issue-pm` agent type, 계약이 필요하면 `tech-leader` agent type을 명시한다.
- ⭐️ `issue-pm`이 티켓을 생성하고 **`<타입>/<이슈번호>-<슬러그>` 브랜치로 자동 전환(`git switch -c`)**하는지 모니터링한다. 타입은 `feature`·`fix`·`chore`·`docs` 중 작업 성격에 맞는 것을 사용한다.
- ⭐️ 오케스트레이터는 `issue-pm`이 생성한 이슈 본문의 범위가 사용자 요청과 일치하는지 **직접 대조 검증**한다. 불일치 시 즉시 `gh issue edit`/`glab issue update`로 정정하고 감사 로그에 편차를 기록한다.
- ⭐️ `issue-pm`의 `tools`에는 TaskBoard·`SendMessage`가 없다(서브 에이전트). 따라서 티켓 내용은 **스폰 프롬프트 본문에 전문을 담아** 전달하고, `TaskCreate`로 등록한 티켓 참조만 남기지 않는다.
- 완료 후 `git commit -m "chore(issue): 티켓 생성 및 인터페이스 계약 완료"` 실행. 커밋 직전에 `handoff/phase-2.md`를 기록한다 (Rule 6). `artifacts`에 이슈 번호·브랜치명·계약 파일 경로를 남긴다.

### Phase 3: 병렬 개발 트랙 (FE/BE/QA/Infra)
- Track A (앱 구현): 선택된 라우트에 맞춰 `backend-qa`, `backend-developer`, `frontend-qa`, `frontend-developer`, `code-reviewer` agent type 중 필요한 역할을 정확히 명시해 스폰하고 P2P 핑퐁 개발을 진행한다.
- ⭐️ 스폰 프롬프트에는 **설계 조회 지시를 넣지 않는다.** 스택·소유권·표준 명령어는 이미 시스템 프롬프트의 `<design_spec>` 블록에 주입되어 있다. 프롬프트에는 **이번 작업의 범위·대상 파일·완료 조건**만 담고, 각 역할이 자기 소유 경로 밖을 건드리지 않도록 경계만 재확인시킨다.
- ⭐️ 각 에이전트가 반환한 `DESIGN_FINGERPRINT`를 기준 지문과 대조한다. 불일치·누락이면 Orchestration Rule 5의 실패 처리(세션 재시작 요청)를 따른다.
- Track B (인프라): `devops-engineer` agent type으로 teammate를 스폰해 설정을 진행한다.
- 완료 후 `git commit -m "feat(app): Phase 3 애플리케이션 및 인프라 구현 완료"` 실행. 커밋 직전에 `handoff/phase-3.md`를 기록한다 (Rule 6). code-reviewer의 미해결 반려는 `open_items`에 남긴다.
- 필요에 따라 `feat`을 제외한 `fix` 등의 커밋 메시지 형태도 허용된다.

### Phase 4: E2E 통합 테스트 (신규)
- `Agent`로 `e2e-tester` agent type을 명시해 호출한다.
- 주입된 `<design_spec>`이 확정한 E2E 도구로 작성된 테스트가 100% 통과(Green)되는지 대기한다.
- 완료 후 `git commit -m "test(e2e): 브라우저 통합 시나리오 검증 완료"` 실행. 커밋 직전에 `handoff/phase-4.md`를 기록한다 (Rule 6).

### Phase 5: 릴리즈(MR) 및 문서화
- 라우팅 결과에 따라 `release-manager`와 `tech-writer` agent type 중 필요한 역할만 명시해 호출한다.
- 생성된 마이크로 커밋들을 모아 원격 저장소에 Push하고 MR/PR을 생성한 뒤 파이프라인을 종료한다. 종료 직전에 `handoff/phase-5.md`를 `status: DONE`으로 기록한다 (Rule 6).

---

## 🔧 주입 스크립트 레퍼런스 (`.claude/tools/inject-design.mjs`)

| 명령 | 용도 |
|---|---|
| `node .claude/tools/inject-design.mjs` | `design.md` 전문을 대상 에이전트 시스템 프롬프트에 주입/갱신 (멱등) |
| `node .claude/tools/inject-design.mjs --check` | 주입 최신성 검증만 수행. 드리프트가 있으면 exit 1 (CI 스테이지용) |
| `node .claude/tools/inject-design.mjs --json` | 결과를 JSON으로 출력 (`fingerprint`, 에이전트별 상태) |
| `node .claude/tools/inject-design.mjs --sections` | `design.md`의 필수 5개 섹션 완결성만 검사. 미충족 시 exit 1 (파일을 쓰지 않는 읽기 전용) |
| `node .claude/tools/inject-design.mjs --dry-run` | 파일을 쓰지 않고 결과만 확인 |
| `node .claude/tools/inject-design.mjs --clear` | 주입 블록을 제거하고 하네스 원본 상태로 복원 |

- 주입 대상은 스크립트의 `TARGETS` 배열이 정의한다. 에이전트를 추가·삭제하면 이 배열도 함께 갱신한다.
- 주입 블록은 자동 생성 영역이다. `.claude/agents/*.md`의 `<!-- DESIGN_SPEC:BEGIN -->` ~ `<!-- DESIGN_SPEC:END -->` 구간을 손으로 편집하지 않는다.

---

## ⚠️ 에러 핸들링 (Error Handling)
- 각 트랙 내의 코드 리뷰 핑퐁 횟수나 스크립트 실행 재시도가 **3회를 초과**하여 `[PASS WITH WARNING]` 플래그가 반환되면, 오케스트레이터는 즉시 해당 파이프라인의 진행을 일시 정지(Pause)한다.
- ⭐️ 일시 정지 시에는 사용자에게 알리기 **전에** Rule 6의 인계 파일을 `status: PAUSED`로 먼저 기록한다. 사람이 개입을 마친 뒤 그 파일 하나만 읽고 재개할 수 있어야 한다.
- `orchestrator-log.jsonl`에 에러 로그를 기록한 뒤 사용자에게 알림을 띄우고 인간 개입(Human Intervention)을 요청한다.
