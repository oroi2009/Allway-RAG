# ADR-0003: 공식 자료를 RAG 후보와 안전 전용으로 분리

- 원본: [Notion](https://app.notion.com/p/3bb499bdd633817daa71d771b514f518)
- 상태: Accepted
- 확정일: 2026-08-13
- 범위: 성형외과, 피부과, 피부미용 시술 후 이상반응 및 사후관리 챗봇

## 상황

질병관리청, E-Gen, 국가법령정보센터, 생활법령정보, 119, 행정안전부 자료를 원본으로 수집했다. 초기 수집에는 심근경색, 뇌졸중, 열사병, 목 이물질 같은 일반 응급 자료도 포함됐다.

이후 서비스 범위가 성형/피부미용 사후관리라는 점이 다시 확인되었다. 따라서 모든 공식 자료를 한 벡터DB에 넣는 대신, 사용 목적별로 분리해야 했다.

## 결정

공식 자료를 네 버킷으로 나눈다.

- `rag_candidate`: 답변 본문 근거로 사용할 수 있는 문서
- `safety_only`: 벡터DB에는 넣지 않고 hard-stop 응급 룰과 CTA 근거로만 쓰는 문서
- `out_of_scope`: 현재 서비스 범위 밖이라 보관만 하고 기본 파이프라인에서 제외하는 문서
- `api_reference`: 향후 API 수집 전환 검토용 문서

원본 HTML과 추출 텍스트는 삭제하지 않고 `raw_official`에 보존한다. 분류 결과는 `official_by_use`에 복사해 파이프라인이 안전한 입력만 읽도록 한다.

## 현재 분류 결과

`rag_candidate` 2개:

- 질병관리청 국가건강정보포털 상처관리와 흉터예방
- 질병관리청 국가건강정보포털 두드러기

`safety_only` 8개:

- 질병관리청 국가건강정보포털 응급상황정보
- 질병관리청 국가건강정보포털 심폐소생술
- 찾기쉬운 생활법령정보 응급의료의 개념 등
- 국가법령정보센터 응급의료에 관한 법률 시행규칙 제2조
- E-Gen 응급상황시 대처요령
- E-Gen 기본 응급처치
- 119 안전신고센터 소소심 캠페인
- 행정안전부 안전 배움터 구조·구급

`out_of_scope` 4개:

- 질병관리청 국가건강정보포털 급성 심근경색증
- 질병관리청 뇌졸중 조기증상 의심되면 즉시 119
- E-Gen 열사병/일사병 응급처치
- E-Gen 목 이물질 응급처치

`api_reference` 2개:

- 공공데이터포털 질병관리청 국가건강정보포털 OpenAPI
- 국가법령정보 공동활용 Open API 가이드

## 근거

- 상처, 흉터, 두드러기/알레르기는 성형/피부시술 후 사용자 질문과 직접 연결된다.
- CPR, 응급 법령, E-Gen 응급 대처 자료는 중요하지만 답변 생성용으로 검색되면 일반 응급 상담으로 흘러갈 수 있다.
- 심근경색, 뇌졸중, 열사병, 목 이물질은 안전상 언급될 수는 있으나 이 서비스의 주 도메인이 아니다.
- 원본을 보존하면 이후 범위 확장이나 설계 리뷰 시 왜 제외했는지 다시 검토할 수 있다.

## 검토한 선택지

1. 공식 자료 전체를 RAG에 넣는다: 출처는 신뢰할 수 있지만 서비스 범위가 흐려지고, 심장/뇌졸중 같은 일반 응급 문서가 과검색될 수 있다.
2. 범위 밖 자료를 삭제한다: 파이프라인은 깔끔하지만 추후 감사와 재검토 근거가 사라진다.
3. 원본은 보존하고 사용 목적별로 분리한다: 감사 가능성과 안전한 색인을 모두 만족하므로 채택한다.

## 구현 기록

- 원본 보존 위치: `/Users/cheonseongjin/Documents/중앙톤/rag_rulebook/sources/raw_official/`
- 분류 결과 위치: `/Users/cheonseongjin/Documents/중앙톤/rag_rulebook/sources/official_by_use/`
- 분류 매니페스트: `/Users/cheonseongjin/Documents/중앙톤/rag_rulebook/sources/official_by_use/manifests/official_sources_by_use.json`
- 원본 추출 스크립트: `/Users/cheonseongjin/Documents/중앙톤/rag_rulebook/tools/extract_official_source_text.py`
- 분류 스크립트: `/Users/cheonseongjin/Documents/중앙톤/rag_rulebook/tools/partition_official_sources.py`

## 후속 과제

- 성형외과 시술별 사후관리 공식/전문 자료를 추가 수집한다.
- 감염, 출혈, 괴사 의심, 혈관 폐색 의심, 필러/보톡스/레이저/박피/코성형 회복 관련 red-flag를 별도 ADR로 확장한다.
- `safety_only` 문서는 룰 검증 테스트의 근거 링크로 연결한다.
