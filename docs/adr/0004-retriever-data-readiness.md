# ADR-0004: 리트리버용 데이터 준비 완료 범위와 남은 전처리

- 원본: [Notion](https://app.notion.com/p/3bd499bdd633810eaceadb612832f1bc)
- 상태: Accepted
- 확정일: 2026-08-15
- 범위: 성형외과, 피부과, 피부미용 시술 후 이상반응 및 사후관리 챗봇

## 상황

RAG 리트리버 개발을 시작하기 전에, 현재 모아둔 원본 데이터와 전처리 산출물이 실제로 벡터DB 색인에 바로 사용할 수 있는지 점검했다. 로컬에는 공식 자료 원본/분류본, MVP curated 케어 지식, 일부 AIHub 기반 JSONL 산출물이 존재한다.

현재 변환된 JSONL 산출물은 다음과 같다.

- `rag_rulebook/rag/mvp_care_knowledge.jsonl`: 17개 curated 사후관리 문서
- `rag_rulebook/derived/source_medical_qa_rag.jsonl`: 3,630개 전문 QA 참고 문서
- `rag_rulebook/derived/problem_skin_makeup_rag.jsonl`: 2,785개 문제성 피부/메이크업 참고 문서

공식 자료는 원본 HTML/텍스트와 사용 목적별 분류본까지 준비되어 있다. 다만 03번 스킨케어 성분-효능 추천 데이터는 아직 별도 JSONL 산출물로 변환되지 않았고, 08번 다운로드 원본 zip 전체가 아니라 Desktop에 풀려 있는 `TL_외과`, `TL_피부과`만 기존 빌드 스크립트가 읽고 있다.

## 결정

현재 상태를 "원본 수집과 1차 분류는 완료, 리트리버 최종 코퍼스 준비는 미완료"로 본다.

즉, MVP 실험용 리트리버는 지금도 `mvp_care_knowledge.jsonl`, 공식 `rag_candidate`, 일부 reference JSONL로 시작할 수 있다. 하지만 운영 수준의 리트리버 데이터셋이라고 부르기 위해서는 03번 성분 데이터 변환, 08번 zip 원본 처리, chunk 단위 메타데이터 정리, 색인 대상 whitelist 생성이 추가로 필요하다.

## 근거

- `mvp_care_knowledge.jsonl`은 바로 벡터DB에 넣기 쉬운 구조를 갖고 있지만 17개라 커버리지가 좁다.
- `source_medical_qa_rag.jsonl`과 `problem_skin_makeup_rag.jsonl`은 이미 JSONL이지만 `reference_only` 성격이므로 환자 답변의 주 근거로 쓰기 전에 필터링이 필요하다.
- 03번 스킨케어 성분-효능 데이터는 zip 원본만 확인되며 현재 `derived` 산출물에 포함되지 않았다.
- 공식 자료 중 `safety_only`, `out_of_scope`, `api_reference`는 벡터DB에 넣으면 검색 노이즈가 생길 수 있으므로 색인 대상에서 제외해야 한다.

## 검토한 선택지

1. 지금 있는 JSONL 전체를 바로 벡터DB에 넣는다: 빠르지만 reference-only 문서와 도메인 밖 문서가 섞여 답변 품질이 흔들릴 수 있다.
2. curated 17개 문서만 넣고 MVP를 시작한다: 안전하지만 커버리지가 너무 좁다.
3. 원본/분류는 완료로 보고, 최종 색인 전처리 단계를 별도 작업으로 둔다: 현재 상태를 정확히 반영하고 품질 관리가 가능하므로 채택한다.

## 영향

- 리트리버 개발은 시작 가능하지만, 색인 대상은 whitelist 방식으로 제한해야 한다.
- `retrieval_use`, `embedding_policy`, `risk_level`, `department`, `procedure`, `phase`, `intent`, `source_refs`를 기준으로 색인 전 필터링이 필요하다.
- 응급/법령/CPR 자료는 계속 safety 룰 근거로만 관리한다.
- 성분/스킨케어 데이터는 별도 변환 후 `skin_care_ingredient_reference` 같은 데이터셋 타입으로 분리하는 것이 좋다.

## 구현 기록

- 전체 룰북 위치: `/Users/cheonseongjin/Documents/중앙톤/rag_rulebook/`
- 바로 사용 가능한 curated 문서: `/Users/cheonseongjin/Documents/중앙톤/rag_rulebook/rag/mvp_care_knowledge.jsonl`
- 기존 JSONL 산출물: `/Users/cheonseongjin/Documents/중앙톤/rag_rulebook/derived/`
- 공식 원본 자료: `/Users/cheonseongjin/Documents/중앙톤/rag_rulebook/sources/raw_official/`
- 공식 분류 자료: `/Users/cheonseongjin/Documents/중앙톤/rag_rulebook/sources/official_by_use/`
- 전처리 스크립트: `/Users/cheonseongjin/Documents/중앙톤/rag_rulebook/tools/`
- 검증 결과: `python3 rag_rulebook/tools/validate_triage.py` 실행 시 5개 시나리오 통과

## 후속 과제

- 03번 스킨케어 성분-효능 zip을 읽어 JSONL 변환 스크립트를 추가한다.
- 08번 전문 의학지식 다운로드 zip에서 피부과/외과 subset을 직접 읽도록 스크립트를 수정한다.
- 공식 `rag_candidate` 텍스트를 chunk 단위 JSONL로 변환한다.
- 최종 색인 whitelist manifest를 만든다.
- 리트리버 평가용 질문 세트를 만들어 top-k 검색 품질을 확인한다.
