# ADR-0013: 응급 룰 단독 키워드 축소와 정규식 매칭 도입

- 원본: [Notion](https://app.notion.com/p/3bd499bdd63381d78fc8d3131ceba636)

## Context

AI 사후관리 챗봇의 응급 hard-stop은 안전상 모델/RAG보다 먼저 실행되어야 한다. 기존 MVP 룰은 `열`, `출혈` 같은 넓은 단독 키워드도 포함하고 있어 오탐 가능성이 있었다.

예를 들어 `열심히 관리하고 있어요`는 실제 발열과 관계없지만 `열`을 포함하고, `출혈은 없어요`는 출혈 부정 문장인데도 `출혈` 단독 키워드에 매칭될 수 있다.

## Decision

응급 룰의 넓은 단독 키워드를 줄이고, 실제 증상 상태가 드러나는 구체 표현 또는 정규식 패턴으로 전환한다.

`RISK-02 fever_or_chills_after_procedure`에서는 `열` 단독 키워드를 제거하고, `38도`, `38.5도`, `열이 나요`, `오한이 있어요`, `식은땀 동반`, `몸살 기운 + 열/오한`처럼 상태가 포함된 표현만 매칭한다.

`RISK-03 uncontrolled_bleeding`에서는 `출혈` 단독 키워드를 제거하고, `피가 안 멈춤`, `출혈이 계속됨`, `지혈이 안 됨`, `압박해도 피가 계속남`처럼 조치 필요성이 드러나는 패턴만 매칭한다.

## Rationale

응급 룰은 false negative를 줄이는 것이 중요하지만, 너무 넓은 단독 키워드는 정상 문의까지 hard-stop으로 막아 사용자 경험과 신뢰를 해칠 수 있다.

정규식은 LLM 분류보다 결정적이고 빠르며, 어떤 표현이 왜 매칭됐는지 로그와 테스트로 추적하기 쉽다. 현재 룰 수가 작기 때문에 성능 부담도 사실상 없다.

## Current Source Of Truth

FastAPI RAG 서비스는 `rag_rulebook/rules/emergency_rules.json`을 읽는다.

Spring Boot는 아직 JSON을 직접 읽지 않고 `AiChatEmergencyRuleService`에 동일 룰을 코드로 내장한다. 즉 현재 응급 룰은 DB에 적재되어 있지 않다. RAG 문서는 `rag_documents`에 적재되어 있지만, 응급 hard-stop 룰은 별도 정책 파일/코드 자산이다.

## Consequences

`열심히 관리하고 있어요`, `출혈은 없어요` 같은 문장이 응급으로 잘못 걸릴 가능성이 줄어든다.

반대로 사용자가 매우 짧게 `열`, `출혈`만 입력하면 이제 즉시 hard-stop이 아니라 추가 확인 또는 일반 답변 경로로 갈 수 있다. 장기적으로는 별도 clarify route를 두어 `열이 있다는 뜻인가요? 체온이 몇 도인가요?`처럼 확인 질문을 던지는 것이 좋다.

## Implementation

- `rag_rulebook/rules/emergency_rules.json` 버전을 `2026-08-15-mvp-rules-v2`로 변경했다.
- `RISK-02`, `RISK-03`에 `trigger_patterns`를 추가했다.
- Spring `AiChatEmergencyRuleService`에 Java 정규식 매칭을 추가했다.
- FastAPI `EmergencyRuleEngine`에 JSON `trigger_patterns` 처리를 추가했다.
- Spring 단위 테스트를 추가해 오탐/정탐을 검증했다.

## Verification

- `sh gradlew test` 통과.
- `python3 -m py_compile ai-rag-service/main.py` 통과.
- `emergency_rules.json` JSON 파싱 통과, rules count 9.
