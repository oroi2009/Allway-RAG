# ADR-0014: 응급 룰 외부 JSON 로딩과 Spring-FastAPI Docker 네트워크 배포

- 원본: [Notion](https://app.notion.com/p/3bd499bdd6338129bd60c05d7fdc6b50)

## Context

응급 hard-stop 룰은 Spring Boot 1차 방어와 FastAPI RAG 서비스 2차 방어에 모두 필요하다. 이전 상태에서는 FastAPI는 `rag_rulebook/rules/emergency_rules.json`을 읽고, Spring은 동일 내용을 Java 코드에 복사해 들고 있었다.

이 구조는 룰이 변경될 때 두 구현이 쉽게 어긋난다. 또한 운영에서는 Spring Boot가 이미 Docker image로 배포되고, FastAPI도 같은 서버의 별도 컨테이너로 띄워 Docker network 내부 통신을 하려는 방향이 정해졌다.

## Decision

응급 룰은 DB에 적재하지 않고 `rag_rulebook/rules/emergency_rules.json` 정책 파일을 source of truth로 둔다.

Spring Boot는 `AI_CHAT_EMERGENCY_RULES_PATH`로 지정된 외부 JSON 파일을 시작 시 로드한다. FastAPI는 기존처럼 `RAG_RULEBOOK_ROOT` 아래의 `rules/emergency_rules.json`을 읽는다.

운영 배포에서는 같은 rulebook 디렉터리를 두 컨테이너에 read-only 볼륨으로 마운트한다.

Spring Boot 컨테이너는 `AI_CHAT_RAG_SERVICE_BASE_URL=http://centerton-rag:8001`로 FastAPI 컨테이너를 호출한다. FastAPI 포트는 외부에 publish하지 않고 Docker network 내부에서만 노출한다.

## Rationale

응급 룰은 RAG 검색 문서와 다르게 hard-stop 정책이다. DB 마이그레이션 중에도 룰 파일만 마운트되면 동작해야 하므로, RAG 문서 테이블과 분리하는 편이 안전하다.

Spring과 FastAPI가 같은 파일을 읽으면 룰 변경 시 양쪽 동작이 일관된다. Docker network의 service name을 쓰면 같은 서버 안에서 포트/호스트 의존성을 줄일 수 있다.

FastAPI를 별도 컨테이너로 둠으로써 KURE 모델, Python dependency, pgvector 검색, OpenAI prompt 실험을 Spring 배포와 분리할 수 있다.

## Implementation

Spring Boot:

- `AiChatEmergencyRuleProperties`를 추가했다.
- `ai-chat.emergency-rules.path=${AI_CHAT_EMERGENCY_RULES_PATH:...}` 설정을 추가했다.
- `AiChatEmergencyRuleService`가 Java 하드코딩 룰 대신 JSON의 `trigger_keywords`, `trigger_patterns`, `frontend_message`를 로드한다.
- 테스트 컨텍스트는 `classpath:rag/emergency_rules.json`을 사용해 외부 파일 의존 없이 로드된다.

FastAPI/Docker:

- `ai-rag-service/Dockerfile`을 추가했다.
- `ai-rag-service/.dockerignore`를 추가했다.
- `docker-compose.rag.example.yml`을 추가해 `centerton-api`와 `centerton-rag`를 같은 `centerton-internal` 네트워크에 배치했다.
- 두 컨테이너 모두 `${RAG_RULEBOOK_HOST_PATH}`를 `/app/rag-rulebook:ro`로 마운트하도록 했다.

## Consequences

운영 서버에는 `rag_rulebook` 디렉터리가 존재해야 하며, 두 컨테이너 모두 read-only로 접근할 수 있어야 한다.

응급 룰을 바꾼 뒤에는 최소한 Spring 컨테이너 재시작이 필요하다. 현재 Spring은 시작 시 룰을 로드하고 런타임 hot reload는 하지 않는다.

FastAPI의 KURE 모델은 컨테이너 시작 시 다운로드/캐시가 필요할 수 있으므로, Hugging Face cache volume을 둔다.

## Verification

- `sh gradlew test` 통과.
- `python3 -m py_compile ai-rag-service/main.py` 통과.
- Docker CLI가 현재 로컬 환경에 없어 `docker compose config`는 실행하지 못했다.

## Follow-up

- 운영 서버에서 Docker CLI가 있는 환경으로 `docker compose -f docker-compose.rag.example.yml config`를 검증한다.
- 실제 서버에 `${RAG_RULEBOOK_HOST_PATH}`를 지정하고 Spring/FastAPI 컨테이너를 함께 띄운다.
- `/health`와 실제 AI 채팅 API로 Docker network 통합 테스트를 수행한다.
