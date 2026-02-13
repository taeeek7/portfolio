# Scrumble - 데일리 스크럼 자동화 봇

> FastAPI + Slack Bolt 기반 대화형 데일리 스탠드업 자동화 서비스
> 프로젝트: `scrumble`

---

## 프로젝트 개요

매일 아침 Slack DM으로 팀원에게 스탠드업 질문을 보내고, 답변을 수집하여 팀 채널에 자동 게시하는 **대화형 데일리 스크럼 봇**.
Geekbot 같은 외부 SaaS 없이 자체 운영 가능한 스탠드업 도구를 구축했습니다.

---

## 기술 스택

| 영역 | 기술 |
|------|------|
| **Language** | Python 3.13 |
| **Web Framework** | FastAPI + Uvicorn |
| **Slack** | Slack Bolt (Socket Mode) |
| **Scheduling** | APScheduler (Cron) |
| **Database** | MySQL (mysql-connector-python) |
| **Validation** | Pydantic + Pydantic Settings |
| **Holiday** | holidays 라이브러리 (한국 공휴일) |
| **Package Manager** | uv (lock file 기반) |
| **Testing** | pytest + pytest-asyncio |
| **Linting** | ruff |
| **Secret Management** | SOPS + Age |
| **Deployment** | Docker → NCloud NCR → ArgoCD |

---

## 아키텍처

```
┌────────────────────────────────────────────────┐
│              FastAPI Application                │
│                                                 │
│  Lifespan Manager                               │
│  ├── Startup: APScheduler + Socket Mode 시작    │
│  └── Shutdown: Graceful 종료                    │
│                                                 │
│  ┌──────────────┐  ┌────────────────────────┐  │
│  │ APScheduler  │  │  Slack Bolt            │  │
│  │              │  │  (Socket Mode)         │  │
│  │ Cron: 07:00  │  │                        │  │
│  │ Mon-Fri      │  │  Event Handlers:       │  │
│  │ Asia/Seoul   │  │  - DM message          │  │
│  └──────┬───────┘  │  - Button actions      │  │
│         │          │  - Modal submissions    │  │
│         ▼          └──────────┬─────────────┘  │
│  ┌──────────────────────────────────────────┐  │
│  │            service.py (Core)             │  │
│  │                                          │  │
│  │  send_standup_notifications()            │  │
│  │       ↓                                  │  │
│  │  start_standup_conversation()            │  │
│  │       ↓ (5-Step DM Conversation)         │  │
│  │  Q1: 오늘 컨디션은?                      │  │
│  │  Q2: 근무 형태는?                        │  │
│  │  Q3: 어제 수행한 업무는?                  │  │
│  │  Q4: 오늘 예정된 업무는?                  │  │
│  │  Q5: 팀에 공유할 내용?                    │  │
│  │       ↓                                  │  │
│  │  complete_standup() → DB + Channel Post  │  │
│  └──────────────────┬───────────────────────┘  │
│                     │                          │
│  ┌──────────────────▼───────────────────────┐  │
│  │         repository.py (MySQL)            │  │
│  │  - select_active_members()               │  │
│  │  - upsert_response()                     │  │
│  │  - select_yesterday_plan() (7일 이내)    │  │
│  └──────────────────────────────────────────┘  │
│                                                 │
│  ┌──────────────────────────────────────────┐  │
│  │  router.py (Manual Trigger API)          │  │
│  │  GET /api/daily-scrum/scheduler          │  │
│  │    ?user_ids=U1,U2&skip_holiday_check    │  │
│  └──────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

---

## 핵심 기능 및 해결한 문제

### 1. 대화형 5단계 스탠드업 수집

**문제:** 기존 스탠드업 도구는 폼 기반으로 딱딱하고 참여율 낮음
**해결:**
```
Bot: "오늘 컨디션은 어떤가요?" 🌤
User: "좋아요!"

Bot: "오늘의 근무 형태는 어떻게 되시나요?"
     [재택근무] [출근] [하이브리드]
User: clicks [출근]

Bot: "어제 수행한 업무는 어떻게 되었나요?"
     (참고: 어제 계획 - "API 개발 마무리")
User: "API 개발 완료하고 테스트 작성했습니다"
...
```
- DM 기반 1:1 대화로 자연스러운 참여 유도
- 이전 계획 리마인더로 연속성 있는 회고
- In-memory 상태 관리 (conversation_state)

### 2. 한국 공휴일 자동 스킵

**문제:** 공휴일에도 스탠드업 알림 발송
**해결:**
```python
# holidays 라이브러리 + 수동 추가
holidays_kr = holidays.KR(observed=True)  # 대체공휴일 포함

# 2026년부터 제헌절(7.17) 수동 추가
if year >= 2026:
    holidays_kr[date(year, 7, 17)] = "제헌절"
```
- 주말(토/일) 자동 제외
- 한국 법정 공휴일 자동 제외 (대체공휴일 포함)
- `skip_holiday_check` 파라미터로 수동 오버라이드 가능

### 3. 수정 가능한 응답

**문제:** 한번 제출하면 수정 불가 → 오타/실수 시 곤란
**해결:**
- 채널 게시 후 "수정하기" 버튼 제공
- Slack Modal로 기존 답변 편집
- 채널 메시지 업데이트 (중복 게시 방지)
- `channel_message_ts` 저장으로 정확한 메시지 타겟팅

### 4. 휴가 등록 기능

**문제:** 휴가 시 스탠드업 미응답 → 팀원 혼란
**해결:**
- "오늘 휴가입니다" 버튼으로 즉시 휴가 등록
- 채널에 휴가 상태 자동 게시
- 휴가 취소 후 스탠드업 재시작 가능
- `is_day_off` 플래그로 DB 기록

### 5. 텍스트 새니타이제이션

**문제:** 사용자 입력의 NULL 바이트, 제어 문자로 DB 오류
**해결:**
```python
def sanitize_text(text):
    text = text.replace('\x00', '')          # NULL 바이트 제거
    text = re.sub(r'[\x01-\x08\x0e-\x1f]', '', text)  # 제어 문자
    text = re.sub(r'(\d{1,2})(:\d{2})', r'\1\u200B\2', text)  # 이모지 파싱 방지
    return text[:field_limit]                # 필드 길이 제한
```

### 6. Socket Mode 자동 재연결

**문제:** WebSocket 연결 끊김 시 봇 중단
**해결:**
- 3회 재시도 + 지수 백오프
- 연결 실패 시 Slack 채널에 알림
- 백그라운드 스레드에서 독립 실행

---

## 테스트 커버리지

```
tests/
├── test_main.py          # FastAPI 앱 생성, 헬스체크
├── test_service.py       # 서비스 레이어 유닛 테스트
├── test_models.py        # Pydantic 모델 검증
├── test_holiday.py       # 공휴일 감지 로직
└── test_socket_mode.py   # 재연결 로직 (3회 재시도, 백오프)
```

---

## 데이터 모델

```sql
-- daily_scrum_member
CREATE TABLE daily_scrum_member (
    slack_user_id VARCHAR(20) PRIMARY KEY,
    name VARCHAR(50),
    is_active BOOLEAN
);

-- daily_scrum_response
CREATE TABLE daily_scrum_response (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id VARCHAR(20),
    user_name VARCHAR(50),
    response_date DATE,
    today_condition TEXT,
    work_type VARCHAR(100),
    done_yesterday TEXT,
    plan_today TEXT,
    share_note TEXT,
    channel_message_ts VARCHAR(50),
    is_day_off BOOLEAN
);
```

---

## 배포

```
GitHub Push (test branch)
    ↓
GitHub Actions
    ├── SOPS 설치 → .env.enc 복호화
    ├── Docker 빌드 (python:3.13-slim, uv 기반)
    ├── NCloud NCR 푸시
    ↓
ArgoCD GitOps → Kubernetes 배포
```

---

## 성과 및 효과

### 팀 커뮤니케이션 개선
- **스탠드업 참여율 향상:** DM 기반 자연스러운 대화로 기존 폼 기반 도구 대비 **참여율 향상**
- **비동기 스탠드업:** 각자 편한 시간에 DM 응답 → 시간대가 다른 원격 팀원도 참여 가능
- **이전 계획 리마인더:** 어제 계획과 오늘 실적을 자연스럽게 연결하여 **업무 연속성 향상**

### 비용 절감
- **SaaS 대체:** Geekbot 등 유료 스탠드업 도구 **월 구독료 절감** (팀 규모에 따라 연간 수십~수백만원)
- **자체 커스터마이징:** 한국 공휴일, 제헌절 등 국내 환경에 맞는 **세밀한 커스텀** 가능
- **데이터 소유권:** 스탠드업 데이터를 자체 DB에 보관하여 **분석 및 활용 가능**

### 기술적 완성도
- **pytest 테스트:** 서비스/모델/공휴일/Socket Mode 재연결 등 핵심 로직 테스트 커버리지 확보
- **Pydantic 기반 설계:** DTO, 설정, 에러 응답 모두 타입 안전하게 설계
- **Graceful Lifecycle:** FastAPI lifespan으로 시작/종료 안정적 관리
