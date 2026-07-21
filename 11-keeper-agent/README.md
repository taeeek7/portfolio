# Keeper Agent - 사내 AI 지식 에이전트 (MCP 서버)

> 흩어진 사내 지식(API 스펙·DB 스키마·서버 Enum·비즈니스 정책)을 MCP로 통합 조회
> 프로젝트: `keeper-agent` (Python 3.13 / uv 모노레포)

---

## 프로젝트 개요

사내에 흩어져 있던 **REST API 스펙, DB 스키마, 서버 Enum·엔티티 ID 규칙, 비즈니스 정책서, 어드민 화면 IA**를
하나의 지식베이스로 정리하고, 이를 **MCP(Model Context Protocol) 서버**로 노출해
Claude Code·Claude Desktop 등 AI 도구에서 자연어로 조회할 수 있게 만든 사내 AI 지식 에이전트입니다.

개발팀·기획자가 IDE를 벗어나지 않고 사내 도메인을 질의할 수 있으며,
서버 레포 배포 시 지식이 자동 현행화되어 "낡은 문서" 문제를 구조적으로 제거했습니다.

| 항목 | 내용 |
|------|------|
| **기간** | 2026.02 ~ 진행중 |
| **규모** | 141 커밋 (본인 134커밋, 1저자) |
| **운영** | NCP 배포, 개발팀 일상 도구로 정착 |
| **지식 범위** | 정책 7개 도메인 26개 파일 + 어드민 화면 IA 10개 그룹 + API/스키마/Enum 전체 |

---

## 1. 문제 정의

**문제 1 — 지식이 흩어지고 낡는다**

- API 스펙·DB 스키마·서버 상태값·비즈니스 정책이 위키와 코드에 분산되어 있고, 최신 상태가 유지되지 않음
- 기획·개발 시 매번 위키를 뒤지거나 담당자에게 물어야 했고, 오래된 문서를 근거로 잘못된 판단을 내릴 위험 존재

**문제 2 — AI 도구가 사내 도메인을 모른다**

- 정책 문서가 사람이 읽기 위한 산문 형태로만 존재해, AI 코딩 도구가 사내 도메인을 이해하지 못함
- 그 결과 **그럴듯하지만 틀린 결과(환각)** 를 생성 → 오히려 검증 비용이 증가

---

## 2. 문제 해결

### 2-1. 지식의 구조화와 단일화

사람이 읽는 산문을 **AI가 파싱하기 쉬운 구조화 포맷**으로 재정리했습니다.

```
knowledge/
├── policies/{domain}/{topic}/     # 비즈니스 정책 (7개 도메인)
│   ├── rules.yaml                 #   └ 기계가 읽는 규칙
│   └── notes.md                   #   └ 사람이 읽는 맥락
├── screens/                       # 어드민 화면·IA 가이드 (10개 그룹)
│   ├── _index.yaml                #   └ 전체 화면 맵
│   └── flows.md                   #   └ 화면 전환 흐름
├── sync/                          # 자동 생성 (git 추적)
│   ├── enums.json                 #   └ 서버 Kotlin Enum
│   └── entity_prefix.json         #   └ 엔티티 ID 프리픽스 규칙
└── private/                       # 민감 정보 (gitignore, 운영 마운트 전용)
```

**설계 의도**: `rules.yaml`(기계용) / `notes.md`(사람용)을 분리해, 하나의 정책이 AI와 사람 양쪽에 동시에 봉사하도록 했습니다.

### 2-2. ★ 핵심 의사결정 — 벡터 RAG를 걷어내고 "파일 직접 반환"으로 전환

초기에는 정석대로 **벡터 RAG**를 구축했습니다.

```
[초기 PoC] Confluence 문서 → 청킹 → TEI 임베딩 → Qdrant → 유사도 검색 → Gemini 답변 생성
```

그런데 정책서를 이 레포에 **파일로 깨끗하게 정리하고 나니 RAG의 주된 사용처가 사라졌습니다.**
문서가 이미 도메인·토픽별로 정돈되어 있는데, 굳이 벡터화해서 유사도로 근사할 이유가 없었습니다.

그래서 **아키텍처를 전환**했습니다.

```
[현재] MCP tool → knowledge/ 파일을 직접 read → JSON/텍스트 그대로 반환 → 검색·요약은 클라이언트 LLM이 수행
```

| 비교 | 벡터 RAG | 파일 직접 반환 (채택) |
|------|----------|----------------------|
| **환각 여지** | 임베딩 유사도가 엉뚱한 청크를 물어오면 그대로 오답 | 원문을 그대로 주므로 **서버가 지어낼 여지 없음** |
| **응답 속도** | 임베딩 + 벡터 검색 + LLM 생성 | 파일 read 1회 |
| **운영 비용** | TEI 컨테이너 + Qdrant Cloud 상시 유지 | 없음 |
| **정확도** | 청킹 경계에서 맥락 손실 | 문서 전체 맥락 보존 |

> **설계 철학**: "키퍼 에이전트는 **멍청한 파일 리더**로 남는다."
> 서버는 판단하지 않고 정확한 원문만 건넨다. 해석과 추론은 이미 그걸 잘하는 클라이언트 LLM에게 위임한다.
> 이것이 환각을 줄이는 가장 확실한 방법이라고 판단했습니다.

RAG 코드는 삭제하지 않고 **되살릴 수 있는 상태로 보존**했습니다. 지식 양이 늘어 자연어 검색이 다시 필요해지면
`/embed/*`, `intent_router.py`, TEI·Qdrant 연결을 복구하는 옵션을 남겨둔 것입니다.

### 2-3. 자동 현행화 파이프라인

"낡은 문서" 문제를 사람의 성실함이 아니라 **구조**로 해결했습니다.

```
server 레포 push
    ↓ GitHub Actions (sync-keeper-agent.yml)
Kotlin 소스 파싱 (Enum / RowIdPrefix 추출)
    ↓
POST /sync/knowledge  → knowledge/sync/*.json 갱신 → 자동 PR 생성
POST /sync/swagger    → swagger.json 갱신
POST /sync/schema     → schema.sql 갱신
```

- Sync 엔드포인트는 `202 Accepted` + `run_id`를 반환하고 **백그라운드로 처리** (긴 파싱 작업이 CI를 붙잡지 않도록)
- 동기화 결과가 기존과 동일하면 `_meta.updated`를 유지해 **노이즈 PR을 억제**

---

## 3. 아키텍처

```
┌──────────────────┐              ┌──────────────────────┐
│  Claude Code     │              │  GitHub Actions      │
│  Claude Desktop  │              │  (server 레포 push)  │
└────────┬─────────┘              └──────────┬───────────┘
         │ Bearer auth                       │ Bearer auth
         │ /mcp (stateless HTTP)             │ POST /sync/*
         ▼                                   ▼
┌─────────────────────────────────────────────────────────┐
│         FastAPI (packages/api, :8001)                    │
│                                                          │
│  ┌────────────────────┐   ┌──────────────────────────┐  │
│  │  MCP Server (/mcp) │   │  REST Routers            │  │
│  │  ├ search_policy   │   │  ├ /sync/*     (지식갱신)│  │
│  │  ├ search_api_spec │   │  ├ /knowledge/* (동일데이터)│
│  │  ├ search_schema   │   │  ├ /confluence/* (프록시)│  │
│  │  ├ search_server_  │   │  └ /embed/*  (RAG, 보존) │  │
│  │  │    enums        │   └──────────────────────────┘  │
│  │  ├ search_entity_  │                                  │
│  │  │    prefix       │            파일 직접 read        │
│  │  └ search_screen   │──────────────┐                   │
│  └────────────────────┘              ▼                   │
│                          ┌──────────────────────┐        │
│                          │  knowledge/          │        │
│                          │   policies/ screens/ │        │
│                          │   sync/    private/  │        │
│                          └──────────────────────┘        │
└─────────────────────────────────────────────────────────┘
```

### MCP Tools (6종)

| Tool | 반환 대상 |
|------|-----------|
| `search_policy` | `policies/{domain}/{topic}/` — rules.yaml + notes.md |
| `search_api_spec` | REST API Swagger 스펙 |
| `search_schema` | DB 스키마 DDL |
| `search_server_enums` | 서버 Enum (summary / detail 선택) |
| `search_entity_prefix` | 엔티티 ID 프리픽스 규칙 |
| `search_screen` | 어드민 화면·IA 가이드 (인자 없으면 전체 맵) |

### 모노레포 구성 (uv workspaces)

| 경로 | 역할 |
|------|------|
| `packages/api/` | FastAPI 서버 + MCP HTTP 서버 (메인 애플리케이션) |
| `packages/clients/slack/` | Slack 봇 클라이언트 (초기 PoC) |
| `packages/toolkit/` | Claude Code 스킬·에이전트 정의 |
| `infra/` | Docker 빌드, 운영 스크립트, 시크릿 |

### 보안 설계

- `/health` 제외 **전 엔드포인트 Bearer 토큰 인증**, MCP는 `BearerAuthMiddleware`로 한 번 더 검증
- 민감 데이터(`schema.sql`, `swagger.json`, 고객사 정보)는 `knowledge/private/`에 격리 → **gitignore + 운영 볼륨 마운트 전용**
- `knowledge/policies/`는 컨테이너에 **read-only 마운트**하여 런타임 변조 차단

---

## 4. 기술 스택

| 영역 | 기술 |
|------|------|
| **Language** | Python 3.13 |
| **Package/Workspace** | uv (workspaces 모노레포) |
| **Framework** | FastAPI + Uvicorn |
| **MCP** | FastMCP (stateless HTTP, `/mcp` mount) |
| **LLM** | Claude (anthropic SDK), Gemini 2.5 Flash (intent router) |
| **Embedding** | TEI + `intfloat/multilingual-e5-base` *(초기 RAG, 현재 보존)* |
| **Vector DB** | Qdrant Cloud *(초기 RAG, 현재 보존)* |
| **Validation** | Pydantic Settings (싱글톤 `.env` 로드) |
| **Test** | pytest |
| **Infra** | Docker (`keeper-net`), NCP Container Registry |
| **CI/CD** | GitHub Actions (git tag `v*` / `fastapi-v*` / `tei-v*` 트리거) |

---

## 5. 성과 / 효과

### 지식 접근성

- 비즈니스 정책 **7개 도메인 26개 파일** + 어드민 화면 IA **10개 그룹** + API·스키마·Enum 전체를 색인해 **사내 지식의 단일 조회 창구** 마련
- 위키 탐색·담당자 문의에 소요되던 **조회 비용 절감** — IDE를 벗어나지 않고 자연어로 질의
- 개발팀 **일상 도구로 정착** (MCP 연결 후 상시 사용)

### 정확도 / 신뢰성

- 벡터 RAG → 파일 직접 반환 전환으로 **서버 측 환각 가능성 제거** (원문 그대로 전달)
- 배포마다 GitHub Actions로 **자동 현행화** → 문서-코드 불일치를 구조적으로 차단
- AI 코딩 도구가 사내 도메인을 정확히 이해하게 되어 **기획·개발 산출물의 초기 정확도 향상**

### 운영 효율

- TEI 컨테이너 + Qdrant Cloud 상시 운영 부담 제거 (**인프라 비용·복잡도 절감**)
- 동일 내용 sync 시 PR 생성을 억제해 **리뷰 노이즈 감소**
- 정리된 정책 지식을 신규 기능 기획의 근거로 활용 → **정책 정합성 확인·영향 범위 파악** 속도 향상

---

## 6. 회고

**잘한 판단**

정석(벡터 RAG)을 구축한 뒤 **걷어낼 줄 안 것**. "RAG를 썼다"는 이력보다 "왜 안 쓰기로 했는지 설명할 수 있는 것"이
실제로는 더 어려운 결정이었습니다. 문제(흩어진 지식)를 데이터 정리로 해결하고 나니 기술적 복잡도가 필요 없어졌고,
이때 매몰비용에 끌려가지 않고 단순한 구조를 택한 것이 운영 부담과 환각을 동시에 줄였습니다.

**남은 과제**

- 정량 임팩트(질의량·사용자 수·절감 시간) 계측 체계가 없음 → 사용 로그 수집 필요
- Slack 봇 클라이언트가 초기 PoC 상태로 방치 — 현재 실사용 채널은 MCP 단일
