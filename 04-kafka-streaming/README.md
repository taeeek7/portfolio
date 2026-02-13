# Kafka Connect 이벤트 스트리밍 플랫폼

> CDC + Data Sink를 위한 Kafka Connect 관리 시스템
> 프로젝트: `kafka`

---

## 프로젝트 개요

MySQL CDC(Change Data Capture)로 실시간 데이터 변경을 감지하고, BigQuery/GCS/OpenSearch로 스트리밍하는 **이벤트 기반 데이터 파이프라인**.
Python REST API 클라이언트로 커넥터를 프로그래밍 방식으로 관리하며, SOPS 암호화로 시크릿을 안전하게 버전 관리합니다.

---

## 기술 스택

| 영역 | 기술 |
|------|------|
| **Streaming** | Kafka Connect 7.9.4 (Confluent Platform) |
| **Source Connector** | Debezium MySQL 2.7.3 (CDC) |
| **Sink Connectors** | GCS (Aiven), BigQuery (WePlay), OpenSearch |
| **Management** | Python 3.13 REST API Client |
| **Secret Management** | SOPS + Age 암호화 |
| **Infra** | Docker, NCloud NKS, GitHub Actions |
| **Deployment** | ArgoCD GitOps |

---

## 아키텍처

```
┌──────────────┐     ┌─────────────────────────────┐
│   MySQL DB   │     │     Kafka Connect 7.9.4     │
│  (운영 DB)  │────▶│                             │
│              │ CDC │  Debezium MySQL Connector    │
└──────────────┘     │  (Source - INSERT/UPDATE/    │
                     │   DELETE 실시간 감지)        │
                     └──────────┬──────────────────┘
                                │
                     ┌──────────▼──────────────────┐
                     │       Kafka Topics           │
                     │  debezium.{db-name}.*        │
                     │  prod-{service}-logs         │
                     │  (서비스별 토픽 분리)        │
                     └──┬──────────┬──────────┬────┘
                        │          │          │
               ┌────────▼──┐ ┌────▼─────┐ ┌──▼────────┐
               │  GCS Sink │ │ BigQuery │ │ OpenSearch │
               │  (Aiven)  │ │  Sink    │ │   Sink    │
               │           │ │ (WePlay) │ │           │
               │ JSONL+gzip│ │ Auto DDL │ │ Real-time │
               │ 날짜 계층 │ │ Schema   │ │ Indexing  │
               └───────────┘ └──────────┘ └───────────┘
                    │              │              │
                    ▼              ▼              ▼
               ┌─────────┐ ┌──────────┐ ┌───────────┐
               │  GCS    │ │ BigQuery │ │ OpenSearch │
               │ Bucket  │ │ Dataset  │ │  Cluster  │
               └─────────┘ └──────────┘ └───────────┘
```

---

## 핵심 기능 및 해결한 문제

### 1. MySQL CDC (Change Data Capture)

**문제:** 운영 DB 변경사항을 실시간으로 다른 시스템에 전파 불가
**해결:**
- Debezium MySQL Connector로 binlog 기반 실시간 변경 감지
- INSERT/UPDATE/DELETE 이벤트를 Kafka 토픽으로 자동 발행
- 스키마 변경 자동 추적 (internal Kafka topics)
- 트랜잭션 메타데이터 포함

### 2. 멀티 싱크 데이터 스트리밍

**문제:** 하나의 데이터 소스를 여러 분석 시스템에 분산 적재
**해결:**

| Sink | 포맷 | 용도 |
|------|------|------|
| **GCS** | JSONL + gzip | 서버 로그 아카이빙 (topic/YYYY/MM/DD/) |
| **BigQuery** | Auto-schema | 실시간 분석 쿼리 |
| **OpenSearch** | JSON | 실시간 로그 검색/인덱싱 |

- GCS: 날짜 계층 구조로 자동 파티셔닝
- BigQuery: 테이블 자동 생성 + 스키마 매핑
- 각 싱크 독립 운영 (하나 실패해도 다른 싱크에 영향 없음)

### 3. Python REST API 커넥터 관리

**문제:** Kafka Connect 커넥터 수동 관리 (curl 반복)
**해결:**
```python
class ConnectorRestApi:
    def create_connector(name, config)    # POST /connectors
    def update_connector(name, config)    # PUT /connectors/{name}/config
    def get_status(name)                  # GET /connectors/{name}/status
    def delete_connector(name)            # DELETE /connectors/{name}
    def list_plugins()                    # GET /connector-plugins
```
- 3개 환경(local/test/live) 엔드포인트 관리
- JSON 설정 파일 기반 일괄 배포
- 커넥터 상태 모니터링

### 4. 환경별 독립 설정

**문제:** 환경별 커넥터 설정 혼재로 인한 실수
**해결:**
```
connectors-local/  (13 configs) → localhost:8083
connectors-test/   (14 configs) → test 환경 엔드포인트
connectors-live/   (prod configs) → 운영 환경 엔드포인트
```
- 환경별 독립 Docker 이미지 (JVM 힙 차등 설정)
- 환경별 독립 Storage Topics (오프셋 격리)
- 환경별 Replication Factor 차등 (live: 3, test: 1)

### 5. SOPS 암호화 시크릿 관리

**문제:** DB 비밀번호, GCP 서비스 계정 키를 Git에 안전하게 저장
**해결:**
```
encrypt/                          secrets/
├── gcp-service-account.enc    → ├── gcp-service-account.json
├── (GCP 서비스 계정 등)        → ├── (복호화된 키 파일들)
├── database-credentials.enc   → ├── database-credentials.properties
└── search-engine-secrets.enc  → └── search-engine-secrets.properties
```
- CI/CD에서 SOPS_AGE_KEY로 자동 복호화
- Docker 이미지 내 `/etc/kafka-connect/secrets/`에 마운트
- Kafka Connect FileConfigProvider로 시크릿 참조

---

## 환경별 Docker 설정

| 환경 | JVM Heap | Group ID | Replication |
|------|----------|----------|-------------|
| **Local** | 2GB | kafka-connect-gcs-local | 1 |
| **Test** | 3.5GB | kafka-connect-gcs-sink | 1 |
| **Live** | 2.5GB | kafka-connect-gcs-sink-live | 3 |

---

## 배포 파이프라인

```
Git Push (test/main branch)
    ↓
GitHub Actions
    ├── SOPS CLI 설치
    ├── Age Key로 시크릿 복호화 (decrypt-all-secrets.sh)
    ├── Docker 이미지 빌드 (커넥터 플러그인 + 시크릿 포함)
    ├── NCloud NCR 푸시
    ↓
ArgoCD GitOps
    ├── argocd-deploy values 업데이트 (yq)
    ├── PR 생성 (automerge)
    ↓
Kubernetes 배포 → Kafka Connect 재시작
```

---

## 커넥터 플러그인

| 플러그인 | 버전 | 용도 |
|----------|------|------|
| Debezium MySQL | 2.7.3 | CDC Source Connector |
| Aiven GCS | 0.13.0 | GCS Sink (로그 아카이빙) |
| WePlay BigQuery | 2.7.0 | BigQuery Sink (분석) |
| OpenSearch | 3.1.1 | OpenSearch Sink (검색) |
| Confluent JDBC | 10.8.4 | JDBC Connector |
| Protobuf Converter | 7.9.0 | 직렬화 |

---

## 성과 및 효과

### 실시간 데이터 동기화
- **CDC 도입 효과:** 배치 ETL 대비 데이터 지연 시간 **수 시간 → 수 초**로 단축
- **멀티 싱크 동시 적재:** 하나의 변경 이벤트를 GCS/BigQuery/OpenSearch에 동시 전파 → **데이터 일관성 확보**
- **스키마 자동 추적:** DB 스키마 변경 시 수동 파이프라인 수정 불필요

### 운영 비용 절감
- **GCS 로그 아카이빙:** 날짜 계층 + gzip 압축으로 스토리지 비용 최소화
- **BigQuery 자동 스키마:** 테이블 수동 DDL 작업 제거
- **커넥터 프로그래밍 관리:** REST API 클라이언트로 13~14개 커넥터 일괄 관리 → 운영 시간 절감

### 확장성 & 안정성
- **싱크 독립 운영:** 하나의 싱크 장애가 다른 싱크에 영향 없는 격리 설계
- **환경 격리:** 환경별 독립 오프셋/Storage Topics로 테스트 ↔ 운영 완전 분리
- **HA 구성:** 운영 환경 Replication Factor 3으로 메시지 유실 방지
