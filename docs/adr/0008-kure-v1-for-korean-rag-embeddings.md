# ADR-0008: 한국어 RAG 임베딩 모델로 KURE-v1 우선 검토

- 원본: [Notion](https://app.notion.com/p/3bd499bdd63381bfbdbaeb584944cb8f)

## 상태

Accepted

## 날짜

2026-08-15

## 맥락

RAG 기본 환자 답변 인덱스 90건을 실제 DB에 적재하기 전에 임베딩 모델을 확정해야 했다. 초기 스크립트는 OpenAI `text-embedding-3-small`을 기본값으로 사용했지만, 서비스 질의는 대부분 한국어 환자 증상 문장이고 KURE-v1은 한국어 검색에 특화된 공개 임베딩 모델이므로 비교가 필요했다.

## 비교 대상

- OpenAI `text-embedding-3-small`: API 기반, 1536차원, 기존 OpenAI 키로 즉시 사용 가능.
- KURE-v1 `nlpai-lab/KURE-v1`: Hugging Face 공개 모델, MIT 라이선스, 1024차원, 로컬 또는 별도 서빙 필요.

## 평가 방법

- 대상 코퍼스: `default_patient_answer_index` 90건.
- 평가 질문: 피코 프락셀, 코성형, 필러, 보톡스, 두드러기, 상처/흉터, 여드름 흉터, 레이저 주의사항 등 대표 한국어 질문 25개.
- 판정 기준: 예상 문서 ID 또는 예상 문서 prefix가 Top-k 안에 들어오는지 확인.
- 평가 스크립트: `/Users/cheonseongjin/Documents/Backend/scripts/evaluate_embedding_retrieval.py`
- 결과 리포트: `/Users/cheonseongjin/Documents/Backend/build/rag-eval/embedding_retrieval_eval.md`

## 결과

| Provider | Hit@1 | Hit@3 | Hit@5 |
|---|---:|---:|---:|
| OpenAI text-embedding-3-small | 20/25 (80%) | 21/25 (84%) | 22/25 (88%) |
| KURE-v1 | 22/25 (88%) | 25/25 (100%) | 25/25 (100%) |

## 관찰

- KURE-v1은 필러 회복, 필러 시야/피부색 경고, 여드름 흉터 사후관리, 레이저 전후 레티노이드/글리콜릭산 질문에서 OpenAI보다 더 관련 문서를 안정적으로 Top-3 안에 배치했다.
- OpenAI는 일부 한국어 의료미용 질의에서 피코 프락셀 일반 문서나 두드러기 문서를 먼저 가져오는 사례가 있었다.
- KURE-v1도 모든 케이스에서 Top-1이 완벽하지는 않았다. 예를 들어 코성형 비대칭 불안 질문은 Top-1에 일반 부기/모양 변화 문서를 가져왔고, 기대 문서는 Top-2에 있었다.

## 결정

기본 환자 답변 인덱스의 1차 임베딩 모델은 OpenAI가 아니라 KURE-v1을 우선 후보로 전환한다. OpenAI `text-embedding-3-small`은 빠른 API 기반 baseline 또는 운영 fallback 후보로 유지한다.

## 결과 영향

- DB 벡터 차원은 OpenAI 기준 `vector(1536)`이 아니라 KURE 기준 `vector(1024)`를 우선 고려해야 한다.
- 적재 스크립트는 OpenAI 전용이 아니라 embedding provider와 dimensions를 명시적으로 기록하는 구조로 수정해야 한다.
- KURE는 무료 공개 모델이지만 운영에서는 로컬/서버 임베딩 실행 환경, 모델 캐시, 배치 처리, 배포 서버 자원 사용량을 따로 관리해야 한다.

## 다음 작업

- KURE 기준 적재 스크립트 또는 임베딩 생성 파이프라인을 준비한다.
- DB에는 `embedding_model`, `embedding_dimensions`, `index_version`, `content_hash`를 반드시 저장한다.
- 실제 챗봇 연결 전에는 Top-k 결과와 threshold를 추가로 튜닝한다.
