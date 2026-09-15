# ADR-0012: Spring 1차 응급 하드스톱과 FastAPI RAG 답변 서비스 분리

- 원본: [Notion](https://app.notion.com/p/3bd499bdd63381ec9edde9cc46d5a13d)

## Context

AI 챗봇은 성형외과/피부미용 시술 후 환자 문의에 대해 응급 상황 여부를 먼저 구분하고, 응급이 아니라면 근거 기반 사후관리 답변을 제공해야 한다. 직전 구현에서는 Spring Boot가 KURE 임베딩 서버 호출, pgvector 검색, OpenAI 프롬프트 조립까지 직접 수행하는 구조였다.

이 구조는 빠르게 붙이기에는 단순하지만, 앞으로 DB를 무료 요금제 상황에 따라 Azure PostgreSQL, Supabase, Neon 등으로 계속 바꿔야 하고, 검색/임베딩/프롬프트 정책도 자주 바뀔 가능성이 높다. Spring이 이 세부 구현을 모두 알면 마이그레이션 때마다 제품 API 서버까지 함께 흔들린다.

## Decision

응급 판단은 Spring Boot와 FastAPI RAG 서비스 양쪽에 둔다.

Spring Boot는 1차 응급 hard-stop을 수행한다. 사용자 메시지를 저장한 뒤 RAG/LLM 호출 전에 룰 매칭을 실행하고, 위험 신호가 있으면 고정 안내문을 assistant 메시지로 저장한다.

FastAPI RAG 서비스는 2차 응급 hard-stop, KURE 임베딩, pgvector 검색, OpenAI 답변 생성을 담당한다. Spring은 `AI_CHAT_ANSWER_PROVIDER=rag-service`일 때 FastAPI의 `/v1/aftercare/answer`만 호출한다.

기존 Spring 내부 pgvector 검색 코드는 제거하고, OpenAI 직결 구현은 `AI_CHAT_ANSWER_PROVIDER=openai` 기본값으로 남겨 로컬/개발 fallback으로 사용한다. 운영에서는 RAG 서비스 장애 시 OpenAI로 자동 우회하지 않는 방향을 우선한다.

## Rationale

응급 판단은 서비스 신뢰성과 무관하게 반드시 동작해야 하는 안전 로직이다. RAG 서버가 죽거나 DB가 이전 중이어도 Spring API 서버가 살아 있다면 최소한의 위험 신호 안내는 반환해야 한다.

동시에 LLM/RAG 서버에도 같은 룰을 두어야 한다. Spring 쪽 룰이 누락되거나 향후 다른 클라이언트가 FastAPI를 직접 호출하더라도, 모델이 응급 증상을 일반 사후관리 답변으로 완화해서 말하지 않도록 막을 수 있다.

RAG 검색과 답변 생성은 변화가 잦은 영역이다. 임베딩 모델, 벡터 DB, 인덱스 버전, top-k, 프롬프트 정책은 FastAPI에 모으는 편이 교체와 실험이 쉽다.

## Options Considered

1. Spring이 응급 판단, 임베딩, pgvector 검색, OpenAI 답변 생성을 모두 담당한다.

   장점은 배포 단위가 하나라 단순하다는 점이다. 단점은 DB/임베딩/검색 정책 변경이 Spring 코드와 배포에 강하게 결합된다는 점이다.

2. FastAPI RAG 서비스가 응급 판단부터 답변 생성까지 모두 담당한다.

   장점은 AI 파이프라인이 한곳에 모인다는 점이다. 단점은 RAG 서비스 장애 시 응급 hard-stop까지 같이 죽을 수 있다는 점이다.

3. Spring 1차 응급 hard-stop + FastAPI 2차 응급 hard-stop/RAG/생성.

   선택한 방식이다. 제품 안전장치는 Spring에 남기고, AI 검색/생성 변경 가능성은 FastAPI에 격리한다.

## Consequences

운영 배포에서는 Spring과 FastAPI 두 서비스를 함께 띄워야 한다. `AI_CHAT_ANSWER_PROVIDER=rag-service`, `AI_CHAT_RAG_SERVICE_BASE_URL` 설정이 필요하다.

응급 룰이 두 곳에 존재하므로 장기적으로는 룰 소스를 단일화해야 한다. 현재 FastAPI는 `rag_rulebook/rules/emergency_rules.json`을 읽고, Spring은 동일 룰의 MVP 키워드 세트를 코드에 내장한다. 다음 단계에서는 Spring도 classpath JSON 또는 별도 설정 파일을 읽도록 바꾸는 것이 좋다.

RAG 서비스 장애 시 운영에서 OpenAI 자동 우회를 하지 않으면 답변 가능성은 낮아지지만, 근거 없는 의료 답변을 줄일 수 있다. 개발 환경에서는 `openai` provider로 빠른 확인이 가능하다.

## Implementation

Spring Boot:

- `AiChatService`에서 AI 호출 전 `AiChatEmergencyRuleService`를 실행한다.
- `FastApiRagChatAnswerService`를 추가해 FastAPI `/v1/aftercare/answer` 호출을 담당한다.
- `OpenAiChatAnswerService`는 `AI_CHAT_ANSWER_PROVIDER=openai`일 때만 활성화한다.
- Spring 내부 `aichat.rag` pgvector 검색 패키지는 제거한다.

FastAPI:

- `/health`로 RAG 인덱스, KURE 모델, 응급 룰 버전을 확인한다.
- `/v1/aftercare/answer`에서 응급 룰 검사 → KURE 임베딩 → `rag_documents` pgvector 검색 → OpenAI Responses API 답변 생성을 수행한다.

## Verification

- `sh gradlew test` 통과.
- `python3 -m py_compile ai-rag-service/main.py` 통과.

## Follow-up

- FastAPI 런타임 의존성 설치 후 실제 `/health`와 `/v1/aftercare/answer` 통합 테스트를 수행한다.
- Spring 응급 룰을 코드 내장 방식에서 `emergency_rules.json` 기반 로딩으로 전환한다.
- 운영에서는 RAG 서비스 장애 시 자동 OpenAI fallback을 켤지 여부를 별도 ADR로 판단한다.
