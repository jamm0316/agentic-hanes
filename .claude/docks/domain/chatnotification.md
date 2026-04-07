# chatnotification 도메인

사용자의 읽지 않은 채팅 알림을 관리하는 경량 도메인. 별도 엔티티 없이 ChatLogEntity의 `seen` 필드를 기반으로 동작한다.

## 패키지 구조

```
chatnotification/
├── presentation/
│   ├── ChatNotificationController.java    # GET /chats/logs/unseen, PATCH /chats/logs/seen
│   └── dto/
│       ├── ChatNotificationResponse.java  # record, from() 팩토리
│       └── MarkChatLogsAsSeenRequest.java # record, chatLogIds
└── application/
    └── ChatNotificationService.java       # 조회 + 읽음 처리
```

## API

| 메서드 | 경로 | 설명 |
|--------|------|------|
| GET | `/chats/logs/unseen` | 미읽은 알림 목록 (최근 2주, NORMAL/ERROR, 최대 100개) |
| PATCH | `/chats/logs/seen` | 지정된 chatLogIds를 읽음 처리 (userId 권한 검증) |

## 의존 도메인

| 도메인 | 사용 클래스 | 용도 |
|--------|-----------|------|
| chatlog | ChatLogDataAdapter | 미읽은 로그 조회, 읽음 처리 (seen 필드) |
| aiagent | AiAgentDataAdapter | ServiceType.AGENT일 때 봇 아이콘, chatType 조회 |
| aitool | AiToolDataAdapter | ServiceType.AI_TOOL일 때 봇 아이콘 조회 |

## 핵심 설계 결정

| 결정 | 이유 |
|------|------|
| 별도 엔티티 없이 ChatLog.seen 필드 사용 | 별도 엔티티는 현 시점 over-engineering (RGD-3773 회의) |
| GET에서 읽음 처리하지 않고 PATCH 분리 | POST→GET→PUT→GET 타이밍 버그로 알림 누락 발생 (RGD-3773 트러블슈팅) |
| 필드명 seen (isNotified 아님) | isNotified는 "보냄/읽음" 두 의미가 혼재. seen은 "사용자가 읽었는가"만 표현 |
| ServiceType.AI_PROJECT 미지원 | 현재 요구사항에 없음. 호출 시 GlobalException |

## 비즈니스 규칙

- 알림 대상: chatLogResult가 NORMAL 또는 ERROR이고 seen=false인 ChatLog
- 조회 범위: 최근 2주 이내 (`UNSEEN_LOOKBACK_WEEKS = 2`)
- 최대 개수: 100개 (`MAX_UNSEEN_CHAT_LOGS_SIZE = 100`)
- 읽음 처리 시 요청자 userId와 chatLog.authUsersId 일치 검증 필수
- logsQData는 30자 초과 시 "..." 처리
