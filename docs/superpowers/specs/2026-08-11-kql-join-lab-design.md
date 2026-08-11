# KQL 멀티테이블 실습 랩 — 설계 문서

- 날짜: 2026-08-11
- 대상 파일: `azure-kql-join-lab.html` (신규 단일 HTML 학습 Artifact — `azure-migration-tracker.html`과 별개 파일)
- 선행 참고: `docs/superpowers/specs/2026-08-06-azure-monitor-design.md`(동일 프로젝트의 Kusto 실행기 1세대 설계 — 단일 테이블·join 미지원으로 스코프를 의도적으로 제한했던 문서), `docs/superpowers/specs/2026-08-07-workbook-lab-design.md`(문서 포맷·CSS 토큰 재사용 대상)

## 배경 및 목적

`azure-migration-tracker.html`의 Kusto 실행기(`runKustoQuery`, `AM_GATEWAY_LOGS`)는 2026-08-06 설계에서 "가상 테이블 1개만 노출, `join` 연산자는 실행 엔진에 넣지 않는다"고 명시적으로 결정했다. 이 결정은 당시 스코프를 지키기 위한 합리적 판단이었지만, 그 결과 이 프로젝트에는 "여러 Log Analytics 테이블을 조합해 Azure Workbook 대시보드용 차트를 만드는" 실습 경로가 없다. 이 문서는 그 갭을 메우는 **완전히 새로운 독립 HTML 파일**을 설계한다.

**핵심 목적은 "반드시 JOIN을 시키는 것"이 아니라 "샘플 데이터 테이블 3개로 할 수 있는 예제를 최대한 다양하고 많이 학습하는 것"이다.** 따라서 예제의 대다수는 테이블 1개만으로 완결되는 집계/차트 실습이며, join은 그 스펙트럼의 상위 난이도 구간을 채우는 예제 유형 중 하나로만 배치한다.

## 범위 및 비목표

**포함:**
- 실제 Azure Monitor Log Analytics 테이블 스키마 3종의 샘플 데이터(JSON)
- `join`(inner/leftouter/rightouter/fullouter) + `isnull()`/`isnotempty()`를 포함해 확장된 미니 Kusto 실행 엔진(다중 테이블 지원)
- `render timechart|barchart|columnchart|piechart` 4종 차트를 순수 SVG로 렌더링
- 예제 17개(단일 테이블 10 · 2테이블 join 5 · 3테이블 조합 2) — 문제 제시 → 사용자 작성 → 정답 비교의 능동회상 구조
- 튜토리얼 모드(난이도 순 6단계 워크스루)
- 빌드 도구 없는 단일 HTML 파일

**비목표(이번 스코프 제외):**
- `azure-migration-tracker.html` 수정 — 그 파일과 이 파일은 완전히 독립적이며 전역 상태·함수를 공유하지 않는다.
- `mv-expand`, `parse`, `extend`, 사용자 정의 함수(`let` 함수형) 등 join 이외의 추가 연산자 확장 — 화이트리스트는 아래 "엔진 설계"에 명시된 것으로 고정.
- 자동 채점(정답 문자열 완전일치 판정) — 기존 KQL 퀴즈·1세대 Kusto 실행기와 동일하게 "정답 쿼리 결과와 내 쿼리 결과를 나란히 비교해 스스로 판단"하는 방식을 유지한다.
- 실시간 Azure 리소스 연동, 실제 워크북 JSON export — 이번 파일은 개념·문법 학습용 시뮬레이터다.

## 데이터 모델 — 실제 스키마 3종

3개 테이블 모두 `OperationId`가 실제 공식 컬럼으로 존재함을 리서치로 확인했다(`learn.microsoft.com/azure/azure-monitor/reference/tables/{apimanagementgatewaylogs,apprequests,appdependencies}`, 2026-08-11 fetch). 이 컬럼을 join 키로 사용하면 "APIM 게이트웨이 → 백엔드 서비스(App Requests) → 데이터 계층 의존성 호출(App Dependencies, Cassandra/PostgreSQL)"이라는 이 프로젝트의 실제 마이그레이션 서사와 정확히 일치하는 3단 상관관계 체인을 만들 수 있다. 각 테이블은 실제 스키마의 부분집합만 사용한다(전체 컬럼을 다 쓰지 않는 것은 정상적인 실무 관행이며 허구가 아니다).

### 1. `ApiManagementGatewayLogs` (게이트웨이 계층 — Kong 대체)

실제 컬럼 사용: `TimeGenerated`(datetime), `OperationId`(string), `ApiId`(string), `Method`(string), `Url`(string), `ResponseCode`(int), `BackendResponseCode`(int), `IsRequestSuccess`(bool), `TotalTime`(long, ms), `CallerIpAddress`(string).

20행(합성 샘플) — 5건은 게이트웨이 단계에서 차단(401/403/429)되어 백엔드에 도달하지 않는다(`AppRequests`에 대응 행 없음 — 실제 APIM 동작과 동일: 인증/구독/속도제한 실패는 백엔드를 호출하지 않는다).

```json
[
  { "TimeGenerated": "2026-08-11T09:00:05Z", "OperationId": "OP-1001", "ApiId": "orders-api", "Method": "GET", "Url": "/orders", "ResponseCode": 200, "BackendResponseCode": 200, "IsRequestSuccess": true, "TotalTime": 82, "CallerIpAddress": "203.0.113.10" },
  { "TimeGenerated": "2026-08-11T09:00:41Z", "OperationId": "OP-1002", "ApiId": "orders-api", "Method": "POST", "Url": "/orders", "ResponseCode": 200, "BackendResponseCode": 200, "IsRequestSuccess": true, "TotalTime": 118, "CallerIpAddress": "203.0.113.11" },
  { "TimeGenerated": "2026-08-11T09:01:12Z", "OperationId": "OP-1003", "ApiId": "orders-api", "Method": "GET", "Url": "/orders", "ResponseCode": 401, "BackendResponseCode": 0, "IsRequestSuccess": false, "TotalTime": 9, "CallerIpAddress": "203.0.113.12" },
  { "TimeGenerated": "2026-08-11T09:01:47Z", "OperationId": "OP-1004", "ApiId": "orders-api", "Method": "GET", "Url": "/orders", "ResponseCode": 429, "BackendResponseCode": 0, "IsRequestSuccess": false, "TotalTime": 4, "CallerIpAddress": "203.0.113.10" },
  { "TimeGenerated": "2026-08-11T09:02:30Z", "OperationId": "OP-1005", "ApiId": "users-api", "Method": "GET", "Url": "/users/1", "ResponseCode": 200, "BackendResponseCode": 200, "IsRequestSuccess": true, "TotalTime": 640, "CallerIpAddress": "203.0.113.20" },
  { "TimeGenerated": "2026-08-11T09:03:01Z", "OperationId": "OP-1006", "ApiId": "users-api", "Method": "GET", "Url": "/users/2", "ResponseCode": 200, "BackendResponseCode": 200, "IsRequestSuccess": true, "TotalTime": 95, "CallerIpAddress": "203.0.113.21" },
  { "TimeGenerated": "2026-08-11T09:03:44Z", "OperationId": "OP-1007", "ApiId": "users-api", "Method": "GET", "Url": "/users/3", "ResponseCode": 403, "BackendResponseCode": 0, "IsRequestSuccess": false, "TotalTime": 6, "CallerIpAddress": "203.0.113.22" },
  { "TimeGenerated": "2026-08-11T09:04:12Z", "OperationId": "OP-1008", "ApiId": "users-api", "Method": "GET", "Url": "/users/4", "ResponseCode": 200, "BackendResponseCode": 200, "IsRequestSuccess": true, "TotalTime": 88, "CallerIpAddress": "203.0.113.20" },
  { "TimeGenerated": "2026-08-11T09:04:55Z", "OperationId": "OP-1009", "ApiId": "orders-api", "Method": "GET", "Url": "/orders", "ResponseCode": 200, "BackendResponseCode": 200, "IsRequestSuccess": true, "TotalTime": 730, "CallerIpAddress": "203.0.113.11" },
  { "TimeGenerated": "2026-08-11T09:05:20Z", "OperationId": "OP-1010", "ApiId": "orders-api", "Method": "POST", "Url": "/orders", "ResponseCode": 200, "BackendResponseCode": 200, "IsRequestSuccess": true, "TotalTime": 101, "CallerIpAddress": "203.0.113.12" },
  { "TimeGenerated": "2026-08-11T09:05:58Z", "OperationId": "OP-1011", "ApiId": "legacy-api", "Method": "GET", "Url": "/legacy/report", "ResponseCode": 403, "BackendResponseCode": 0, "IsRequestSuccess": false, "TotalTime": 3, "CallerIpAddress": "203.0.113.30" },
  { "TimeGenerated": "2026-08-11T09:06:33Z", "OperationId": "OP-1012", "ApiId": "legacy-api", "Method": "GET", "Url": "/legacy/report", "ResponseCode": 403, "BackendResponseCode": 0, "IsRequestSuccess": false, "TotalTime": 3, "CallerIpAddress": "203.0.113.30" },
  { "TimeGenerated": "2026-08-11T09:07:02Z", "OperationId": "OP-1013", "ApiId": "users-api", "Method": "GET", "Url": "/users/5", "ResponseCode": 200, "BackendResponseCode": 200, "IsRequestSuccess": true, "TotalTime": 92, "CallerIpAddress": "203.0.113.21" },
  { "TimeGenerated": "2026-08-11T09:07:40Z", "OperationId": "OP-1014", "ApiId": "orders-api", "Method": "GET", "Url": "/orders", "ResponseCode": 200, "BackendResponseCode": 200, "IsRequestSuccess": true, "TotalTime": 79, "CallerIpAddress": "203.0.113.10" },
  { "TimeGenerated": "2026-08-11T09:08:15Z", "OperationId": "OP-1015", "ApiId": "orders-api", "Method": "GET", "Url": "/orders", "ResponseCode": 500, "BackendResponseCode": 500, "IsRequestSuccess": false, "TotalTime": 850, "CallerIpAddress": "203.0.113.11" },
  { "TimeGenerated": "2026-08-11T09:08:50Z", "OperationId": "OP-1016", "ApiId": "users-api", "Method": "GET", "Url": "/users/6", "ResponseCode": 200, "BackendResponseCode": 200, "IsRequestSuccess": true, "TotalTime": 100, "CallerIpAddress": "203.0.113.22" },
  { "TimeGenerated": "2026-08-11T09:09:25Z", "OperationId": "OP-1017", "ApiId": "orders-api", "Method": "POST", "Url": "/orders", "ResponseCode": 200, "BackendResponseCode": 200, "IsRequestSuccess": true, "TotalTime": 113, "CallerIpAddress": "203.0.113.12" },
  { "TimeGenerated": "2026-08-11T09:10:02Z", "OperationId": "OP-1018", "ApiId": "legacy-api", "Method": "GET", "Url": "/legacy/report", "ResponseCode": 200, "BackendResponseCode": 200, "IsRequestSuccess": true, "TotalTime": 410, "CallerIpAddress": "203.0.113.30" },
  { "TimeGenerated": "2026-08-11T09:10:47Z", "OperationId": "OP-1019", "ApiId": "users-api", "Method": "GET", "Url": "/users/7", "ResponseCode": 200, "BackendResponseCode": 200, "IsRequestSuccess": true, "TotalTime": 91, "CallerIpAddress": "203.0.113.21" },
  { "TimeGenerated": "2026-08-11T09:11:30Z", "OperationId": "OP-1020", "ApiId": "orders-api", "Method": "GET", "Url": "/orders", "ResponseCode": 200, "BackendResponseCode": 200, "IsRequestSuccess": true, "TotalTime": 85, "CallerIpAddress": "203.0.113.10" }
]
```

### 2. `AppRequests` (백엔드 서비스 계층)

실제 컬럼 사용: `TimeGenerated`, `OperationId`, `Name`, `Url`, `Success`(bool), `ResultCode`(string — 게이트웨이의 `ResponseCode`(int)와 타입이 다름을 튜토리얼에서 명시적으로 짚는다), `DurationMs`(real), `AppRoleName`(string).

15행(합성 샘플) — 게이트웨이를 통과한 15개 `OperationId`와 1:1 대응. 그중 `OP-1006`, `OP-1019`는 캐시 히트로 뒤 단계(`AppDependencies`)에 대응 행이 없다.

```json
[
  { "TimeGenerated": "2026-08-11T09:00:05Z", "OperationId": "OP-1001", "Name": "GET Orders/List", "Url": "/orders", "Success": true, "ResultCode": "200", "DurationMs": 75, "AppRoleName": "orders-api-backend" },
  { "TimeGenerated": "2026-08-11T09:00:41Z", "OperationId": "OP-1002", "Name": "POST Orders/Create", "Url": "/orders", "Success": true, "ResultCode": "200", "DurationMs": 108, "AppRoleName": "orders-api-backend" },
  { "TimeGenerated": "2026-08-11T09:02:30Z", "OperationId": "OP-1005", "Name": "GET Users/GetById", "Url": "/users/1", "Success": true, "ResultCode": "200", "DurationMs": 600, "AppRoleName": "users-api-backend" },
  { "TimeGenerated": "2026-08-11T09:03:01Z", "OperationId": "OP-1006", "Name": "GET Users/GetById", "Url": "/users/2", "Success": true, "ResultCode": "200", "DurationMs": 40, "AppRoleName": "users-api-backend" },
  { "TimeGenerated": "2026-08-11T09:04:12Z", "OperationId": "OP-1008", "Name": "GET Users/GetById", "Url": "/users/4", "Success": true, "ResultCode": "200", "DurationMs": 78, "AppRoleName": "users-api-backend" },
  { "TimeGenerated": "2026-08-11T09:04:55Z", "OperationId": "OP-1009", "Name": "GET Orders/List", "Url": "/orders", "Success": true, "ResultCode": "200", "DurationMs": 700, "AppRoleName": "orders-api-backend" },
  { "TimeGenerated": "2026-08-11T09:05:20Z", "OperationId": "OP-1010", "Name": "POST Orders/Create", "Url": "/orders", "Success": true, "ResultCode": "200", "DurationMs": 90, "AppRoleName": "orders-api-backend" },
  { "TimeGenerated": "2026-08-11T09:07:02Z", "OperationId": "OP-1013", "Name": "GET Users/GetById", "Url": "/users/5", "Success": true, "ResultCode": "200", "DurationMs": 80, "AppRoleName": "users-api-backend" },
  { "TimeGenerated": "2026-08-11T09:07:40Z", "OperationId": "OP-1014", "Name": "GET Orders/List", "Url": "/orders", "Success": true, "ResultCode": "200", "DurationMs": 68, "AppRoleName": "orders-api-backend" },
  { "TimeGenerated": "2026-08-11T09:08:15Z", "OperationId": "OP-1015", "Name": "GET Orders/List", "Url": "/orders", "Success": false, "ResultCode": "500", "DurationMs": 830, "AppRoleName": "orders-api-backend" },
  { "TimeGenerated": "2026-08-11T09:08:50Z", "OperationId": "OP-1016", "Name": "GET Users/GetById", "Url": "/users/6", "Success": true, "ResultCode": "200", "DurationMs": 85, "AppRoleName": "users-api-backend" },
  { "TimeGenerated": "2026-08-11T09:09:25Z", "OperationId": "OP-1017", "Name": "POST Orders/Create", "Url": "/orders", "Success": true, "ResultCode": "200", "DurationMs": 100, "AppRoleName": "orders-api-backend" },
  { "TimeGenerated": "2026-08-11T09:10:02Z", "OperationId": "OP-1018", "Name": "GET Legacy/Report", "Url": "/legacy/report", "Success": true, "ResultCode": "200", "DurationMs": 380, "AppRoleName": "legacy-api-backend" },
  { "TimeGenerated": "2026-08-11T09:10:47Z", "OperationId": "OP-1019", "Name": "GET Users/GetById", "Url": "/users/7", "Success": true, "ResultCode": "200", "DurationMs": 35, "AppRoleName": "users-api-backend" },
  { "TimeGenerated": "2026-08-11T09:11:30Z", "OperationId": "OP-1020", "Name": "GET Orders/List", "Url": "/orders", "Success": true, "ResultCode": "200", "DurationMs": 72, "AppRoleName": "orders-api-backend" }
]
```

### 3. `AppDependencies` (데이터 계층 — Cassandra/PostgreSQL 호출)

실제 컬럼 사용: `TimeGenerated`, `OperationId`, `Target`(string), `DependencyType`(string — `Cassandra`/`SQL`은 SDK가 자동 계측하는 대표값이며 커스텀 계측 시 표기가 다를 수 있음을 튜토리얼에서 안내), `Name`(string), `Success`(bool), `ResultCode`(string), `DurationMs`(real).

15행(합성 샘플) — `AppRequests` 13건(캐시 히트 2건 제외) + APIM을 거치지 않고 직접 실행된 백그라운드 작업 2건(`OP-BG01`, `OP-BG02` — 어느 `OperationId`와도 매칭되지 않는 고아 행. `rightouter`/`fullouter` join과 "게이트웨이를 거치지 않은 직접 DB 접근 탐지" 예제에 사용).

```json
[
  { "TimeGenerated": "2026-08-11T09:00:06Z", "OperationId": "OP-1001", "Target": "cassandra-orders-cluster", "DependencyType": "Cassandra", "Name": "SELECT orders_by_customer", "Success": true, "ResultCode": "0", "DurationMs": 60 },
  { "TimeGenerated": "2026-08-11T09:00:42Z", "OperationId": "OP-1002", "Target": "cassandra-orders-cluster", "DependencyType": "Cassandra", "Name": "INSERT orders_by_customer", "Success": true, "ResultCode": "0", "DurationMs": 90 },
  { "TimeGenerated": "2026-08-11T09:00:30Z", "OperationId": "OP-BG01", "Target": "cassandra-orders-cluster", "DependencyType": "Cassandra", "Name": "compaction-health-check", "Success": true, "ResultCode": "0", "DurationMs": 1200 },
  { "TimeGenerated": "2026-08-11T09:02:31Z", "OperationId": "OP-1005", "Target": "postgres-users-db", "DependencyType": "SQL", "Name": "SELECT users_by_id", "Success": true, "ResultCode": "0", "DurationMs": 560 },
  { "TimeGenerated": "2026-08-11T09:04:13Z", "OperationId": "OP-1008", "Target": "postgres-users-db", "DependencyType": "SQL", "Name": "SELECT users_by_id", "Success": true, "ResultCode": "0", "DurationMs": 55 },
  { "TimeGenerated": "2026-08-11T09:04:56Z", "OperationId": "OP-1009", "Target": "cassandra-orders-cluster", "DependencyType": "Cassandra", "Name": "SELECT orders_by_customer", "Success": true, "ResultCode": "0", "DurationMs": 660 },
  { "TimeGenerated": "2026-08-11T09:05:21Z", "OperationId": "OP-1010", "Target": "cassandra-orders-cluster", "DependencyType": "Cassandra", "Name": "INSERT orders_by_customer", "Success": true, "ResultCode": "0", "DurationMs": 70 },
  { "TimeGenerated": "2026-08-11T09:07:03Z", "OperationId": "OP-1013", "Target": "postgres-users-db", "DependencyType": "SQL", "Name": "SELECT users_by_id", "Success": true, "ResultCode": "0", "DurationMs": 58 },
  { "TimeGenerated": "2026-08-11T09:07:41Z", "OperationId": "OP-1014", "Target": "cassandra-orders-cluster", "DependencyType": "Cassandra", "Name": "SELECT orders_by_customer", "Success": true, "ResultCode": "0", "DurationMs": 50 },
  { "TimeGenerated": "2026-08-11T09:08:16Z", "OperationId": "OP-1015", "Target": "cassandra-orders-cluster", "DependencyType": "Cassandra", "Name": "SELECT orders_by_customer", "Success": false, "ResultCode": "Timeout", "DurationMs": 800 },
  { "TimeGenerated": "2026-08-11T09:08:51Z", "OperationId": "OP-1016", "Target": "postgres-users-db", "DependencyType": "SQL", "Name": "SELECT users_by_id", "Success": true, "ResultCode": "0", "DurationMs": 62 },
  { "TimeGenerated": "2026-08-11T09:09:00Z", "OperationId": "OP-BG02", "Target": "postgres-users-db", "DependencyType": "SQL", "Name": "nightly-backup-verify", "Success": true, "ResultCode": "0", "DurationMs": 4200 },
  { "TimeGenerated": "2026-08-11T09:09:26Z", "OperationId": "OP-1017", "Target": "cassandra-orders-cluster", "DependencyType": "Cassandra", "Name": "INSERT orders_by_customer", "Success": true, "ResultCode": "0", "DurationMs": 75 },
  { "TimeGenerated": "2026-08-11T09:10:03Z", "OperationId": "OP-1018", "Target": "postgres-legacy-db", "DependencyType": "SQL", "Name": "SELECT legacy_reports", "Success": true, "ResultCode": "0", "DurationMs": 350 },
  { "TimeGenerated": "2026-08-11T09:11:31Z", "OperationId": "OP-1020", "Target": "cassandra-orders-cluster", "DependencyType": "Cassandra", "Name": "SELECT orders_by_customer", "Success": true, "ResultCode": "0", "DurationMs": 48 }
]
```

**행 수 요약(계층별 자연 감소 — 그 자체가 교육 포인트):** 게이트웨이 20건 → 백엔드 도달 15건(5건 게이트웨이 차단) → DB 호출 13건 상관 + 2건 고아(백그라운드) = 15건. "왜 세 테이블의 행 수가 다른가"를 튜토리얼 1단계에서 명시적으로 설명한다.

## 미니 Kusto 실행 엔진 설계 (다중 테이블 + join)

기존 `azure-migration-tracker.html`의 `runKustoQuery` 구현(단일 테이블, 정규식 기반 스테이지 파서)의 코드 스타일과 에러 메시지 관례(어느 스테이지에서 실패했는지 명시, 조용히 무시 금지)를 그대로 계승하되, 아래 3가지를 확장한다.

### 확장 1 — 테이블 이름 3종 지원

파이프라인 첫 토큰을 `KUSTO_TABLES = { ApiManagementGatewayLogs, AppRequests, AppDependencies }` 맵에서 조회. 없으면 기존과 동일하게 `알 수 없는 테이블: ...` 에러.

### 확장 2 — `join` 스테이지

문법: `join kind=(inner|innerunique|leftouter|rightouter|fullouter) (<RightTable>) on <OnClause>` — 실제 Kusto 문법(`docs.microsoft.com/kusto/query/join-operator` 기준)을 따른다. `<OnClause>`는 두 형태를 지원한다: 컬럼명이 동일할 때의 `on OperationId`, 컬럼명이 다를 때의 `on $left.ColA == $right.ColB`.

**결과 컬럼 병합 규칙(실제 Kusto 동작 재현):** 두 테이블의 컬럼을 모두 유지하되, join 키를 제외한 컬럼명이 겹치면 오른쪽 테이블의 컬럼에 `1` 접미사를 붙인다(예: 양쪽에 `TimeGenerated`가 있으면 오른쪽은 `TimeGenerated1`). `leftouter`/`rightouter`/`fullouter`에서 매칭되지 않은 쪽의 컬럼은 `null`로 채운다.

```js
function kustoApplyJoin(leftRows, clause) {
  var m = clause.match(/^kind\s*=\s*(inner|innerunique|leftouter|rightouter|fullouter)\s*\(\s*(\w+)\s*\)\s+on\s+(.+)$/);
  if (!m) throw new Error("join 문법을 해석할 수 없습니다: " + clause);
  var kind = m[1], rightTableName = m[2], onClause = m[3].trim();
  var rightTable = KUSTO_TABLES[rightTableName];
  if (!rightTable) throw new Error("알 수 없는 테이블: " + rightTableName);
  var rightRows = rightTable.slice();

  var onMatch = onClause.match(/^\$left\.(\w+)\s*==\s*\$right\.(\w+)$/);
  var leftKey = onMatch ? onMatch[1] : onClause;
  var rightKey = onMatch ? onMatch[2] : onClause;

  var leftCols = leftRows.length ? Object.keys(leftRows[0]) : [];
  var rightCols = rightRows.length ? Object.keys(rightRows[0]) : [];
  function mergeRow(l, r) {
    var out = {};
    leftCols.forEach(function (c) { out[c] = l ? l[c] : null; });
    rightCols.forEach(function (c) {
      if (c === rightKey && leftCols.indexOf(leftKey) !== -1) return; // 조인 키 중복 생략
      var outKey = leftCols.indexOf(c) !== -1 ? c + "1" : c;
      out[outKey] = r ? r[c] : null;
    });
    return out;
  }

  var rightByKey = {};
  rightRows.forEach(function (r) {
    var k = r[rightKey];
    if (!rightByKey[k]) rightByKey[k] = [];
    rightByKey[k].push(r);
  });

  var matchedRightKeys = {};
  var result = [];
  leftRows.forEach(function (l) {
    var matches = rightByKey[l[leftKey]] || [];
    if (matches.length) {
      matches.forEach(function (r) { matchedRightKeys[r[rightKey]] = true; result.push(mergeRow(l, r)); });
    } else if (kind === "leftouter" || kind === "fullouter") {
      result.push(mergeRow(l, null));
    }
  });
  if (kind === "rightouter" || kind === "fullouter") {
    rightRows.forEach(function (r) {
      if (!matchedRightKeys[r[rightKey]]) result.push(mergeRow(null, r));
    });
  }
  return result;
}
```

`KUSTO_STAGE_HANDLERS`에 `join: function (rows, clause) { return kustoApplyJoin(rows, clause); }` 등록. `kustoExtractColRefs`의 컬럼 존재 검증은 join 스테이지 직후에는 건너뛴다(병합 후 컬럼 목록이 동적으로 바뀌므로 `currentColumns = null` 처리 — 기존 `summarize` 이후 처리와 동일 패턴).

### 확장 3 — `where` 절의 `isnull()` / `isnotempty()` 단항 조건

leftouter/rightouter join 이후 "매칭되지 않은 행 찾기" 예제(B4·B5)에 필요. `kustoApplyWhere`의 조건 분해 정규식에 단항 함수 형태를 추가 지원:

```js
function kustoApplyWhere(rows, clause) {
  var conditions = clause.split(/\s+and\s+/i).map(function (c) {
    var unary = c.match(/^(isnull|isnotempty)\((\w+)\)$/);
    if (unary) return { fn: unary[1], col: unary[2] };
    var m = c.match(/^(\w+)\s*(==|!=|>=|<=|>|<|!contains|contains)\s*(.+)$/);
    if (!m) throw new Error("조건을 해석할 수 없습니다: " + c);
    return { col: m[1], op: m[2], val: kustoParseLiteral(m[3]) };
  });
  return rows.filter(function (row) {
    return conditions.every(function (c) {
      if (c.fn === "isnull") return row[c.col] === null || row[c.col] === undefined;
      if (c.fn === "isnotempty") return row[c.col] !== null && row[c.col] !== undefined && row[c.col] !== "";
      return kustoCompare(row[c.col], c.op, c.val);
    });
  });
}
```

### 확장 4 — `render` 4종

기존은 `chartHint: true`(불리언, timechart 전용)만 부여했다. 이를 문자열 `chartType: "timechart"|"barchart"|"columnchart"|"piechart"|null`로 확장하고, 렌더러가 이 값으로 분기한다. `render` 뒤에 `with (title="...")` 같은 실제 Kusto 부가 옵션은 이번 화이트리스트에서 무시(파싱만 하고 반영은 안 함)하되, 무시했다는 사실을 결과 화면에 안내 문구로 표시한다(정확성을 가장하지 않는다는 기존 원칙 유지).

## 차트 렌더러 설계

외부 라이브러리 없이 순수 SVG. 입력은 `runKustoQuery` 결과(`{ columns, rows, chartType }`)이며, 렌더러는 **첫 번째 컬럼을 카테고리/시간 축, 나머지 숫자형 컬럼을 계열(series)**로 취급한다(Kusto `render`의 기본 컬럼 추론 규칙과 동일한 단순화).

- `timechart` → x축 시간(`Date` 파싱), y축 값, `<polyline>` 라인 차트. 계열이 여러 개(3테이블 조합 예제 C1처럼 게이트웨이/백엔드/DB 소요시간 3계열)면 색상을 구분해 다중 라인.
- `barchart` → 가로 막대(카테고리가 y축), `columnchart` → 세로 막대(카테고리가 x축). 둘 다 `<rect>`.
- `piechart` → 값 비율에 따른 `<path>` 부채꼴(`strokeDasharray` 트릭이 아니라 실제 각도 계산으로 정확히 그린다).
- 데이터 0건이면 차트 대신 "결과 0건" 안내, 숫자형 컬럼이 없으면 "차트로 표현할 숫자 컬럼이 없습니다" 안내(조용히 빈 차트를 그리지 않는다).

## 예제 17개 — 그룹별 분포

**단일 테이블(10개, 전체의 59%)**

| ID | 테이블 | 문제 | 정답 쿼리 요지 | 차트 |
|---|---|---|---|---|
| A1 | Gateway | 5분 단위 요청 수 추이 | `summarize count() by bin(TimeGenerated,5m)` | timechart |
| A2 | Gateway | API별 요청 수 | `summarize count() by ApiId` | columnchart |
| A3 | Gateway | 응답코드 분포 | `summarize count() by ResponseCode` | piechart |
| A4 | Gateway | 처리시간 상위 5건 | `top 5 by TotalTime desc` | barchart |
| A5 | Gateway | API별 평균 처리시간 | `summarize avg(TotalTime) by ApiId` | columnchart |
| A6 | AppRequests | 성공/실패 비율 | `summarize count() by Success` | piechart |
| A7 | AppRequests | 백엔드 서비스별 평균 응답시간 | `summarize avg(DurationMs) by AppRoleName` | columnchart |
| A8 | AppDependencies | 의존성 타입별 호출 건수 | `summarize count() by DependencyType` | piechart |
| A9 | AppDependencies | Target별 평균 소요시간(느린 순) | `summarize avg(DurationMs) by Target \| sort by avg_DurationMs desc` | barchart |
| A10 | Gateway | 5분 단위 실패 건수 추이 | `where IsRequestSuccess == false \| summarize count() by bin(TimeGenerated,5m)` | timechart |

**2테이블 join(5개)**

| ID | 조인 | 문제 | 학습 포인트 |
|---|---|---|---|
| B1 | Gateway ⨝ AppRequests | 게이트웨이는 200인데 백엔드는 실패로 기록한 요청 찾기 | `inner` join + 상태 불일치 탐지 |
| B2 | Gateway ⨝ AppRequests | API별 "게이트웨이 오버헤드"(TotalTime − DurationMs) 평균 | join 후 계산 컬럼 |
| B3 | AppRequests ⨝ AppDependencies | 백엔드 응답시간 중 DB 호출이 차지하는 비율 | `leftouter`(캐시 히트로 DB 미호출 행 포함) |
| B4 | Gateway ⨝ AppRequests | 게이트웨이 차단율(백엔드 미도달 비율) | `leftouter` + `isnull()` |
| B5 | AppDependencies ⨝ AppRequests | APIM을 거치지 않은 백그라운드 DB 호출 찾기 | `rightouter`/`fullouter` + 고아 행 |

**3테이블 조합(2개)**

| ID | 조인 | 문제 | 차트 |
|---|---|---|---|
| C1 | Gateway ⨝ AppRequests ⨝ AppDependencies | API별 "게이트웨이/백엔드/DB" 3단 소요시간 분해 | 3계열 columnchart(스택형 대체 — 계열별 막대 나열) |
| C2 | Gateway ⨝ AppRequests ⨝ AppDependencies | 3계층 모두 성공한 완전 성공 요청 비율(API별) | piechart |

## 예제-정답 UI 흐름

1. 예제 카드: 제목 + 자연어 문제 + (2·3테이블 예제는) 참여 테이블 배지.
2. 사용자가 `<textarea>`에 KQL 작성 → "내 쿼리 실행" → 결과 테이블/차트 렌더링(에러 시 몇 번째 스테이지인지 표시).
3. "정답 보기" 클릭 시에만 정답 쿼리와 그 실행 결과(표+차트)를 나란히 펼침(자동 채점 없음 — CLAUDE.md·1세대 랩과 동일 철학, 사용자가 스스로 비교).
4. 정답을 펼치기 전 내 쿼리를 최소 1회 실행하지 않으면 "정답 보기" 비활성화(추측만으로 정답을 먼저 보는 것을 방지 — 능동회상 원칙 강제).

## 튜토리얼 모드 — 난이도 순 6단계 워크스루

각 단계는 설명 텍스트 + 미리 채워진 예시 쿼리(수정 가능) + "실행해서 확인" 버튼 + "다음" 이동으로 구성(정답을 숨기지 않는 안내형 — 예제 뱅크의 능동회상형과 구분).

1. 왜 세 테이블인가 — 게이트웨이→백엔드→DB 3단 구조와 행 수가 자연 감소하는 이유(20→15→15) 설명.
2. 단일 테이블 기초: `where` / `project`
3. 집계와 시간 버킷: `summarize ... by bin()` → `render timechart`
4. 순위와 정렬: `top` / `sort by` → `render barchart`
5. join 문법 소개: `on Col` vs `on $left.X == $right.Y`, join 종류 4가지 차이(벤 다이어그램 텍스트 설명)
6. 실전 조합: 2테이블 → 3테이블로 확장하며 워크북 대시보드 패턴(여러 차트를 한 화면에 배치) 소개

## 에러 처리 / 엣지 케이스

- 화이트리스트 밖 연산자(`mv-expand`, `parse` 등): 1세대와 동일한 차단 메시지 패턴("이 시뮬레이터는 지원하지 않지만 실제 Kusto에서는 지원한다").
- `join`의 오른쪽 테이블명이 3종에 없으면 즉시 에러.
- 존재하지 않는 컬럼 참조는 join 이전 단계와 동일하게 차단(단, join 직후 스테이지는 병합된 새 컬럼 목록 기준으로 재검증).
- 좌/우 테이블 모두 0건이거나 매칭 0건인 `inner`/`innerunique` join → "결과 0건"으로 정상 처리(에러 아님).
- 차트 렌더링 시 숫자형 컬럼이 여러 개면 그중 컬럼 순서상 첫 번째만 사용하고, 나머지는 무시했다는 안내 문구 표시(단, C1처럼 3계열이 의도된 예제는 렌더러가 예제별 `seriesCols` 힌트를 받아 다계열 렌더링).
- `TimeGenerated`가 join 이후 `TimeGenerated1`로 접미사가 붙는 경우, timechart 렌더러가 첫 번째 `TimeGenerated*` 컬럼을 자동 선택.

## 테스트 방법

이 프로젝트는 빌드/테스트 러너가 없다. 모든 검증은 `azure-kql-join-lab.html`을 브라우저로 직접 열어 수동 확인한다: (1) 튜토리얼 6단계가 순서대로 실행되는지, (2) 예제 17개를 정답 쿼리 그대로 입력했을 때 모두 에러 없이 기대한 차트 타입으로 렌더링되는지, (3) 화이트리스트 밖 연산자·잘못된 컬럼명·빈 쿼리 등 엣지 케이스가 명확한 에러 메시지를 내는지.
