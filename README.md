# Data 360 MCP Server Playground

[forcedotcom/d360-mcp-server](https://github.com/forcedotcom/d360-mcp-server) — Salesforce Data 360 REST API를 3개 파사드 툴(`search`, `payload_examples`, `execute`)로 감싸는 로컬 MCP 서버를 기능별로 하나씩 테스트하고 기록하는 저장소.

- 설치/연동 가이드: [SETUP.md](./SETUP.md)
- 기능별 테스트 기록: [`/tests`](./tests) — 패밀리 하나씩 PR로 추가됨

## 진행 상태

| 패밀리 | 상태 | 설명 | 가능한 기능 |
|---|---|---|---|
| DLO | [✅ 테스트 완료](tests/01-dlo-to-sdm-pipeline/README.md) | 원시 데이터가 적재되는 컨테이너. 보통 DataStreams가 자동 생성 | 조회, 생성, 수정, 삭제 |
| DMO | [✅ 테스트 완료](tests/01-dlo-to-sdm-pipeline/README.md) | 고객/상품/주문 등 비즈니스 엔티티의 타겟 스키마. 이름은 `__dlm`로 끝남 | 조회, 생성, 수정, 삭제 |
| Mappings | [✅ 테스트 완료](tests/01-dlo-to-sdm-pipeline/README.md) | 소스 필드 ↔ DMO 필드 매핑. 모든 인입 파이프라인에 필수 | 매핑 CRUD, 필드 단위 추가/삭제 |
| SDM | [✅ 테스트 완료](tests/01-dlo-to-sdm-pipeline/README.md) | DMO 위에 얹는 BI/리포팅용 시멘틱 레이어 | 모델·데이터객체·관계·계산차원·계산측정값·메트릭 CRUD, 시멘틱 쿼리, 검증·복제·의존성 조회 |
| Query | [✅ 테스트 완료](tests/01-dlo-to-sdm-pipeline/README.md) | Data 360에 대한 SQL 실행 및 메타데이터/프로필/인사이트 조회 | SQL 실행·상태조회·취소·행조회, 메타데이터 검색, 프로필/인사이트/데이터그래프 쿼리 |
| CalculatedInsights | 🔲 예정 | SQL로 LTV·이탈위험 등 지표·스코어 계산. Segment가 쓰려면 ACTIVE 필요 | CRUD, 활성/비활성화, 실행·실행상태조회, 검증 |
| Connection | 🔲 예정 | 외부 시스템(Snowflake/S3/DB/CRM/ERP/POS)과의 연결. 모든 파이프라인의 첫 단계 | 연결 CRUD·테스트, 커넥터 목록/메타데이터, Snowflake 스키마·오브젝트·필드 탐색 |
| DataStreams | 🔲 예정 | 외부 데이터를 실제로 인입하는 파이프라인. Connection이 선행되어야 함 | 스트림 CRUD·실행, SFDC/S3/Snowflake/서드파티 커넥터별 생성 |
| IdentityResolution | 🔲 예정 | 여러 소스의 고객 프로필을 매칭·통합해 단일 뷰(360 프로필) 생성 | CRUD, 퍼블리시, 실행 |
| Segment | 🔲 예정 | 마케팅용 오디언스 세그먼트 생성. 활성 CalculatedInsights 필요 | CRUD, 퍼블리시(멤버십 계산), 비활성화 |
| Activation | 🔲 예정 | 세그먼트/오디언스를 이메일·SMS·푸시·광고 등 외부 채널로 푸시 | 액티베이션 CRUD, 액티베이션 타겟 CRUD |
| DataAction | 🔲 예정 | 데이터 조건 충족 시 실시간 알림·워크플로우 트리거 | 액션 목록/조회/생성, 액션 타겟 CRUD |
| DataTransform | 🔲 예정 | 스케줄 또는 수동 실행되는 SQL 기반 데이터 변환 | CRUD, 실행, 검증, 스케줄 조회/설정 |
| StandardMappings | 🔲 예정 | 500개+ 사전정의 표준 매핑 템플릿으로 소스 오브젝트 전체를 일괄 매핑 | 매핑 미리보기, 일괄 생성 |
| Smart | 🔲 예정 | AI 보조로 필드 매핑·이벤트 날짜 필드 등을 자동 추천/생성 | 매핑 추천, 필드 매칭 미리보기, 스마트 데이터스트림 생성, 이벤트 날짜 추천 |
| Dataspace | 🔲 예정 | 데이터 격리용 워크스페이스(테넌트 파티션). 대부분 작업에 dataspace 이름 필요 | CRUD, 멤버(권한) 추가/삭제/조회 |
| DataKit | 🔲 예정 | 사전 패키징된 데이터 모델 템플릿을 한 번에 배포 | 목록/조회/매니페스트, 배포·배포취소, 배포상태·컴포넌트상태·의존성 조회 |
| Retriever | 🔲 예정 | RAG/AI 검색용 retriever 엔드포인트 생성·관리 (Data 360 데이터에 대한 시멘틱 검색) | retriever CRUD, retriever config CRUD |
| SearchIndex | 🔲 예정 | Data 360 엔티티에 대한 검색 인덱스 관리 (Retriever가 사용하는 인덱스) | CRUD, 설정, 처리 이력 조회 |

> 테스트 예시의 데이터/필드명은 모두 가상 데이터입니다. 실제 회사 데이터는 포함하지 않습니다.
