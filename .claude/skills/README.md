# /lets-get-to-work — 멀티에이전트 워크플로우 기술 문서

> 대화에서 합의된 요구사항을 입력으로 받아 `plan-writer → implementer → test-writer → reviewer → spec 동기화` 파이프라인을 자동 실행하는 슬래시 커맨드. 메인 세션이 오케스트레이터 역할을 하고, 각 Phase는 전용 서브에이전트가 격리된 컨텍스트에서 책임을 처리한다.

---

## 1. 핵심 기능 — 도메인 spec 동기화로 context를 정적 문서로 유지

이 워크플로우의 **존재 이유는 도메인 spec(`.claude/docs/domains/<도메인>.md`)을 코드의 영속적 진실로 유지하는 것**이다. 다른 모든 단계(plan, implement, test, review)는 결국 spec을 정확하게 갱신하기 위한 선행 게이트다.

### 1.1 왜 spec이 핵심인가

LLM 기반 개발 흐름의 가장 큰 비용은 **매번 코드베이스를 full-scan 해야 한다는 점**이다. 같은 도메인을 두 번째로 건드릴 때도 plan-writer 가 처음부터 Controller → Service → DataAdapter → Repository 트리를 다시 읽으면, 토큰은 누적되고 같은 추론을 반복한다.

`/lets-get-to-work` 는 이 비용을 spec 문서로 **고정한다**:

- **읽기 단계 (Phase 1)**: plan-writer 는 도메인 spec을 **탐색 전에 먼저** 읽는다. spec이 정확하고 최신이면 코드 full-scan 없이 plan 을 세울 수 있다 → 토큰 ↓, 정확성 ↑.
- **쓰기 단계 (Phase 5)**: 코드와 테스트가 PASS 된 후 메인 세션이 spec을 갱신한다. 이번 변경의 결과를 다음 작업의 baseline 으로 박아둔다 → 다음 plan-writer 호출이 더 가벼워진다.

이 사이클이 반복되면 도메인을 만질수록 spec이 두꺼워지고, 새 작업의 진입 비용이 단조 감소한다. **워크플로우 전체가 spec 자산을 누적하는 도구**다.

### 1.2 영속적 진실 vs 일회성 지시서

| 산출물 | 위치 | 수명 | 역할 |
|--------|------|------|------|
| **도메인 spec** | `.claude/docs/domains/<도메인>.md` | **영구** | 검증된 코드 상태의 정의 — plan-writer 1차 참조원 |
| `requirements.md` | `.claude/plans/<식별자>/` | 워크플로우 1회 | 사용자 합의의 충실한 정리본 |
| `plan.md` | `.claude/plans/<식별자>/` | 워크플로우 1회 | implementer 용 일회성 실행 지시서 |

**plan.md 는 워크플로우 종료 시 디렉토리째 삭제된다.** 영속적으로 남는 건 spec 뿐이다 (Phase 6).

### 1.3 spec 갱신은 Terminal Action

> 코드와 테스트가 reviewer PASS를 받기 **전에** 는 절대 spec을 건드리지 않는다.

근거가 검증되지 않은 상태에서 spec을 갱신하면, spec 자체가 오염되어 다음 plan-writer 가 잘못된 baseline 위에서 plan을 세운다. 따라서:

- plan-writer (Phase 1): spec **읽기만** — 부재 시 `## Spec 부재 도메인` 섹션으로 backfill 신호만 남김
- implementer (Phase 2), test-writer (Phase 3), reviewer (Phase 4): spec 파일에 **절대 손대지 않음**
- 메인 세션 (Phase 5): PASS 확인 후 **단독으로** 갱신 — backfill 신호가 있으면 `_TEMPLATE.md` 기반 신규 작성, spec 존재 시 git history 기반 incremental 갱신

### 1.4 도메인 변경 6가지 기준 (Phase 5 트리거)

다음 중 하나라도 해당하면 spec 갱신 대상이다:

1. 구조적 변경 — 도메인 엔티티 필드 추가/삭제/타입 변경
2. API 계약 변경 — Controller 경로/메서드/요청·응답 DTO 변경
3. 의존성 변경 — 다른 도메인에 대한 의존 추가/제거
4. 비즈니스 규칙 변경 — 핵심 상수, 정책 분기 로직 변경
5. 이벤트/비동기 계약 변경 — 도메인 간 이벤트 페이로드, 알림 구조, 메시지 포맷 변경
6. 에러 계약 변경 — 에러 코드 체계, 예외 타입, 에러 응답 DTO 변경

> 인프라/설정 변경(MongoDB 컬렉션명, 인덱스, Redis 키, 환경변수)은 도메인 spec이 아니라 별도 위치(향후 `infra.md`)에서 관리한다.

---

## 2. /lets-get-to-work 개요

### 2.1 진입 경로 비교

| 슬래시 커맨드 | 입력 소스 | Phase 0 처리 | 적합한 상황 |
|-------------|----------|------------|-----------|
| `/zebra2` | Jira 티켓 | `zebra2-requirements` 가 티켓 본문에서 추출 | 티켓이 명확히 정의된 정식 작업 |
| **`/lets-get-to-work`** | **현재 대화 컨텍스트** | **이미 합의된 것으로 간주, 메인 세션이 정리** | **자유 논의 후 바로 구현 진입** |

두 커맨드 모두 Phase 1 이후의 파이프라인은 동일하다. 다른 점은 요구사항을 어디서 가져오느냐 뿐.

### 2.2 입력 인자

```
/lets-get-to-work [티켓번호?]
```

- **티켓 번호 (선택)**: `RGD-1234` 같은 형식. 명시적으로 전달된 경우에만 사용. 대화에서 우연히 등장한 패턴은 자동 추출하지 않는다.
- 인자가 없으면 대화 요약을 kebab-case 슬러그로 변환해 식별자로 사용 (예: `aitool-partial-update`).

### 2.3 작업 식별자와 산출물 위치

```
.claude/plans/<식별자>/
├── requirements.md   # Phase 0 산출물 (대화 정리본)
└── plan.md           # Phase 1 산출물 (Task 분할된 실행 지시서)
```

식별자는 티켓 번호 또는 슬러그. 워크플로우 성공 종료 시 이 디렉토리는 삭제된다.

### 2.4 Jira 코멘트 정책

`/lets-get-to-work` 는 대화 컨텍스트 기반이므로 **Jira 에 자동 코멘트하지 않는다.** (`zebra2-finish` 가 완료 코멘트를 push 하는 것과 다른 점.) 사용자가 명시적으로 요청할 때만 push.

---

## 3. 파이프라인 개요

```
[Phase 0] 메인 세션: 대화 → requirements.md 추출
   ↓
[Phase 1] plan-writer: spec 읽기 + 코드 탐색 + plan.md 작성
   ↓
[Phase 2] implementer (병렬/순차): Task별 코드 변환
   ↓
[Phase 3] test-writer: 변경 클래스에 대한 테스트 작성
   ↓
[Phase 4] reviewer: Trust-5 + 성공 기준 + 테스트 통과 검증
   ↓ (FAIL 시 retry_target 에 따라 분기)
[Phase 5] 메인 세션: 도메인 spec 동기화  ← Terminal Action
   ↓
[Phase 6] 메인 세션: 빌드 점검 + 보고 + plan/ 디렉토리 삭제
```

### 3.1 사용자 개입 포인트 (4개로 한정)

워크플로우는 휴면 시간을 최소화한다. 다음 4개 시점에만 사용자에게 묻는다:

1. **브랜치 결정** (Phase 0 직전, 무조건 1회) — `AskUserQuestion` 으로 현재 브랜치 유지/신규 생성 선택
2. **plan-writer 의 `[질문 #N]` 일괄 질문** (Phase 1 종료 직후) — 표식이 있을 때만, 한 번에 묶어서 질문
3. **보안 결정** — 인증/인가, 권한 우회 가능성에 대한 결정 필요 시
4. **에스컬레이션** — 재시도 한도 초과 시

**자동 진행되는 항목**: Phase 1 plan.md 사전 승인 생략, Phase 2/3/4 진입 전 확인 생략, FAIL 시 한도 내 자동 재시도, Phase 5 spec 갱신 자동 판단, Phase 6 디렉토리 삭제.

---

## 4. Phase별 상세

### Phase 0 — 요구사항 추출 (메인 세션)

대화에서 다음 항목을 추출하여 `.claude/plans/<식별자>/requirements.md` 에 저장한다:

- **Goal** — 비즈니스/도메인 맥락
- **아키텍처 방향** — 어느 레이어에 어떤 책임
- **보안 고려사항** — 인증/인가, 데이터 접근 범위
- **성공 기준** — 검증 가능한 구체적 조건
- **실패 기준** — 절대 하지 않을 것
- **범위 제한** — 이번 작업 외 사항
- **대화 요약** — 합의에 이르기까지의 핵심 결정

#### 절대 규칙: 합의되지 않은 정책 추가 금지

requirements.md 는 **대화 트랜스크립트의 충실한 정리본**이다. 메인 세션이 "이렇게 하면 좋을 것 같다"고 떠올린 정책 fallback 을 임의로 채워 넣지 않는다. 추가하면 plan → implement → 코드까지 흘러가서 사용자가 합의하지 않은 동작이 production 에 박힌다.

다음 "정책 냄새" 항목이 떠오르면 멈추고 사용자에게 질문한다:

- null / 빈 값 처리 정책 (`null → ""`, `null → 기본값`)
- 누락 데이터 fallback (조회 실패 시 빈 문자열 / 빈 리스트 / throw)
- 에러 처리 분기 (일부 실패 시 부분 성공 vs 전체 실패)
- 재시도/타임아웃/캐시 정책
- 권한·인가 정책의 미합의 부분
- 성능 임계값 (batch 크기, 페이지 크기)

논의되지 않은 항목은 `(논의되지 않음)` 으로 표기한다 — Phase 1 plan-writer 가 사전 검증에서 이 마커를 발견하면 plan 작성을 중단하고 메인 세션에 일괄 질문을 요청한다 (이중 방어).

### Phase 1 — 계획 수립 (plan-writer)

자세한 사항은 §5.1 에이전트 카탈로그 참조. 산출물은 `.claude/plans/<식별자>/plan.md`.

핵심:
- 도메인 spec 부재 시 `## Spec 부재 도메인` 섹션으로 Phase 5 backfill 신호 남김
- API 계약 변경 / N-requirement(반드시 유지) 섹션 명시
- Task 분할 계약 (haiku 컨텍스트 보호: 단일 파일·참고 코드 ≤1·신규 메서드 ≤3·로직 ≤5단계)
- 불명확 항목은 `[질문 #N]` 표식으로 비워둠 (plan 당 최대 3개)

### Phase 2 — 구현 (implementer, 병렬/순차)

plan.md 의 Task 별로 implementer 를 호출한다. 서로 다른 파일을 만지는 Task 는 단일 메시지에 묶어 병렬 호출한다. import 의존이 있는 Task 는 순차.

Task 마다 plan-writer 가 `모델: haiku` 또는 `모델: sonnet` 을 명시했으므로 메인 세션이 호출 시 `model` 파라미터로 전달한다.

### Phase 3 — 테스트 작성 (test-writer)

implementer 가 변경한 파일 목록을 받아 각 클래스에 단위 테스트를 작성한다. 프로젝트 규칙(TestObjectFactory, BDD 스타일, 네이밍 컨벤션)을 그대로 적용.

### Phase 4 — 검증 (reviewer)

PASS / FAIL 판정 + FAIL 시 `retry_target` 명시. 자세한 사항은 §5.4.

### Phase 5 — 도메인 Spec 동기화 (메인 세션, Terminal Action)

§1 핵심 기능 참조. **분기 기준은 spec 파일 존재 여부** (도메인이 신규냐 legacy 냐가 아님):

#### 분기 A: spec 파일 부재 — 신규 작성

- baseline 이 없으므로 코드베이스 **full-scan 불가피**
- `.claude/docs/domains/_TEMPLATE.md` 구조 그대로 복제 → 신규 작성
- 도메인 자체가 신규든, 기존 도메인 backfill 이든 동일 처리
- 작성 직후 이 spec 이 차후 갱신의 baseline

#### 분기 B: spec 파일 존재 — Incremental 갱신

git history 기반으로 full-scan 회피:

```bash
# Step B-1: 현재 커밋 변경분 반영
# implementer 변경 파일을 spec 과 대조해 §1.4 6가지 항목 갱신

# Step B-2: 누락분 검출 (다른 팀원 변경 흡수)
git log -1 --format=%H -- .claude/docs/domains/<도메인>.md   # spec 마지막 커밋
git log <spec_last_commit>..HEAD --name-only -- src/main/java/.../<도메인>/
# 변경 파일들의 diff 만 읽어 spec 반영 여부 판단
```

자가 확인 — `_TEMPLATE.md` 의 핵심 항목(엔티티 필드, API 시그니처, 의존 도메인, 핵심 설계 결정, 비즈니스 규칙)이 모두 채워졌는지 점검.

### Phase 6 — 최종 보고 + 정리 (메인 세션)

1. `./gradlew build` 실행. 실패 시 원인 분석 후 수정 → 재시도.
2. 사용자에게 보고 — 변경 사항 요약, 성공 기준 충족 확인, 갱신된 spec 경로, 후속 작업 제안.
3. `.claude/plans/<식별자>/` 디렉토리 삭제 (성공 종료 시에만. 실패로 중단된 경우 디버깅용으로 유지).

---

## 5. 에이전트 카탈로그

| 에이전트 | Phase | 모델 (기본) | 도구 | permissionMode |
|----------|-------|------------|------|----------------|
| `plan-writer` | 1 | **opus** | Read, Grep, Glob, Bash, Agent(Explore) | plan |
| `implementer` | 2 | **haiku** (Task 별 sonnet 오버라이드 가능) | Read, Write, Edit, Bash, Grep, Glob | acceptEdits |
| `test-writer` | 3 | **sonnet** | Read, Edit, Write, Glob, Grep, Bash | — |
| `reviewer` | 4 | **sonnet** | Read, Grep, Glob, Bash | plan |

각 에이전트는 격리된 컨텍스트로 호출되며 정의 파일은 `~/.claude/agents/<이름>.md` 에 있다. 메인 세션은 **Agent 도구**로 `subagent_type` 파라미터를 지정해 호출한다.

### 5.1 plan-writer

**역할**: 합의된 요구사항을 받아 코드베이스를 탐색하고 implementer 가 판단 없이 코드로 옮길 수 있는 상세 계획을 작성한다.

**Waterfall 절차**:

1. **사전 검증 (1.0)** — requirements.md 에서 `(논의되지 않음)` 마커, 정책 냄새, 합의 근거 누락 시 즉시 중단 및 메인 세션에 보고. plan.md 를 만들지 않는다.
2. **요구사항 분석 (1.1)** — P-requirement(구현 항목) + N-requirement(반드시 유지) 추출, API 계약 변경 식별.
3. **코드베이스 탐색** — `Explore` 서브에이전트에 위임 (직접 파일 읽기 X, 토큰 절약). 도메인 spec 을 탐색 전에 먼저 읽는다.
4. **아키텍처 설계** — 클래스 목록, 레이어 의존성, 기존 패턴 일관성.
5. **Task 분할 계약** — haiku 컨텍스트 보호:
   - 단일 파일 수정/생성 원칙
   - 참고 코드 ≤ 1개
   - 신규 메서드 ≤ 3개
   - 구현 로직 ≤ 5단계
   - 초과 시 분할 또는 sonnet 상향
6. **상세 설계** — Task 별로 모델/대상 파일/메서드 시그니처/구현 로직 단계/참고 코드/성공 기준/실패 기준 명시.
7. **Task 순서 결정** — 의존성 기반 병렬/순차 구분.

**핵심 산출물**: `.claude/plans/<식별자>/plan.md`

**plan.md 표준 섹션**:
- `## Spec 부재 도메인` (해당 시) — Phase 5 backfill 신호
- `## API 계약 변경` (해당 시) — 엔드포인트/메서드/DTO 변경 명시
- `## Task 목록` — Task 별 상세 설계
- `## Task 순서` — 병렬/순차 구분
- `## 반드시 유지 (N-requirement)` — reviewer 가 코드에서 직접 확인할 항목
- `## 하지 않는 것 (범위 외)`

**불명확 항목 표식**: `[질문 #N]` 으로 plan.md 본문에 남기고 자리를 비워둠. 메인 세션이 종료 후 표식을 모아 `AskUserQuestion` 으로 일괄 질문. plan 당 최대 3개.

**경계**:
- Always: Explore 위임, 도메인 문서 먼저 읽기, plan.md 저장, spec 부재 신호 남기기
- Never: 도메인 spec 파일 수정, plan 에 spec 갱신 Task 포함, plan 에 테스트 작성 Task 포함, 사용자와 직접 대화, 코드 직접 구현, `[질문]` 표식 자리를 추측으로 메우기

### 5.2 implementer

**역할**: 전달받은 Task 계획을 그대로 코드로 변환한다. **설계 판단을 하지 않는다.**

**핵심 원칙**:

1. **계획을 그대로 따른다** — 메서드 시그니처, 구현 로직 순서, 참고 코드 패턴 그대로. 계획에 없는 메서드/클래스/필드를 임의로 추가하지 않는다.
2. **DDD 기반 구현** — 참고 코드의 기존 패턴과 동일한 스타일. 네이밍/예외 처리/로깅 일치.
3. **모르면 구현하지 않는다** — 모호한 부분은 `// TODO: ...` 주석으로 남기고 나머지를 먼저 구현. 추측 금지.
4. **기존 경로 임의 삭제 금지** — plan 에 명시적으로 제거하라고 적혀있지 않은 분기/메서드/조건문은 그대로 유지. plan 의 N-requirement 섹션 항목은 코드에서 살아있어야 함 (구현 후 자가 확인).
5. **API 계약 정확히 따르기** — plan.md "API 계약 변경" 섹션의 엔드포인트/HTTP 메서드/메서드명을 한 글자도 다르지 않게 일치.

**경계**:
- Read/Write 가능: Task 의 plan 섹션, 참고 코드, 수정 대상 파일, `src/main/`
- **절대 금지**: 도메인 spec 파일(`.claude/docs/domains/*`) 수정 (Phase 5 책임), 테스트 파일(`src/test/*`) 생성/수정 (Phase 3 책임), 다른 Task 정보 참조, 사용자 대화 참조, 테스트 실행 (Phase 4 책임)

**모델 정책**: 기본 haiku. plan.md Task 마다 plan-writer 가 `모델: sonnet` 으로 명시한 경우 그것에 따라 sonnet 으로 호출. 컨텍스트가 얕은 haiku 의 한계를 Task 분할 계약(§5.1 5번)으로 보장.

### 5.3 test-writer

**역할**: implementer 가 변경한 클래스에 대해 프로젝트 규칙에 맞는 단위 테스트를 작성한다.

**작업 순서**:
1. 대상 클래스 찾기 — `Test` 접미어 제거해 소스 클래스 식별. `src/main/java/com/posicube/robi/g/` 하위 검색.
2. 대상 클래스 분석 — public 메서드, 의존성, Entity/DTO 파악.
3. **TestObjectFactory 확인** — `src/test/java/com/posicube/robi/g/testutils/TestObjectFactory.java` 읽고 사용 가능한 빌더 확인.
4. 테스트 작성 — `src/test/java/` 하위 대응 경로에 생성/수정.

**프로젝트 테스트 규칙** (요약):

- **클래스 구조**: `@DisplayName("ClassName 테스트") @ExtendWith(MockitoExtension.class) class ClassNameTest`
- **Nested 패턴**: 메서드별 `@Nested class MethodNameTest`, `@DisplayName("methodName 메서드는")`
- **메서드명**: 영어 `action_whenCondition` (예: `throwsException_whenIdNotFound`)
- **메서드 `@DisplayName`**: 한글 시나리오 (예: `"존재하지 않는 ID면 예외를 던진다"`)
- **객체 생성**:
  - Entity → 반드시 TestObjectFactory 사용. private 헬퍼 절대 금지.
  - DTO/Record/SearchQuery → 실제 생성자 사용. `mock()` 절대 금지.
  - Page → `PageImpl` 에 실제 데이터 포함. 빈 리스트 금지, `@SuppressWarnings("unchecked")` 금지.
  - Response DTO → TestObjectFactory 에 없을 때만 private 헬퍼 허용.
- **Mocking BDD 스타일**: 반환값 있는 메서드는 `given().willReturn()`, void 는 `willDoNothing().given()`. 반환값 메서드에 `willDoNothing()` 절대 금지.
- **검증 분기**: Command(save/delete/publish) → `then().should()` 검증. 조건부 호출 → `should(never()/times(N))`. Query 메서드 → 결과 assertion 으로 충분 (`should()` 추가 금지).
- **메서드 순서**: 성공 케이스 먼저 → 실패/예외 케이스 → private 헬퍼.

**경계**:
- Read 가능: plan.md 전체, implementer 변경 파일 전체, 기존 테스트 파일, TestObjectFactory
- **절대 금지**: `.claude/docs/domains/*` 참조, `src/main/*` 수정 (읽기만), 사용자 대화 참조

### 5.4 reviewer

**역할**: implementer 코드 + test-writer 테스트를 plan.md 기준으로 검증하고 PASS/FAIL 판정. FAIL 시 `retry_target` 명시.

**리뷰 프로세스**:

1. **requirements.md 대조** — 요구사항의 모든 항목(특히 API 계약, N-requirement)이 plan.md 에 반영됐는지 확인. 누락 시 `retry_target=plan-writer`.
2. plan.md 의 성공/실패 기준 + API 계약 변경 + N-requirement 섹션 확인.
3. implementer 산출물(`src/main/*`) + test-writer 산출물(`src/test/*`) 모두 읽기.
4. 성공 기준 하나씩 검증.
5. **API 계약 검증** — 엔드포인트/HTTP 메서드/메서드명이 코드와 정확히 일치하는지. 불일치 = Unified 치명적 위반 → FAIL.
6. **N-requirement 검증** — 반드시 유지 항목이 코드에서 살아있는지. 제거된 분기 있으면 FAIL.
7. 실패 기준 위반 확인.
8. **Trust-5 검증**:
   - **T(Tested)** — 핵심 경로 테스트 존재, 통과 여부, 기존 테스트 깨지지 않음, 프로젝트 규칙 준수
   - **R(Readable)** — 메서드/변수명이 의도를 드러내고 plan.md 네이밍과 일치
   - **U(Unified)** — 기존 패턴/참고 코드 스타일 일치, DDD 레이어 책임 준수
   - **S(Secured)** — PermissionAdapter 사용, 데이터 접근 범위 적절, 인가 우회 가능성 없음
9. **`./gradlew test` 실행** — test-writer 작성 테스트 통과 여부.
10. FAIL 시 `retry_target` 판단 (아래 표).

**retry_target 판단 기준**:

| retry_target | 판단 기준 |
|-------------|----------|
| `implementer` | 소스 코드 자체가 잘못됨: 로직 오류, plan 시그니처 불일치, 패턴/네이밍 위반, 레이어 책임 위반. 테스트는 정상이고 코드만 고치면 해결. |
| `test-writer` | 소스 코드는 정상인데 테스트가 문제: 누락, 잘못된 assertion, TestObjectFactory 미사용, BDD 스타일 위반, 커버리지 부족. 테스트만 고치면 해결. |
| `implementer+test-writer` | 코드와 테스트 둘 다 문제 — 코드를 바꾸면 테스트도 다시 써야 함. 메인 세션이 Phase 2 → Phase 3 순차 재실행. |
| `plan-writer` | 설계가 근본적으로 문제: plan 으로는 성공 기준 충족 불가, 존재하지 않는 클래스/메서드 참조, 아키텍처적 오판. **같은 plan 으로는 해결 불가** — Phase 1 부터 재수립. |

**판정 원칙**:

- **성공 기준이 절대 기준**. 코드가 좋아 보여도 성공 기준 미충족이면 FAIL. 코드가 아쉬워도 성공 기준 모두 충족이면 PASS (개선 제안은 별도).
- **과도한 완벽주의 경계** — Trust-5 보조 항목만으로는 FAIL 안 함. 성공 기준 위반 또는 Trust-5 **치명적** 위반일 때만 FAIL.
- **구체적으로 지적** — "코드가 깔끔하지 않다" 금지. "XxxService.java 42행에서 Repository 를 직접 호출하나, plan 에서는 DataAdapter 를 통하도록 명시" 형식.
- 판정 전 자문: **"이 이슈가 없어도 성공 기준은 충족되는가?"** 예 → 개선 제안, 아니요 → FAIL.

**경계**:
- Read 가능: plan.md, implementer 산출물, test-writer 산출물, 변경 파일 전체
- **쓰기 없음** (read-only 검증)
- 참조 금지: 사용자 대화, 이전 재시도 결과 (현재 시도만), `.claude/docs/domains/*` (코드+테스트만 검증)

---

## 6. 핸드오프 컨트랙트

| From → To | 전달 내용 |
|----------|----------|
| 메인 세션 → plan-writer | requirements.md 경로, 식별자(티켓 또는 슬러그) |
| plan-writer → 메인 세션 | plan.md 경로, Task 목록, 병렬/순차 정보, `[질문 #N]` 표식 (있으면) |
| 메인 세션 → implementer | 해당 Task 만 추출한 plan 섹션 + CLAUDE.md 컨벤션 준수 지시 |
| implementer → 메인 세션 | 변경 파일 목록, 구현 결과 요약, TODO/불확실 사항, API 계약/N-requirement 준수 체크 |
| 메인 세션 → test-writer | plan.md 전체 + implementer 변경 파일 목록 |
| test-writer → 메인 세션 | 작성한 테스트 파일 목록, 대상 클래스, 커버된 성공 기준, 스킵한 클래스(있으면 사유) |
| 메인 세션 → reviewer | plan.md 전체, implementer 산출물, test-writer 산출물 |
| reviewer → 메인 세션 | PASS/FAIL, Trust-5 결과, retry_target(FAIL 시), 재구현 가이드 |
| (Phase 5 내부) | 변경 파일 → 메인 세션이 도메인 spec 에 diff 적용 |

**원칙**:
- 핸드오프 페이로드는 **각 에이전트의 책임 범위 내 정보만** 포함. 메인 세션이 게이트키퍼.
- 사용자 대화 원문은 plan-writer 이후 누구도 받지 않음 (requirements.md/plan.md 만 전달).
- plan-writer 재호출 시 전체 plan 재전달 생략 — 경로 + delta(변경 섹션·이유)만 전달 (토큰 절감).

---

## 7. 재시도 & 에스컬레이션

### 재시도 한도

| 항목 | 한도 |
|------|------|
| implementer 재호출 | 최대 2회 (총 3번 시도) |
| test-writer 재호출 | 최대 2회 (총 3번 시도) |
| plan-writer 재호출 (plan 재수립) | 최대 2회 |

### 재시도 분기 (retry_target 기반)

| retry_target | 재실행 범위 |
|-------------|------------|
| `implementer` | Phase 2 implementer 만 재호출 (피드백 포함) → Phase 4 reviewer 재호출. Phase 3 test-writer 건너뜀. |
| `test-writer` | Phase 3 test-writer 만 재호출 (피드백 포함) → Phase 4 reviewer 재호출. |
| `implementer+test-writer` | Phase 2 → Phase 3 → Phase 4 전체 재실행. |
| `plan-writer` | Phase 1 plan-writer 재호출 → Phase 2 부터 전체 재실행. |

### 자동 재시도 원칙

- FAIL 시 사용자에게 묻지 않고 한도 내 자동 재시도.
- 한도 초과 시에만 사용자 개입(에스컬레이션).
- plan.md 처리: 재시도 중 유지(재사용), 에스컬레이션 후 유지(디버깅), 성공 종료 시 삭제(Phase 6).

### 에스컬레이션 보고 형식

- 실패한 Phase
- 시도한 접근법 (각 시도별)
- reviewer 피드백 및 retry_target 이력
- 막힌 지점에 대한 가설

---

## 8. 사용자 개입 — `AskUserQuestion` 표준

`AskUserQuestion` 호출 시 다음 형식 필수:

1. **옵션 2~3개 + 마지막 "더 논의하기" 옵션 1개** = 총 3~4개.
2. 각 옵션의 `description` 에 **추천 사유** 명시 (한 문장).
3. 추천 옵션 1개의 `description` 첫 단어를 `[추천]` 으로 시작.
4. 마지막 옵션:
   - label: `이것에 대해 더 논의하기`
   - description: `워크플로우를 일시 중단하고 자유 대화로 결정한다.`

### "더 논의하기" 선택 시 동작

- 워크플로우 **일시 중단** (Phase 2 진입 X).
- 메인 세션이 자유 대화 모드로 전환하여 사용자와 결정 논의.
- 재개는 사용자가 **명시적으로 트리거** (`/lets-get-to-work continue [식별자]` 또는 "이어서 진행해").
- 자동 재개 X. 기존 requirements.md + plan.md 재사용.

### 사용자 답변 처리

- 옵션 선택 시 메인 세션이 plan.md 의 표식 자리에 답변을 직접 채워 넣음 (plan-writer 재호출 X).
- **예외**: 답변이 plan 본문 다수를 흔드는 큰 결정이면 plan-writer 재호출 (delta 만 전달).

---

## 9. 로깅 & 추적

각 Phase 는 메인 세션에 한 줄 로그를 남긴다:

```
[/lets-get-to-work 시작] 식별자: <식별자>, 요구사항: .claude/plans/<식별자>/requirements.md → Phase 1 plan-writer 진입

[Phase 1 시작] plan-writer 호출
[Phase 1 완료] plan.md 생성, Task K개

[Phase 2 시작] implementer 호출 (Task 1, 2 병렬)
[Phase 2 완료] 변경 파일 N개

[Phase 3 시작] test-writer 호출 (변경 파일 N개 기반)
[Phase 3 완료] 테스트 파일 M개 작성

[Phase 4 시작] reviewer 호출
[Phase 4 완료] PASS (또는 FAIL, retry_target: <대상>) — 사유 요약

[Phase 5 시작] 도메인 spec 동기화 판단
[Phase 5 완료] spec 갱신: <도메인>.md (또는: 도메인 변경 없음 — 갱신 스킵)

[Phase 6 완료] 최종 보고 + .claude/plans/<식별자>/ 삭제
```

핸드오프 시 한 줄 요약도 남긴다:
```
[핸드오프 P2→P3] test-writer 에 변경 파일 3개 전달 (ChatNotificationService, ChatNotificationResponse, MarkChatLogsAsSeenRequest)
```

---

## 10. 적용 범위

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

> 비적용 대상 작업에 `/lets-get-to-work` 를 호출하면 파이프라인 오버헤드가 작업 자체보다 커진다. 메인 세션이 직접 처리하는 것이 적합하다.

---

## 11. 커밋 단계 — 티켓 prefix

Phase 6 보고 단계 또는 사용자가 `/cpr` / `/push-pr` 등으로 커밋·PR 명령을 트리거할 때:

- `/lets-get-to-work` 인자로 받은 티켓이 있으면 `[RGD-XXXX]` prefix 그대로 사용.
- 티켓이 없으면 **그 시점에 1회만** `AskUserQuestion`:
  - 옵션 1: `티켓 번호 입력` — 사용자가 직접 입력
  - 옵션 2: `티켓 없이 커밋` — prefix 없이 진행 (`<요약>` 형식)
  - 옵션 3: `이것에 대해 더 논의하기`

작업 진입 시점에는 묻지 않는다.

---

## 12. 관련 문서

- `~/.claude/agents/workflow.md` — 멀티에이전트 워크플로우 정의 (이 문서의 원본)
- `~/.claude/agents/plan-writer.md` / `implementer.md` / `test-writer.md` / `reviewer.md` — 각 에이전트 정의
- `~/.claude/skills/lets-get-to-work/SKILL.md` — 슬래시 커맨드 진입 정의
- `.claude/docs/domains/_TEMPLATE.md` — 도메인 spec 표준 템플릿
- `.claude/docs/architecture.md` — DDD 레이어 책임 규칙
- `.claude/docs/code-style.md` — 코드 스타일 컨벤션
