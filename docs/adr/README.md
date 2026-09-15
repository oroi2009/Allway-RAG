# Architecture Decision Records

원본 Notion 페이지: [RAG](https://app.notion.com/p/3bb499bdd633806d98f3c97fd159e778)

이 디렉터리는 RAG 개발 중 생기는 주요 의사결정을 ADR 형식으로 남기는 기록소다. 새 결정은 `ADR-000N: 제목` 형식으로 추가하고, 각 ADR에는 상황, 결정, 근거, 검토한 선택지, 영향, 구현 기록, 후속 과제를 남긴다.

## Decision Records

| ADR | Title | File |
|---|---|---|
| ADR-0001 | RAG 도입과 안전 룰 우선 구조 | [0001-rag-adoption-and-safety-rule-first.md](0001-rag-adoption-and-safety-rule-first.md) |
| ADR-0002 | RAG 데이터셋 선택 기준 | [0002-rag-dataset-selection-criteria.md](0002-rag-dataset-selection-criteria.md) |
| ADR-0003 | 공식 자료를 RAG 후보와 안전 전용으로 분리 | [0003-separate-official-materials-by-use.md](0003-separate-official-materials-by-use.md) |
| ADR-0004 | 리트리버용 데이터 준비 완료 범위와 남은 전처리 | [0004-retriever-data-readiness.md](0004-retriever-data-readiness.md) |
| ADR-0005 | 확장 코퍼스 변환과 Trust Tier 색인 정책 | [0005-expanded-corpus-and-trust-tier-indexing.md](0005-expanded-corpus-and-trust-tier-indexing.md) |
| ADR-0006 | 추가 신뢰 출처 수집과 안전 룰 보강 | [0006-trusted-sources-and-safety-rule-expansion.md](0006-trusted-sources-and-safety-rule-expansion.md) |
| ADR-0007 | 배포 DB 적재와 마이그레이션 가능한 RAG 인덱스 전략 | [0007-deploy-db-ingest-and-migratable-rag-index.md](0007-deploy-db-ingest-and-migratable-rag-index.md) |
| ADR-0008 | 한국어 RAG 임베딩 모델로 KURE-v1 우선 검토 | [0008-kure-v1-for-korean-rag-embeddings.md](0008-kure-v1-for-korean-rag-embeddings.md) |
| ADR-0009 | Azure PostgreSQL pgvector 제한과 JSONB 임베딩 저장 fallback | [0009-azure-pgvector-limit-and-jsonb-fallback.md](0009-azure-pgvector-limit-and-jsonb-fallback.md) |
| ADR-0010 | KURE 임베딩 저장소를 JSONB fallback에서 pgvector로 전환 | [0010-switch-kure-embedding-storage-to-pgvector.md](0010-switch-kure-embedding-storage-to-pgvector.md) |
| ADR-0011 | AI 챗봇 응답 생성에 pgvector RAG 컨텍스트 주입 | [0011-inject-pgvector-rag-context-into-chat-answer.md](0011-inject-pgvector-rag-context-into-chat-answer.md) |
| ADR-0012 | Spring 1차 응급 하드스톱과 FastAPI RAG 답변 서비스 분리 | [0012-spring-hard-stop-and-fastapi-rag-service-separation.md](0012-spring-hard-stop-and-fastapi-rag-service-separation.md) |
| ADR-0013 | 응급 룰 단독 키워드 축소와 정규식 매칭 도입 | [0013-reduce-broad-emergency-keywords-and-use-regex.md](0013-reduce-broad-emergency-keywords-and-use-regex.md) |
| ADR-0014 | 응급 룰 외부 JSON 로딩과 Spring-FastAPI Docker 네트워크 배포 | [0014-external-emergency-rule-json-and-docker-network.md](0014-external-emergency-rule-json-and-docker-network.md) |
| ADR-0015 | FastAPI Triage Classifier를 RAG 검색 앞단에 배치 | [0015-place-fastapi-triage-classifier-before-rag.md](0015-place-fastapi-triage-classifier-before-rag.md) |
| ADR-0016 | 검색 근거 부족 시 생성 중단과 보수 안내 반환 | [0016-stop-generation-when-evidence-is-insufficient.md](0016-stop-generation-when-evidence-is-insufficient.md) |
| ADR-0017 | 응급 룰북은 정확도를 우선하고 재현율은 답변 가드레일이 담당 | [0017-precision-first-emergency-rulebook.md](0017-precision-first-emergency-rulebook.md) |
| ADR-0018 | 응급 룰 매칭을 compact 및 spaced 이중 정규화로 분리 | [0018-compact-and-spaced-normalization-for-rule-matching.md](0018-compact-and-spaced-normalization-for-rule-matching.md) |
| ADR-0019 | 부정 억제는 명시적 비발생 형태로 한정하고 안을 배제 | [0019-negation-guards-only-for-explicit-non-occurrence.md](0019-negation-guards-only-for-explicit-non-occurrence.md) |
| ADR-0020 | 다국어 입력은 번역 정규화로 처리하고 단일 한국어 룰북을 유지 | [0020-translate-multilingual-inputs-to-korean-rulebook.md](0020-translate-multilingual-inputs-to-korean-rulebook.md) |
| ADR-0021 | 응급 룰북 커버리지 확장의 종료 기준 | [0021-stop-criteria-for-emergency-rulebook-coverage.md](0021-stop-criteria-for-emergency-rulebook-coverage.md) |
| ADR-0022 | FastAPI RAG 서비스를 Rag-Lab으로 이관하고 응급 룰 매칭을 단일 구현으로 통합 | [0022-move-fastapi-rag-service-into-rag-lab.md](0022-move-fastapi-rag-service-into-rag-lab.md) |
| ADR-0023 | pre-RAG triage classifier 제거와 상담 CTA를 답변 속성으로 전환 | [0023-remove-pre-rag-triage-classifier-and-use-cta.md](0023-remove-pre-rag-triage-classifier-and-use-cta.md) |
