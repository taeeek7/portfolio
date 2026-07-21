# Portfolio - 숙박/청소 운영 플랫폼 엔지니어링

> 17개 도메인으로 정리한 기술 포트폴리오

---

## 프로젝트 개요

숙박/청소 운영 관리 플랫폼(Keeper)의 **백엔드, 데이터 파이프라인, 인프라, 챗봇, 프론트엔드**를 설계하고 구축한 프로젝트 모음입니다.
2026년부터는 **AI 에이전트·프로토타이핑 기반의 기획 워크플로우**로 역할을 확장했습니다.

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
| 09 | [Text-to-SQL](./09-text-to-sql/) | task-keeper (feature branch) | 자연어 → SQL 질의 시스템 *(학습용 실험)* | LangChain, ChromaDB, Gemini, FastAPI |
| 10 | [UX & i18n](./10-ux-i18n/) | 키퍼 서비스 전체 | UX 라이팅 250+ 개선 + 10,000+ 문자열 다국어 자동화 | Lokalise, Python, CLI Automation |
| 11 | [Keeper Agent](./11-keeper-agent/) | keeper-agent | 사내 지식을 MCP로 통합 조회하는 AI 에이전트 | MCP, FastAPI, Claude, Python 3.13 |
| 12 | [Keeper Prototype](./12-keeper-prototype/) | keeper-prototype | AI 기반 기획-프로토타이핑 사이클 | React 19, TypeScript, Vite, shadcn/ui |
| 13 | [AI Agent Platform](./13-ai-agent-platform/) | .claude/{agents,skills,mcp-servers} | 반복업무 위임 에이전트 생태계 (16 에이전트 + 21 스킬) | Claude Code, MCP, Python |
| 14 | [Log Lakehouse](./14-log-lakehouse/) | kafka, kube-airflow | 일 300만 건 로그 장기보관·분석 파이프라인 | Kafka, Airflow, GCS, Parquet, BigQuery |
| 15 | [n8n Automation](./15-n8n-automation/) | integrations/n8n | 로우코드 운영 워크플로우 자동화 (live/test 이중화) | n8n, Webhook, Google Sheets, Slack |
| 16 | [Channel Talk](./16-channel-talk/) | integrations/channel-talk | CS 문의 태그 기반 자동 라우팅 | AWS Lambda, DynamoDB, Slack API |
| 17 | [Analytics](./17-analytics/) | integrations/{google-analytics,redash} | 가설 기반 GA4 이벤트 설계 + 전사 대시보드 | GA4, BigQuery, Redash, Python |

---

## 역할 변화

```
2023 ─────────────── 2025 ─────────────── 2026 ───────▶
운영 자동화        데이터 엔지니어링       AI · 프로덕트 기획
(02,07,08)      (01,03,04,05,14)       (11,12,13,17)
```

| 시기 | 중심 역할 | 대표 프로젝트 |
|------|-----------|---------------|
| **2023~2024** | 운영 자동화 | Slack 봇 3종, 스크럼 봇, 운영 백오피스 |
| **2024~2025** | 데이터 엔지니어링 | Airflow 파이프라인, Kafka CDC, GitOps 인프라, 로그 Lakehouse |
| **2026~** | AI · 프로덕트 기획 | Keeper Agent, Prototype 사이클, 에이전트 생태계, 애널리틱스 |

---

## 기술 스택 전체 맵

### Languages & Frameworks
- **Python** 3.12~3.13 (FastAPI, Slack Bolt, Airflow DAGs, MCP)
- **Kotlin** 1.9 (Spring Boot 3.3, JDK 21)
- **TypeScript** 5.9 + **React** 19 (Vite, shadcn/ui, Tailwind v4)
- **JavaScript** ES6 (Vanilla, YAML 내장 로직)
- **YAML** (Helm Charts, Admin Console Config, CI/CD, 정책 정의)

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
- **MCP** (Model Context Protocol) — 사내 지식 에이전트 서버 구축·운영
- **Claude** (anthropic SDK, Claude Code 에이전트·스킬 설계)
- **Google Gemini** 2.0-flash / 2.5-pro (Text-to-SQL, 태스크 배정 최적화, intent routing)
- **LangChain** 1.1 + **ChromaDB** / **Qdrant** + **TEI** (RAG 기반 자연어 질의)
- **HuggingFace** Embeddings (`multilingual-e5`, 다국어 시맨틱 검색)
- **OpenAI API** (이미지 검수, 클레임 분석)
- **Figma MCP** (디자인 → 다국어 키 자동 생성)

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

### AI 기획 사이클 (2026~)
```
사내 지식 (API 스펙 / DB 스키마 / 정책 / 화면 IA)
    ↓ GitHub Actions 자동 현행화
Keeper Agent (MCP 서버) ──▶ Claude Code / Desktop
    ↓ 도메인 파악
기획 초안 (spec.md) ──▶ Keeper Prototype (동작하는 화면 + 정책서 split panel)
    ↓ 이해관계자 조기 검증
개발 착수 ──▶ GA4 이벤트로 가설 검증
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
7. **안전한 실패** - 폴백을 기본값으로. 자동화의 실패 모드가 "누락"이 아니라 "덜 정확한 처리"가 되게 설계
8. **권한 경계 우선** - AI 에이전트는 할 수 있는 것보다 **못 하게 할 것**을 먼저 정의 (advisor는 read-only)
9. **결정의 근거를 남긴다** - 아키텍처 선택뿐 아니라 **폐기한 결정과 그 이유**까지 기록
