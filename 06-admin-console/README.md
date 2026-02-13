# 운영 관리 Admin Console

> YAML 선언형 Low-Code 어드민 대시보드
> 프로젝트: `keeper-console` (키퍼 운영 관리), `connect-admin` (제안서 관리)

---

## 프로젝트 개요

**SelectFromUser** 플랫폼 기반의 선언형 어드민 대시보드 2종.
YAML 설정만으로 복잡한 어드민 UI, DB 쿼리, API 연동, 역할 기반 접근 제어를 구현합니다.

| 프로젝트 | 용도 | YAML 규모 |
|----------|------|-----------|
| **keeper-console** | 키퍼 배정/스케줄/세탁/린넨 운영 관리 | 21,000+ 줄 |
| **connect-admin** | 파트너 제안서/견적 관리 | 3,520 줄 |

---

## 기술 스택

| 영역 | 기술 |
|------|------|
| **Platform** | SelectFromUser v2.4.2 (Low-Code Admin SaaS) |
| **Configuration** | YAML (페이지, 쿼리, 워크플로우 정의) |
| **Embedded Logic** | JavaScript ES6 (submitFn, responseFn, formatFn) |
| **Database** | MySQL (NCloud VPC + AWS RDS) |
| **API Protocol** | Protocol Buffers 3 (connect-admin) |
| **Runtime** | Node.js 16 Alpine |
| **Deployment** | Docker → NCloud NCR → ArgoCD |
| **CI/CD** | GitHub Actions |

---

## 아키텍처

```
┌─────────────────────────────────────────────┐
│           SelectFromUser Platform            │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │           YAML Configuration         │   │
│  │                                     │   │
│  │  index.yml → 메뉴 구조, 리소스 정의 │   │
│  │  prod/*.yml → 페이지별 기능 정의    │   │
│  │  guide/*.yml → 사용자 가이드        │   │
│  │  release/*.yml → 릴리즈 노트        │   │
│  └──────────────┬──────────────────────┘   │
│                 │                           │
│  ┌──────────────▼──────────────────────┐   │
│  │          YAML Block Types           │   │
│  │                                     │   │
│  │  query: MySQL SELECT/INSERT/UPDATE  │   │
│  │  http: External API 호출            │   │
│  │  submitFn: JS 비즈니스 로직         │   │
│  │  responseFn: 응답 후처리            │   │
│  │  formatFn: 데이터 포맷팅            │   │
│  └──────────────┬──────────────────────┘   │
│                 │                           │
│  ┌──────────────▼──────────────────────┐   │
│  │        Database Resources           │   │
│  │                                     │   │
│  │  mysql.cleanops (운영 DB)           │   │
│  │  mysql.keeper (키퍼 Live DB)        │   │
│  │  mysql.dev (개발 DB)                │   │
│  └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

---

## 핵심 기능 및 해결한 문제

### 1. AI 배정 관리 대시보드 (keeper-console)

**문제:** AI 배정 결과를 확인하고 조정할 수 있는 관리 도구 부재
**해결:**
- `assign-optimization-ai.yml` (1,786줄) + renewal 버전 (3,433줄)
- 지점 선택 → AI 배정 실행 → 결과 확인 → 수동 조정 → 확정
- 패널티 점수 시각화 (면적, 층수, 업무량 편차)
- Airflow/data-api 연동으로 AI 모델 실행 트리거

### 2. 키퍼 스케줄 관리

**문제:** 수십 명 키퍼의 주간 일정 수기 관리
**해결:**
- `keeper-schedules.yml` (1,761줄)
- 키퍼별 근무 가능 일자 조회/등록
- 일일/주간 단위 스케줄 관리
- 키퍼 등급별 필터링 (Diamond ~ Bronze)

### 3. 배정 자동화 워크플로우

**문제:** 배정 프로세스의 여러 단계를 수동 실행
**해결:**
- `assign-automation.yml` (3,254줄)
- 자동화 룰 설정 → 배치 실행 → 결과 검증
- 단계별 상태 추적 및 롤백

### 4. 세탁 CS 관리 시스템

**문제:** 세탁 관련 고객 요청을 체계적으로 관리할 도구 부재
**해결:**
- 세탁 CS 접수 → 처리 → 결과 보고 워크플로우
- 파트너별 CS 현황 대시보드
- 월별 정산 명세서 자동 생성

### 5. 린넨 재고 관리

**문제:** 린넨 이동/폐기 수기 관리
**해결:**
- `rental-inventory-transaction.yml` (1,034줄)
- 린넨 이관 요청 접수 → 처리 → 이력 관리
- 지점별 재고 현황 추적

### 6. 제안서/견적 관리 (connect-admin)

**문제:** 숙박 파트너 제안서와 견적을 관리할 시스템 부재
**해결:**
- Protocol Buffers 기반 API 21개 정의
- 제안서 생성 → 견적 제출 → 미팅 확정 워크플로우
- 파트너 정보 관리 (호텔, 모텔, 생활숙박 등)
- 상태별 필터링 (검토중, 제안완료, 확정, 일정잡힘)

### 7. 역할 기반 접근 제어 (RBAC)

**문제:** 관리자/파트너/고객별로 다른 화면 필요
**해결:**
```yaml
roles:
  list: [eleven, partners, client, laundry]
  view: [eleven, partners]
```
- 역할별 메뉴/페이지 접근 제한
- 데이터 필터링 (자기 소속 데이터만 조회)

---

## 코드로서의 설정 (Configuration as Code)

YAML만으로 전체 어드민 UI를 정의하는 패턴:

```yaml
# 예시: 데이터 조회 블록
- type: query
  resource: mysql.ops
  sqlType: select
  sql: >
    SELECT t.id, t.name, t.grade, t.phone
    FROM staff t
    WHERE t.branch_code = :branch_code
    AND t.status = 'active'
  params:
    - key: branch_code
      valueFromRow: branch_code
  columns:
    id: { label: "ID" }
    name: { label: "이름" }
    grade: { label: "등급", format: badge }
    phone: { label: "연락처", masking: true }

# 예시: API 호출 블록
- type: http
  axios:
    method: POST
    url: "{{API_ENDPOINT}}/ExecuteAssignmentOptimization"
  responseFn: |
    if (result.status === 'SUCCESS') {
      alert('AI 배정이 완료되었습니다.');
    }
```

---

## 프로젝트 규모

### keeper-console
| 영역 | 파일 수 | 총 줄 수 |
|------|---------|----------|
| 배정 관리 (prod/assign/) | 7 | ~12,352 |
| 키퍼 관리 (prod/keeper/) | 3 | ~2,504 |
| 세탁 관리 (prod/laundry/) | 5 | ~2,244 |
| 린넨/CS (prod/rental/) | 6 | ~3,895 |
| 사용자 가이드 (guide/) | 12 | ~2,200 |
| 릴리즈 노트 (release/) | 6 | ~900 |
| **총계** | **56** | **~24,000+** |

### connect-admin
| 영역 | 파일 수 | 총 줄 수 |
|------|---------|----------|
| 페이지 설정 (prod/) | 10 | ~3,520 |
| Proto 정의 (proto/) | 21+3 | ~500 |
| **총계** | **34** | **~4,020** |

---

## 성과 및 효과

### 개발 생산성
- **Low-Code 어드민 구축:** 전통적인 React/Vue 어드민 개발 대비 **개발 기간 70~80% 단축**
- **YAML 기반 빠른 반복:** 코드 컴파일/빌드 없이 YAML 수정만으로 UI/로직 변경 즉시 반영
- **24,000줄 YAML:** 동급 기능의 커스텀 어드민 대비 훨씬 적은 코드량으로 복잡한 운영 기능 구현

### 운영 효율화
- **AI 배정 대시보드:** 관리자가 AI 배정 결과를 시각적으로 확인하고 즉시 조정 가능 → **배정 확정 시간 단축**
- **세탁 CS 워크플로우:** 접수 → 처리 → 결과 보고 전 과정 디지털화 → **처리 누락 방지**
- **정산 자동 생성:** 월별 정산 명세서를 데이터 기반 자동 생성 → **정산 오류 감소**

### 접근성 & 보안
- **RBAC 적용:** 관리자/파트너/고객/세탁사별 역할 기반 접근 제어로 **데이터 보안 강화**
- **사용자 가이드 내장:** 12개 가이드 페이지로 별도 교육 없이 즉시 사용 가능
- **릴리즈 노트 관리:** 버전별 변경사항 추적으로 **변경 관리 체계화**
