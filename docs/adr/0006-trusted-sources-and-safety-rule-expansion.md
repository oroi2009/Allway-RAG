# ADR-0006: 추가 신뢰 출처 수집과 안전 룰 보강

- 원본: [Notion](https://app.notion.com/p/3bd499bdd63381f89325e5cafd1958bd)

## Context

성형·피부미용 사후관리 RAG는 증상 범위가 넓어 데이터가 많을수록 좋아 보이지만, 무차별 수집은 응급 판단 오염과 과잉 일반 의학 답변을 만들 수 있다. 이번에는 사용자가 요청한 "추가 수집 필요한 정보 중 직접 수집 가능한 것"을 서비스 범위 안에서 보강했다.

## Decision

추가 출처는 식약처, FDA, AAD, ASPS처럼 공신력 있는 기관 또는 전문 단체의 피부미용·성형 사후관리 자료로 제한한다. 수집한 원본은 `sources/raw_trusted/`에 보관하고, `sources/trusted_by_use/`에서 `rag_candidate`와 `safety_only`로 분리한다.

## Rationale

- 필러, 레이저, 보툴리눔 톡신, 코성형은 현재 서비스 도메인과 직접 연결된다.
- 필러 후 시야 이상·피부 창백/푸른 변화·편측 약화, 보툴리눔 톡신 후 호흡/삼킴/말 어눌함, 레이저 후 과도한 물집/참기 힘든 통증은 RAG 답변보다 hard-stop 룰이 우선해야 한다.
- 정상 회복 설명과 중대 위험 신호가 섞인 자료는 `allow_with_safety_filters`로 표기해, 응급 룰 미매칭 시에만 환자 답변 색인에서 사용한다.

## Options Considered

1. 모든 신뢰 출처를 기본 RAG에 넣기: 커버리지는 넓지만 안전 신호가 일반 답변에 섞일 위험이 크다.
2. 안전 문서를 모두 제외하기: 답변은 안정적이지만 즉시 진료 룰의 근거와 키워드 보강이 약해진다.
3. RAG 후보와 safety-only를 분리하기: 답변 근거와 hard-stop 근거를 동시에 확보할 수 있어 채택했다.

## Consequences

- `derived/trusted_rag_candidate_chunks.jsonl` 58개 chunk가 기본 환자 답변 색인에 추가된다.
- `sources/trusted_by_use/safety_only/`는 임베딩에서 제외하고 hard-stop 룰 근거로 사용한다.
- 응급 룰은 RISK-07 필러 혈류/신경학적 위험, RISK-08 보툴리눔 톡신 확산 의심, RISK-09 레이저 중증 피부반응을 추가한다.

## Implementation

- 원본 HTML 13개와 식약처 PDF 2개를 `sources/raw_trusted/`에 보관했다.
- `tools/extract_trusted_source_text.py`로 HTML/PDF 텍스트와 manifest를 생성했다.
- `tools/partition_trusted_sources.py`로 `rag_candidate` 10개, `safety_only` 3개를 분류했다.
- `tools/build_rag_datasets.py`가 `derived/trusted_rag_candidate_chunks.jsonl`을 생성하도록 확장했다.
- `tools/validate_triage.py` 기준 8개 통합 시나리오가 통과했다.

## Source Links

- [MFDS 성형용 필러 안전사용 안내서](https://www.mfds.go.kr/brd/m_465/view.do?seq=27164)
- [MFDS 의료용레이저 안전사용 안내서](https://www.mfds.go.kr/brd/m_465/view.do?seq=27162)
- [MFDS 성형용필러](https://www.mfds.go.kr/brd/m_464/view.do?seq=28280)
- [MFDS 의료용 레이저(피부치료용)](https://www.mfds.go.kr/brd/m_464/view.do?seq=28291)
- [FDA Dermal Filler Do's and Don'ts](https://www.fda.gov/consumers/consumer-updates/dermal-filler-dos-and-donts-wrinkles-lips-and-more)
- [FDA Dermal Fillers Soft Tissue Fillers](https://www.fda.gov/medical-devices/aesthetic-cosmetic-devices/dermal-fillers-soft-tissue-fillers)
- [AAD Acne scars aftercare](https://www.aad.org/public/diseases/acne/derm-treat/scars/self-care)
- [AAD Laser treatment for scars](https://www.aad.org/public/cosmetic/scars-stretch-marks/laser-treatment-scar)
- [ASPS Dermal fillers safety](https://www.plasticsurgery.org/cosmetic-procedures/dermal-fillers/safety)
- [ASPS Dermal fillers recovery](https://www.plasticsurgery.org/cosmetic-procedures/dermal-fillers/recovery)
- [ASPS Rhinoplasty recovery](https://www.plasticsurgery.org/cosmetic-procedures/rhinoplasty/recovery)
- [ASPS Botulinum toxin safety](https://www.plasticsurgery.org/cosmetic-procedures/botulinum-toxin/safety)
- [ASPS Botulinum toxin recovery](https://www.plasticsurgery.org/cosmetic-procedures/botulinum-toxin/recovery)

## Follow-up

- 실제 서비스 적용 전, `allow_with_safety_filters` chunk가 정상 상담에서 과도하게 무서운 답변을 만들지 않는지 검색 평가셋으로 확인한다.
- 한국어 환자 표현 확장을 위해 필러/보톡스/레이저 이상반응 동의어를 추가 수집한다.
