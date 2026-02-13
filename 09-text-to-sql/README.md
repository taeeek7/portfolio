# Text-to-SQL 자연어 질의 시스템

> LangChain + ChromaDB + Gemini 기반 자연어 → SQL → 자연어 응답 파이프라인
> 프로젝트: `task-keeper` (feature/langchain-model 브랜치)

---

## 프로젝트 개요

운영 담당자가 **자연어로 데이터를 조회**할 수 있는 Text-to-SQL 시스템.
"고양이천국 지점의 오늘 대기중인 업무 조회해줘" 같은 질문을 SQL로 변환하고, 결과를 자연어로 요약하여 응답합니다.

20개 테이블 스키마와 250+ 엔티티 매핑을 Vector DB에 저장하여 정확한 SQL을 생성합니다.

---

## 기술 스택

| 영역 | 기술 |
|------|------|
| **Framework** | FastAPI + Uvicorn |
| **LLM** | Google Gemini 2.0-flash (SQL 생성: temp 0.0, 응답 요약: temp 0.3) |
| **Orchestration** | LangChain 1.1.0 |
| **Vector DB** | ChromaDB 1.3.5 (Persistent Storage) |
| **Embeddings** | HuggingFace `paraphrase-multilingual-MiniLM-L12-v2` |
| **Database** | MySQL (운영 DB 20개 테이블) |
| **Validation** | Pydantic |
| **Language** | Python 3.12 |

---

## 아키텍처

```
User Question (자연어)
    ↓
┌──────────────────────────────────────────────┐
│  FastAPI Endpoint (/v2/text-to-sql)          │
│                                               │
│  1. VectorDB 검색 (ChromaDB, k=5)            │
│     ├─ 관련 테이블 스키마 검색               │
│     └─ 엔티티/코드 매핑 검색                 │
│                                               │
│  2. 컨텍스트 조립                             │
│     ├─ System Prompt (ERD, 쿼리 패턴)        │
│     ├─ 검색된 스키마 정보                     │
│     └─ 엔티티 매핑 (지점명→ID, 상태코드 등)  │
│                                               │
│  3. Gemini API → SQL 생성 (temp: 0.0)        │
│                                               │
│  4. MySQL 실행 (SELECT only)                  │
│                                               │
│  5. Gemini API → 자연어 응답 (temp: 0.3)     │
│                                               │
│  6. Background: chat_history 저장             │
└──────────────────────────────────────────────┘
    ↓
Response: { question, answer, sql, data, row_count }
```

---

## 핵심 기능 및 해결한 문제

### 1. 시맨틱 스키마 검색 (RAG)

**문제:** 20개 테이블 전체를 LLM에 넣으면 토큰 낭비 + 정확도 저하
**해결:**
- ChromaDB에 테이블 스키마를 벡터화하여 저장
- 질문과 가장 관련 높은 **상위 5개 스키마**만 검색하여 컨텍스트에 포함
- HuggingFace 다국어 임베딩 모델로 한국어 질문 지원

### 2. 엔티티 매핑 (자연어 → DB 값)

**문제:** "고양이천국", "데이빗" 같은 자연어를 DB의 ID/코드로 변환해야 함
**해결:**
```
[지점 매핑]
"고양이천국" → RGRP01KA05VQZRXVGN6K8CJV4BX672
"비카호텔"   → RGRP01KA0AKZHTDVDJ8J1MRQWT2B4V

[상태 매핑]
"대기중" → PENDING
"배정됨" → ASSIGNED
"진행중" → STARTED
"완료"   → RESOLVED

[업무 유형]
"일반"   → TICKET_TYPE1
"검수"   → TICKET_TYPE4
```
- 250+ 엔티티 매핑을 VectorDB에 저장
- 질문에서 자연어 → DB 값 자동 변환

### 3. 프롬프트 엔지니어링

**문제:** LLM이 잘못된 JOIN, 존재하지 않는 컬럼 참조
**해결:**
- ERD 관계를 시스템 프롬프트에 명시 (예: ticket에 room_group_id 없음 → room 경유 필수)
- 컬럼 네이밍 규칙 명시 (키퍼 이름 = `display_name`, NOT `name`)
- 6개 이상 자주 쓰는 쿼리 패턴을 예시로 제공
- Soft Delete 안전장치: `WHERE deleted_at IS NULL` 항상 포함

### 4. 메타데이터 자동 생성 파이프라인

**문제:** 20개 테이블의 컬럼 설명을 수동으로 작성하기 어려움
**해결:**
```
Phase 1: Schema Sync
  INFORMATION_SCHEMA → metadata 테이블 (테이블/컬럼 자동 동기화)

Phase 2: LLM Enhancement
  NULL description → Gemini가 의미 있는 설명 자동 생성

Phase 3: Build Embedding
  metadata → JSON → ChromaDB VectorDB 빌드

Phase 4: Rebuild VectorDB
  JSON 파일 → LangChain Documents → ChromaDB 인덱스
```
- 배치 API로 전체 파이프라인 자동화
- 신규 테이블 추가 시 재빌드만으로 반영

### 5. 안전한 쿼리 실행

**문제:** LLM이 DELETE, UPDATE 등 위험한 쿼리를 생성할 수 있음
**해결:**
- SELECT 쿼리만 실행 허용
- SQL 생성 temperature 0.0으로 결정적(deterministic) 출력
- 응답 데이터 최대 10행만 LLM에 전달 (환각 방지)
- 모든 쿼리를 chat_history에 비동기 로깅

---

## 지원 테이블 (20개)

| 도메인 | 테이블 | 설명 |
|--------|--------|------|
| **업무 관리** | ticket | 업무(청소, 검수 등) |
| | ticket_request | 업무 템플릿/정의 |
| | ticket_request_group | 업무 요청 그룹핑 |
| | ticket_keeper_assign | 키퍼 배정 |
| | ticket_report | 이슈/문제 보고 |
| **공간 관리** | room | 개별 객실 |
| | room_group | 지점/시설 |
| | room_category | 객실 유형 |
| | room_item_group | 소모품/비품 |
| | room_property_group | 고정 설비 |
| **키퍼 관리** | keeper | 작업자 |
| | keeper_group | 작업팀 |
| | keeper_registration | 등급 배정 |
| | keeper_group_registration | 팀 소속 |
| | keeper_level | 등급 정의 |
| | keeper_level_group | 등급 체계 |
| **반복 업무** | recurring_rule | 반복 규칙 |
| | recurring_rule_room_registration | 반복 대상 객실 |
| **조직** | staff | 관리자 |
| | workspace | 워크스페이스 |

---

## 성과 및 효과

### 데이터 접근성 향상
- **비개발자 데이터 조회:** SQL을 모르는 운영 담당자도 자연어로 실시간 데이터 조회 가능
- **조회 시간 단축:** 개발팀에 데이터 요청 후 대기(수 시간~1일) → **자연어 질의 즉시 응답(수 초)**
- **20개 테이블 커버:** 업무/공간/키퍼/보고서 등 운영에 필요한 핵심 데이터 전체 지원

### 기술적 성과
- **RAG 기반 정확도:** 20개 테이블 중 관련 스키마만 선별하여 SQL 생성 정확도 향상
- **250+ 엔티티 매핑:** 한국어 자연어 → DB ID/코드 자동 변환으로 실용적 질의 가능
- **메타데이터 자동화:** LLM 기반 스키마 설명 자동 생성으로 유지보수 비용 최소화

### 안정성
- **SELECT Only:** 읽기 전용 쿼리만 허용하여 운영 DB 안전성 보장
- **Temperature 0.0:** 동일 질문 → 동일 SQL 보장 (결정적 출력)
- **실행 시간 추적:** 모든 질의를 chat_history에 기록하여 성능 모니터링 가능