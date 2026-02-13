# Work Scheduler - 키퍼 주간 일정 등록 웹앱

> Vanilla JavaScript 기반 모바일 최적화 스케줄 등록 서비스
> 프로젝트: `work-scheduler`

---

## 프로젝트 개요

키퍼(하우스키핑 스태프)가 **모바일에서 다음 주 근무 가능 일자를 등록**하는 경량 웹 애플리케이션.
프레임워크 없이 Vanilla JavaScript만으로 구현하여 699줄의 코드로 완성한 실용적인 프론트엔드 프로젝트입니다.

---

## 기술 스택

| 영역 | 기술 |
|------|------|
| **Frontend** | HTML5, CSS3, Vanilla JavaScript (ES6 Modules) |
| **Backend** | data-api (별도 프로젝트, REST API) |
| **Deployment** | Netlify (Static Site) |
| **Monitoring** | WhaTap Browser RUM |
| **Package** | 없음 (Zero Dependencies) |

---

## 아키텍처

```
┌─────────────────────────────────────────┐
│          Netlify CDN (Static)           │
│                                         │
│  index.html                             │
│    ├── css/style.css                    │
│    ├── js/env.js (환경변수)             │
│    ├── js/main.js (이벤트 핸들링)       │
│    ├── js/services.js (API 통신)        │
│    └── js/uiController.js (DOM 제어)    │
│                                         │
└──────────────┬──────────────────────────┘
               │ HTTPS
               ▼
┌──────────────────────────────────────────┐
│       Backend API Server                │
│                                          │
│  POST /GetScheduleToken                  │
│    → 토큰 검증 + 키퍼 정보 반환         │
│                                          │
│  POST /InsertWorkSchedules               │
│    → 주간 일정 등록/휴가 신청            │
└──────────────────────────────────────────┘
```

---

## 핵심 기능 및 해결한 문제

### 1. 토큰 기반 보안 인증

**문제:** 키퍼 개인 링크로 접근 시 본인 확인 필요
**해결:**
```
URL: https://{service-domain}/?token=abc123
    ↓
services.fetchMemberKeeperInfo(token)
    ↓
Backend: Fernet 토큰 복호화 → memberKeeperId 매핑
    ↓
유효: 일정 등록 폼 표시
무효: 에러 메시지 표시
```
- 개인 고유 토큰으로 URL 공유만으로 인증
- 별도 로그인 없이 간편한 접근

### 2. 동적 날짜 계산

**문제:** 매주 바뀌는 "다음 주" 날짜를 정확히 표시
**해결:**
```javascript
getNextMonday() {
    const today = new Date();
    const dayOfWeek = today.getDay();
    const daysUntilMonday = (8 - dayOfWeek) % 7 || 7;
    // 일요일에 접속해도 정확한 다음 주 월요일 계산
}
```
- 월~일 7일간 체크박스 동적 생성
- "2025-02-17 ~ 2025-02-23" 형식 날짜 범위 표시
- "전체 선택" 체크박스로 편의성

### 3. 듀얼 제출 모드

**문제:** 근무 등록과 휴가 신청이 서로 다른 워크플로우
**해결:**
| 모드 | 동작 | API 호출 |
|------|------|----------|
| **근무 등록** | 선택된 날 = "scheduled", 미선택 = "canceled" | InsertWorkSchedules |
| **휴가 신청** | 전체 7일 = "canceled" | InsertWorkSchedules |

- 확인 다이얼로그로 실수 방지
- 3초간 성공 메시지 표시 후 폼 리셋

### 4. 모바일 최적화 UI

**문제:** 키퍼가 현장에서 스마트폰으로 등록해야 함
**해결:**
- Max-width 500px 모바일 퍼스트 디자인
- 터치 친화적 체크박스 크기
- 직관적인 색상 코딩 (파란색: 등록, 빨간색: 휴가)
- Flexbox 기반 반응형 레이아웃

---

## 코드 구조 (699줄)

| 파일 | 줄 수 | 역할 |
|------|-------|------|
| `js/services.js` | 147 | API 통신 (토큰 검증, 일정 등록) |
| `js/main.js` | 134 | 이벤트 핸들링, 폼 제출 로직 |
| `js/uiController.js` | 117 | DOM 조작, 날짜 계산, UI 렌더링 |
| `css/style.css` | 216 | 모바일 최적화 스타일링 |
| `index.html` | 68 | 메인 HTML 구조 |
| `js/env.js` | 4 | 환경 변수 (API 엔드포인트) |
| `js/generate-env.js` | 13 | Netlify 환경 변수 주입 스크립트 |

---

## 배포

- **Netlify** 정적 사이트 호스팅
- `netlify.toml`로 SPA 리다이렉트 설정
- `generate-env.js`로 빌드 시 환경 변수 주입
- WhaTap Browser RUM으로 실사용자 모니터링 (100% 샘플링)

---

## 성과 및 효과

### 운영 효율화
- **자가 일정 등록:** 키퍼가 직접 모바일로 일정 등록 → **관리자 수동 입력 작업 완전 제거**
- **URL 기반 간편 접근:** 토큰 링크만 공유하면 별도 앱 설치/로그인 없이 즉시 사용
- **제출 즉시 반영:** Backend API 연동으로 등록 즉시 운영 시스템에 반영

### 기술적 성과
- **Zero Dependencies:** npm 패키지 없이 699줄로 완성 → **빌드 시간 0초, 보안 취약점 0개**
- **모바일 퍼스트:** 현장 키퍼 사용 환경에 최적화된 터치 친화적 UI
- **Netlify 배포:** 정적 사이트로 서버 비용 0원, CDN 기반 글로벌 빠른 로딩
