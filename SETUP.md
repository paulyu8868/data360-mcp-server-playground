# Data 360 MCP Server — 설치 & 트러블슈팅 가이드

> 이 문서는 Claude Code에 그대로 붙여넣고 "이 가이드대로 셋업해줘"라고 시키면 됩니다.
> `install.py`는 대화형(interactive) 스크립트라 Claude Code가 직접 실행할 수 없습니다 — 아래는 그걸 단계별 비대화형 작업으로 풀어낸 버전입니다.

## 0. 무엇을 만드는가

Salesforce Data 360 REST API ~190개를 3개 파사드 툴(`search`, `payload_examples`, `execute`)로 감싸는 로컬 Java MCP 서버. stdio 방식으로 동작하며 Claude Code의 `~/.claude.json`에 `data360`이라는 이름으로 등록한다.

## 1. 사전 요구사항

- Java 17+
- Maven 3.9+
- Git
- Salesforce Data 360 활성화된 org + Connected App (Client ID/Secret)

Windows에서 Java/Maven이 없으면:

```powershell
# Java 17 (winget)
winget install --id EclipseAdoptium.Temurin.17.JDK -e --silent --accept-package-agreements --accept-source-agreements

# Maven은 winget에 없으므로 직접 다운로드
$mavenUrl = "https://archive.apache.org/dist/maven/maven-3/3.9.6/binaries/apache-maven-3.9.6-bin.zip"
Invoke-WebRequest -Uri $mavenUrl -OutFile "$env:TEMP\maven.zip"
Expand-Archive -Path "$env:TEMP\maven.zip" -DestinationPath "C:\tools" -Force
# 이후 PATH에 C:\tools\apache-maven-3.9.6\bin 추가 필요 (세션마다 또는 시스템 PATH에 영구 등록)
```

설치 후 새 PowerShell 세션에서 `java -version`, `mvn -version`으로 확인.

## 2. JAR 빌드

```powershell
git clone --depth 1 --branch main https://github.com/forcedotcom/d360-mcp-server "$env:TEMP\d360-mcp-build"
mvn clean package -DskipTests -f "$env:TEMP\d360-mcp-build\pom.xml"

$installDir = "$env:USERPROFILE\.data360-mcp-server"
New-Item -ItemType Directory -Force $installDir
Copy-Item "$env:TEMP\d360-mcp-build\target\data360-mcp-server-*.jar" $installDir
# .original 파일은 제외하고 복사할 것
```

## 3. Salesforce Connected App 설정 (★ 가장 중요, 가장 많이 막히는 구간)

기존 Connected App을 재사용해도 되고, 새로 만들어도 된다. **Client Credentials Flow로 자동 토큰 갱신**을 쓰려면 체크박스 하나로 끝나지 않고 아래 4가지가 전부 필요하다. 하나라도 빠지면 각기 다른 에러가 난다.

| 단계 | 위치 | 빠졌을 때 에러 |
|---|---|---|
| 1. Client Credentials Flow 활성화 | App Manager → Edit → OAuth 설정 → Enable Client Credentials Flow 체크 | (체크 안 하면 애초에 grant_type=client_credentials 요청 자체가 거부) |
| 2. Run As 사용자 지정 | App Manager → **Manage** → Client Credentials Flow → Edit → Run As 사용자 선택 | `"no client credentials user enabled"` |
| 3. Permitted Users + Profile/Permission Set 할당 | Edit → Permitted Users를 **"Admin approved users are pre-authorized"**로 변경 → Manage 화면에서 Run As 사용자의 Profile 또는 Permission Set을 **Manage Profiles / Manage Permission Sets**로 명시적 추가 | `"user is not admin approved to access this app"` |
| 4. OAuth 스코프 | Edit → Selected OAuth Scopes에 추가: `api`, `cdp_api`, `cdp_query_api`. **"Full access (full)" 스코프는 다른 세부 스코프와 같이 있으면 충돌나므로 제거** | 스코프 부족 시 실제 API 호출에서 `401 INVALID_SCOPES` / 충돌 시 토큰 발급 자체가 `"too many scopes requested"` |

설정 반영까지 캐시 때문에 즉시 안 될 수 있다 — 안 되면 5~10분 후 재시도.

**검증 방법** (PowerShell):

```powershell
$loginUrl = "https://YOUR_ORG.my.salesforce.com"
$clientId = "YOUR_CLIENT_ID"
$clientSecret = "YOUR_CLIENT_SECRET"

$body = "grant_type=client_credentials&client_id=$([Uri]::EscapeDataString($clientId))&client_secret=$([Uri]::EscapeDataString($clientSecret))"
Invoke-RestMethod -Uri "$loginUrl/services/oauth2/token" -Method Post -Body $body -ContentType "application/x-www-form-urlencoded"
```

응답의 `"scope"` 값에 `cdp_api`, `cdp_query_api`, `api`가 다 들어있어야 한다.

## 4. Claude Code에 MCP 서버 등록

`claude` CLI가 PATH에 없는 환경이 많으므로, `~/.claude.json`을 직접 수정하는 게 더 안정적이다 (Python으로 처리 권장 — PowerShell의 `ConvertFrom-Json`은 이 파일의 일부 비-ASCII 콘텐츠에서 깨질 수 있음):

```python
import json, os

claude_json = os.path.expanduser("~/.claude.json")
with open(claude_json, "r", encoding="utf-8") as f:
    cfg = json.load(f)

jar_path = "C:/Users/YOUR_USER/.data360-mcp-server/data360-mcp-server-1.0.0.jar"

cfg.setdefault("mcpServers", {})["data360"] = {
    "command": "java",
    "args": ["-jar", jar_path],
    "env": {
        "DATA360_AUTH_FLOW": "client_credentials",
        "DATA360_CLIENT_ID": "YOUR_CLIENT_ID",
        "DATA360_CLIENT_SECRET": "YOUR_CLIENT_SECRET",
        "DATA360_LOGIN_URL": "https://YOUR_ORG.my.salesforce.com",
        "DATA360_API_VERSION": "66.0"
    }
}

tmp = claude_json + ".tmp"
with open(tmp, "w", encoding="utf-8") as f:
    json.dump(cfg, f, indent=2, ensure_ascii=False)
    f.write("\n")
os.replace(tmp, claude_json)
```

등록 후 **Claude Code 재시작 필수** (실행 중인 MCP 프로세스는 세션 시작 시점 환경변수를 메모리에 들고 있어서, 설정만 바꾸고 재시작 안 하면 옛 값으로 계속 동작함).

### access_token 방식 (대안, Client Credentials Flow를 못 쓰는 경우)

Connected App 설정을 못 건드리는 상황이면 refresh_token으로 access_token을 매번 발급받아 쓰는 방식도 가능하지만, **토큰이 ~2시간마다 만료**되어 수동 갱신이 필요하다. 운영에는 Client Credentials Flow를 강력 추천.

```powershell
$body = "grant_type=refresh_token&client_id=$clientId&client_secret=$clientSecret&refresh_token=$refreshToken"
$resp = Invoke-RestMethod -Uri "$loginUrl/services/oauth2/token" -Method Post -Body $body -ContentType "application/x-www-form-urlencoded"
# $resp.access_token 을 DATA360_ACCESS_TOKEN으로 ~/.claude.json에 반영
```

## 5. 동작 검증

Claude Code 재시작 후:

```
search("semantic data model dimensions")  → 패밀리/툴 목록 반환되는지 확인
execute("d360_dmo_list", {})              → 실제 401/200 여부로 인증 통과 확인
```

`401 INVALID_SCOPES`가 나오면 3단계(OAuth 스코프)를 다시 확인.

## 6. 사용법 — 3개 파사드 툴

1. `search(query)` — 의도/키워드로 21개 툴 패밀리 중 관련 패밀리 탐색 (DMO, DLO, DataStreams, Mappings, IdentityResolution, CalculatedInsights, SDM, Query, Segment, Activation 등)
2. `payload_examples(toolName)` — 특정 툴의 정확한 파라미터 스키마 + 예시 확인 (실행 전 항상 먼저 볼 것)
3. `execute(toolName, paramsJson)` — 실제 실행. `paramsJson`은 JSON **문자열**로 전달

## 7. 알려진 이슈 / 주의사항

- **개발자 프리뷰 단계** — 프로덕션 사용 비권장, 단일 org/단일 사용자 구조
- `d360_dmo_create`의 `fields[]`에 `creationType`을 넣으면 `400 JSON_PARSER_ERROR` 발생 — 빼고 보낼 것
- DMO 생성 시 `KQ_*`, `DataSource__c`, `DataSourceObject__c`, `InternalOrganization__c`는 Data Cloud가 시스템 필드로 자동 추가함 — 수동으로 만들지 말 것. 매핑도 자동으로 연결됨
- **`d360_dmo_update`로 이미 존재하는 필드의 `description`을 수정해도 반영되지 않는 것으로 확인됨** (API는 200 OK를 반환하지만 Setup → Data Model UI에서 실제로는 비어있음). 현재로선 Setup UI에서 수동 입력이 유일하게 확인된 방법. SDM(시멘틱 모델)의 일반 차원/측정값(계산 필드 아님)에 description을 설정하는 전용 API 툴도 현재 노출되어 있지 않음
- `d360_sdm_query`의 `structuredSemanticQuery`는 객체가 아니라 **JSON 문자열로 직렬화**해서 전달해야 함 (payload_examples의 예시는 객체 형태로 보이지만 실제 스키마 타입은 string)

## 8. 트러블슈팅 표

| 에러 메시지 | 원인 | 해결 |
|---|---|---|
| `no client credentials user enabled` | Run As 사용자 미지정 | App Manager → Manage → Client Credentials Flow에 Run As 지정 |
| `user is not admin approved to access this app` | Permitted Users 설정 또는 Profile/Permission Set 할당 누락 | Permitted Users를 Admin approved로, Manage Profiles/Permission Sets에서 사용자 권한 추가 |
| `too many scopes requested` | "Full access(full)" 스코프와 세부 스코프 동시 선택 | full 스코프 제거, 필요한 세부 스코프만 유지 |
| `401 INVALID_SCOPES` / `INVALID_AUTH_HEADER` (실제 API 호출 시) | OAuth 스코프에 `cdp_api`/`cdp_query_api` 없음 | Connected App에 해당 스코프 추가 |
| `Unrecognized field "creationType"` | DMO 생성 시 입력 불가 필드 포함 | 필드 배열에서 `creationType` 제거 |
| MCP 도구 호출이 계속 옛 인증으로 401 | Claude Code 재시작 안 함 | 설정 변경 후 반드시 완전 재시작 |
