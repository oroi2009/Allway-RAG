# ADR-0022: FastAPI RAG 서비스를 Rag-Lab으로 이관하고 응급 룰 매칭을 단일 구현으로 통합

- 원본: [Notion](https://app.notion.com/p/3bf499bdd63380c795e9e24b60f9dda8)

## Context

FastAPI RAG 서비스(`main.py`)는 Spring과 같은 Backend 저장소에 있었고, `rag_rulebook`을 read-only 볼륨으로 마운트해 `rules/emergency_rules.json`을 읽었다. 룰북 데이터는 공유했지만 매칭 구현은 서비스가 자체적으로 작성했다.

그 결과 구현이 룰북 계약과 어긋났다. 서비스의 `EmergencyRuleEngine`을 그대로 재현해 Rag-Lab 회귀 스위트 66건으로 채점한 결과는 다음과 같다.

| 구분 | 건수 |
|---|---:|
| 미탐 (응급인데 통과) | 0 |
| 오탐 (정상 문의인데 hard-stop) | 18 |

오탐은 두 원인으로 나뉜다.

서비스의 `normalize()`는 모든 공백을 제거한 뒤 `trigger_patterns`를 평가한다. ADR-0018에서 패턴은 공백 보존 형태에서 평가하기로 정했으므로 `(?:고열|발열)`이 어절 경계를 넘어 매칭된다. `약을 먹고 열이 내렸어요`가 발열 응급으로 분류된다.

서비스에 ADR-0019의 `negation_guards`가 구현되어 있지 않다. `고름은 없어요`, `진물이 나지 않아요` 같은 증상 부재 보고 14건이 모두 hard-stop 된다.

또한 `find_match()`는 첫 매칭 룰에서 반환한다. `recommendedAction`이 그 룰의 `frontend_message`이므로 환자가 받는 안내 문구가 달라진다. 필러 혈관 폐색 의심 입력이 RISK-07 대신 RISK-05로 분류되어 필러 전용 안내가 아닌 일반 호흡·시야 안내가 전달된다.

한편 저장소 분리의 실익을 측정해 보면 다음과 같다.

| 대상 | 크기 | 런타임 필요 |
|---|---:|---|
| `rules/emergency_rules.json` | 15K | 필요 |
| `tools/emergency_matcher.py` | 11K | 필요 |
| 회귀 스위트와 러너 | 30K | CI에서 필요 |
| `derived/` | 64M | 불필요, pgvector에 적재됨 |
| `sources/` | 13M | 불필요 |

서비스가 파일시스템에서 읽는 것은 JSON 하나이며 90개 문서는 이미 DB에 있다. 즉 결합은 60K 규모다. 동시에 이 서비스는 룰북 없이 존재 이유가 없고 룰북은 이 서비스 외에 소비자가 없다.

## Decision

FastAPI RAG 서비스를 Rag-Lab 저장소로 이관하고 `centerton_rag` 패키지로 둔다.

응급 룰 매칭은 `rag_rulebook.tools.emergency_matcher`를 단일 구현으로 사용한다. 서비스는 정규화, 키워드 매칭, 패턴 매칭, 부정 억제를 재구현하지 않는다.

Spring은 응급 룰을 구현하지 않는다. FastAPI를 호출해 결과를 전달하고, FastAPI에 도달할 수 없으면 보수 안내로 종결한다.

회귀 스위트를 서비스 코드에 대해 실행하는 계약 테스트를 둔다. 룰북을 바꾸는 변경과 서비스를 바꾸는 변경이 같은 검증을 통과해야 한다.

컨테이너 이미지에서는 `derived/`와 `sources/`를 제외한다.

## Rationale

같은 저장소에 두면 드리프트가 구조적으로 불가능해진다. 룰북과 매처와 계약 테스트가 한 커밋에서 함께 변경되므로 버전 동기화 절차가 필요 없다.

결합도가 1:1인 두 산출물을 저장소로 분리하면 동기화 작업만 늘고 얻는 것이 없다. 실제로 분리 상태에서 데이터는 공유되고 구현은 갈라졌다.

또한 서비스를 별도 저장소로 두면 Spring, FastAPI, Rag-Lab으로 저장소가 세 개가 된다. 이관하면 두 개로 끝나고 Spring과는 도커 네트워크와 배포로만 연결된다는 원래 의도가 유지된다.

Spring이 응급 룰을 갖지 않는 편이 더 안전하다. FastAPI가 정지하면 답변 자체를 만들 수 없으므로, 그 상황에서는 보수 안내로 종결하는 것이 중복 구현보다 단순하고 ADR-0016의 fail-closed 원칙과 일관된다. 구현이 두 곳에 있으면 두 곳이 서로 다르게 동작하는 것을 막을 방법이 없다.

## Options Considered

- 룰북만 볼륨으로 마운트하고 매처는 각자 구현: 현재 상태이며 오탐 18건과 안내 문구 오류를 만들었으므로 유지하지 않는다.
- 매처와 룰북을 60K 패키지로 분리해 버전 핀으로 관리: 드리프트를 버전으로 통제할 수 있고 Rag-Lab을 순수 데이터 저장소로 유지할 수 있다. 다만 태그와 설치 절차가 추가되고 오래된 핀이 남으면 여전히 어긋날 수 있어 채택하지 않았다.
- git submodule로 커밋 핀 관리: 패키지화와 유사하나 submodule 관리 부담이 더 크고 이득이 없어 채택하지 않았다.
- 서비스를 Rag-Lab으로 이관: 드리프트가 구조적으로 불가능하고 저장소 수가 줄어들어 채택했다.

## Consequences

Rag-Lab의 성격이 데이터 큐레이션 저장소에서 배포 대상으로 바뀐다. 의료 검토가 필요한 룰북 변경과 서비스 배포 변경이 한 저장소에 섞인다. 현재 팀 규모에서는 감당 가능하다고 판단했으나, 검토 주체가 늘어나면 CODEOWNERS로 경로별 리뷰어를 분리한다.

저장소에 운영 환경 변수 목록이 들어온다. 실제 비밀값은 저장하지 않고 예시 파일과 문서로만 관리한다.

`derived/`와 `sources/`가 저장소에는 남지만 이미지에는 포함되지 않는다. 이미지 크기는 60K 수준의 룰북 자산만 늘어난다.

핸드오프 문서의 저장소 경계 설명이 더 이상 맞지 않으므로 갱신해야 한다.

## Implementation

- `centerton_rag/` 패키지를 만들고 `main.py`를 이관했다. 계약에 관여하는 로직은 표준 라이브러리만 사용하는 모듈로 분리해 웹 프레임워크 없이 검증할 수 있게 했다.
- `centerton_rag/emergency.py`가 `rag_rulebook.tools.emergency_matcher.EmergencyMatcher`를 사용한다. 기존 `EmergencyRuleEngine`과 `normalize()`는 삭제했다.
- 매칭된 모든 룰을 반환하도록 바꿨다. 안내 문구는 가장 먼저 선언된 룰이 아니라 매칭된 룰 전체를 근거로 구성한다.
- `rag_rulebook/__init__.py`와 `rag_rulebook/tools/__init__.py`를 추가해 패키지로 import 가능하게 했다. 기존 검증 스크립트의 동작은 바뀌지 않는다.
- `tests/test_emergency_contract.py`가 `test_cases/emergency_rule_regression.json` 66건을 서비스 모듈에 대해 실행한다.
- `Dockerfile`과 `.dockerignore`를 추가했다. `.dockerignore`가 `derived/`와 `sources/`를 제외한다.
- Spring 쪽 응급 룰 제거는 Backend 저장소 작업으로 남아 있다.

## Related Decision

- [ADR-0016: 검색 근거 부족 시 생성 중단과 보수 안내 반환](0016-stop-generation-when-evidence-is-insufficient.md)
- [ADR-0018: 응급 룰 매칭을 compact 및 spaced 이중 정규화로 분리](0018-compact-and-spaced-normalization-for-rule-matching.md)
- [ADR-0019: 부정 억제는 명시적 비발생 형태로 한정하고 안을 배제](0019-negation-guards-only-for-explicit-non-occurrence.md)
- [ADR-0023: pre-RAG triage classifier 제거와 상담 CTA를 답변 속성으로 전환](0023-remove-pre-rag-triage-classifier-and-use-cta.md)

ADR-0018과 ADR-0019가 정의한 계약을 서비스가 지키지 않은 것이 이 결정의 직접적인 계기다.
