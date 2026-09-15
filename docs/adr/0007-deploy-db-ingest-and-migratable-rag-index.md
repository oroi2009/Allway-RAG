# ADR-0007: 배포 DB 적재와 마이그레이션 가능한 RAG 인덱스 전략

- 원본: [Notion](https://app.notion.com/p/3bd499bdd63381589579c8939330d052)

## Context

무료 요금제 기반으로 여러 DB를 바꿔가며 사용할 가능성이 높다. 따라서 현재 배포 DB에 RAG 데이터를 바로 적재해도 되는지, 그리고 향후 DB 변경 시 마이그레이션을 어떻게 관리할지 결정해야 한다.

## Decision

현재 배포 DB에 바로 적재해도 된다. 단, 배포 DB는 원본 데이터 저장소가 아니라 **재생성 가능한 RAG 검색 인덱스**로 취급한다. 원본과 정규화된 데이터의 source of truth는 GitHub repo의 `rag_rulebook` JSONL/manifest/rule files로 둔다.

## Rationale

- 무료 요금제나 운영 상황 때문에 DB가 바뀌어도 같은 JSONL에서 다시 적재할 수 있어야 한다.
- DB별 vector 타입, metadata filter 문법, distance function이 달라도 canonical dataset이 안정적이면 마이그레이션 비용이 낮아진다.
- 의료/사후관리 RAG는 출처, safety policy, embedding model, chunk version이 답변 신뢰성과 직결되므로 metadata를 함께 저장해야 한다.
- safety-only/out-of-scope/eval 데이터가 실수로 답변 색인에 들어가지 않도록 ingest 단계에서 manifest 기반 필터링이 필요하다.

## Options Considered

1. 현재 배포 DB를 원본 저장소처럼 사용한다: 빠르지만 DB 변경 시 데이터 일관성과 출처 추적이 깨질 수 있다.
2. 매번 수동으로 DB별 적재 파일을 만든다: 초기에는 가능하지만 마이그레이션 때 반복 실수가 생긴다.
3. `rag_rulebook`을 원본으로 두고 DB는 disposable index로 운영한다: DB 교체가 쉬워지고 감사 가능성이 유지되어 채택한다.

## Consequences

- 배포 DB에는 `content`, `embedding`, `metadata`, `doc_id`, `content_hash`, `index_version`, `embedding_model`을 저장한다.
- `doc_id`와 `content_hash`를 기준으로 idempotent upsert를 구현한다.
- DB를 바꿀 때는 기존 DB에서 export하지 않고, 가능하면 `rag_rulebook`에서 새 DB로 재적재한다.
- `safety_only`, `out_of_scope`, `api_reference`, `*_eval.jsonl`은 기본 환자 답변 인덱스에 넣지 않는다.

## Implementation

- ingest 대상은 `derived/retriever_index_manifest.json`의 `default_patient_answer_index`를 기준으로 한다.
- app code는 특정 DB SDK에 직접 묶지 않고 `VectorStoreAdapter` 같은 얇은 인터페이스를 둔다.
- migration 시에는 `index_version` 또는 `namespace`를 새로 만들고, 새 인덱스 검증 후 traffic을 전환한다.
- metadata filter는 최소한 `dataset_type`, `retrieval_use`, `department`, `procedure`, `phase`, `risk_level`, `source_refs`, `embedding_policy`를 보존한다.

## Follow-up

- 현재 사용하는 배포 DB에 맞춘 ingest adapter를 만든다.
- 적재 후 샘플 질의 10~20개로 검색 결과와 source citation을 확인한다.
- DB/embedding model 변경 시마다 ADR 또는 changelog에 `index_version`, 모델명, chunk count, 검증 결과를 남긴다.
