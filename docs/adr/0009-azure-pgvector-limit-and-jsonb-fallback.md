# ADR-0009: Azure PostgreSQL pgvector 제한과 JSONB 임베딩 저장 fallback

- 원본: [Notion](https://app.notion.com/p/3bd499bdd6338129aae5eb1a339d9372)

## 상태

Accepted

## 날짜

2026-08-15

## 맥락

KURE-v1 기준 RAG 적재 파이프라인을 실제 서버 DB에 적용하려고 했다. DB는 Azure Database for PostgreSQL이며, `pg_available_extensions`에는 `vector`가 보였지만 실제 `CREATE EXTENSION vector` 실행은 `azure_pg_admin` 사용자 allow-list 제한으로 실패했다.

## 문제

pgvector를 바로 사용할 수 없으면 `embedding vector(1024)` 컬럼과 HNSW 인덱스를 만들 수 없다. 하지만 현재 기본 환자 답변 인덱스는 90건 규모이므로, 벡터 검색을 DB 인덱스에 맡기지 않고 애플리케이션에서 cosine similarity를 계산하는 방식으로도 운영 검증을 시작할 수 있다.

## 결정

현재 배포 DB에는 KURE-v1 임베딩을 `jsonb` 컬럼 `embedding_values`에 저장한다. 적재 파이프라인은 `--storage auto`를 기본값으로 두어, pgvector가 사용 가능하면 `pgvector`, 막혀 있으면 `jsonb` fallback으로 적재한다.

## 결과

- `rag_documents` 테이블이 생성되었다.
- `rag_ingest_runs` 테이블이 생성되었다.
- `default_patient_answer_index` 90건이 `embedding_provider = kure`, `embedding_model = nlpai-lab/KURE-v1`, `embedding_dimensions = 1024`, `embedding_storage = jsonb`로 적재되었다.
- 데이터셋별 검증 결과: official 15, trusted 58, curated/symptom/safety/video/makeup 합산 17.

## 영향

- 당장 DB에서 pgvector HNSW 검색은 사용할 수 없다.
- 초기 챗봇 연결은 `rag_documents`에서 후보 문서를 읽어 애플리케이션 메모리에서 cosine similarity를 계산하는 방식이 적합하다.
- 문서 수가 수천~수만 건으로 늘어나면 JSONB 전수 비교는 한계가 있으므로 pgvector allow-list 설정 또는 별도 벡터 DB가 필요하다.

## 대안

- Azure 서버 파라미터에서 `vector` 확장을 allow-list에 추가한 뒤 pgvector로 재적재한다.
- Supabase/Neon 등 pgvector가 기본적으로 쉬운 DB로 이전한다.
- 별도 벡터 DB를 사용하고 현재 PostgreSQL에는 문서 메타데이터만 둔다.

## 다음 작업

- 백엔드 RAG retriever 구현 시 현재는 JSONB 임베딩을 읽어 cosine similarity를 계산한다.
- pgvector가 가능해지는 시점에 같은 원본과 KURE 모델로 재적재한다.
- 적재 스크립트: `/Users/cheonseongjin/Documents/Backend/scripts/ingest_rag_documents.py`
