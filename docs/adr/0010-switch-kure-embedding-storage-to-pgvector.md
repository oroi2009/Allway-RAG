# ADR-0010: KURE 임베딩 저장소를 JSONB fallback에서 pgvector로 전환

- 원본: [Notion](https://app.notion.com/p/3bd499bdd633810e97abcae4b4165ea0)

## 상태

Accepted

## 날짜

2026-08-15

## 맥락

ADR-0009에서 Azure PostgreSQL의 `vector` extension allow-list 제한 때문에 KURE-v1 임베딩을 임시로 `jsonb`에 저장했다. 이후 Azure 서버 파라미터 `azure.extensions`에 `vector`가 허용되어 pgvector를 사용할 수 있게 되었다.

## 결정

RAG 임베딩 저장소를 `jsonb` fallback에서 pgvector `vector(1024)` 컬럼으로 전환한다. KURE-v1은 1024차원 임베딩을 생성하므로 `embedding vector(1024)`를 사용하고, cosine distance 검색을 위해 HNSW 인덱스를 생성한다.

## 실행 결과

- `CREATE EXTENSION vector` 성공.
- `rag_documents.embedding` 컬럼 생성: `vector` 타입.
- 기존 `embedding_values jsonb`는 nullable fallback 컬럼으로 남겨두었다.
- `idx_rag_documents_embedding_hnsw` 인덱스 생성.
- 90건 전체가 `embedding_storage = pgvector`, `embedding_dimensions = 1024`로 재적재되었다.
- 최근 적재 run id: 2.

## 검증

- `rag_documents` 내 `pgvector/1024` 레코드 수: 90건.
- `vector` extension 버전: 0.8.2.
- smoke query 결과, `GUIDE-DERM-PICO-D0-D1-WASH` 임베딩으로 cosine distance 검색 시 자기 자신이 distance 0으로 Top-1 반환되었다.

## 영향

- 다음 백엔드 retriever 구현은 DB에서 `embedding <=> :queryEmbedding` 형태의 cosine distance 정렬을 사용할 수 있다.
- 대량 문서로 확장할 때 JSONB 전수 비교보다 훨씬 유리하다.
- 운영 중 DB를 마이그레이션할 때도 `embedding_model`, `embedding_dimensions`, `embedding_storage`, `index_version`, `content_hash`를 함께 유지해야 한다.

## 관련 파일

- `/Users/cheonseongjin/Documents/Backend/scripts/ingest_rag_documents.py`
- `/Users/cheonseongjin/Documents/Backend/build/rag-ingest/rag_ingest_summary.json`
