# ADR-0023: pre-RAG triage classifier 제거와 상담 CTA를 답변 속성으로 전환

- 원본: [Notion](https://app.notion.com/p/3bf499bdd63380e384a7dcac68fc9919)

## Context

ADR-0015로 FastAPI에 pre-RAG triage classifier를 두었다. 응급 룰에 매칭되지 않은 문의를 OpenAI로 분류해 `hard_stop`, `urgent_clinic`, `video_consult`, `clarify`, `rag_answer` 중 하나를 선택한다.

구현에서 이 분류 결과가 검색 실행 여부를 결정한다.

```python
triage = run_triage(request, settings)
if not triage.allowRagAnswer:
    return AnswerResponse(...)      # 검색 없이 반환
documents = retrieve_documents(question, settings)
```

`sanitize_triage()`는 `route`가 `rag_answer`가 아니면 `allowRagAnswer`를 거짓으로 강제하고, 추가로 위험도가 높거나 알 수 없으면 `rag_answer`를 강등한다.

```python
if route == "rag_answer" and risk_level in {"high", "urgent"}:
    route = "urgent_clinic"
if route == "rag_answer" and risk_level == "unknown":
    route = "clarify"
```

즉 위험도가 높다고 판단될수록 검색이 실행되지 않는다. 이는 핸드오프가 명시적으로 금지한 동작이다.

> 별도 pre-RAG LLM triage classifier가 `urgent_clinic`, `video_consult`, `clarify`라고 판단했다는 이유만으로 검색을 차단하지 않습니다.

문제는 정책 위반에 그치지 않는다.

`test_cases/integration_scenarios.json`의 `TC-RHINO-01`은 `expected_route`가 `video_consult`이면서 `expected_docs`로 `SYM-RHINO-ASYMMETRY-ANXIETY`와 `COMMON-LOW-CONFIDENCE-PHOTO`를 기대한다. 현재 구현에서는 `video_consult`로 분류되면 검색이 실행되지 않으므로 이 시나리오는 통과할 수 없다. 코성형 비대칭 불안을 위해 작성한 큐레이션 문서와 `answer_template`이 사용되지 않는다.

비용 구조도 의도와 어긋난다. 단순 문의 한 건에 OpenAI 호출이 두 번 발생하고, triage 호출은 첨부 이미지를 `detail: "high"`로 함께 전송한다. 단순 CS를 저비용으로 방어한다는 명분과 상충한다.

가용성 측면에서도 `clarify`가 기본 fallback이다. OpenAI 응답이 스키마를 벗어나면 `route`가 `clarify`가 되어 검색 없이 정보 보완 요청으로 종결된다.

마지막으로 답변 가드레일의 위치가 뒤집혀 있다. `DEVELOPER_PROMPT`에 위험 신호 안내 지침이 있으나 이 경로는 triage가 저위험이라고 판단했을 때만 실행된다. 위험 신호가 있을 때는 실행되지 않는다.

## Decision

응급 룰에 매칭되지 않은 모든 문의는 검색을 실행한다. 검색 실행 여부를 판단하는 pre-RAG LLM 분류기를 제거하고 `/v1/aftercare/triage` 엔드포인트도 제거한다.

상담 유도는 라우트가 아니라 답변에 부착되는 CTA 속성으로 다룬다. CTA는 검색된 근거의 메타데이터에서 결정적으로 도출한다.

| 조건 | CTA |
|---|---|
| 근거 중 `risk_level`이 `urgent` | `urgent_clinic` |
| 근거 중 `dataset_type`이 `video_consult_trigger` | `video_consult` |
| 근거 중 `risk_level`이 `watch` | `video_consult` |
| 그 외 | 없음 |

`urgent_clinic`이 `video_consult`보다 우선한다.

근거 부족 시 동작은 ADR-0016을 그대로 유지한다. 임계값을 통과한 문서가 없으면 생성 모델을 호출하지 않고 보수 안내를 반환한다.

답변 생성 단계의 가드레일을 강화한다. 응급 룰에 매칭되지 않은 모든 문의가 이 경로를 지나므로, 생성 모델이 근거에 없는 위험 신호를 발견하면 일반 사후관리로 마무리하지 않고 병원 확인을 안내한다.

## Rationale

검색은 판단이 아니라 근거 수집이다. 위험해 보이는 문의일수록 근거가 더 필요하므로, 위험도를 이유로 검색을 막는 것은 순서가 뒤바뀐 것이다. 검색을 먼저 하고 그 근거를 받은 답변이 보수적 안내와 CTA를 구성하면 두 목적이 모두 충족된다.

CTA를 근거에서 도출하면 LLM 호출이 한 번으로 줄고 판정이 결정적이 된다. `risk_level`과 `dataset_type`은 이미 큐레이션 단계에서 부여한 값이고 검색 쿼리가 이미 조회하고 있으므로 스키마 변경이 필요하지 않다. `SYM-RHINO-ASYMMETRY-ANXIETY`는 `dataset_type`이 `video_consult_trigger`이므로 이 문서가 검색되면 화상상담 CTA가 자동으로 붙는다.

또한 이 변경으로 ADR-0017의 두 번째 계층이 실제로 성립한다. 답변 생성 경로가 모든 비응급 문의를 지나므로 룰북이 놓친 위험 신호를 받을 자리가 생긴다. 현재 구조에서는 그 자리가 존재하지 않았다.

ADR-0015를 폐기하는 것이 아니라 대체한다. 위험도 분류라는 목적은 유지되지만 그 판단 시점을 검색 이전에서 검색 이후로 옮긴다.

## Options Considered

- 현행 유지: 핸드오프 정책을 위반하고 `TC-RHINO-01`이 통과할 수 없으며 유료 전환 시나리오에서 근거가 0건이므로 유지하지 않는다.
- triage classifier는 유지하되 검색은 항상 실행: LLM 호출 두 번이 유지되어 비용 문제가 남고, 분류 결과와 근거 기반 판단이 충돌할 때 우선순위를 정해야 한다. 이득이 불명확해 채택하지 않았다.
- triage classifier 제거하고 CTA를 근거 메타데이터에서 도출: 호출이 한 번으로 줄고 판정이 결정적이며 기존 큐레이션 메타데이터를 활용하므로 채택했다.
- CTA도 생성 모델이 판단: 구조화 출력을 추가로 요구해야 하고 결정성이 떨어져 채택하지 않았다.

## Consequences

이미지 첨부만으로 상담 유도를 판단하던 경로가 없어진다. 기존에는 triage classifier가 이미지를 보고 `video_consult`를 선택할 수 있었다. 이제는 검색된 근거가 상담 유도 대상이어야 CTA가 붙는다. `COMMON-LOW-CONFIDENCE-PHOTO` 문서가 이 역할을 하도록 작성되어 있으나 이미지 신뢰도 점수가 요청 스키마에 없어 현재는 활용되지 않는다. 이미지 품질 기반 유도는 별도 결정으로 남긴다.

`riskLevel` 응답 값의 출처가 바뀐다. 생성 모델 판단이 아니라 검색된 근거의 최대 위험도를 사용한다.

`/v1/aftercare/triage`를 사용하는 호출자가 있으면 함께 제거해야 한다. Spring 연동 확인이 필요하다.

`TC-RHINO-01`의 `expected_route`가 라우트 개념과 맞지 않게 된다. 통합 시나리오의 필드를 `expected_route`에서 CTA 기대값으로 조정해야 한다.

## Implementation

- `centerton_rag/main.py`에서 `run_triage()` 게이트를 제거했다. 응급 룰 미매칭이면 항상 `retrieve_documents()`를 호출한다.
- `generate_openai_triage()`, `create_triage_input()`, `triage_json_schema()`, `sanitize_triage()`, `create_triage_answer()`를 삭제했다.
- `centerton_rag/consultation.py`에 `derive_consultation_cta()`와 `derive_risk_level()`을 구현했다. 표준 라이브러리만 사용하며 근거 목록만 입력으로 받는다.
- ADR-0016의 `insufficient_evidence` 분기는 유지했다. `tests/test_generation_gate.py`가 근거 0건일 때 생성 함수가 호출되지 않는 것을 검증한다.
- `DEVELOPER_PROMPT`에 근거에 없는 위험 신호 발견 시 처리 지침을 명시했다.
- `integration_scenarios.json` 조정은 하지 않았다. 라우트 개념 정리와 함께 별도로 처리한다.

## Related Decision

- [ADR-0015: FastAPI Triage Classifier를 RAG 검색 앞단에 배치](0015-place-fastapi-triage-classifier-before-rag.md)
- [ADR-0016: 검색 근거 부족 시 생성 중단과 보수 안내 반환](0016-stop-generation-when-evidence-is-insufficient.md)
- [ADR-0017: 응급 룰북은 정확도를 우선하고 재현율은 답변 가드레일이 담당](0017-precision-first-emergency-rulebook.md)
- [ADR-0022: FastAPI RAG 서비스를 Rag-Lab으로 이관하고 응급 룰 매칭을 단일 구현으로 통합](0022-move-fastapi-rag-service-into-rag-lab.md)

이 결정은 ADR-0015를 대체한다.
