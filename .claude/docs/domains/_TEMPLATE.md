# [도메인명] 도메인

> **이 문서는 도메인 spec 표준 템플릿이다.** 신규 도메인 spec을 작성하거나 legacy backfill 시 이 구조를 그대로 복제해서 채운다.
> 작성 후 이 안내 인용문(>)은 제거한다.

---

[도메인 한 줄 정의 — 책임/저장소/aggregate를 1~2문장으로. 예: "사용자의 읽지 않은 채팅 알림을 관리하는 경량 도메인. 별도 엔티티 없이 ChatLogEntity의 `seen` 필드를 기반으로 동작한다."]

## 패키지 구조

> 디렉토리 트리 + 각 파일에 **한 줄 설명**을 우측 주석으로 단다. plan-writer가 이 트리만 보고 어느 파일을 만져야 할지 판단할 수 있어야 한다.

```
[도메인]/
├── presentation/
│   ├── [Controller].java          # [경로 + 한 줄 책임]
│   └── dto/
│       ├── [Request].java         # [record / class — 주요 필드]
│       └── [Response].java        # [핵심 필드 또는 from() 팩토리 여부]
├── application/
│   ├── [Service].java             # [핵심 책임]
│   └── data/
│       └── [DataAdapter].java     # [영속 게이트웨이 책임]
├── entity/
│   ├── [Aggregate].java           # [collection명] — aggregate root
│   └── [ValueObject].java         # [embedded인지 별도 collection인지]
└── repository/
    └── [Repository].java          # [MongoRepository / Custom 여부]
```

## API

> 메서드/경로/설명 3열 표. 도메인이 큰 경우 컨트롤러별로 sub-section으로 쪼갠다 (`### [Controller명]`).

| 메서드 | 경로 | 설명 |
|--------|------|------|
| POST | `/path` | [한 줄 책임 — 부수 효과(연쇄 생성/삭제)는 명시] |
| GET | `/path` | [조회 범위·정렬·페이지네이션 등 핵심 제약] |
| PATCH | `/path/{id}` | [null-skip 여부, 부분 수정 정책] |
| DELETE | `/path/{id}` | [cascade 대상 명시] |

## 의존 도메인

> 이 도메인이 **호출하는** 다른 도메인. 역방향 의존(다른 도메인이 이 도메인을 호출)도 중요한 경우 표 아래 인용문으로 보강.

| 도메인 | 사용 클래스 | 용도 |
|--------|-----------|------|
| [도메인명] | [DataAdapter / Entity 클래스명] | [무엇을 위해 호출하는지] |
| common | `BaseTimeEntity`, `GlobalException`, `ErrorCode` | 공통 인프라 |

> (해당 시) 역방향 의존 노트: [예: chatlog 도메인이 `AgentSubInfoDto.from(AiProject)` 로 이 도메인을 역참조한다.]

## 핵심 설계 결정

> **이 섹션이 가장 중요하다.** "왜 이렇게 됐는지"를 보존. 코드만 봐서는 알 수 없는 trade-off, 과거 incident, 거부된 대안을 기록한다. 플랜 단계의 설계 품질이 여기서 결정된다.

| 결정 | 이유 |
|------|------|
| [구체적 결정 — 패턴/네이밍/구조 선택] | [왜 그렇게 했는지 — 거부된 대안, 과거 incident, 외부 제약] |
| [예: 컬렉션명 `services` 유지 (entity명: `AiAgent`)] | [예: 1.10.X 이전 GptService → AiAgent 리네이밍, 컬렉션 마이그레이션은 미수행] |
| [예: 엔티티 수정은 null-skip 패턴] | [예: 호출자가 기존 값 조회 없이 필드별 선택 갱신 가능] |

## 비즈니스 규칙

> 정책 상수, 한도, 분기 로직, 검증 순서. **숫자 상수는 코드 상수명과 함께 기록**해 plan-writer가 grep할 수 있게 한다.

- [규칙 1 — 상수/조건 + 코드 식별자] (예: 최대 100개 — `MAX_PROJECT_COUNT`)
- [규칙 2 — 검증 순서가 중요한 경우 ①②③ 형태로 기록]
- [규칙 3 — 권한 검사 조건식, cascade 순서 등]

## 주의 사항 (선택)

> 함정, deprecated 예정 항목, 직관적이지 않은 명명 규칙. 신규 코드가 빠지기 쉬운 함정을 기록한다.

- [예: `isPublicAgent`의 정의가 `chatType` null/empty라는 점은 직관적이지 않다 — 신규 코드에서 반드시 이 메서드를 호출.]
- [예: `uiAccessEnabled`는 향후 제거 예정. 새 검증 로직에 의존성 추가 금지.]
- [예: 도메인 A가 패키지 B에 일부 응답 DTO를 가진다 — 변경 시 양쪽 함께 확인.]

---

## 작성 가이드 (작성자용 — 작성 후 이 섹션은 삭제)

**spec의 목적**: plan-writer가 이 문서만 읽고 plan을 세울 수 있어야 한다 (코드베이스 full-scan 회피 → 토큰↓·정확성↑).

**작성 순서 권장:**
1. 패키지 구조 — 디렉토리 트리 먼저 (plan-writer가 어디를 만질지 판단할 수 있게)
2. API — Controller 메서드 시그니처 그대로
3. 의존 도메인 — `import` 통계로 추출
4. 핵심 설계 결정 — 가장 중요. git log + 커밋 메시지 + RGD 티켓 메모에서 "왜"를 발굴
5. 비즈니스 규칙 — 상수와 정책 분기
6. 주의 사항 — 도메인 작업해본 팀원의 암묵지를 명문화

**채우지 말아야 할 것:**
- 메서드별 라인 단위 구현 — 코드를 보면 됨 (spec은 "왜"와 "구조"가 핵심)
- 변경 이력 — git log가 진실
- 향후 작업 계획 — Jira 또는 plan.md 책임
