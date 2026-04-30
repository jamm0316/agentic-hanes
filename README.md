# agentic-hanes

> Brownfield 환경을 위한 spec 기반 멀티에이전트 워크플로우. spec 문서를 워크플로우의 일등시민으로 두어 LLM이 코드베이스를 매번 다시 스캔하지 않도록 한다.

---

## 핵심 원칙

> **Sessions are disposable. Specs are permanent.**

세션은 일회성이지만 spec 문서는 누적되는 자산이다. 이 원칙 위에서 워크플로우 전체가 동작한다.

| 산출물 | 위치 | 수명 | 역할 |
|--------|------|------|------|
| 도메인 spec | `.claude/docs/domains/<도메인>.md` | **영구** | 검증된 코드 상태의 정의 |
| `requirements.md` | `.claude/plans/<식별자>/` | 워크플로우 1회 | 사용자 합의 정리본 |
| `plan.md` | `.claude/plans/<식별자>/` | 워크플로우 1회 | implementer용 실행 지시서 |

`requirements.md`와 `plan.md`는 워크플로우 종료 시 디렉토리째 삭제된다. 영속적으로 남는 건 spec뿐이다.

---

## 워크플로우

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
   ↓ (FAIL 시 retry_target에 따라 분기)
[Phase 5] 메인 세션: 도메인 spec 동기화 ← Terminal Action
   ↓
[Phase 6] 메인 세션: 빌드 점검 + 보고 + plan/ 디렉토리 삭제
```

**진입점**: `/lets-get-to-work [티켓번호?]` — 대화 컨텍스트에서 합의된 요구사항을 그대로 입력으로 받아 Phase 0부터 자동 실행한다.

---

## 에이전트

| 에이전트 | Phase | 모델 | 역할 |
|---------|-------|------|------|
| **plan-writer** | 1 | opus | spec과 코드를 탐색해 implementer가 판단 없이 따를 수 있는 상세 계획 작성 |
| **implementer** | 2 | haiku (Task별 sonnet 오버라이드 가능) | 계획을 그대로 코드로 변환. 설계 판단 없음 |
| **test-writer** | 3 | sonnet | 변경 클래스에 대한 단위 테스트 작성 (TestObjectFactory + BDD 스타일) |
| **reviewer** | 4 | sonnet | 성공 기준 + Trust-5 검증, FAIL 시 `retry_target` 명시 |

각 에이전트는 격리된 컨텍스트에서 호출되며, 메인 세션이 오케스트레이터 역할을 한다.

### Trust-5 품질 프레임워크

reviewer가 검증하는 4개 항목 (Trackable은 사람의 책임):

- **T**ested — 테스트 통과, 기존 테스트 깨지지 않음
- **R**eadable — 네이밍이 의도를 드러내고 plan과 일치
- **U**nified — 기존 코드베이스 패턴과 일치, DDD 레이어 준수
- **S**ecured — 권한 체크, 데이터 접근 범위 적절
- ~~**T**rackable~~ — 의미 있는 커밋 (사람 책임)

---

## Spec 문서: Bounded Context 단위

spec은 도메인 주도 설계의 Bounded Context를 단위로 한다. 각 spec 파일은 한 컨텍스트를 외부에서 이해하기 위한 모든 것을 담는다.

### spec.md 구성 항목

**컨텍스트 내부**
- 도메인 엔티티의 필드와 타입 (Aggregate / Entity)
- 비즈니스 규칙 (정책 분기, 핵심 상수)
- 핵심 설계 결정 (왜 이 구조를 잡았는가)

**외부 노출 (Published Language)**
- API 시그니처 (경로, HTTP 메서드, 요청·응답 DTO)
- 이벤트·비동기 계약 (페이로드, 알림 구조)
- 에러 계약 (에러 코드 체계, 예외 타입)

**다른 컨텍스트와의 관계 (Context Map)**
- 다른 도메인에 대한 의존과 그 방식

### Spec 갱신 6가지 트리거 (Phase 5)

다음 중 하나라도 해당하면 spec 갱신 대상이다:

1. 구조적 변경 — 도메인 엔티티 필드 추가/삭제/타입 변경
2. API 계약 변경 — Controller 경로/메서드/요청·응답 DTO 변경
3. 의존성 변경 — 다른 도메인에 대한 의존 추가/제거
4. 비즈니스 규칙 변경 — 핵심 상수, 정책 분기 로직 변경
5. 이벤트/비동기 계약 변경 — 도메인 간 이벤트 페이로드, 알림 구조 변경
6. 에러 계약 변경 — 에러 코드 체계, 예외 타입, 에러 응답 DTO 변경

> 인프라/설정 변경(MongoDB 컬렉션명, 인덱스, Redis 키, 환경변수)은 도메인 spec이 아니라 별도 위치에서 관리한다.

---

## Brownfield 진입 전략

대부분의 시스템은 spec이 0개인 상태에서 시작한다. 처음부터 모든 도메인의 spec을 만들 수는 없으므로, **작업 흐름 속에서 자연스럽게 누적**되도록 설계했다.

분기 기준은 도메인이 신규냐 legacy냐가 아니라 **spec 파일이 존재하느냐**다.

### 분기 A — spec 파일 부재

- baseline이 없으므로 코드 full-scan 1회 수행 (불가피한 비용)
- `.claude/docs/domains/_TEMPLATE.md` 구조를 그대로 복제해 신규 작성
- 작성 직후 이 spec이 차후 갱신의 baseline

### 분기 B — spec 파일 존재

git history 기반 incremental 갱신:

```bash
# Step 1: 현재 커밋 변경분을 spec에 반영
# implementer 변경 파일을 spec과 대조해 6가지 항목 갱신

# Step 2: 누락분 검출 (다른 팀원 변경 흡수)
git log -1 --format=%H -- .claude/docs/domains/<도메인>.md
git log <spec_last_commit>..HEAD --name-only -- src/main/java/.../<도메인>/
# 변경 파일들의 diff만 읽어 spec 반영 여부 판단
```

이 분기 덕분에 도메인을 처음 건드릴 때만 비싸게 비용을 치르고, 그 다음부터는 가벼워진다. 팀원들과 GitHub로 spec을 공유하면 몇 주 안에 시스템 대부분의 spec이 누적된다.

---

## 디렉토리 구조

```
.claude/
├── agents/
│   ├── plan-writer.md      # Waterfall 계획 에이전트
│   ├── implementer.md      # DDD 기반 구현 에이전트
│   ├── test-writer.md      # 테스트 작성 에이전트
│   └── reviewer.md         # Trust-5 검증 에이전트
├── docs/
│   ├── architecture.md     # 프로젝트 레이어 규칙
│   ├── code-style.md       # 코드 스타일 컨벤션
│   └── domains/            # 도메인 spec (시간이 지날수록 누적)
│       ├── _TEMPLATE.md
│       └── <도메인>.md
├── plans/                  # 워크플로우별 임시 작업 디렉토리 (성공 종료 시 삭제)
│   └── <식별자>/
│       ├── requirements.md
│       └── plan.md
└── skills/
    └── lets-get-to-work/
        └── SKILL.md
```

---

## 사용법

### `/lets-get-to-work` — 대화 컨텍스트 진입

대화에서 충분히 논의를 마친 뒤 호출한다. Phase 0부터 자동으로 진행된다.

```
/lets-get-to-work              # 슬러그 자동 생성
/lets-get-to-work RGD-1234     # 티켓 번호 명시
```

### 워크플로우가 사용자에게 묻는 시점 (4개로 한정)

1. **브랜치 결정** — 무조건 1회, 작업 진입 전
2. **plan-writer의 `[질문 #N]` 일괄 질문** — Phase 1 종료 직후, 표식이 있을 때만
3. **보안 결정** — 인증/인가, 권한 우회 가능성에 대한 결정 필요 시
4. **에스컬레이션** — 재시도 한도 초과 시

그 외에는 자동 진행. plan.md 사전 승인, Phase 진입 전 확인, FAIL 시 재시도 등은 모두 자동.

---

## 적용 범위

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

비적용 대상에 `/lets-get-to-work`를 호출하면 파이프라인 오버헤드가 작업 자체보다 커진다. 메인 세션이 직접 처리하는 것이 적합하다.

---

## 설계 결정

| 결정 | 이유 |
|------|------|
| spec 문서를 세션 컨텍스트보다 우선 | 세션은 일회성, spec은 영속 |
| 계획은 Waterfall, 구현은 XP 흐름 | 충분한 사전 설계가 reviewer 부담과 재시도 비용을 줄임 |
| 기존 프로젝트는 DDD 우선 | 이미 확립된 패턴 위에 작업 — 도메인 분석 후 구현 |
| plan-writer는 opus | 계획 품질이 전체 품질을 결정 — 재시도 줄여 총 토큰 절약 |
| implementer는 haiku/sonnet | 상세 계획을 따르는 작업은 비싼 추론이 불필요 |
| Spec 갱신은 Terminal Action | 검증 안 된 결과가 spec에 들어가면 다음 plan-writer가 잘못된 baseline에서 시작 |

---

## 관련 문서

- [`.claude/agents/`](./.claude/agents/) — 4개 에이전트 정의
- [`.claude/skills/lets-get-to-work/SKILL.md`](./.claude/skills/lets-get-to-work/SKILL.md) — 슬래시 커맨드 정의
- `~/.claude/agents/workflow.md` — 멀티에이전트 워크플로우 상세 (Phase별 정의, 핸드오프 컨트랙트, 재시도 정책)
- `.claude/docs/domains/_TEMPLATE.md` — 도메인 spec 표준 템플릿

---

## 라이선스

본인 프로젝트의 컨벤션에 맞게 수정해서 사용하시면 됩니다.
