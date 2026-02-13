# Backend API 서버

> 운영 데이터 관리 + AI 기반 태스크 배정 최적화 API
> 프로젝트: `data-api` (Kotlin/Spring Boot), `task-keeper` (Python/FastAPI)

---

## 프로젝트 개요

숙박/청소 운영 시스템의 핵심 백엔드 API 서버 2종.
**data-api**는 Airflow 및 외부 시스템과의 데이터 연동을 담당하고, **task-keeper**는 AI 기반 태스크 배정 최적화와 데일리 스크럼 봇 기능을 제공합니다.

| 프로젝트 | 역할 | 주요 클라이언트 |
|----------|------|----------------|
| **data-api** | 운영 데이터 CRUD + Protocol Buffers API | Airflow, Work Scheduler |
| **task-keeper** | AI 배정 최적화 + Daily Scrum Bot | Keeper Console, Slack |

---

## 기술 스택

### data-api

| 영역 | 기술 |
|------|------|
| **Language** | Kotlin 1.9.25 (JDK 21) |
| **Framework** | Spring Boot 3.3.3 |
| **ORM** | Spring Data JPA + Hibernate |
| **Database** | MySQL (NCloud VPC) |
| **Serialization** | Protocol Buffers (protobuf-java-util 4.26.1) |
| **HTTP Client** | OkHttp3 |
| **Encryption** | Fernet-Java8 |
| **Build** | Gradle 8.10 |
| **Runtime** | Amazon Corretto 21 Alpine |

### task-keeper

| 영역 | 기술 |
|------|------|
| **Language** | Python 3.12 |
| **Framework** | FastAPI + Uvicorn |
| **Database** | MySQL (NCloud + AWS RDS, 멀티 DB) |
| **AI** | Google Gemini 2.5-pro, OpenAI API |
| **Slack** | Slack Bolt (Socket Mode) |
| **Scheduling** | APScheduler |
| **Vector DB** | ChromaDB |
| **Validation** | Pydantic |
| **Secret Management** | SOPS + Age |

---

## 아키텍처

### data-api (Layered Architecture)

```
HTTP Request (Protocol Buffers)
    ↓
┌──────────────────────────────────┐
│  Controller Layer (17 endpoints) │
│  - HealthCheck, Assignment,      │
│    Linen, Schedule, Branch...    │
└──────────────┬───────────────────┘
               ↓
┌──────────────────────────────────┐
│  Service Layer (12 services)     │
│  - TaskAssignmentOptimization    │
│  - RentalLinenTransaction        │
│  - WorkSchedules, Token...       │
└──────────────┬───────────────────┘
               ↓
┌──────────────────────────────────┐
│  Repository Layer (16 repos)     │
│  - Spring Data JPA               │
│  - Custom QueryDSL impl          │
└──────────────┬───────────────────┘
               ↓
┌──────────────────────────────────┐
│  Entity Layer (20 entities)      │
│  - JPA Entities → MySQL Tables   │
└──────────────────────────────────┘
```

### task-keeper (Versioned API)

```
HTTP Request / Slack Event
    ↓
┌──────────────────────────────────┐
│  Router Layer                    │
│  v1/ assignments, laundrys       │
│  v2/ assignments, presigned,     │
│      daily_scrum                 │
└──────────────┬───────────────────┘
               ↓
┌──────────────────────────────────┐
│  Service Layer                   │
│  v1/ assignment_service          │
│      (Gemini AI 연동)            │
│  v2/ daily_scrum_service         │
│      (Slack DM 대화형 봇)        │
│      presigned_url_service       │
└──────────────┬───────────────────┘
               ↓
┌──────────────────────────────────┐
│  Repository Layer (DAO)          │
│  - Raw SQL + mysql-connector     │
│  - Multi-DB (cleanops, keeper,   │
│    biz, eleven)                  │
└──────────────────────────────────┘
```

---

## 핵심 기능 및 해결한 문제

### 1. AI 기반 태스크 배정 최적화 (task-keeper)

**문제:** 수백 개 객실을 수십 명 키퍼에게 수동 배정 → 비효율적 분배
**해결:**
```
1. DB에서 키퍼 정보 + 티켓(객실) 정보 조회
2. Gemini 2.5-pro에 프롬프트 + 데이터 전송
   - 제약조건: 객실 면적, 층수, 건물, 체크인/아웃 시간
   - 가중치: 이동거리, 업무량 균등 분배
3. AI 최적 배정 결과 수신
4. 패널티 점수 계산 (면적 편차, 층 이동, 업무량 초과)
5. 결과를 assign_ai_result, assign_ai_penalty_score 테이블에 저장
6. 확정 시 assign_ai_adjustment_result로 반영
```

**프롬프트 관리:** `/prompt/assignment_v4.txt`로 버전 관리

### 2. 외부 AI 서비스 연동 (data-api)

**문제:** AI 최적화 모델 실행 시 긴 응답 시간
**해결:**
```kotlin
// 비동기 폴링 패턴
fun executeOptimization() → 외부 AI 모델 실행 요청
fun getOptimizationResult() → 최대 100회 재시도 폴링
fun updateFinalAssignmentResult() → 확정 결과 반영
```
- OkHttp3 Read Timeout: 120초
- 폴링 기반 비동기 결과 조회
- 실행 버전 관리로 여러 번 실행 비교 가능

### 3. Protocol Buffers API 설계 (data-api)

**문제:** Airflow와의 데이터 교환에서 타입 안전성 부족
**해결:**
- 11개 .proto 파일로 API 계약 정의
- 요청/응답 스키마 명시적 정의
- 언어 독립적 직렬화 (Python Airflow ↔ Kotlin API)

| Endpoint | Proto | 용도 |
|----------|-------|------|
| `/ExecuteAssignmentOptimization` | InsertAssignmentOptimization.proto | AI 배정 실행 |
| `/InsertWorkSchedules` | InsertWorkSchedules.proto | 근무 일정 등록 |
| `/InsertLinenTransactionLog` | InsertLinenTransactionLog.proto | 린넨 거래 기록 |
| `/GetBranchInfo` | GetBranchInfo.proto | 지점 정보 조회 |
| `/GetScheduleToken` | GetScheduleToken.proto | 일정 토큰 조회 |

### 4. Fernet 토큰 암/복호화 (data-api)

**문제:** 키퍼 일정 등록 링크의 보안
**해결:**
- Fernet 대칭 암호화로 토큰 생성/검증
- Work Scheduler 프론트엔드에서 토큰으로 인증
- 토큰 ↔ memberKeeperId 매핑

### 5. 멀티 데이터베이스 전략 (task-keeper)

**문제:** 여러 서비스의 DB를 하나의 API에서 접근
**해결:**
```python
# ENV 기반 자동 전환
if ENV == "live":
    ops_db = "운영 DB (NCloud VPC)"
    keeper_db = "키퍼 DB (AWS RDS)"
    biz_db = "비즈니스 DB"
    butler_db = "버틀러 DB"
else:
    # test 환경 DB 자동 선택
```

| DB | 용도 | 인프라 |
|----|------|--------|
| 운영 DB | 청소 운영 데이터 | NCloud VPC |
| 키퍼 DB | 키퍼 관리 | AWS RDS |
| 비즈니스 DB | 트랜잭션 데이터 | NCloud VPC |
| 버틀러 DB | 서비스 데이터 | NCloud VPC |

### 6. 이미지 Presigned URL (task-keeper)

**문제:** 청소 전/후 사진 업로드를 위한 안전한 URL 필요
**해결:**
- NCloud Object Storage Presigned URL 생성
- boto3 기반 S3 호환 API
- CS(고객 세그먼트)별 이미지 추적

---

## 배포 파이프라인

### data-api (Multi-stage Docker)
```dockerfile
# Build Stage: gradle:jdk21-corretto
./gradlew clean build -x test → bootJar

# Runtime Stage: amazoncorretto:21-alpine
COPY app.jar → Port 8089
```

### task-keeper (Python + SOPS)
```dockerfile
# python:3.12-slim
SOPS decrypt → config.py 생성
uvicorn main:app → Port 8000
```

공통: GitHub Actions → NCloud NCR → ArgoCD GitOps 배포

---

## 성과 및 효과

### AI 배정 최적화 효과
- **배정 시간 단축:** 수십 명 키퍼 × 수백 개 객실의 수동 배정(30분~1시간) → **AI 자동 배정 2~3분으로 단축**
- **배정 품질 향상:** 패널티 점수 기반 최적화로 업무량 편차 최소화 → **키퍼 만족도 및 서비스 품질 향상**
- **프롬프트 버전 관리:** v1~v4 반복 개선으로 AI 배정 정확도 지속 향상

### 시스템 연동 효과
- **Protocol Buffers 도입:** Python(Airflow) ↔ Kotlin(API) 간 타입 안전한 통신으로 **런타임 오류 감소**
- **멀티 DB 전략:** 4개 DB를 환경 변수 기반으로 자동 전환하여 **개발/운영 환경 전환 즉시 가능**
- **Fernet 토큰 인증:** 키퍼 개인 링크로 별도 로그인 없이 일정 등록 → **사용성 대폭 개선**

### 운영 안정성
- **비동기 폴링 패턴:** AI 모델 장시간 실행에도 안정적인 결과 수신 (최대 100회 재시도)
- **Multi-stage Docker:** 빌드 이미지 ≠ 런타임 이미지로 **컨테이너 크기 최적화**
- **Presigned URL:** 서버 부하 없이 클라이언트에서 직접 스토리지 업로드
