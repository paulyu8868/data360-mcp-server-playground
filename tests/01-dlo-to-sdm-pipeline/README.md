# DLO → SDM 파이프라인 테스트

Data Cloud의 데이터 인입 흐름 — **DLO 스키마 확인 → DMO 생성 → 필드 매핑 → SDM(시멘틱 모델) 생성 → 시멘틱 쿼리** — 를 data360 MCP 서버의 5개 패밀리(DLO, DMO, Mappings, SDM, Query)로 엔드투엔드 검증한 기록.

## 가상 시나리오

실제 회사명/스키마/데이터는 포함하지 않음. 대신 가상의 소매업체 **Northwind**의 주문 마스터 데이터를 예시로 사용:

- DLO: `northwind_order_master__dll`
- DMO: `northwind_order_master__dlm`
- SDM 모델: `NorthwindOrderModel` ("Northwind Order Master Model")

실제 테스트는 45개 필드(커스텀 42 + 시스템 3)를 가진 스키마로 진행했으나, 아래 예시에서는 패턴 확인용으로 10개 내외의 대표 필드만 표기했다. payload 전체는 [`payloads/`](./payloads)에 있다.

## 파이프라인 단계별 결과

### 1. DLO 스키마 조회 — `d360_dlo_get`

DLO에 어떤 필드가 들어와 있는지 확인. 시스템 필드(`KQ_fm_pk__c`, `DataSource__c`, `DataSourceObject__c`)가 커스텀 필드와 함께 같이 내려온다.

→ [`payloads/01_dlo_get.json`](./payloads/01_dlo_get.json)

### 2. DMO 생성 — `d360_dmo_create`

DLO 필드 목록을 그대로 가지고 DMO를 생성. `fields[]` 각 항목에 `creationType`을 넣으면 `400 JSON_PARSER_ERROR`로 거부된다 — 빼고 보내야 성공한다. 성공 응답에는 요청하지 않은 시스템 필드(`KQ_*`, `DataSource__c`, `DataSourceObject__c`, `InternalOrganization__c`)가 자동으로 추가되어 돌아온다.

→ [`payloads/02_dmo_create.json`](./payloads/02_dmo_create.json)

### 3. 필드 매핑 생성/조회 — `d360_dmo_mapping_create` / `d360_dmo_mapping_get`

DLO ↔ DMO 필드를 1:1로 매핑. 매핑 객체 이름은 `{source}_map_{target}_{epoch-ms}` 패턴으로 자동 생성되고, 필드 단위 매핑 ID는 `{source필드}_fieldmap_{target필드}` 패턴이다. 시스템 필드는 별도 요청 없이 자동으로 매핑된다.

→ [`payloads/03_dmo_mapping_create.json`](./payloads/03_dmo_mapping_create.json)

### 4. DMO 필드 description 수정 — `d360_dmo_update` (알려진 한계)

기존 필드의 `description`을 수정하면 `200 OK`가 돌아오지만, 읽기 계열 툴(`d360_dmo_get`, `d360_sdm_dimensions_list`, `d360_metadata`) 중 어느 것도 응답에 `description` 필드 자체를 노출하지 않는다. 즉 **API만으로는 실제로 반영됐는지 확인할 방법이 없다.** Setup → Data Model UI에서 직접 확인하거나 입력하는 것이 현재 유일하게 검증 가능한 경로.

→ [`payloads/04_dmo_update_description.json`](./payloads/04_dmo_update_description.json)

### 5. SDM 모델 + 데이터 객체 생성 — `d360_sdm_create` → `d360_sdm_data_object_create`

먼저 `d360_sdm_create`로 빈 모델 셸을 만들고, `d360_sdm_data_object_create`에 `shouldIncludeAllFields: true`로 DMO를 통째로 얹는다. 텍스트/날짜 계열 필드는 dimension, 숫자 계열 필드는 measurement로 자동 분류되며 숫자 필드에는 기본 `SUM` aggregation이 자동 할당된다. 실제 테스트에서는 38개 dimension + 7개 measurement가 자동 분류됨.

데이터 객체를 갱신하는 in-place update API는 없고, `d360_sdm_data_object_delete` 후 재생성하는 것이 갱신 우회 방법으로 쓰였다.

→ [`payloads/05_sdm_create.json`](./payloads/05_sdm_create.json), [`payloads/06_sdm_data_object_create.json`](./payloads/06_sdm_data_object_create.json)

### 6. 시멘틱 쿼리 — `d360_sdm_query` (주의: structuredSemanticQuery는 문자열)

`payload_examples('d360_sdm_query')`의 예시는 `structuredSemanticQuery`가 객체처럼 보이지만, 실제 입력 스키마 타입은 **JSON 문자열**이다. 객체를 그대로 넣으면 `Cannot deserialize value of type java.lang.String from Object value (token JsonToken.START_OBJECT)` 에러가 난다. `JSON.stringify`한 문자열로 감싸서 전달하면 성공.

→ [`payloads/07_sdm_query.json`](./payloads/07_sdm_query.json)

### 7. Query 패밀리 — 메타데이터 조회만 검증 (`d360_metadata`)

이번 테스트에서는 `d360_metadata`(엔티티 스키마 조회)까지만 확인했고, 실제 SQL 실행 툴(`d360_query_sql`, `d360_query_sql_rows`)은 호출하지 않았다. Query 패밀리를 "완료"로 표시하지만, **SQL 실행 자체는 별도로 검증이 필요**하다는 점을 명확히 해 둔다.

→ [`payloads/08_metadata.json`](./payloads/08_metadata.json)

## 발견된 이슈 요약

| 패밀리 | 이슈 | 비고 |
|---|---|---|
| DMO | `d360_dmo_create`의 `fields[].creationType`은 입력 불가 → `400 JSON_PARSER_ERROR` | [SETUP.md §7](../../SETUP.md)에도 명시 |
| DMO | `d360_dmo_update`로 description 수정해도 API 응답으로는 반영 확인 불가 | UI에서만 확인 가능 |
| SDM | `structuredSemanticQuery`는 객체가 아니라 JSON 문자열로 전달 | `payload_examples`의 예시 표기와 실제 스키마 타입이 다름 |
| SDM | 데이터 객체 in-place 갱신 API 없음 | delete → re-create로 우회 |
| Query | 이번 테스트는 메타데이터 조회만 검증, SQL 실행은 미검증 | 후속 테스트 필요 |

## 참고

- 서버 설치/인증 트러블슈팅: [SETUP.md](../../SETUP.md)
- 다음 테스트 대상: README 진행 테이블의 나머지 패밀리(CalculatedInsights, Connection, DataStreams 등)
