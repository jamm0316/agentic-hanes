---
name: lets-get-to-work
description: 지금까지의 대화에서 합의된 요구사항을 그대로 입력으로 삼아 ~/.claude/agents/workflow.md 멀티에이전트 파이프라인(plan-writer → implementer → test-writer → reviewer → spec 동기화)을 실행한다. 사용자가 충분히 논의를 마친 뒤 /lets-get-to-work 로 호출한다. zebra2가 Jira 티켓 입력용이라면 lets-get-to-work는 대화 컨텍스트 입력용이다.
---

# /lets-get-to-work — 대화 컨텍스트 기반 멀티에이전트 워크플로우

지금까지의 대화에서 합의된 내용을 요구사항으로 추출하여, `~/.claude/agents/workflow.md` 의 파이프라인을 그대로 실행한다.

**Phase 0(요구사항 합의)는 이미 대화로 끝난 것으로 간주하고 건너뛴다.** 합의 내용을 파일로 고정하고 곧장 Phase 1부터 진입한다.

---

## 입력

- `$ARGUMENTS`: 티켓 번호 (예: `RGD-1234`). 선택. 명시적으로 전달된 경우에만 사용.
  - **자동 추출 금지** — 대화에 `RGD-\d+` 가 우연히 등장해도 자동으로 잡지 않는다.
  - **티켓 없이 진행이 기본값.** `$ARGUMENTS` 가 비어있으면 슬러그 기반으로 진행하고 묻지 않는다.
  - 티켓 prefix 가 필요한 시점(커밋 단계)에 1회만 묻는다.

---

## 절차

### 1. 브랜치 결정 (필수, 첫 번째 질문, 예외 없음)

작업 진입 전에 **무조건** 브랜치 전략을 확정한다. 티켓보다 먼저 결정한다 — 어디에 코드를 쌓을지가 implementer 진입의 전제 조건이기 때문.

**Skip 조건 없음.** 대화에서 사용자가 이미 브랜치를 언급했더라도, 워크트리 자동 진입 상태이더라도, 무조건 `AskUserQuestion` 으로 1회 확인한다. 이미 결정된 경우 옵션 1("현재 브랜치 유지")이 자동 추천되므로 비용은 클릭 한 번. 휴리스틱 추론으로 단계를 건너뛰는 것은 금지한다 — 워크트리 자동 생성 브랜치 같은 사각지대를 방지하기 위함.

#### 1.1 현재 브랜치 분류

`git rev-parse --abbrev-ref HEAD` 로 현재 브랜치를 확인하고 다음으로 분류한다:

- **보호 브랜치**: `main`, `master`, `develop`, `release/*`
- **작업 브랜치**: 그 외 모두 (`feature/*`, `fix/*`, `hotfix/*`, `refactor/*` 등)

#### 1.2 추천 브랜치명 생성

대화에서 합의된 작업 내용의 한 줄 요약을 kebab-case 로 변환해 prefix 와 결합한다:

- 형식: `<prefix>/<요구사항-요약>` (예: `feature/aitool-partial-update`)
- **티켓 번호는 포함하지 않는다** — 커밋 메시지에만 사용.
- prefix 추천: 신규 기능 → `feature`, 버그/장애 → `fix` 또는 `hotfix`, 리팩토링 → `refactor`
- 추천 옵션은 prefix 만 다르게 하여 2~3개 생성 (예: `feature/aitool-partial-update`, `fix/aitool-partial-update`)

#### 1.3 분기 처리

**(분기 A) 현재가 보호 브랜치** — 자동으로 새 브랜치 생성 흐름:

`AskUserQuestion` 으로 한 번에 묻는다 (옵션 3~4개):
- 옵션 1: `[추천] feature/<요약>` — root: main
- 옵션 2: `fix/<요약>` (또는 `hotfix/<요약>`) — root: main
- 옵션 3: `직접 입력` — 브랜치명과 root 브랜치 모두 사용자가 입력
- 옵션 4: `이것에 대해 더 논의하기`

**(분기 B) 현재가 작업 브랜치** — 기본은 현재 브랜치 유지:

`AskUserQuestion` 으로 한 번에 묻는다:
- 옵션 1: `[추천] 현재 브랜치 유지: <현재 브랜치명>` — 예: `feature/aitool-abc`
- 옵션 2: `feature/<요약>` (자동 추천) — root: main
- 옵션 3: `직접 입력` — 브랜치명과 root 브랜치 모두 사용자가 입력
- 옵션 4: `이것에 대해 더 논의하기`

#### 1.4 브랜치 생성 실행

새 브랜치를 만드는 경우:

1. **dirty working tree 처리**: 변경사항이 있으면 그대로 새 브랜치로 carry over (git default 동작 — `git checkout -b` 사용). stash 하지 않는다.
2. root 브랜치를 최신화하지 않는다 (사용자가 직접 처리). root 브랜치에서 분기만 한다:
   ```
   git checkout -b <새 브랜치명> <root 브랜치명>
   ```
3. 생성 결과를 한 줄 로그로 남긴다: `[브랜치 생성] <새 브랜치명> (from <root>)`.

현재 브랜치 유지를 선택한 경우 아무 작업도 하지 않는다.

---

### 2. 작업 식별자 확정

산출물 폴더(`.claude/plans/<식별자>/`) 와 시작 로그에 쓸 식별자를 정한다. **이 단계에서 사용자에게 묻지 않는다.**

우선순위:

1. `$ARGUMENTS` 가 비어있지 않으면 그대로 사용 (예: `RGD-1234`).
2. 비어있으면 대화에서 합의된 작업 내용의 한 줄 요약을 kebab-case 슬러그로 변환해 사용 (예: `group-management-datasource-name`). 길이는 최대 60자로 자른다.

티켓 번호 자동 추출(정규식)은 하지 않는다 — 사용자가 명시적으로 인자로 넘겼을 때만 티켓으로 인식한다.

---

### 3. requirements.md 생성

`.claude/plans/<식별자>/requirements.md` 에 다음 항목을 추출하여 저장한다. 추측 금지 — 대화에 있는 내용만 옮긴다. 빠진 항목은 `(논의되지 않음)` 으로 표기.

#### 3.1 절대 규칙 — 합의되지 않은 정책 추가 금지

requirements.md 는 **대화 트랜스크립트의 충실한 정리본**이다. 메인 세션이 "이렇게 하면 좋을 것 같다" 고 떠올린 정책·정책 fallback·에러 처리 방침 등을 **임의로 채워 넣지 않는다.** 추가하면 그대로 plan-writer → implementer → 코드까지 흘러가서 사용자가 합의하지 않은 동작이 production 에 박힌다.

**작성 중 다음과 같은 "정책 냄새" 항목이 떠오르면 멈추고 사용자에게 질문한다** (`AskUserQuestion` 1회):
- null / 빈 값 처리 정책 (예: `null → ""`, `null → 기본값`)
- 누락 데이터 fallback (예: 조회 실패 시 빈 문자열 / 빈 리스트 / throw)
- 에러 처리 분기 (예: 일부 실패 시 부분 성공 vs 전체 실패)
- 재시도/타임아웃/캐시 정책
- 권한·인가 정책의 미합의 부분
- 성능 임계값 (예: batch 크기, 페이지 크기)

질문 형식:
- 옵션마다 trade-off 한 줄
- 마지막 옵션은 항상 "이것에 대해 더 논의하기"
- 답변 받기 전까지 해당 항목은 `(논의되지 않음)` 으로 둔다.

제목 줄 규칙:
- 티켓이 있으면 `# [RGD-XXXX] 요구사항`
- 없으면 `# <한 줄 요약> 요구사항`

```markdown
# <제목> 요구사항

## Goal
(비즈니스/도메인 맥락)

## 아키텍처 방향
(어느 레이어에 어떤 책임)

## 보안 고려사항
(인증/인가, 데이터 접근 범위)

## 성공 기준
- (검증 가능한 구체적 조건)

## 실패 기준
- (절대 하지 않을 것)

## 범위 제한
- (이번 작업 외 사항)

## 대화 요약
(사용자와 합의에 이르기까지의 핵심 결정 — 한두 단락)
```

---

### 4. workflow.md 진입

`~/.claude/agents/workflow.md` 의 **Phase 1부터** 그대로 따른다.

- Phase 1: `plan-writer` 호출 (model: opus, subagent_type=plan-writer). 입력은 위 requirements.md 경로 + 식별자(티켓이 있으면 티켓, 없으면 슬러그).
- Phase 2: plan.md의 Task별로 `implementer` 호출 (병렬/순차 규칙은 plan.md 따름).
- Phase 3: `test-writer` 호출 (변경 파일 목록 전달).
- Phase 4: `reviewer` 호출. FAIL 시 `retry_target` 에 따라 분기 (재시도 한도는 workflow.md §5 준수).
- Phase 5: 메인 세션이 도메인 spec 동기화 (workflow.md §6 6가지 기준).
- Phase 6: 빌드 점검 + 사용자 보고 + `.claude/plans/<식별자>/` 정리.

---

### 5. 커밋 단계 — 티켓 prefix 확인

Phase 6 보고 단계 또는 사용자가 `/cpr`/`/push-pr` 등으로 커밋·PR 명령을 트리거할 때:

- `$ARGUMENTS` 로 받은 티켓이 있으면 그대로 `[RGD-XXXX]` prefix 사용.
- 티켓이 없으면 **그 시점에 1회만** `AskUserQuestion` 으로 묻는다:
  - 옵션 1: `티켓 번호 입력` — 사용자가 직접 입력
  - 옵션 2: `티켓 없이 커밋` — prefix 없이 진행 (`<요약>` 형식)
  - 옵션 3: `이것에 대해 더 논의하기`

작업 진입 시점에는 묻지 않는다.

---

### 6. Jira 코멘트 정책

`/lets-get-to-work` 는 대화 컨텍스트 기반이므로 **Jira에 자동 코멘트하지 않는다.** (zebra2-finish 와 다른 점.) 사용자가 명시적으로 요청하면 그때만 push.

---

## 사용자 개입 최소화

workflow.md §7 의 개입 포인트만 유지:

1. **브랜치 결정** (위 §1) — **무조건 1회 질문**. 첫 번째 질문이자 작업 진입 전 유일한 필수 질문. skip 조건 없음.
2. **plan-writer 종료 직후, `[질문 #N]` 표식 일괄 질문** — `AskUserQuestion` 으로, 옵션마다 추천 사유 명시 + 마지막은 항상 "이것에 대해 더 논의하기" 옵션 (workflow.md §7 참조).
3. 보안 결정 (인가/권한 우회 가능성).
4. 에스컬레이션 (재시도 한도 초과).
5. **커밋 단계 티켓 prefix** (위 §5) — 티켓 없이 시작했고 커밋 시점에 도달했을 때만 1회 질문.

그 외에는 자동 진행. plan.md 사전 승인도 생략. implementer/test-writer/reviewer 단계의 추가 질문 금지. **작업 진입 시점에 티켓 번호를 묻지 않는다** — 슬러그 기반으로 자동 진행.

### "더 논의하기" 선택 시 재개

- 워크플로우 일시 중단 → 자유 대화 모드.
- 재개는 사용자가 명시적으로 트리거: `/lets-get-to-work continue [식별자]` 또는 "이어서 진행해".
- `/lets-get-to-work continue` 호출 시 기존 `requirements.md` + `plan.md` 를 그대로 이어서 사용 (재생성 X).

---

## 시작 로그 형식

진입 시 한 줄 로그 (티켓 유무에 따라 분기):

```
[/lets-get-to-work 시작] 식별자: RGD-XXXX, 요구사항: .claude/plans/RGD-XXXX/requirements.md → Phase 1 plan-writer 진입
```

티켓 없이 시작한 경우:

```
[/lets-get-to-work 시작] 식별자(슬러그): group-management-datasource-name, 요구사항: .claude/plans/group-management-datasource-name/requirements.md → Phase 1 plan-writer 진입
```
