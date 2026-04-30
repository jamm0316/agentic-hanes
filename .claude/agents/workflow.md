# Multi-Agent Workflow

robi-g-datahub의 코드 변경 작업은 plan-writer → implementer → test-writer → reviewer 파이프라인을 따른다.
DDD 흐름(요구사항 분석 → 도메인 spec 읽기 → 구현 → 테스트 → 도메인 spec 작성)을 유지하며, 메인 세션이 오케스트레이터 역할을 수행한다.

**핵심 원칙:**
- **도메인 spec은 영속적 진실이자 워크플로우의 중추** (`.claude/docs/domains/`) — 검증된 코드 상태의 최종 정의이며 **plan-writer의 1차 참조원**. spec이 정확하고 최신일수록 plan-writer는 코드베이스 전체를 다시 탐색하지 않아도 되어 **정확성 ↑ + 토큰 ↓**. spec 누락·노후는 곧 후속 워크플로우 비용 증가로 직결된다.
- **plan.md는 일회성 실행 지시서** (`.claude/plans/[티켓]/plan.md`) — 워크플로우 완료 시 삭제된다
- **도메인 spec 갱신은 terminal action** — 코드와 테스트가 PASS된 후 메인 세션이 수행한다. 신규 도메인뿐 아니라 **기존 도메인 중 spec 파일이 부재한 경우(legacy backfill)도 같은 단계에서 신규 작성한다.**
- **DDD 흐름 유지:** 도메인 이해(spec 읽기)로 시작, 도메인 갱신(spec 작성)으로 종료

---

## 1. 적용 범위

### 적용 대상
- 새로운 기능 구현 (Controller/Service/DataAdapter 신규 추가)
- 기존 기능 수정 (도메인 로직, API 스펙 변경)
- 버그 수정 중 코드 변경이 2개 파일 이상에 걸치는 경우
- 리팩토링 (도메인 구조에 영향을 주는 경우)

### 비적용 대상
- 단순 질문, 코드 탐색, 설명
- 단일 파일 1~2줄 수정 (오타, 상수 값 변경)
- 문서만 수정하는 작업
- 의존성 업데이트, 빌드 설정 변경

---

## 2. 파이프라인 개요

```
[Phase 0] 메인 세션: 요구사항 합의
   ↓
[Phase 1] plan-writer: 탐색 + 도메인 spec 읽기 + 상세 계획 작성
   ↓
[Phase 2] implementer (병렬/순차): Task별 코드 변환
   ↓
[Phase 3] test-writer: 변경 클래스에 대한 테스트 작성
   ↓
[Phase 4] reviewer: Trust-5 + 성공 기준 + 테스트 통과 검증
   ↓ (FAIL 시 §5 — retry_target에 따라 분기)
[Phase 5] 메인 세션: 도메인 spec 동기화 ← terminal action
   ↓
[Phase 6] 메인 세션: 최종 보고 + plan.md 정리
```

### 팀 진입 경로

팀원은 **`zebra2` skill 체인**으로 이 파이프라인에 진입한다.

| skill | 담당 Phase |
|-------|-----------|
| `zebra2` (entry) | 티켓 번호 입력·세션 초기화 (cloudId 1회 획득) |
| `zebra2-requirements` | Phase 0 — 비즈니스 요구사항 → `.claude/plans/[티켓]/requirements.md` |
| `zebra2-plan` | Phase 1 — plan-writer 에이전트 dispatch → `.claude/plans/[티켓]/plan.md` |
| `zebra2-implement` | Phase 2~4 — implementer / test-writer / reviewer 오케스트레이션 + retry_target 분기 |
| `zebra2-finish` | Phase 5~6 — 도메인 spec 동기화 + 빌드 점검 + Jira 완료 코멘트 1회 push + plan.md 정리 |

**Jira MCP 호출 상한: 3회** (cloudId 1 + `getJiraIssue` 1 + `addCommentToJiraIssue` 1). 중간 단계 산출물은 모두 로컬 파일에 저장한다. Jira description은 **절대 수정하지 않는다** (ADF 콘텐츠 손상 리스크).

메인 세션이 수동으로 오케스트레이션할 때도 동일한 Phase 정의를 따른다 — skill 체인은 진입 편의이지 별도 규칙이 아니다.

---

## 2.1 에이전트 카탈로그

파이프라인에서 사용하는 서브에이전트 목록. 정의 파일은 `.claude/agents/[이름].md`에 있고, 메인 세션은 **Agent 도구**로 `subagent_type` 파라미터를 지정해 호출한다.

| 에이전트 | Phase | 모델 (기본) | 도구 | permissionMode |
|----------|-------|------------|------|----------------|
| `plan-writer` | Phase 1 | **opus** | Read, Grep, Glob, Bash, Agent(Explore) | plan |
| `implementer` | Phase 2 | **haiku** (Task별로 sonnet 오버라이드 가능) | Read, Write, Edit, Bash, Grep, Glob | acceptEdits |
| `test-writer` | Phase 3 | **sonnet** | Read, Edit, Write, Glob, Grep, Bash | — |
| `reviewer` | Phase 4 | **sonnet** | Read, Grep, Glob, Bash | plan |

### 호출 방법

메인 세션이 Agent 도구를 사용해 다음과 같이 호출한다:

```
Agent(
  description="[간단 설명]",
  subagent_type="plan-writer",  // 또는 "implementer" / "test-writer" / "reviewer"
  prompt="[전달 페이로드 — §4 핸드오프 Contract 참조]"
)
```

**모델 오버라이드:** plan-writer가 plan.md에 각 Task의 복잡도에 따라 `모델: haiku` 또는 `모델: sonnet`을 명시한다. 메인 세션이 implementer를 호출할 때 `model` 파라미터로 이를 전달하여 per-Task 모델 선택을 수행한다.

**병렬 호출:** 서로 독립적인 Task에 대해 implementer를 병렬로 호출할 때는 **단일 메시지에 여러 Agent 도구 호출**을 포함한다. 순차 의존이 있는 Task는 선행 호출 완료 후 후속 호출.

---

## 3. Phase별 정의

### Phase 0: 요구사항 합의 (메인 세션)

**처리:**
1. 사용자와 다음 항목을 합의한다 — 추측 금지:
   - **Goal**: 비즈니스/도메인 맥락
   - **아키텍처 방향**: 어느 레이어에 어떤 책임
   - **보안 고려사항**: 인증/인가, 데이터 접근 범위
   - **성공 기준**: 검증 가능한 구체적 조건
   - **실패 기준**: 절대 하지 않을 것
   - **범위 제한**: 이번 작업 외 사항
2. **확인이 필요한 항목은 반드시 `AskUserQuestion` 도구로 질문한다** — 텍스트 질문 금지. 선택지(2~4개)와 추천 옵션 제시.
3. 합의 결과는 `.claude/plans/[티켓]/requirements.md`에 저장한다 (zebra2 체인을 거치면 `zebra2-requirements` skill이 담당).
4. 합의 완료 후 사용자에게 "계획을 수립합니다" 알린 뒤 Phase 1로 진행

**컨텍스트:**
- 읽기: 사용자 대화, Jira 티켓 본문(zebra2-requirements skill 경유 시), CLAUDE.md, workflow.md, `.claude/docs/domains/*`
- 쓰기: `.claude/plans/[티켓]/requirements.md`

**다음 단계:** Phase 1 plan-writer 호출

---

### Phase 1: 계획 수립 (plan-writer)

**호출 대상:** `plan-writer` (model: **opus**) — Agent 도구로 `subagent_type="plan-writer"` 지정

**처리:**
1. 영향받는 도메인의 `.claude/docs/domains/[도메인].md`를 탐색 전에 먼저 읽는다 (DDD 요구사항: 기존 도메인 이해).
   - **spec 파일이 부재하면**: plan.md 상단에 `## Spec 부재 도메인` 섹션을 만들어 도메인 이름과 사유(`신규 도메인` 또는 `기존 도메인 backfill 필요`)를 명시한다. 이 섹션은 Phase 5 메인 세션이 backfill 신호로 사용한다.
   - spec 부재 시에도 Phase 1 작업은 계속 진행 — 코드베이스 탐색으로 정보를 보충해 plan을 완성한다.
2. Explore 에이전트로 코드베이스 탐색.
3. Task별 상세 계획 작성 (메서드 시그니처, 구현 로직, 참고 코드, 성공/실패 기준).
4. **Task 분할 계약**: haiku 컨텍스트 포화를 막기 위해 단일 파일·참고 코드 ≤1·신규 메서드 ≤3·로직 ≤5단계 기준으로 Task를 쪼갠다. 초과 시 분할 또는 sonnet 상향 (plan-writer.md §4 참조).
5. **코드 변경 Task만 작성한다.** 테스트 작성은 Phase 3 test-writer 책임, 도메인 spec 갱신은 Phase 5 메인 세션 책임이다. plan에 포함하지 않는다.
6. plan.md를 `.claude/plans/[티켓]/plan.md`에 저장.

**컨텍스트:**
- 읽기: `.claude/plans/[티켓]/requirements.md`, CLAUDE.md, workflow.md, `.claude/docs/domains/*`, 소스 코드 전체
- 쓰기: `.claude/plans/[티켓]/plan.md`
- 참조 금지: 사용자 대화 원문 (`requirements.md`만 전달받음)

**다음 단계:** 메인 세션이 plan을 받아 Phase 2로 전달

---

### Phase 2: 구현 (implementer, 병렬/순차)

**호출 대상:** `implementer` (model: **haiku** 기본, plan.md Task 설정에 따라 **sonnet** 오버라이드) — Agent 도구로 `subagent_type="implementer"` 지정

**처리:**
1. 해당 Task의 plan 섹션을 그대로 코드로 변환.
2. 메서드 시그니처, 구현 로직 순서, 참고 코드 패턴을 그대로 따른다.
3. **테스트 파일은 작성하지 않는다** (Phase 3 test-writer 책임).
4. **도메인 spec 파일은 절대 건드리지 않는다** (Phase 5 책임).

**컨텍스트:**
- 읽기: 자기 Task의 plan 섹션, 참고 코드 파일, 수정 대상 기존 파일
- 쓰기: Task에 명시된 소스 코드 파일만 (`src/main/`)
- 참조 금지: 다른 Task 정보, 사용자 대화, plan의 다른 섹션, `.claude/docs/domains/*`, `src/test/*`

**병렬/순차:**
- 병렬: 서로 다른 파일을 수정하는 Task → 동시 호출
- 순차: Task B가 Task A의 산출물을 import하는 경우 → 선행 Task 완료 후

**다음 단계:** 모든 Task 완료 후 메인 세션이 결과를 모아 Phase 3로 전달

---

### Phase 3: 테스트 작성 (test-writer)

**호출 대상:** `test-writer` (model: **sonnet**) — Agent 도구로 `subagent_type="test-writer"` 지정

**처리:**
1. plan.md의 성공 기준을 읽는다 (테스트해야 할 behavior 추출).
2. implementer가 변경한 파일 목록을 받는다 (테스트 대상 클래스 식별).
3. 각 변경 클래스에 대해 단위 테스트를 작성한다 — `test-writer.md`의 프로젝트 테스트 규칙(TestObjectFactory, BDD 스타일, 네이밍 등)을 그대로 적용.
4. 테스트 파일을 `src/test/java/` 하위 대응 경로에 생성/수정한다.
5. 대상 클래스가 private 메서드만 있거나 테스트 불가 구조면 TODO 주석으로 표기 후 다음 클래스로 진행.

**컨텍스트:**
- 읽기: `.claude/plans/[티켓]/plan.md`, implementer 변경 파일 전체, 기존 테스트 파일들, `src/test/java/com/posicube/robi/g/testutils/TestObjectFactory.java`
- 쓰기: `src/test/java/` 하위 테스트 파일만
- 참조 금지: 사용자 대화, `.claude/docs/domains/*`, `src/main/*` 수정 (읽기만 가능)

**다음 단계:** 메인 세션이 결과를 받아 Phase 4로 전달

---

### Phase 4: 검증 (reviewer)

**호출 대상:** `reviewer` (model: **sonnet**) — Agent 도구로 `subagent_type="reviewer"` 지정

**처리:**
1. plan.md의 성공/실패 기준을 절대 기준으로 사용.
2. implementer가 생성/수정한 코드 파일 + test-writer가 생성/수정한 테스트 파일을 모두 읽는다.
3. 성공 기준 검증 + 실패 기준 위반 확인.
4. Trust-5 검증 수행 (테스트 존재 여부와 통과 여부는 **T(Tested)** 항목에서 확인).
5. `./gradlew test` 실행 — test-writer가 작성한 테스트 통과 여부 확인.
6. PASS / FAIL 판정 + FAIL 시 **retry_target** 명시 + 재구현 가이드 작성.

**retry_target 값 (FAIL 시 필수):**
- `implementer` — 소스 코드 자체가 잘못됨 (로직, 패턴, 네이밍)
- `test-writer` — 테스트가 누락되었거나 잘못 작성됨 (커버리지 부족, 잘못된 assertion, 프로젝트 테스트 규칙 위반)
- `implementer+test-writer` — 코드와 테스트 둘 다 문제 (순차 재호출 필요)
- `plan-writer` — 설계가 근본적으로 문제, 같은 plan으로는 해결 불가 (Phase 1부터 재수립)

**컨텍스트:**
- 읽기: plan.md 전체, implementer 산출물, test-writer 산출물, 변경 파일 전체
- 쓰기: (없음 — read-only 검증)
- 참조 금지: 사용자 대화, 이전 재시도의 결과 (현재 시도만), `.claude/docs/domains/*` (reviewer는 코드+테스트만 검증)

**다음 단계:** PASS → Phase 5 / FAIL → §5 재시도 규칙 (retry_target 기반 분기)

---

### Phase 5: 도메인 Spec 동기화 (메인 세션) ← Terminal Action

**전제:** Phase 4에서 PASS 판정을 받은 후에만 실행된다. 코드와 테스트가 모두 검증되지 않은 상태에서는 절대 spec을 건드리지 않는다.

**목적:** spec 문서를 "코드의 영속적 진실"로 유지한다. plan-writer가 이 spec만 읽고도 정확한 plan을 세울 수 있을 정도의 정보 밀도를 보장 → 코드베이스 full-scan 회피 → 토큰↓·설계 품질↑.

**처리:**
1. implementer가 변경한 파일 목록과 plan.md(특히 `## Spec 부재 도메인` 섹션 — Phase 1에서 plan-writer가 남긴 backfill 신호)를 확인한다.
2. §6의 6가지 도메인 변경 기준에 해당하는 변경이 있는지 판단한다.
3. 다음 분기 중 하나로 처리한다 — **spec 문서 존재 여부**가 분기 기준 (도메인이 신규냐 legacy냐가 아님):

   **(분기 A) spec 파일 부재** — Phase 1에서 신호를 남겼거나 메인 세션이 직접 부재 확인:
   - 해당 도메인의 코드베이스를 **full-scan** 한다 (baseline이 없으므로 불가피).
   - `.claude/docs/domains/_TEMPLATE.md` 구조를 그대로 복제해 spec을 **신규 작성** 한다.
   - 도메인 자체가 신규든, 기존 도메인의 legacy backfill이든 동일하게 처리.
   - 작성 직후 이 spec이 차후 갱신의 baseline이 된다.

   **(분기 B) spec 파일 존재** — git history 기반 incremental 갱신:
   - **Step B-1 (현재 커밋 변경분 반영):** implementer가 변경한 파일 목록을 spec과 대조해 §6의 6가지 변경 항목을 갱신.
   - **Step B-2 (누락분 검출 — 다른 팀원 변경 흡수):**
     - spec.md의 마지막 커밋 시점 추출: `git log -1 --format=%H -- .claude/docs/domains/[도메인].md`
     - 그 시점부터 현재까지 도메인 디렉토리 변경 파일 목록: `git log [spec_last_commit]..HEAD --name-only -- src/main/java/.../[도메인]/`
     - 변경 파일들의 diff만 읽어 spec 반영 여부 판단 (full-scan 불필요).
   - 갱신 후 spec.md를 커밋 가능한 상태로 둔다 (커밋 자체는 사용자 책임).

4. 영향받는 도메인이 전혀 없고 spec 부재 신호도 없으면 손대지 않는다.
5. 작성/갱신 후 자가 확인 — `_TEMPLATE.md`의 핵심 항목(엔티티 필드, API 시그니처, 의존 도메인, 핵심 설계 결정, 비즈니스 규칙)이 모두 채워졌는지 점검.

**컨텍스트:**
- 읽기: 변경된 파일들, plan.md, 해당 도메인 spec
- 쓰기: `.claude/docs/domains/[도메인].md`

**다음 단계:** Phase 6 최종 보고

---

### Phase 6: 최종 보고 + 정리 (메인 세션)

**처리:**
1. **빌드 점검:** `./gradlew build` 실행. 실패 시 원인을 분석하고 수정 후 재시도.
2. **Jira 완료 코멘트 push (zebra2 경유 시):** 변경 파일 목록, 테스트 결과, spec 갱신 내역, plan.md 요약을 `[zebra2] 작업 완료` 코멘트로 1회 addCommentToJiraIssue 호출. description은 절대 수정하지 않는다.
3. 사용자에게 보고:
   - 변경 사항 요약 (코드 파일, 테스트 파일 분리)
   - 성공 기준 충족 확인
   - Phase 5에서 갱신한 도메인 spec (있으면 파일 경로 명시)
   - Jira 코멘트 링크 (push한 경우)
   - 후속 작업 제안
4. **plan.md 정리:** 워크플로우가 성공 종료했으므로 `.claude/plans/[티켓]/` 디렉토리를 삭제한다. (실패로 중단된 경우 디버깅을 위해 유지)

**컨텍스트:**
- 읽기: 각 Phase 결과 요약, 변경 파일 목록
- 쓰기: Jira 코멘트(1회, zebra2 경유 시), 사용자 보고 메시지, `.claude/plans/[티켓]/` 삭제

---

## 4. 핸드오프 Contract

| From → To | 전달 내용 |
|----------|----------|
| 메인 세션 → plan-writer | 합의된 요구사항(Goal, 성공/실패 기준, 범위), 티켓 번호 |
| plan-writer → 메인 세션 | `plan.md` 경로(`.claude/plans/[티켓]/plan.md`), Task 목록, 병렬/순차 정보 |
| 메인 세션 → implementer | 해당 Task만 추출한 plan 섹션 + CLAUDE.md 컨벤션 준수 지시 |
| implementer → 메인 세션 | 변경 파일 목록, 구현 결과 요약, TODO/불확실 사항 |
| 메인 세션 → test-writer | plan.md 전체 + implementer가 변경한 파일 목록(테스트 대상) |
| test-writer → 메인 세션 | 작성한 테스트 파일 목록, 테스트 대상 클래스, 커버된 성공 기준, 스킵한 클래스(있으면 사유) |
| 메인 세션 → reviewer | plan.md 전체, implementer 산출물, test-writer 산출물 |
| reviewer → 메인 세션 | PASS/FAIL, Trust-5 결과, **retry_target**(FAIL 시), 재구현 가이드 |
| (Phase 5 내부) | 변경 파일 → 메인 세션이 도메인 spec에 diff 적용 |

---

## 5. 재시도 & 에스컬레이션

### 재시도 한도

| 항목 | 한도 |
|------|------|
| implementer 재호출 | 최대 2회 (총 3번 시도) |
| test-writer 재호출 | 최대 2회 (총 3번 시도) |
| plan-writer 재호출 (plan 재수립) | 최대 2회 |

### 재시도 분기 (reviewer의 retry_target 기반)

Phase 4 reviewer가 FAIL 판정 시 `retry_target` 값에 따라 재실행 대상과 범위가 달라진다:

| retry_target | 재실행 범위 |
|-------------|------------|
| `implementer` | Phase 2 implementer만 재호출 (피드백 포함) → Phase 4 reviewer 재호출. Phase 3 test-writer는 건너뛴다 (기존 테스트 유지) |
| `test-writer` | Phase 3 test-writer만 재호출 (피드백 포함) → Phase 4 reviewer 재호출 |
| `implementer+test-writer` | Phase 2 → Phase 3 → Phase 4 전체 재실행 (implementer가 코드를 바꿨으므로 테스트도 다시 작성) |
| `plan-writer` | Phase 1 plan-writer 재호출 → Phase 2부터 전체 재실행 |

### 자동 재시도 원칙

- FAIL 시 사용자에게 묻지 않고 재시도 (한도 내에서)
- 한도 초과 시에만 사용자 개입

### 에스컬레이션 보고 형식

- 실패한 Phase
- 시도한 접근법 (각 시도별)
- reviewer 피드백 및 retry_target 이력
- 막힌 지점에 대한 가설

### 실패 시 plan.md 처리

- 재시도 중: plan.md 유지 (재사용)
- 에스컬레이션 후: plan.md **유지** (사용자 디버깅용)
- 성공 종료 시: plan.md 삭제 (Phase 6에서)

---

## 6. 도메인 Spec 책임 매트릭스

### "도메인 변경"의 정의

다음 중 하나라도 해당하면 도메인 변경으로 간주한다:

1. **구조적 변경** — 도메인 엔티티 필드 추가/삭제/타입 변경
2. **API 계약 변경** — Controller 경로/메서드/요청·응답 DTO 변경
3. **의존성 변경** — 다른 도메인에 대한 의존 추가/제거
4. **비즈니스 규칙 변경** — 핵심 상수, 정책 분기 로직 변경
5. **이벤트/비동기 계약 변경** — 도메인 간 이벤트 페이로드, 알림 구조, 메시지 포맷 변경
6. **에러 계약 변경** — 에러 코드 체계, 예외 타입, 에러 응답 DTO 변경

> 인프라/설정 변경(MongoDB 컬렉션명, 인덱스, Redis 키, 환경변수)은 도메인 spec이 아니라 별도 위치(향후 `infra.md`)에서 관리한다.

### 책임 분배

| 단계 | 책임 |
|------|------|
| **plan-writer (Phase 1)** | 영향받는 도메인의 spec을 탐색 전 먼저 **읽기만** 한다. spec 파일을 수정하지 않는다. plan에 spec 갱신 Task를 포함하지 않는다. **spec 파일이 부재하면** plan.md 상단에 `## Spec 부재 도메인` 섹션을 추가해 도메인명과 사유(신규/backfill)를 기록한다 (Phase 5 backfill 신호). |
| **implementer (Phase 2)** | 소스 코드 파일만 수정. spec 파일/테스트 파일은 절대 건드리지 않는다. |
| **test-writer (Phase 3)** | 테스트 파일만 생성/수정. spec 파일/소스 코드는 절대 건드리지 않는다. |
| **reviewer (Phase 4)** | 코드 + 테스트만 검증. spec 동기화 여부는 검증 범위 밖. |
| **메인 세션 (Phase 5)** | **도메인 spec 갱신의 단일 책임자.** Phase 4 PASS 후 위 6가지 기준으로 변경을 감지하고, 해당 도메인 spec 파일을 갱신한다. 신규 도메인이면 신규 작성. |

### 도메인 spec 표준 템플릿

`.claude/docs/domains/_TEMPLATE.md`를 표준 템플릿으로 한다. spec 신규 작성/backfill 시 이 구조를 그대로 복제해 채운다. 포함 항목:

- 도메인 한 줄 정의 (책임/저장소/aggregate)
- 패키지 구조 (디렉토리 트리 + 파일별 한 줄 설명)
- API 표 (메서드/경로/설명 — 큰 도메인은 컨트롤러별 sub-section)
- 의존 도메인 표 (호출 방향 + 역방향 의존 노트)
- 핵심 설계 결정 (결정/이유 — **trade-off, 거부된 대안, 과거 incident** 보존)
- 비즈니스 규칙 (상수명·정책·검증 순서)
- 주의 사항 (선택 — 함정, deprecated 예정 항목)

> 참고 spec: `chatnotification.md` (소형 도메인 예시), `aiproject.md` (단일 aggregate 예시), `aiagent.md` (대형 도메인 + 서브 컨트롤러 예시).

### MECE 검증

위 6가지 분류는 초기 정의이며, 최초 6주 운영 후 실제 사용 데이터로 회고한다. 매핑 안 되는 변경 케이스가 발견되면 카테고리를 추가한다.

---

## 7. 사용자 개입 포인트

워크플로우는 인간 개입을 최소화한다. **다음 4가지 경우에만** 사용자에게 묻는다:

1. **Phase 0 요구사항 합의** — Goal, 성공/실패 기준, 범위. 확인 항목은 **반드시 `AskUserQuestion` 도구** 사용 (텍스트 질문 금지)
2. **보안 결정** — 인증/인가 정책, 데이터 접근 범위, 권한 우회 가능성
3. **정책 결정** — 도메인 간 책임 분할, 아키텍처 방향(DDD 레이어), 트레이드오프 선택
4. **에스컬레이션** — implementer/test-writer 재시도 2회 + plan 재수립 2회 모두 실패한 경우

### 자동 진행되는 항목 (사용자 개입 없음)

- **Phase 1 완료 후 plan.md 사용자 사전 승인 — 생략**. Phase 0 합의를 plan-writer가 충실히 반영했다고 가정. plan의 구조적 오류는 Phase 4 reviewer의 `retry_target=plan-writer`로 자동 보정.
- Phase 2 각 Task 시작 전 확인
- Phase 3 test-writer 시작 전 확인
- Phase 4 FAIL 시 재시도 여부 확인 (한도 내 자동 재시도)
- Phase 5 도메인 spec 갱신 (메인 세션이 §6 6가지 기준으로 자동 판단)
- Phase 6 plan.md 삭제
- 병렬/순차 호출 결정
- plan-writer 재호출 시 전체 plan 재전달 — **생략**. 경로 + delta(변경 섹션·이유)만 전달 (토큰 절감).

### 예외: plan-writer가 스스로 플래그한 불명확 항목

plan-writer가 plan.md 내부에 `[질문 #N]` 표식을 남긴 경우에만 메인 세션이 해당 항목을 `AskUserQuestion` 으로 확인한다. 그 외에는 자동 전이.

**호출 시점**: plan-writer 종료 직후, Phase 2 implementer 진입 **전**. implementer/test-writer/reviewer 단계에서는 추가 질문 금지 — 그 단계의 의문은 plan 부실로 간주하고 reviewer의 `retry_target=plan-writer` 로 처리한다.

#### AskUserQuestion 사용 규칙 (필수)

`AskUserQuestion` 을 호출할 때는 다음 형식을 반드시 지킨다:

1. **옵션 2~3개 + 마지막 "더 논의하기" 옵션 1개** = 총 3~4개.
2. 각 옵션의 `description` 에 **추천 사유** 명시 (한 문장).
3. 추천 옵션 1개의 `description` 첫 단어를 `[추천]` 으로 시작.
4. 마지막 옵션은 항상:
   - label: `이것에 대해 더 논의하기`
   - description: `워크플로우를 일시 중단하고 자유 대화로 결정한다.`

#### "더 논의하기" 선택 시 동작

- 워크플로우를 **일시 중단** (Phase 2 진입 X).
- 메인 세션은 자유 대화 모드로 전환하여 사용자와 해당 결정을 논의.
- 재개는 사용자가 **명시적으로 트리거** 한 경우에만 (예: `/lets-get-to-work continue [티켓]` 또는 "이어서 진행해").
- 자동 재개 X.

#### 사용자 답변 처리

- 사용자가 옵션 중 하나를 선택하면 메인 세션이 plan.md 의 표식 자리에 **답변을 직접 채워 넣는다** (plan-writer 재호출 X).
- **예외**: 답변이 plan 본문 다수를 흔드는 큰 결정이면 plan-writer 재호출 (메인 세션 판단). 단, delta(변경 섹션·이유)만 전달.

#### 남용 방지

- 명백한 결정·기존 패턴 답습으로 충분한 경우는 묻지 않는다 (plan-writer가 자체 결정).
- 한 plan 당 표식 권장 상한: **3개**. 그 이상이면 요구사항 자체가 부실 → 메인 세션이 사용자에게 "추가 논의 필요" 보고.

---

## 8. 로깅 & 추적

### Phase 시작/종료 로그

각 Phase는 메인 세션에 한 줄 로그를 남긴다:

```
[Phase 0 시작] 요구사항 합의 (RGD-XXXX: 작업 제목)
[Phase 0 완료] 성공 기준 N개, 실패 기준 M개 합의

[Phase 1 시작] plan-writer 호출
[Phase 1 완료] plan.md 생성 (.claude/plans/RGD-XXXX/), Task K개

[Phase 2 시작] implementer 호출 (Task 1, 2 병렬)
[Phase 2 완료] 변경 파일 N개

[Phase 3 시작] test-writer 호출 (변경 파일 N개 기반)
[Phase 3 완료] 테스트 파일 M개 작성

[Phase 4 시작] reviewer 호출
[Phase 4 완료] PASS (또는 FAIL, retry_target: [대상]) — 사유 요약

[Phase 5 시작] 도메인 spec 동기화 판단
[Phase 5 완료] spec 갱신: [도메인명].md (또는: 도메인 변경 없음 — 갱신 스킵)

[Phase 6 완료] 최종 보고 + .claude/plans/RGD-XXXX/ 삭제
```

### 핸드오프 페이로드 요약

각 핸드오프 시 무엇을 넘겼는지 한 줄 요약을 남긴다.
예: `[핸드오프 P2→P3] test-writer에 변경 파일 3개 전달 (ChatNotificationService, ChatNotificationResponse, MarkChatLogsAsSeenRequest)`

### 실패 시 보고

FAIL 또는 에스컬레이션 시 다음을 남긴다:
- 어느 Phase에서 실패
- 무엇이 실패했는지 (성공 기준 위반, Trust-5 위반, 테스트 실패, retry_target 등)
- 시도 횟수 (Phase별 재시도 카운트)
- plan.md 위치 (유지된 경우 — 디버깅용)
