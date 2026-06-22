# Data 360 MCP Server Playground

[forcedotcom/d360-mcp-server](https://github.com/forcedotcom/d360-mcp-server) — Salesforce Data 360 REST API를 3개 파사드 툴(`search`, `payload_examples`, `execute`)로 감싸는 로컬 MCP 서버를 기능별로 하나씩 테스트하고 기록하는 저장소.

- 설치/연동 가이드: [SETUP.md](./SETUP.md)
- 기능별 테스트 기록: [`/tests`](./tests) — 패밀리 하나씩 PR로 추가됨
- 아래 각 패밀리 항목을 클릭하면 설명/가능한 기능/테스트 내용이 펼쳐집니다.

## 진행 상태

| 패밀리 | 상태 |
|---|---|
| DLO | ✅ 테스트 완료 |
| DMO | ✅ 테스트 완료 |
| Mappings | ✅ 테스트 완료 |
| SDM | ✅ 테스트 완료 |
| Query | ✅ 테스트 완료 |
| CalculatedInsights | 🔲 예정 |
| Connection | 🔲 예정 |
| DataStreams | 🔲 예정 |
| IdentityResolution | 🔲 예정 |
| Segment | 🔲 예정 |
| Activation | 🔲 예정 |
| DataAction | 🔲 예정 |
| DataTransform | 🔲 예정 |
| StandardMappings | 🔲 예정 |
| Smart | 🔲 예정 |
| Dataspace | 🔲 예정 |
| DataKit | 🔲 예정 |
| Retriever | 🔲 예정 |
| SearchIndex | 🔲 예정 |

> 테스트 예시의 데이터/필드명은 모두 가상 데이터입니다. 실제 회사 데이터는 포함하지 않습니다.

## 패밀리별 상세

<details>
<summary><b>DLO</b> — ✅ 테스트 완료</summary>

원시 데이터가 적재되는 컨테이너. 보통 DataStreams가 자동 생성.

**가능한 기능:** 조회, 생성, 수정, 삭제

**테스트 내용** ([전체 payload](tests/01-dlo-to-sdm-pipeline/payloads/01_dlo_get.json)):
- `execute("d360_dlo_get", {"dloName": "northwind_order_master__dll"})` 호출 → 필드 목록 확인. 커스텀 필드와 시스템 필드(`KQ_fm_pk__c`, `DataSource__c`, `DataSourceObject__c`)가 함께 반환됨. 실제 테스트는 45개 필드(커스텀 42 + 시스템 3)였음.
</details>

<details>
<summary><b>DMO</b> — ✅ 테스트 완료</summary>

고객/상품/주문 등 비즈니스 엔티티의 타겟 스키마. 이름은 `__dlm`로 끝남.

**가능한 기능:** 조회, 생성, 수정, 삭제

**테스트 내용** ([전체 payload](tests/01-dlo-to-sdm-pipeline/payloads/02_dmo_create.json), [description 수정 payload](tests/01-dlo-to-sdm-pipeline/payloads/04_dmo_update_description.json)):
- `execute("d360_dmo_create", {request:{fields:[{...,"creationType":"Custom"}]}})` → **실패** (`400 JSON_PARSER_ERROR: Unrecognized field "creationType"`)
- 같은 요청에서 `creationType` 키 제거 후 재호출 → **성공**, DMO 생성됨. 응답에 요청하지 않은 시스템 필드(`KQ_fm_pk__c`, `DataSource__c`, `DataSourceObject__c`, `InternalOrganization__c`)가 자동으로 추가되어 돌아옴
- `execute("d360_dmo_update", {request:{fields:[{apiName:"order_amount__c", description:"..."}]}})` → `200 OK` 반환되지만, `d360_dmo_get` / `d360_sdm_dimensions_list` / `d360_metadata` 어느 응답에도 description 필드가 노출되지 않아 **API로는 실제 반영 여부를 확인할 방법이 없음** (알려진 한계)
</details>

<details>
<summary><b>Mappings</b> — ✅ 테스트 완료</summary>

소스 필드 ↔ DMO 필드 매핑. 모든 인입 파이프라인에 필수.

**가능한 기능:** 매핑 CRUD, 필드 단위 추가/삭제

**테스트 내용** ([전체 payload](tests/01-dlo-to-sdm-pipeline/payloads/03_dmo_mapping_create.json)):
- `execute("d360_dmo_mapping_create", {request:{sourceEntityDeveloperName:"northwind_order_master__dll", targetEntityDeveloperName:"northwind_order_master__dlm", fieldMapping:[...]}})` 호출 → 매핑 생성 확인
- 매핑 객체 이름은 `{source}_map_{target}_{epoch-ms}` 패턴, 필드 단위 매핑 ID는 `{source필드}_fieldmap_{target필드}` 패턴으로 자동 생성됨을 `d360_dmo_mapping_get`으로 확인
- 시스템 필드(`KQ_*`, `DataSource__c` 등)는 별도 요청 없이 자동으로 매핑됨
</details>

<details>
<summary><b>SDM</b> — ✅ 테스트 완료</summary>

DMO 위에 얹는 BI/리포팅용 시멘틱 레이어.

**가능한 기능:** 모델·데이터객체·관계·계산차원·계산측정값·메트릭 CRUD, 시멘틱 쿼리, 검증·복제·의존성 조회

**테스트 내용** ([sdm_create](tests/01-dlo-to-sdm-pipeline/payloads/05_sdm_create.json), [data_object_create](tests/01-dlo-to-sdm-pipeline/payloads/06_sdm_data_object_create.json), [sdm_query](tests/01-dlo-to-sdm-pipeline/payloads/07_sdm_query.json)):
- `execute("d360_sdm_create", {request:{apiName:"NorthwindOrderModel", label:"...", dataspace:"default"}})` → 모델 셸 생성, Salesforce 내부 레코드 id 반환 확인
- `execute("d360_sdm_data_object_create", {modelApiNameOrId:"NorthwindOrderModel", request:{dataObjectName:"...", shouldIncludeAllFields:true}})` → 텍스트/날짜 필드는 dimension, 숫자 필드는 measurement로 자동 분류, 숫자 필드엔 기본 `SUM` aggregation 자동 할당 확인 (실제 테스트는 38 dimension + 7 measurement). in-place 갱신 API가 없어 `d360_sdm_data_object_delete` 후 재생성하는 방식으로 우회
- `execute("d360_sdm_query", {structuredSemanticQuery: {...객체...}})` → **실패** (`Cannot deserialize value of type java.lang.String from Object value`)
- 같은 쿼리를 `structuredSemanticQuery`를 JSON 문자열로 직렬화해서 재호출 → **성공**, region별 집계 결과 반환 확인 (`payload_examples`의 예시 표기와 실제 입력 스키마 타입이 다른 케이스)
</details>

<details>
<summary><b>Query</b> — ✅ 테스트 완료 (SQL 실행 자체는 미검증)</summary>

Data 360에 대한 SQL 실행 및 메타데이터/프로필/인사이트 조회.

**가능한 기능:** SQL 실행·상태조회·취소·행조회, 메타데이터 검색, 프로필/인사이트/데이터그래프 쿼리

**테스트 내용** ([전체 payload](tests/01-dlo-to-sdm-pipeline/payloads/08_metadata.json)):
- `execute("d360_metadata", {"entityName":"northwind_order_master__dlm","entityType":"DataModelObject"})` 호출 → 엔티티 필드 스키마(이름/타입/businessType) 조회 확인
- **`d360_query_sql` / `d360_query_sql_rows`(실제 SQL 실행)는 이번 테스트에서 호출하지 않음** — 메타데이터 조회까지만 검증됨, 후속 테스트 필요
</details>

<details>
<summary><b>CalculatedInsights</b> — 🔲 예정</summary>

SQL로 LTV·이탈위험 등 지표·스코어 계산. Segment가 쓰려면 ACTIVE 필요.

**가능한 기능:** CRUD, 활성/비활성화, 실행·실행상태조회, 검증
</details>

<details>
<summary><b>Connection</b> — 🔲 예정</summary>

외부 시스템(Snowflake/S3/DB/CRM/ERP/POS)과의 연결. 모든 파이프라인의 첫 단계.

**가능한 기능:** 연결 CRUD·테스트, 커넥터 목록/메타데이터, Snowflake 스키마·오브젝트·필드 탐색
</details>

<details>
<summary><b>DataStreams</b> — 🔲 예정</summary>

외부 데이터를 실제로 인입하는 파이프라인. Connection이 선행되어야 함.

**가능한 기능:** 스트림 CRUD·실행, SFDC/S3/Snowflake/서드파티 커넥터별 생성
</details>

<details>
<summary><b>IdentityResolution</b> — 🔲 예정</summary>

여러 소스의 고객 프로필을 매칭·통합해 단일 뷰(360 프로필) 생성.

**가능한 기능:** CRUD, 퍼블리시, 실행
</details>

<details>
<summary><b>Segment</b> — 🔲 예정</summary>

마케팅용 오디언스 세그먼트 생성. 활성 CalculatedInsights 필요.

**가능한 기능:** CRUD, 퍼블리시(멤버십 계산), 비활성화
</details>

<details>
<summary><b>Activation</b> — 🔲 예정</summary>

세그먼트/오디언스를 이메일·SMS·푸시·광고 등 외부 채널로 푸시.

**가능한 기능:** 액티베이션 CRUD, 액티베이션 타겟 CRUD
</details>

<details>
<summary><b>DataAction</b> — 🔲 예정</summary>

데이터 조건 충족 시 실시간 알림·워크플로우 트리거.

**가능한 기능:** 액션 목록/조회/생성, 액션 타겟 CRUD
</details>

<details>
<summary><b>DataTransform</b> — 🔲 예정</summary>

스케줄 또는 수동 실행되는 SQL 기반 데이터 변환.

**가능한 기능:** CRUD, 실행, 검증, 스케줄 조회/설정
</details>

<details>
<summary><b>StandardMappings</b> — 🔲 예정</summary>

500개+ 사전정의 표준 매핑 템플릿으로 소스 오브젝트 전체를 일괄 매핑.

**가능한 기능:** 매핑 미리보기, 일괄 생성
</details>

<details>
<summary><b>Smart</b> — 🔲 예정</summary>

AI 보조로 필드 매핑·이벤트 날짜 필드 등을 자동 추천/생성.

**가능한 기능:** 매핑 추천, 필드 매칭 미리보기, 스마트 데이터스트림 생성, 이벤트 날짜 추천
</details>

<details>
<summary><b>Dataspace</b> — 🔲 예정</summary>

데이터 격리용 워크스페이스(테넌트 파티션). 대부분 작업에 dataspace 이름 필요.

**가능한 기능:** CRUD, 멤버(권한) 추가/삭제/조회
</details>

<details>
<summary><b>DataKit</b> — 🔲 예정</summary>

사전 패키징된 데이터 모델 템플릿을 한 번에 배포.

**가능한 기능:** 목록/조회/매니페스트, 배포·배포취소, 배포상태·컴포넌트상태·의존성 조회
</details>

<details>
<summary><b>Retriever</b> — 🔲 예정</summary>

RAG/AI 검색용 retriever 엔드포인트 생성·관리 (Data 360 데이터에 대한 시멘틱 검색).

**가능한 기능:** retriever CRUD, retriever config CRUD
</details>

<details>
<summary><b>SearchIndex</b> — 🔲 예정</summary>

Data 360 엔티티에 대한 검색 인덱스 관리 (Retriever가 사용하는 인덱스).

**가능한 기능:** CRUD, 설정, 처리 이력 조회
</details>
