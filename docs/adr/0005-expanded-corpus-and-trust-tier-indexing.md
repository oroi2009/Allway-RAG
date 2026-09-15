# ADR-0005: 확장 코퍼스 변환과 Trust Tier 색인 정책

- 원본: [Notion](https://app.notion.com/p/3bd499bdd6338145b22aca95d8d0320d)
- 상태: Accepted
- 확정일: 2026-08-15
- 범위: 성형외과, 피부과, 피부미용 시술 후 이상반응 및 사후관리 챗봇

## 상황

사용자는 "정보는 많을수록 좋은 것 아닌가"와 "어설픈 RAG는 생성형 AI만 쓰는 것보다 나쁠 수 있다"는 문제를 제기했다. 이 지적은 맞다. RAG는 데이터를 많이 넣는 기술이 아니라, 검색되어도 되는 정보를 정확한 용도와 신뢰도에 따라 분리해 쓰는 구조여야 한다.

로컬 zip 원본을 확인한 결과 압축 해제 없이 직접 읽을 수 있었다. 따라서 사용자가 별도로 zip을 풀지 않아도 전처리 스크립트가 다운로드 원본 zip 내부의 JSON/JSONL을 직접 읽도록 확장했다.

## 결정

Training 데이터는 RAG 후보로 변환하고, Validation 데이터는 평가용 holdout으로 분리한다.

색인은 두 단계로 나눈다.

- 기본 환자 답변 색인: curated 사후관리 문서와 공식 RAG 후보 chunk만 사용
- 확장 참고 색인: 스킨케어 성분, 문제성 피부/메이크업, 전문 QA를 포함하되 reference-only로 제한

즉, 가능한 많은 데이터를 준비하되, 모든 데이터를 같은 신뢰도로 섞지 않는다. 응급 판단, 진단 확정, 처방/복약 지시에는 확장 참고 데이터를 사용하지 않는다.

## 근거

- 데이터가 많아도 노이즈가 많으면 retriever가 엉뚱한 문서를 가져오고, 생성 모델은 그 근거를 그럴듯하게 포장할 수 있다.
- 공식 문서와 curated 문서는 환자 답변의 주 근거로 적합하다.
- AIHub 기반 QA/화장품/성분 데이터는 커버리지 확장에는 유용하지만, 법적/임상적 최종 근거로는 약하다.
- Validation 데이터를 전부 색인하면 검색 품질 평가용 샘플이 사라진다.
- `chain_of_thought`는 RAG 근거가 아니므로 03번 스킨케어 성분 데이터 산출물에서 제외한다.

## 구현 결과

생성된 RAG 후보/평가 산출물은 다음과 같다.

- `rag/mvp_care_knowledge.jsonl`: curated 사후관리 문서 17개
- `derived/official_rag_candidate_chunks.jsonl`: 공식 RAG 후보 chunk 15개
- `derived/skin_care_ingredient_rag.jsonl`: 스킨케어 성분/효능 Training 문서 8,000개
- `derived/problem_skin_makeup_rag.jsonl`: 문제성 피부/메이크업 Training 문서 8,031개
- `derived/source_medical_qa_rag.jsonl`: 피부과/외과 전문 QA Training 문서 3,630개
- `derived/skin_care_ingredient_eval.jsonl`: 스킨케어 성분/효능 Validation holdout 1,000개
- `derived/problem_skin_makeup_eval.jsonl`: 문제성 피부/메이크업 Validation holdout 1,000개
- `derived/source_medical_qa_eval.jsonl`: 피부과/외과 전문 QA Validation holdout 436개

검증 결과 JSONL 파싱은 정상이며, `chain_of_thought` 필드는 산출물에 남기지 않았다. 기존 triage 검증 스크립트도 5개 시나리오를 통과했다.

## 검토한 선택지

1. 모든 Training/Validation 데이터를 한 번에 색인한다: 커버리지는 넓지만 평가셋이 사라지고 노이즈가 증가한다.
2. 공식/curated 문서만 색인한다: 안전하지만 사용자 질문 커버리지가 좁다.
3. Training은 RAG 후보, Validation은 평가용으로 분리하고 trust tier를 둔다: 커버리지와 품질 검증을 동시에 확보하므로 채택한다.

## 영향

- 리트리버는 `retriever_index_manifest.json`의 색인 세트를 따라야 한다.
- 답변 생성 시 `retrieval_use`와 `trust_level`을 필터로 사용해야 한다.
- reference-only 데이터가 공식/curated 근거를 이기면 안 된다.
- 안전 룰과 응급 CTA는 계속 RAG보다 먼저 동작한다.

## 구현 기록

- 전처리 스크립트: `/Users/cheonseongjin/Documents/중앙톤/rag_rulebook/tools/build_rag_datasets.py`
- 데이터셋 manifest: `/Users/cheonseongjin/Documents/중앙톤/rag_rulebook/derived/dataset_manifest.json`
- 색인 manifest: `/Users/cheonseongjin/Documents/중앙톤/rag_rulebook/derived/retriever_index_manifest.json`
- 검색 스키마: `/Users/cheonseongjin/Documents/중앙톤/rag_rulebook/rag/retrieval_schema.md`
- 추가 수집 후보: `/Users/cheonseongjin/Documents/중앙톤/rag_rulebook/sources/additional_source_candidates.md`

## 추가 수집 후보

- 식품의약품안전처 의료용 레이저 안전사용 안내서
- 식품의약품안전처 성형용 필러 안전사용 안내서
- AAD 레이저/여드름 흉터 치료 후 self-care 자료
- ASPS 필러 safety/recovery, 코성형 recovery 자료

## 후속 과제

- 추가 후보 출처의 PDF/HTML을 실제 원본으로 저장하고 source manifest에 편입한다.
- 필러 시야 이상, 피부 창백/괴사 의심, 심한 통증은 hard-stop 후보로 룰화한다.
- 색인 이후 Validation holdout으로 top-k 검색 품질을 평가한다.
