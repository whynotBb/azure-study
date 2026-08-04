# 학습 콘텐츠 팩트체크 — 2026-08-04

`azure-migration-tracker.html`의 기존 8개 패널(구조 비교, APIM 정책 레퍼런스/퀴즈, KQL 레퍼런스/퀴즈, 데이터·인프라 레퍼런스/퀴즈, AIDD·Jira)을 공식 문서(Microsoft Learn, Kong Docs, Elastic Docs)와 커뮤니티 자료(Microsoft Q&A, Stack Overflow)로 팩트체크한 결과다. "로컬 테스트 랩"(9번째 그룹)은 구현 직후 별도의 전체 리뷰·수정 사이클을 이미 거쳤으므로 이번 검토 대상에서 제외했다.

검토 방법: 항목을 5개 영역으로 나눠 병렬 조사 에이전트가 각 항목을 공식 문서 URL로 직접 확인(WebFetch)했다. "확인됨"이라고 보고된 항목은 이 문서에 싣지 않았다 — 아래는 문제가 발견된 항목만이다.

## 요약

| 심각도 | 건수 | 의미 |
|---|---|---|
| Critical | 9 | 그대로 따라 하면 Azure/CLI에서 실제로 실패하는 문법 오류 |
| Important | 7 | 문법은 맞지만 오해를 유발하는 누락·낡은 서술 |
| Minor | 1 | 참고용, 수정 선택사항 |

---

## Critical — 그대로 사용하면 실패하는 문법

### C1. `validate-graphql-request` 정책의 `<authorize><rule>` 속성이 존재하지 않음
- **위치**: `APIM_POLICIES` → "내용 유효성 검사" → `validate-graphql-request`
- **현재**: `<rule field="*" roles="*" />`, 필수 속성 `max-size` 없음
- **근거**: 공식 문법은 `<rule path="..." action="allow|remove|reject|ignore" />`이며, `field`/`roles`는 스키마에 없는 속성이다. `max-size`는 루트 요소의 필수 속성이라 빠지면 저장 자체가 안 된다. — https://learn.microsoft.com/en-us/azure/api-management/validate-graphql-request-policy
- **수정안**: `<validate-graphql-request error-variable-name="graphQLValidationError" max-size="102400"><authorize><rule path="/__*" action="reject" /></authorize></validate-graphql-request>`

### C2. `xml-to-json` 정책에 필수 속성 `kind` 누락
- **위치**: `APIM_POLICIES` → "변환" → `xml-to-json`
- **현재**: `<xml-to-json apply="always" consider-accept-header="false" />`
- **근거**: `kind`(`javascript-friendly | direct`)는 필수이며 기본값이 없다. `json-to-xml`(대칭 정책)에는 `kind`가 없어서 그대로 복사하다 빠뜨린 것으로 보인다. — https://learn.microsoft.com/en-us/azure/api-management/xml-to-json-policy
- **수정안**: `<xml-to-json kind="direct" apply="always" consider-accept-header="false" />`

### C3. `sql-data-source` 예시가 스키마상 유효하지 않음
- **위치**: `APIM_POLICIES` → "GraphQL 해결 프로그램" → `sql-data-source`
- **현재**: `<sql-data-source><sql-statement>SELECT...</sql-statement></sql-data-source>`
- **근거**: 필수 요소 `<connection-info>`가 없고, `<sql-statement>`는 `<request>` 안에 있어야 한다. — https://learn.microsoft.com/en-us/azure/api-management/sql-data-source-policy
- **수정안**:
```xml
<sql-data-source>
  <connection-info>
    <connection-string>{{my-connection-string}}</connection-string>
  </connection-info>
  <request single-result="true">
    <sql-statement>SELECT * FROM Users WHERE Id = @id</sql-statement>
    <parameters><parameter name="@id">@(context.GraphQL.Arguments["id"])</parameter></parameters>
  </request>
</sql-data-source>
```

### C4. `cosmosdb-data-source`의 `<cosmosdb-parameters>`는 존재하지 않는 요소
- **위치**: `APIM_POLICIES` → "GraphQL 해결 프로그램" → `cosmosdb-data-source`
- **현재**: `<cosmosdb-parameters><parameter name="id">...</parameter></cosmosdb-parameters>`
- **근거**: 실제로는 `<connection-info>`(connection-string/database-name/container-name) + `<query-request>`/`<read-request>`/`<delete-request>`/`<write-request>` 중 하나가 필요하다. `parameters`는 `<query-request>` 안에서만 쓰인다. — https://learn.microsoft.com/en-us/azure/api-management/cosmosdb-data-source-policy
- **수정안**:
```xml
<cosmosdb-data-source>
  <connection-info>
    <connection-string use-managed-identity="true">AccountEndpoint=https://contoso-cosmosdb.documents.azure.com:443/;</connection-string>
    <database-name>myDatabase</database-name>
    <container-name>myContainer</container-name>
  </connection-info>
  <read-request><id>@(context.GraphQL.Arguments["id"].ToString())</id></read-request>
</cosmosdb-data-source>
```

### C5. `publish-event`의 `<target>`은 잘못된 자식 요소
- **위치**: `APIM_POLICIES` → "GraphQL 해결 프로그램" → `publish-event`
- **현재**: `<targets><target>orderCreated</target></targets>`
- **근거**: 실제 요소는 `<graphql-subscription id="..." />`다. `<target>`이라는 텍스트 콘텐츠 요소는 스키마에 없다. — https://learn.microsoft.com/en-us/azure/api-management/publish-event-policy
- **수정안**: `<targets><graphql-subscription id="orderCreated" /></targets>`

### C6. PostgreSQL NSG 방화벽 규칙 CLI 명령의 플래그가 틀림
- **위치**: `DATA_INFRA` → "PostgreSQL" → NSG 항목
- **현재**: `az postgres flexible-server firewall-rule create --resource-group rg-migration --name myserver --rule-name allow-office --start-ip-address ... --end-ip-address ...`
- **근거**: 서버 이름 플래그는 `--server-name`이고(`--name`이 아님), `--rule-name`이라는 플래그 자체가 없다(규칙 이름은 `--name`). — https://learn.microsoft.com/en-us/cli/azure/postgres/flexible-server/firewall-rule
- **수정안**: `az postgres flexible-server firewall-rule create --resource-group rg-migration --server-name myserver --name allow-office --start-ip-address 1.2.3.4 --end-ip-address 1.2.3.4`

### C7. PostgreSQL 고가용성 CLI의 `--high-availability` 플래그가 제거됨
- **위치**: `DATA_INFRA` → "PostgreSQL" → 고가용성(HA) 항목
- **현재**: `az postgres flexible-server create ... --high-availability ZoneRedundant`
- **근거**: 현재 CLI 레퍼런스에서 이 플래그는 사라졌고 `--zonal-resiliency`(Enabled/Disabled) + `--zone`/`--standby-zone`으로 대체됐다("Same-zone"/"Zone-redundant"라는 개념 용어 자체는 여전히 유효). — https://learn.microsoft.com/en-us/cli/azure/postgres/flexible-server
- **수정안**: `az postgres flexible-server create --resource-group rg-migration --name myserver --zonal-resiliency Enabled --zone 1 --standby-zone 2`

### C8. RBAC 역할 할당 CLI에 존재하지 않는 플래그 사용
- **위치**: `DATA_INFRA` → "Azure 기초 개념" → RBAC 항목
- **현재**: `az role assignment create --assignee user@contoso.com --role Contributor --resource-group rg-migration`
- **근거**: `az role assignment create`에는 `--resource-group` 플래그가 없다(list/delete 명령 전용). `--scope`에 전체 리소스 ID를 넘겨야 한다. — https://learn.microsoft.com/en-us/cli/azure/role/assignment#az-role-assignment-create
- **수정안**: `az role assignment create --assignee user@contoso.com --role Contributor --scope /subscriptions/<sub-id>/resourceGroups/rg-migration`

### C9. `az apim identity assign`은 존재하지 않는 명령
- **위치**: `DATA_INFRA` → "Azure 기초 개념" → 관리 ID 항목
- **현재**: `az apim identity assign --resource-group rg-migration --name my-apim`
- **근거**: `az apim` 명령 그룹에 `identity` 서브그룹 자체가 없다(공식 문서 페이지가 404). 관리 ID는 `az apim create`/`az apim update`의 `--enable-managed-identity` 플래그로 켠다. — https://learn.microsoft.com/en-us/cli/azure/apim
- **수정안**: `az apim update --resource-group rg-migration --name my-apim --enable-managed-identity true`

---

## Important — 낡았거나 오해를 유발하는 서술

### I1. `quota-by-key`/`rate-limit-by-key`에 티어 제약 안내 없음
- **위치**: `APIM_POLICIES` → "속도 제한 및 할당량"
- **근거**: `quota-by-key`는 Developer/Basic/Standard/Premium(클래식)에서만 지원되고 v2·Consumption에서는 지원되지 않는다. `rate-limit-by-key`도 Consumption 미지원. — https://learn.microsoft.com/en-us/azure/api-management/quota-by-key-policy , .../rate-limit-by-key-policy
- **수정안**: 두 항목의 `when`에 "단, ~ 티어에서는 사용할 수 없습니다" 한 줄 추가

### I2. "AI 게이트웨이" 태그 정책군에 Consumption 미지원 안내 없음
- **위치**: `llm-token-limit`, `llm-content-safety`, `llm-semantic-cache-lookup`, `llm-semantic-cache-store`
- **근거**: 넷 다 Consumption 티어를 지원 티어 목록에서 제외하고 있다. — 각 정책의 공식 문서 "APPLIES TO" 배너
- **수정안**: "AI 게이트웨이" 태그 옆에 공통 각주 "Consumption 티어에서는 사용할 수 없습니다" 추가

### I3. `publish-to-dapr`/`invoke-dapr-binding`에 "자체 호스팅 게이트웨이 전용" 안내 누락
- **위치**: 같은 카테고리의 `set-backend-service-dapr`에는 이 안내가 있는데 이 둘에는 없음
- **근거**: 둘 다 공식 문서상 `Gateways: self-hosted` 전용. — https://learn.microsoft.com/en-us/azure/api-management/publish-to-dapr-policy , .../invoke-dapr-binding-policy
- **수정안**: 두 `when`에 "(자체 호스팅 게이트웨이 전용)" 추가

### I4. `send-service-bus-message`에 classic 게이트웨이 전용 안내 없음
- **위치**: `APIM_POLICIES` → "통합 및 외부 통신"
- **근거**: v2/Consumption/자체 호스팅에서 지원 안 됨(classic 전용, Developer/Basic/Standard/Premium만). — https://learn.microsoft.com/en-us/azure/api-management/send-service-bus-message-policy
- **수정안**: `when`에 "classic 게이트웨이에서만 지원되며 v2·Consumption·자체 호스팅에서는 사용할 수 없습니다" 추가

### I5. 구조 비교의 APIM 티어 목록이 v2 제품군을 축약 표기
- **위치**: `GATEWAY_ROWS[0]` (실행 모델)
- **현재**: "Consumption/Developer/Basic/Standard/Premium/StandardV2 등 티어 선택"
- **근거**: 현재 티어는 클래식(Consumption/Developer/Basic/Standard/Premium) + v2 제품군 3종(Basic v2/Standard v2/Premium v2)이며, "StandardV2" 하나만 적으면 v2가 하나뿐인 것처럼 보인다. 표기도 "Standard v2"(띄어쓰기)가 맞다. — https://learn.microsoft.com/en-us/azure/api-management/v2-service-tiers-overview
- **수정안**: "클래식 티어(Consumption/Developer/Basic/Standard/Premium) + v2 티어(Basic v2/Standard v2/Premium v2) 중 선택"

### I6. PostgreSQL "Single Server" 서술이 시제상 낡음
- **위치**: `DATA_INFRA` → "PostgreSQL" → Flexible Server 항목
- **현재**: "예전 '단일 서버(Single Server)'는 지원 종료 수순이라"
- **근거**: Single Server는 이미 **2025-03-28부로 완전히 지원 종료**됐다(오늘 기준 1년 이상 지남). "수순"이 아니라 이미 끝난 사실이다. 남은 인스턴스는 자동으로 Flexible Server로 마이그레이션됐다. — Microsoft Tech Community 공지(techcommunity.microsoft.com), Microsoft Q&A
- **수정안**: "예전 '단일 서버(Single Server)'는 2025년 3월 28일부로 이미 완전히 지원 종료되었고, 남아있던 인스턴스는 자동으로 유연한 서버로 전환됐습니다. 지금은 신규든 기존이든 유연한 서버만 존재합니다."

### I7. Key Vault 접근 권한 안내가 최신 기본값을 반영하지 않음
- **위치**: `DATA_INFRA` → "SSL 인증서 / 방화벽 기초" → Key Vault 비밀 사용자 항목
- **근거**: 역할 이름("Key Vault 비밀 사용자") 자체는 맞지만, API 2026-02-01부터 **신규 Key Vault는 RBAC 전용이 기본값**이 됐다. `az keyvault set-policy`는 `--enable-rbac-authorization false`로 명시적으로 만든 vault에서만 동작한다. — https://learn.microsoft.com/en-us/azure/key-vault/general/access-control-default
- **수정안**: "신규 Key Vault는 기본적으로 RBAC 전용이라, `set-policy`는 `--enable-rbac-authorization false`로 만든 vault에만 적용됩니다. 기본 설정이라면 `az role assignment create --role \"Key Vault Secrets User\" --scope <vault-id> --assignee <apim-identity-object-id>`를 대신 씁니다." 각주 추가

---

## Minor (선택)

### M1. `cqlsh` 접속 호스트 접미사 표기
- **위치**: `DATA_INFRA` → "Cassandra" → cqlsh 항목
- **근거**: Microsoft 공식 문서 자체가 페이지마다 `cassandra.cosmos.azure.com`/`cassandra.cosmosdb.azure.com`으로 표기가 갈린다(문서 쪽의 불일치이지 이 트래커의 오류는 아님).
- **제안**: "정확한 호스트는 포털의 연결 문자열에서 그대로 복사하세요" 각주만 추가(선택사항, 급하지 않음)

---

## KQL 퀴즈

### K1. "상태 코드 + 서비스 필터" 정답의 컬럼명이 실존하지 않음
- **위치**: `KQL_QUIZ` → "상태 코드 + 서비스 필터"
- **현재**: `AppRequests | where toint(ResultCode) >= 500 | where CloudRoleName == "payment-api"`
- **근거**: `AppRequests` 테이블에 `CloudRoleName` 컬럼은 없다. 실제 컬럼은 `AppRoleName`이다(`ResultCode`가 string이라 `toint()`가 필요하다는 부분은 맞음). — https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/apprequests
- **수정안**: `AppRequests | where toint(ResultCode) >= 500 | where AppRoleName == "payment-api"`

---

## 검증되어 문제 없음 (참고)

- 정책 퀴즈(POLICY_QUIZ) 4문항 전체, KQL_OPERATORS 31개 항목 전체, 라우팅/캐싱/인증 카테고리의 나머지 정책들, Kong GATEWAY_ROWS 설명 전체, MONITOR_ROWS 7개 행 전체, Cosmos DB Cassandra 매핑(RU/일관성수준/GRANT-REVOKE/COMPACT STORAGE 등), App Gateway↔APIM 아키텍처 패턴, PROC_ITEMS/Jira 템플릿.

## 다음 단계

Critical 9건은 그대로 따라 하면 실제 Azure에서 실패하므로 우선 수정을 권장한다. Important 7건은 동작은 하지만 티어 제약이나 시점 정보가 빠져 있어 학습자가 잘못된 가정을 할 수 있다. 원하시면 이 문서를 기준으로 `azure-migration-tracker.html`을 직접 수정하겠습니다.
