# ADR-0018: 응급 룰 매칭을 compact 및 spaced 이중 정규화로 분리

- 원본: [Notion](https://app.notion.com/p/3bf499bdd633808a829bd4562394096a)

## Context

`2026-08-15-mvp-rules-v2` 룰북은 입력 텍스트를 NFC 정규화하고 소문자로 바꾼 뒤 모든 공백을 제거한 형태에서 트리거를 검사했다. 공백 제거는 환자의 불규칙한 띄어쓰기를 흡수하기 위한 조치였다.

그러나 공백을 제거하면 어절 경계가 사라지고 환자가 쓰지 않은 토큰이 만들어진다. 탐침 결과 오탐 8건이 이 원인으로 확인됐다.

| 입력 | 공백 제거 결과 | 생성된 토큰 | 오탐 룰 |
|---|---|---|---|
| 약을 먹고 열이 내렸어요 | 약을먹고열이내렸어요 | `고열이` | RISK-02 |
| 회복시기가 언제쯤인지 궁금해요 | 회복시기가언제쯤인지궁금해요 | `복시` | RISK-05 |
| 발목이 붓어서 걷기 불편해요 | 발목이붓어서걷기불편해요 | `목이붓` | RISK-06 |

첫 번째 사례는 열이 내려갔다는 안심 보고인데 고열 룰이 매칭되어 응급실 안내로 종결됐다. 원인은 `먹고`의 종성 `고`와 뒤 어절 `열이`가 붙어 `고열이`가 만들어진 것이다.

경계 정보가 이미 소실된 상태이므로 lookbehind로는 복구할 수 없다. `(?<![가-힣])고열`을 적용하면 `심한 고열이`처럼 앞에 한글이 오는 정상 매칭까지 차단된다.

같은 시기에 통합 시나리오 `TC-RISK-01`이 이 결함으로 통과하고 있던 것도 확인됐다. 입력 `수술 부위에서 노란 고름이 나오고 열이 펄펄 나요`에서 RISK-02는 `열이 펄펄`이 아니라 `나오고`와 `열이`가 붙어 생긴 `고열이`로 매칭됐다.

## Decision

정규화 형태를 두 가지로 분리하고 트리거 종류에 따라 다른 형태에서 평가한다.

- `compact`: NFC 변환 후 소문자화, 모든 공백 제거, `number_aliases` 적용. `trigger_keywords` 평가에 사용한다.
- `spaced`: NFC 변환 후 소문자화, 연속 공백을 단일 공백으로 축약하고 trim, `number_aliases` 적용. `trigger_patterns` 평가에 사용한다.

키워드는 사용자의 띄어쓰기 편차를 흡수해야 하므로 `compact`가 적합하다. 정규식은 어절 경계가 필요하므로 `spaced`에서 평가한다.

패턴은 공백이 선택적인 자리에 `\s*`를 쓰고, 인접이 필수인 자리에는 `\s`를 쓰지 않는 방식으로 작성한다. `고열`은 한 단어여야 하고 `열 이`는 나뉠 수 있다.

`normalization` 블록에 `keyword_text_form`과 `pattern_text_form`을 명시해 Backend가 같은 계약을 따르도록 한다.

## Rationale

`spaced` 형태에서는 `먹고 열이`에 공백이 남아 있으므로 `고열` 패턴이 구조적으로 매칭될 수 없다. 정규식 작성자가 인접 여부를 직접 통제하게 되어, 개별 오탐을 사후에 막는 대신 오탐이 발생할 수 없는 형태로 계약을 바꾼다.

`복시`와 `목이 붓`은 키워드에서 패턴으로 옮기고 `(?<![가-힣])` lookbehind를 붙였다. `spaced` 형태에서는 앞 어절과 공백으로 분리되므로 lookbehind가 의도대로 동작한다.

## Options Considered

- `compact` 형태 유지하고 lookbehind로 방어: 공백 제거로 경계 정보가 이미 파괴되어 정상 매칭과 오탐을 구별할 수 없으므로 채택하지 않았다.
- 패턴에 `\s*`를 자동 삽입: `고열`이 `고\s*열`이 되어 `먹고 열이`를 다시 매칭한다. 결함을 재도입하므로 채택하지 않았다.
- 키워드와 패턴을 서로 다른 형태에서 평가: 각 트리거 종류의 요구사항에 맞고 결함을 구조적으로 제거하므로 채택했다.

## Consequences

기존 `trigger_patterns`는 공백 제거를 전제로 작성돼 있어 전부 재작성이 필요했다. RISK-02의 9개, RISK-03의 5개 패턴을 `spaced` 형태로 다시 썼다.

재작성 과정에서 미탐 1건이 드러났다. `열이 안 떨어져요`는 v2에서 `이안떨어` 대안으로 매칭되고 있었으나 `spaced` 형태에서는 공백 때문에 매칭되지 않았다. 발열 지속을 뜻하는 실제 응급이므로 `열\s*이?\s*안\s*떨어`를 명시적으로 추가했다.

패턴 작성 난이도가 올라간다. 작성자가 인접과 공백 허용을 매번 판단해야 한다. 이 부담은 회귀 스위트로 보완한다.

## Implementation

- `rag_rulebook/tools/emergency_matcher.py`에 `normalize_compact`와 `normalize_spaced`를 구현하고 `match`가 트리거 종류에 따라 다른 형태를 쓰도록 했다.
- `rag_rulebook/rules/emergency_rules.json`을 `2026-08-15-mvp-rules-v3`으로 올리고 `keyword_text_form`, `pattern_text_form`을 선언했다.
- RISK-05의 `복시`, RISK-06의 `목이 붓`을 패턴으로 이동하고 `match_policy`를 `any_keyword_or_pattern`으로 변경했다.
- RISK-02에 `열\s*이?\s*펄\s*펄`을 추가해 `TC-RISK-01`이 정당한 근거로 통과하도록 했다.
- `rag_rulebook/tools/validate_emergency_rules.py`가 `pattern_text_form=spaced` 선언과 패턴 도달 가능성을 검사한다. 패턴을 선언했는데 `match_policy`가 키워드 전용이면 실패한다.
- 회귀 스위트에 `word_boundary_false_positive` 10건을 고정했다. 패턴을 v2로 되돌리면 `IS-01`, `NT-01`, `NT-08`이 실패한다.

## Related Decision

- [ADR-0017: 응급 룰북은 정확도를 우선하고 재현율은 답변 가드레일이 담당](0017-precision-first-emergency-rulebook.md)
- [ADR-0019: 부정 억제는 명시적 비발생 형태로 한정하고 안을 배제](0019-negation-guards-only-for-explicit-non-occurrence.md)
