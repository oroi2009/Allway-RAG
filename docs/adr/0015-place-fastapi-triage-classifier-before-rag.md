# ADR-0015: FastAPI Triage Classifier를 RAG 검색 앞단에 배치

- 원본: [Notion](https://app.notion.com/p/3bd499bdd633819ca6d4ce38474133a6)

## Context

성형외과/피부미용 사후관리 챗봇은 환자의 증상 질문에 답변하기 전에 위험 여부를 먼저 판단해야 한다. 기존 구조는 Spring 1차 응급 hard-stop과 FastAPI 2차 응급 룰 검사 후 바로 RAG 검색과 OpenAI 답변 생성을 수행했다.

그러나 응급 룰은 명백한 위험 신호를 빠르게 차단하는 데 적합하지만, 애매한 증상, 표현이 다양한 위험 신호, 정보 부족 상황까지 모두 판단하기에는 부족하다. 위험 판단은 챗봇의 핵심 품질 요소이므로 단순 키워드 룰만으로 끝내지 않고 별도 triage classifier를 둔다.

## Decision

FastAPI에 `/v1/aftercare/triage` endpoint를 추가한다.

`/v1/aftercare/answer`는 내부에서 먼저 triage를 실행하고, triage route가 `rag_answer`일 때만 KURE 임베딩과 pgvector RAG 검색을 수행한다.

Triage route는 다음 중 하나로 제한한다.

- `hard_stop`: 고정 응급 안내, RAG 금지
- `urgent_clinic`: 빠른 병원 확인 권장, RAG 금지
- `video_consult`: 화상상담/사진 확인 권장, RAG 금지
- `clarify`: 추가 정보 요청, RAG 금지
- `rag_answer`: 일반 사후관리 RAG 답변 허용

Triage는 OpenAI Responses API structured output JSON schema를 사용해 자유 답변이 아니라 정해진 JSON 필드만 반환하도록 한다.

## Rationale

RAG는 근거 문서를 검색해 답변 품질과 출처 기반성을 높이는 데 적합하다. 그러나 임베딩 검색 자체가 환자 위험도를 직접 판단하는 도구는 아니다. 따라서 위험도 판단은 RAG 검색 전 별도 단계로 분리해야 한다.

응급 룰은 확정적이고 빠른 hard-stop 안전장치로 유지한다. Triage classifier는 룰에 걸리지 않은 애매한 케이스를 보수적으로 라우팅한다. route가 `rag_answer`가 아니면 일반 관리 답변을 생성하지 않는다.

## Implementation

- FastAPI `AnswerResponse`에 `allowRagAnswer`, `triageReason`, `matchedSignals`, `recommendedAction`, `confidence`를 추가했다.
- `/v1/aftercare/triage` endpoint를 추가했다.
- `/v1/aftercare/answer` 내부에서 `run_triage()`를 먼저 실행하도록 바꿨다.
- `allowRagAnswer=false`이면 RAG 검색 없이 route별 고정/보수 안내를 반환한다.
- `allowRagAnswer=true`이면 KURE 임베딩, pgvector 검색, OpenAI 답변 생성을 진행한다.
- `riskLevel=high|urgent`인데 route가 `rag_answer`로 충돌하면 `urgent_clinic`으로 보수 보정한다.
- `OPENAI_TRIAGE_MODEL`, `OPENAI_TRIAGE_MAX_OUTPUT_TOKENS` 설정을 추가했다.

## Consequences

RAG 답변 전 OpenAI triage 호출이 하나 추가되므로 지연시간과 비용은 증가한다. 대신 위험 판단이 단순 키워드 룰보다 넓은 표현을 다룰 수 있고, 안전하지 않은 일반 사후관리 답변을 줄일 수 있다.

Spring은 여전히 FastAPI `/v1/aftercare/answer`만 호출해도 된다. `/triage`는 디버깅, 평가, 향후 프론트 분리 호출을 위해 제공한다.

## Verification

- `sh gradlew test` 통과.
- `ai-rag-service/main.py` Python compile 통과.

## Follow-up

- triage 평가셋을 최소 50~100개 작성한다.
- route별 기대 결과를 자동 평가하는 스크립트를 추가한다.
- 운영 로그에 route, riskLevel, confidence, matchedSignals를 저장할지 결정한다.
- Spring 응답 DTO에 triage metadata를 노출할지 결정한다.
