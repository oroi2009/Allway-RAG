# ADR-0016: 검색 근거 부족 시 생성 중단과 보수 안내 반환

- 원본: [Notion](https://app.notion.com/p/3bd499bdd63381088d57e4c56e43d868)

## Context

RAG 검색에는 `RAG_MIN_SIMILARITY` 기준이 있었지만, 기준을 통과한 문서가 0건이어도 빈 컨텍스트로 OpenAI 답변 생성을 호출할 수 있었다. 이 경우 RAG의 근거 기반 목적이 무너지고 일반 생성 답변이나 추측이 섞일 위험이 있다.

또한 평가셋의 역할이 운영 중 환자를 분류하는 별도 데이터로 오해될 수 있어 구분이 필요하다. 평가셋은 임계값과 검색 품질을 배포 전에 확인하는 개발용 시험 자료이며 실시간 응답 경로에는 참여하지 않는다.

## Decision

`RAG_MIN_SIMILARITY` 이상인 문서가 하나도 없으면 OpenAI 답변 생성을 호출하지 않는다.

FastAPI는 다음 값을 반환한다.

- `route=insufficient_evidence`
- `riskLevel=unknown`
- `allowRagAnswer=false`
- `confidence=0.0`
- 빈 `ragDocuments`
- 시술 정보 보완 및 시술 병원 확인을 권하는 고정 보수 안내

현재 기본 임계값은 `0.50`으로 유지한다. 이 값은 의료적 확신도가 아니라 KURE 벡터 검색 결과를 채택할지 결정하는 운영 기준이다.

## Rationale

검색 근거가 없을 때 생성 모델을 호출하지 않으면 근거 없는 사후관리 지침을 만들 가능성을 줄일 수 있다. 사용자는 질문을 구체화하거나 시술 병원에 문의하는 안전한 다음 행동을 안내받는다.

평가셋은 이 임계값을 정하는 데 도움을 주는 개발 도구다. 예를 들어 관련 질문이 0.49로 탈락하거나 무관한 질문이 0.55로 통과하는지 확인해 임계값과 문서 구성을 조정한다. 평가셋이 없어도 파이프라인은 실행되지만 `0.50`의 실제 품질을 객관적으로 설명할 수 없다.

## Options Considered

- 빈 검색 결과에도 생성 모델 호출: 답변률은 높지만 근거 없는 생성 위험 때문에 채택하지 않았다.
- 일반 OpenAI 답변으로 fallback: RAG 도입 목적과 충돌해 채택하지 않았다.
- 임계값 미달 시 보수 안내 반환: 근거성 및 안전 원칙에 부합해 채택했다.

## Consequences

관련 문서가 실제로 있어도 임계값보다 낮으면 답변을 거절할 수 있다. 반대로 임계값이 너무 낮으면 관련성이 약한 문서가 통과할 수 있다. 초기 운영 로그와 소규모 검토용 질문 모음으로 임계값을 조정해야 한다.

## Implementation

- `ai-rag-service/main.py`에서 검색 결과가 비어 있으면 즉시 `insufficient_evidence` 응답을 반환한다.
- 이 분기에서는 `generate_openai_answer()`를 호출하지 않는다.
- `ai-rag-service/README.md`에 동작과 반환값을 기록했다.

## Related Decision

- [ADR-0015: FastAPI Triage Classifier를 RAG 검색 앞단에 배치](0015-place-fastapi-triage-classifier-before-rag.md)

ADR-0015의 사전 triage classifier 유지 여부는 별도 구조 변경 결정으로 남아 있다.
