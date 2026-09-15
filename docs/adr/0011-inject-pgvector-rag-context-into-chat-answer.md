# ADR-0011: AI 챗봇 응답 생성에 pgvector RAG 컨텍스트 주입

- 원본: [Notion](https://app.notion.com/p/3bd499bdd633817b872ddd54c37f6451)

## 상태

Accepted

## 날짜

2026-08-15

## 맥락

RAG 문서 90건이 KURE-v1 임베딩과 pgvector `vector(1024)`로 적재되었다. 다음 단계는 환자 질문에 대해 관련 근거 문서를 검색하고, OpenAI 답변 생성 시 해당 근거를 프롬프트에 주입하는 것이다.

## 결정

백엔드 Java 서버는 RAG 검색과 답변 생성 흐름을 담당하고, KURE 임베딩 생성은 별도 HTTP 서비스로 분리한다. 운영 이미지가 Java JRE 기반이므로 백엔드 프로세스 안에서 PyTorch/SentenceTransformer를 직접 실행하지 않는다.

## 구현 방식

- `RAG_ENABLED=false`를 기본값으로 두어 기존 챗봇 동작을 보존한다.
- `RAG_ENABLED=true`일 때 `RAG_EMBEDDING_SERVICE_BASE_URL`의 `/embed` 엔드포인트로 질문 임베딩을 요청한다.
- 응답 임베딩은 1024차원인지 검증한다.
- DB에서는 `rag_documents.embedding <=> CAST(:queryVector AS vector)` 기준으로 cosine distance Top-k를 조회한다.
- 검색된 문서 제목, 출처, dataset type, retrieval use, risk level, similarity, content, answer template을 OpenAI 프롬프트의 `RAG 검색 근거` 블록에 넣는다.
- 근거가 없으면 구체 사후관리 지침을 단정하지 말고 병원 문의를 안내하도록 프롬프트를 제한한다.

## 관련 설정

- `rag.enabled`
- `rag.index-version`
- `rag.top-k`
- `rag.min-similarity`
- `rag.max-document-characters`
- `rag.embedding.service-base-url`
- `rag.embedding.path`
- `rag.embedding.model`
- `rag.embedding.dimensions`

## 관련 파일

- `/Users/cheonseongjin/Documents/Backend/src/main/java/com/centerton/centerton/domain/aichat/rag/`
- `/Users/cheonseongjin/Documents/Backend/src/main/java/com/centerton/centerton/domain/aichat/client/OpenAiChatAnswerService.java`
- `/Users/cheonseongjin/Documents/Backend/scripts/serve_kure_embeddings.py`
- `/Users/cheonseongjin/Documents/Backend/scripts/query_rag_documents.py`

## 검증

- `sh gradlew test` 성공.
- `scripts/query_rag_documents.py --samples --top-k 3`로 실제 DB pgvector 검색 성공.
- `scripts/serve_kure_embeddings.py`의 `/health`, `/embed` 응답 확인. `/embed`는 1024차원 벡터를 반환했다.

## 남은 작업

- 응급 hard-stop 룰을 RAG/LLM 호출보다 먼저 적용하는 코드가 필요하다.
- 실제 운영에서는 KURE 임베딩 HTTP 서비스를 배포 환경에 맞춰 상시 실행해야 한다.
- 통합 테스트에서는 `RAG_ENABLED=true`와 임베딩 서비스 URL을 넣고 실제 챗봇 응답을 확인한다.
