# Portfolio - 숙박/청소 운영 플랫폼 엔지니어링

> 14개 프로젝트를 10개 도메인으로 분류한 기술 포트폴리오

---

## 프로젝트 개요

숙박/청소 운영 관리 플랫폼(Keeper)의 **백엔드, 데이터 파이프라인, 인프라, 챗봇, 프론트엔드**를 설계하고 구축한 프로젝트 모음입니다.

---

## 프로젝트 목록

| # | 폴더 | 프로젝트 | 핵심 역할 | 기술 스택 |
|---|------|----------|-----------|-----------|
| 01 | [Data Pipeline](./01-data-pipeline/) | airflow, kube-airflow | 117개 DAG 데이터 파이프라인 자동화 | Airflow, Python, BigQuery, GCS |
| 02 | [Slack Bots](./02-slack-bots/) | request-bot, partner-request-bot, eleven-request-bot | 멀티 테넌트 업무 요청 접수 자동화 | Slack Bolt, Python, MySQL, SQS |
| 03 | [GitOps Infra](./03-gitops-infra/) | argocd-deploy, helm-deploy | 20개 서비스 K8s 배포 관리 | ArgoCD, Helm, K8s, SOPS |
| 04 | [Kafka Streaming](./04-kafka-streaming/) | kafka | CDC + 멀티 싱크 이벤트 스트리밍 | Kafka Connect, Debezium, GCS, BigQuery |
| 05 | [Backend API](./05-backend-api/) | data-api, task-keeper | AI 배정 최적화 + 운영 데이터 API | Kotlin/Spring Boot, Python/FastAPI, Gemini |
| 06 | [Admin Console](./06-admin-console/) | keeper-console, connect-admin | YAML 선언형 어드민 대시보드 | SelectFromUser, YAML, MySQL |
| 07 | [Scrumble](./07-scrumble/) | scrumble | 대화형 데일리 스크럼 자동화 | FastAPI, Slack Bolt, APScheduler |
| 08 | [Work Scheduler](./08-work-scheduler/) | work-scheduler | 키퍼 주간 일정 등록 웹앱 | Vanilla JS, Netlify |
| 09 | [Text-to-SQL](./09-text-to-sql/) | task-keeper (feature branch) | 자연어 → SQL 질의 시스템 | LangChain, ChromaDB, Gemini, FastAPI |
| 10 | [UX & i18n](./10-ux-i18n/) | 키퍼 서비스 전체 | UX 라이팅 250+ 개선 + 10,000+ 문자열 다국어 자동화 | Lokalise, Python, CLI Automation |

---

## 기술 스택 전체 맵

### Languages & Frameworks
- **Python** 3.12~3.13 (FastAPI, Slack Bolt, Airflow DAGs)
- **Kotlin** 1.9 (Spring Boot 3.3, JDK 21)
- **JavaScript** ES6 (Vanilla, YAML 내장 로직)
- **YAML** (Helm Charts, Admin Console Config, CI/CD)

### Data & Messaging
- **Apache Airflow** 2.9.3 (CeleryExecutor, 117 DAGs)
- **Apache Kafka** Connect 7.9.4 (Debezium CDC, GCS/BigQuery/OpenSearch Sink)
- **Google BigQuery** (데이터 웨어하우스)
- **Google Cloud Storage** (로그 아카이빙, Parquet 변환)
- **AWS SQS** (비동기 메시지 큐)
- **MySQL** (운영 DB, 멀티 인스턴스)
- **PostgreSQL** (Airflow 메타데이터)
- **Redis** (Celery Broker, 캐시)

### Infrastructure & DevOps
- **Kubernetes** (NCloud NKS)
- **ArgoCD** (GitOps 자동 배포 + Self-Healing)
- **Helm v3** (20개 서비스 패키지 관리)
- **Docker** (전 서비스 컨테이너화)
- **GitHub Actions** (CI/CD 파이프라인)
- **SOPS + Age** (시크릿 암호화)
- **Naver Cloud Platform** (NKS, NCR, VPC DB)
- **AWS** (RDS, SQS, S3)

### AI/ML
- **Google Gemini** 2.0-flash / 2.5-pro (Text-to-SQL, 태스크 배정 최적화)
- **LangChain** 1.1 + **ChromaDB** (RAG 기반 자연어 질의)
- **HuggingFace** Embeddings (다국어 시맨틱 검색)
- **OpenAI API** (이미지 검수, 클레임 분석)
- **Dalpha AI** (배정 최적화 모델)

### Communication & Monitoring
- **Slack SDK/Bolt** (Socket Mode 봇 4개 운영)
- **Ncloud AlimTalk** (SMS 알림)
- **Whatap** (APM + Browser RUM)
- **Google Sheets API** (자동 리포팅)
- **Lokalise** (10,000+ 문자열 다국어 로컬라이제이션)

---

## 아키텍처 하이라이트

### End-to-End 데이터 흐름
```
MySQL (운영 DB)
    ↓ Debezium CDC (실시간)
Kafka Topics
    ↓ Sink Connectors
BigQuery / GCS / OpenSearch
    ↓ Airflow ETL (117 DAGs)
Google Sheets / Slack / AlimTalk
```

### GitOps 배포 파이프라인
```
Code Push → GitHub Actions → Docker Build → NCloud NCR
    → ArgoCD GitOps Repo Update → PR (automerge)
    → ArgoCD Sync → K8s Rolling Update → Self-Heal
```

### Slack 봇 요청 흐름
```
User @mention → 2-Stage Modal → Reflection 기반 동적 폼
    → MySQL 저장 → SQS 큐잉 → Airflow DAG 트리거
    → 담당자 리액션 → 결과 Modal → Thread 답글
```

---

## 핵심 설계 원칙

1. **GitOps First** - Git을 Single Source of Truth로, ArgoCD로 자동 배포
2. **Event-Driven** - SQS, Kafka, Webhook으로 시스템 간 느슨한 결합
3. **Configuration as Code** - Helm values, YAML Admin, Proto 정의로 선언적 관리
4. **Secret Management** - SOPS + Age로 시크릿을 암호화하여 Git 추적 가능
5. **Multi-Environment** - live/test/staging 환경 분리, 동일 코드베이스
6. **Observability** - Slack 알림, Whatap APM, Airflow 실패 모니터링
