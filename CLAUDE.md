# CLAUDE.md

{제품에 대한 설명}

**기술 스택:** ex) Spring Boot 3.3 / Java 21 / MongoDB (primary) / Redis / ElasticSearch

> 아키텍처·코드 스타일·레이어 책임·주의사항 상세 규칙은 `.claude/docs/` 참조 (`architecture.md`, `code-style.md`)
> 해당 파일이 없을 시 기존 코드를 분석하여 새로 생성한다.

---

## 주요 명령어

```bash
# 빌드 (Java 21 필수 — Java 11/17로 실행 시 빌드 실패)
./gradlew build

# 전체 테스트
./gradlew test

# 단일 테스트 클래스
./gradlew test --tests "com.xxx.xxx.credit.application.CreditUsageServiceTest"

# 단일 테스트 메서드
./gradlew test --tests "com.xxx.xxx.credit.application.CreditUsageServiceTest.메서드명"

# 로컬 실행 (src/main/resources/application-local.yml 필요)
./gradlew bootRun --args='--spring.profiles.active=local'
```

> 별도의 lint/checkstyle Gradle task는 없다. 코드 포맷은 IntelliJ 코드 스타일(`intellij-java-google-style.xml`)을 기준으로 한다 — 들여쓰기 4스페이스, wildcard import 금지.

---

## 작업 워크플로우

1. **코드 수정 전**: 같은 도메인의 기존 패턴 먼저 확인
2. **신규 기능**: 구현 → 테스트 작성 → 검증 순서 (test-after / DDD 흐름 — 상세는 `.claude/workflow.md` Phase 2~4 참조)
3. **커밋 메시지**: `[RGD-XXXX] 작업 내용 요약` 형식 (Jira 티켓 링크: `https://xxx.atlassian.net/browse/XXX-XXXX`).
4. **단계별 진행**: 다단계 작업 시 각 단계 완료 후 확인하고 다음 단계로 진행. 모든 단계를 한번에 수행하지 않음

---

## Claude와의 협업 방식

- 사용자 의견에 동조하지 않고 객관적 의견 제시 (trade-off, 제약사항, 대안 포함). 더 단순한 방법이 있으면 반론을 제기한다.
- 요구사항이 불명확하거나 여러 해석이 가능하면 추측하지 말고 사용자에게 먼저 확인한다. 가정이 있으면 명시적으로 말한다.

### 단순하게

- 기존 패턴(서비스 계층 if 검사, 기존 유틸 활용 등)으로 충분한 문제에 새로운 구조(커스텀 어노테이션, 프레임워크, 추상 클래스 등)를 도입하지 않는다.
- 해결책은 가장 단순한 방식부터 시도한다. 복잡한 접근이 필요하면 구현 전에 사용자에게 확인한다.
- 불가능한 시나리오에 대한 에러 핸들링을 추가하지 않는다.
- 구현이 반복적으로 테스트 실패하면 접근 방식 자체를 재검토한다. 같은 방향으로 3회 이상 수정-실패를 반복하지 않는다.

### 정확한 범위만

- 인접한 코드, 주석, 포맷팅을 "개선"하지 않는다. 기존 스타일이 내 방식과 달라도 따른다.
- 모든 변경된 줄은 사용자 요청으로 추적 가능해야 한다.
