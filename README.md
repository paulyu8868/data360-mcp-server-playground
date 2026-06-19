# Data 360 MCP Server Playground

[forcedotcom/d360-mcp-server](https://github.com/forcedotcom/d360-mcp-server) — Salesforce Data 360 REST API를 3개 파사드 툴(`search`, `payload_examples`, `execute`)로 감싸는 로컬 MCP 서버를 기능별로 하나씩 테스트하고 기록하는 저장소.

- 설치/연동 가이드: [SETUP.md](./SETUP.md)
- 기능별 테스트 기록: [`/tests`](./tests) — 패밀리 하나씩 PR로 추가됨

## 진행 상태

| 패밀리 | 상태 |
|---|---|
| DLO | [✅ 테스트 완료](tests/01-dlo-to-sdm-pipeline/README.md) |
| DMO | [✅ 테스트 완료](tests/01-dlo-to-sdm-pipeline/README.md) |
| Mappings | [✅ 테스트 완료](tests/01-dlo-to-sdm-pipeline/README.md) |
| SDM | [✅ 테스트 완료](tests/01-dlo-to-sdm-pipeline/README.md) |
| Query | [✅ 테스트 완료](tests/01-dlo-to-sdm-pipeline/README.md) |
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
