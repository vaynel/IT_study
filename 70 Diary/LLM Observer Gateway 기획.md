# 🚀 프로젝트 기획서: LLM Observability Gateway (가칭: Prompt-Shield)

## 1. 프로젝트 개요 (Project Overview)
**Prompt-Shield**는 개발자 및 보안 연구원이 일상적으로 사용하는 AI 코딩 도구(Cursor, Claude Code 등)와 외부 LLM API(OpenAI, Anthropic 등) 사이에 위치하는 **경량화된 지능형 프록시 게이트웨이**입니다. 사용자의 프롬프트 트래픽을 가로채어(Intercept) 민감 정보를 보호하고, 상호작용 로그를 Elasticsearch에 적재하여 프롬프트 효율성을 분석 및 최적화하는 것을 목표로 합니다.

## 2. 기획 배경 및 문제 정의 (Background & Problem Statement)
* **가시성 부재(Lack of Visibility):** CLI 도구나 IDE 내장 AI 에이전트를 사용할 때 발생하는 API 호출, 토큰 사용량, 지연 시간에 대한 추적이 어렵습니다.
* **프롬프트 자산화 실패:** "어떤 맥락(Context)을 제공했을 때 AI가 가장 좋은 코드를 생성했는지"에 대한 귀중한 데이터가 로컬 환경에서 휘발됩니다.
* **보안 위협(Security Risks):** 개발 과정에서 소스코드와 함께 사내 API Key, PII(개인식별정보) 등 민감 데이터가 무의식적으로 외부 LLM으로 전송될 위험이 존재합니다.

## 3. 핵심 목표 및 기대 효과 (Core Objectives)
1. **비용 및 성능 최적화:** 토큰 사용량 추적 및 지연 시간 모니터링을 통한 API 비용 통제.
2. **보안성 강화 (Security First):** 자체 서버(FastAPI)를 통한 데이터 전처리 및 민감 정보 마스킹 적용 후 안전하게 인덱싱.
3. **프롬프트 엔지니어링 피드백 루프:** 과거의 성공/실패 프롬프트 사례를 분석하여 더 나은 프롬프트 작성을 유도(LLM-as-a-Judge 도입 기반 마련).

---

## 4. 시스템 아키텍처 및 기술 스택 (Architecture & Tech Stack)

### 4.1. 시스템 구성도 (Data Flow)
```text
[ Client (Cursor, Claude Code) ]
       │ (1) Proxy Request (Base URL = VM_IP:4000)
       ▼
[ LiteLLM Proxy (Router & Gateway) ] ── (2) Relay ──▶ [ External LLM APIs ]
       │
       │ (3) Async Webhook (Success/Fail Events)
       ▼
[ FastAPI (Log Processor & Security) ]
       │ (4) Data Parsing, Masking, and Formatting
       ▼
[ Elasticsearch 8.x (Vector & Text DB) ] ◀── [ Kibana 8.x (Dashboard) ]
```

### 4.2. 기술 스택 (Tech Stack)
* **Infrastructure:** OpenStack VM (Ubuntu 22.04 / 4 Core / 8GB RAM / 100GB Disk)
* **Containerization:** Docker & Docker Compose
* **API Gateway / Proxy:** LiteLLM (LLM 라우팅 및 규격 통일)
* **Backend Processor:** Python 3.11, FastAPI, Uvicorn (비동기 데이터 가공 및 웹훅 수신)
* **Data Layer & BI:** Elasticsearch 8.x, Kibana 8.x (로그 적재, 시맨틱 검색, 시각화)

---

## 5. 핵심 기능 명세 (Core Features)

### Phase 1: 로깅 및 모니터링 (PoC 핵심)
* **통합 엔드포인트 제공:** 모든 로컬 AI 툴의 API 엔드포인트를 LiteLLM(포트 4000)으로 일원화.
* **비동기 로그 수집:** LiteLLM의 웹훅을 FastAPI가 수신하여 LLM 응답 지연 없이 백그라운드에서 로그 처리.
* **통계 대시보드:** Kibana를 활용한 모델별 토큰 사용량, 지연 시간, 요청 횟수 시각화.

### Phase 2: 데이터 전처리 및 보안 강화
* **민감 데이터 마스킹:** FastAPI 프로세서 내에 정규식/NER(개체명 인식) 로직을 추가하여 비밀번호, IP 주소, API 키 등을 `[REDACTED]` 처리 후 ES에 저장.
* **프롬프트 메타데이터 추출:** 요청 도구(Cursor, Claude), 프로젝트명, 사용자 ID 등의 메타데이터 자동 분류.

### Phase 3: 분석 및 AI 피드백 루프 (Advanced)
* **시맨틱 클러스터링:** Elasticsearch의 `semantic_text` 기능을 활용하여 유사한 의도를 가진 프롬프트 군집화.
* **품질 자동 평가 (LLM-as-a-Judge):** 특정 주기마다 백그라운드 워커가 로그를 분석하여 AI 응답의 품질 점수를 산정하고 개선 리포트 생성.

---

## 6. 마일스톤 및 일정 계획 (Milestones)

* **[Milestone 1] PoC 인프라 구축 및 연동 테스트 (W1)**
  * OpenStack VM 생성 및 Docker 환경 세팅.
  * `docker-compose.yml` 작성 (LiteLLM, FastAPI, ES, Kibana).
  * Claude Code를 이용한 로컬 연동 및 Elasticsearch 인덱싱 확인.
* **[Milestone 2] 데이터 파이프라인 고도화 (W2)**
  * FastAPI 내 데이터 마스킹 로직(정규식 등) 구현.
  * Kibana 대시보드(Data View) 구성 및 대시보드 패널 생성.
* **[Milestone 3] 평가 및 시맨틱 텍스트 적용 (W3)**
  * Elasticsearch 인덱스 매핑 최적화 (벡터 필드 추가).
  * 과거 프롬프트 기반 유사도 검색 테스트.

---

## 7. 향후 확장성 (Future Roadmap)
* **팀 단위 도입 (B2B/Team Ops):** 다수의 개발자가 접속할 수 있도록 Proxy 서버에 API Key 발급 및 RBAC(역할 기반 접근 제어) 시스템 추가.
* **Slack / Discord 에이전트 연동:** 슬랙 봇을 만들어 사용자 피드백(👍/👎)을 직접 수집하고 이를 Elasticsearch 로그와 조인하여 평가 데이터 보강.
* **오픈소스 패키징:** 누구나 쉽게 로컬 환경에 띄울 수 있도록 CLI 설치 스크립트(`curl -sSL ... | bash`) 제공 및 GitHub 공개.
```