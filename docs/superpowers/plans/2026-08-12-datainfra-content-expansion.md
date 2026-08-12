# 데이터·인프라 콘텐츠 보강 (2군) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `DATA_INFRA` 배열(`index.html`)의 5개 카테고리에 시니어 리뷰에서 지적된 공백을 메우는 신규 항목 27개를 추가해 23개 → 50개로 확장한다.

**Architecture:** 순수 데이터 추가. `DATA_INFRA`는 이미 `renderRefPage`(레퍼런스)와 `buildFullQuiz`(퀴즈)에서 공용 재사용되므로, 각 카테고리의 `items` 배열에 객체를 추가하기만 하면 레퍼런스 페이지와 퀴즈 양쪽에 자동 반영된다. JS 로직/CSS 변경 없음.

**Tech Stack:** 순수 HTML/CSS/Vanilla JS 단일 파일. 테스트 프레임워크 없음 — Node 구문 체크 + Playwright로 검증.

## Global Constraints

- `docs/superpowers/specs/2026-08-12-datainfra-content-expansion-design.md`의 설계를 따른다.
- 모든 CLI 명령/설정값은 실제 Azure CLI 문법이어야 한다(허구 금지) — 아래 각 항목은 이미 리서치 에이전트가 Microsoft Learn 공식 문서로 검증했으며, 각 항목 앞에 검증 출처 URL을 주석으로 남긴다.
- 기존 5개 카테고리 구조와 기존 23개 항목은 변경하지 않는다(순수 추가).
- 커밋 메시지는 한국어.

---

### Task 1: Cassandra 카테고리에 5개 항목 추가

**Files:** Modify `index.html` — `DATA_INFRA`의 `cat: "Cassandra → Azure Cosmos DB for Apache Cassandra"` 항목

**Interfaces:** 없음(독립적 데이터 추가, 다른 Task와 무관).

- [ ] **Step 1: 5개 항목 추가**

`index.html`에서 Cassandra 카테고리의 마지막 기존 항목(`지원되지 않는 기능 확인`) 뒤, 해당 카테고리를 닫는 `] },` 직전에 추가:

```js
      ,
      { name: "TTL 지원 범위", code: "TTL", url: "https://learn.microsoft.com/ko-kr/azure/cosmos-db/cassandra/support",
        easy: "ttl() 함수와 INSERT/UPDATE의 USING TTL 절 자체는 지원되지만, 순정 Cassandra와 달리 행(row) 단위로만 적용되고 셀(컬럼) 단위 TTL은 지원되지 않습니다. 또한 CREATE TABLE의 WITH 옵션은 gc_grace_seconds를 제외하면 전부 무시되므로, 테이블 생성 시 지정하는 default_time_to_live 옵션은 적용되지 않습니다.",
        when: "기존 스키마가 테이블 레벨 default_time_to_live에 의존한 만료 정책을 쓰고 있어, 마이그레이션 시 애플리케이션 레벨 TTL 지정으로 바꿔야 하는지 점검할 때",
        example: "INSERT INTO uprofile.user (id, message) VALUES (1, 'hi') USING TTL 3600;\n-- CREATE TABLE ... WITH default_time_to_live = 3600; 은 구문은 통과하지만 실제로는 무시됨(N/A)" },
      { name: "컬렉션 타입 제약", code: "list/set/map", url: "https://learn.microsoft.com/ko-kr/azure/cosmos-db/cassandra/support",
        easy: "list, set, map 컬렉션 타입은 모두 지원됩니다. 다만 frozen 컬렉션에는 보조 인덱스(secondary index)를 걸 수 없어, 컬렉션 자체보다 '컬렉션에 대한 인덱싱'에서 제약이 생깁니다.",
        when: "기존 스키마의 frozen<list<...>>/frozen<map<...>> 컬럼에 인덱스가 걸려 있어 쿼리 패턴 재검토가 필요할 때",
        example: "CREATE TABLE uprofile.tags (id UUID PRIMARY KEY, labels set<text>, attrs map<text, text>);\n-- CREATE INDEX ON uprofile.tags (labels); 는 가능하나 frozen<set<text>> 전체 동결 컬렉션 인덱싱은 미지원" },
      { name: "경량 트랜잭션(LWT) 지원 현황", code: "IF NOT EXISTS / LWT", url: "https://learn.microsoft.com/ko-kr/azure/cosmos-db/cassandra/support",
        easy: "일반적인 '미지원' 통념과 달리 INSERT IF NOT EXISTS, UPDATE IF EXISTS, DELETE IF EXISTS 등 경량 트랜잭션 구문은 모두 지원됩니다. 단, 계정에 멀티 리전 쓰기(multi-region writes)가 활성화되어 있으면 LWT를 사용할 수 없다는 명시적 제약이 있습니다.",
        when: "다중 리전 쓰기(active-active) 구성과 LWT 기반 유일성 보장 로직을 동시에 쓰려 할 때, 둘 중 하나를 선택해야 함을 사전에 인지",
        example: "INSERT INTO uprofile.user (id, user) VALUES (1, 'theo') IF NOT EXISTS;\n-- 단일 리전 쓰기 계정에서만 보장됨. 멀티 리전 쓰기 활성화 계정에서는 미지원" },
      { name: "파티션 키 재설계", code: "logical partition", url: "https://learn.microsoft.com/ko-kr/azure/cosmos-db/cassandra/partitioning",
        easy: "네이티브 Cassandra는 murmur3 해시 기반 토큰 링을 노드에 분배하는 방식이지만, Cosmos DB는 같은 토큰 범위를 유지하면서 내부적으로 논리적 파티션(최대 20GB)을 물리적 파티션(파티션당 최대 10000 RU)에 자동 배치합니다. 파티션 키 카디널리티가 낮으면 하나의 논리 파티션에 쓰기가 몰려 핫 파티션 문제가 생기므로, 복합 파티션 키로 재설계해 분산도를 높여야 합니다.",
        when: "기존에 단일 컬럼 파티션 키(예: user_id)를 쓰던 테이블을 이전할 때, RU 소비가 특정 파티션에 쏠리지 않도록 키 설계를 다시 검토할 때",
        example: "-- 복합 파티션 키로 분산도 향상: firstname+lastname 조합을 파티션 키로 사용\nCREATE TABLE uprofile.user (\n  firstname text, lastname text, id int, message text,\n  PRIMARY KEY ((firstname, lastname), id)\n) WITH cosmosdb_provisioned_throughput=2000;" },
      { name: "드라이버 호환성", code: "CQL protocol v4", url: "https://learn.microsoft.com/ko-kr/azure/cosmos-db/cassandra/support",
        easy: "Cosmos DB Cassandra API는 CQL Binary Protocol v4를 사용하며 CQL v3.11 API와 호환됩니다. DataStax 계열 드라이버는 Java 3.5+, C# 3.5+, Node.js 3.5+, Python 3.15+, C++ 2.9, PHP 1.3 이상을 요구하며, DSE나 Cassandra 4.0용 cqlsh는 연결이 되지 않으므로 반드시 오픈소스 Cassandra 3.11 버전의 cqlsh를 써야 합니다.",
        when: "기존에 쓰던 드라이버/cqlsh 버전이 낮거나 DSE 전용 드라이버를 쓰고 있어 연결 실패가 예상될 때, 사전에 드라이버 버전을 올려야 하는지 확인",
        example: "cqlsh <account>.cassandra.cosmos.azure.com 10350 -u <account> -p <password> --ssl --protocol-version=4\n-- Java 3.5+, C# 3.5+, Node.js 3.5+, Python 3.15+, C++ 2.9, PHP 1.3+ 드라이버 필요" }
```

- [ ] **Step 2: 구문 검사**

Run: `node -e "const fs=require('fs');const html=fs.readFileSync('D:/99. S/20260730_Azure/index.html','utf8');const scripts=[...html.matchAll(/<script>([\s\S]*?)<\/script>/g)].map(m=>m[1]);scripts.forEach((s,i)=>{try{new Function(s);console.log('block '+i+': OK');}catch(e){console.log('block '+i+': SYNTAX ERROR -> '+e.message);}});"`
Expected: `block 0: OK`, `block 1: OK`.

- [ ] **Step 3: 커밋**

```bash
git add index.html
git commit -m "docs: Cassandra 레퍼런스에 TTL/컬렉션/LWT/파티션키/드라이버 항목 5개 추가"
```

---

### Task 2: PostgreSQL 카테고리에 5개 항목 추가

**Files:** Modify `index.html` — `DATA_INFRA`의 `cat: "PostgreSQL → Azure Database for PostgreSQL"` 항목

**Interfaces:** 없음.

- [ ] **Step 1: 5개 항목 추가**

PostgreSQL 카테고리의 마지막 기존 항목(`NSG 트래픽 허용`) 뒤, 카테고리를 닫는 `] },` 직전에 추가:

```js
      ,
      { name: "내장 커넥션 풀링(PgBouncer)", code: "pgbouncer.enabled", url: "https://learn.microsoft.com/ko-kr/azure/postgresql/connectivity/concepts-pgbouncer",
        easy: "Flexible Server에는 PgBouncer가 이미 내장되어 있고, 별도 서버 없이 포트 6432로 바로 접속할 수 있습니다. pgbouncer.enabled 파라미터를 true로 바꾸기만 하면 재시작 없이 켜지고, 기본 풀 모드는 transaction입니다.",
        when: "짧은 커넥션을 대량으로 맺고 끊는 트랜잭션형 애플리케이션에서 유휴 연결 때문에 서버 리소스가 부족할 때",
        example: "az postgres flexible-server parameter set --resource-group rg-migration --server-name myserver --name pgbouncer.enabled --value true" },
      { name: "읽기 복제본(Read Replica)", code: "az postgres flexible-server replica create", url: "https://learn.microsoft.com/ko-kr/cli/azure/postgres/flexible-server/replica",
        easy: "읽기 복제본은 비동기 복제로 동작하는 읽기 전용 서버라서 쓰기는 할 수 없습니다. 복제를 끊고 싶으면 replica promote 명령을 쓰는데, standalone(독립 서버화)과 switchover(주 서버로 승격) 두 모드가 있고 planned/forced 옵션에 따라 데이터 동기화 여부가 달라집니다.",
        when: "리포팅·분석 쿼리를 운영 DB에서 분리해 읽기 트래픽을 확장하고 싶을 때",
        example: "az postgres flexible-server replica create --name myserver-replica --resource-group rg-migration --source-server myserver --location koreacentral" },
      { name: "자동 유지관리(Maintenance Window)", code: "--maintenance-window", url: "https://learn.microsoft.com/ko-kr/azure/postgresql/configure-maintain/how-to-configure-scheduled-maintenance",
        easy: "매달 보안 패치·마이너 버전 업그레이드가 자동으로 진행되며, '요일:시:분' 형식으로 원하는 1시간 창을 직접 지정하거나 시스템이 밤 11시~오전 7시 사이 임의 시간을 고르게 둘 수 있습니다. 유지관리 중에는 서버 재시작 등으로 일부 다운타임이 발생할 수 있습니다.",
        when: "새벽 배치 작업 시간과 겹치지 않도록 유지관리 창을 특정 요일 새벽으로 고정하고 싶을 때",
        example: "az postgres flexible-server update --resource-group rg-migration --name myserver --maintenance-window Sun:03:00" },
      { name: "백업 및 시점 복원(PITR)", code: "az postgres flexible-server restore", url: "https://learn.microsoft.com/ko-kr/azure/postgresql/backup-restore/concepts-backup-restore",
        easy: "기본 백업 보존 기간은 7일이고 최대 35일까지 늘릴 수 있습니다. 지역 중복 백업(geo-redundant)은 서버를 만들 때만 선택할 수 있고 이후에는 변경할 수 없습니다. 복원은 항상 새 서버를 만드는 방식이라 기존 서버는 덮어쓰지 않습니다.",
        when: "실수로 테이블이나 데이터베이스를 삭제해 사고 발생 직전 시점으로 되돌려야 할 때",
        example: "az postgres flexible-server restore --resource-group rg-migration --name myserver-restored --source-server myserver --restore-time \"2026-08-10T02:00:00+00:00\"" },
      { name: "서버 매개변수 커스터마이징(STATIC/DYNAMIC)", code: "az postgres flexible-server parameter set", url: "https://learn.microsoft.com/ko-kr/azure/postgresql/parameters/how-to-parameters-list-read-write-static",
        easy: "Flexible Server는 별도 파라미터 그룹 리소스가 없고 서버마다 직접 파라미터를 설정합니다. Static 파라미터(예: max_connections)는 값을 바꾼 뒤 서버 재시작이 있어야 적용되고, Dynamic 파라미터는 재시작 없이 바뀌지만 이미 연결된 세션이 아니라 새로 맺는 연결부터 반영됩니다.",
        when: "동시 접속자 수 한도(max_connections)처럼 재시작이 필요한 값을 변경할 때 배포 창(다운타임)을 미리 계획해야 하는 경우",
        example: "az postgres flexible-server parameter set --resource-group rg-migration --server-name myserver --name max_connections --value 200" }
```

- [ ] **Step 2: 구문 검사** (Task 1과 동일 명령)

- [ ] **Step 3: 커밋**

```bash
git add index.html
git commit -m "docs: PostgreSQL 레퍼런스에 커넥션풀링/read replica/유지관리/백업/파라미터 항목 5개 추가"
```

---

### Task 3: Application Gateway vs API Management 카테고리에 5개 항목 추가

**Files:** Modify `index.html` — `DATA_INFRA`의 `cat: "Application Gateway vs API Management"` 항목

**Interfaces:** 없음.

- [ ] **Step 1: 5개 항목 추가**

App Gateway 카테고리의 마지막 기존 항목(`호스트 헤더 처리 주의점`) 뒤, 카테고리를 닫는 `] },` 직전에 추가:

```js
      ,
      { name: "WAF_v2 티어와 OWASP 규칙 세트", code: "WAF_v2 SKU / Detection vs Prevention", url: "https://learn.microsoft.com/ko-kr/azure/web-application-firewall/ag/application-gateway-crs-rulegroups-rules",
        easy: "WAF_v2는 방화벽 기능이 포함된 Application Gateway SKU입니다. 정책은 Detection(탐지만, 차단 안 함) 또는 Prevention(실제 차단) 모드로 동작하며, 공격 패턴은 OWASP Core Rule Set(예: 3.1) 버전으로 관리됩니다. 특정 규칙만 끄거나 커스텀 규칙도 추가할 수 있습니다.",
        when: "SQL 삽입·XSS 같은 웹 공격을 자동 차단하는 방화벽을 Application Gateway에 붙이려 할 때",
        example: "az network application-gateway waf-policy create --name MyWAFPolicy --resource-group MyResourceGroup\naz network application-gateway waf-policy managed-rule rule-set add --policy-name MyWAFPolicy --resource-group MyResourceGroup --type OWASP --version 3.1 --group-name REQUEST-921-PROTOCOL-ATTACK --rule rule-id=921110\naz network application-gateway waf-policy policy-setting update --policy-name MyWAFPolicy --resource-group MyResourceGroup --mode Prevention --state Enabled" },
      { name: "경로 기반 라우팅", code: "URL Path-based Routing", url: "https://learn.microsoft.com/ko-kr/azure/application-gateway/tutorial-url-route-cli",
        easy: "URL 경로별로 서로 다른 백엔드 풀에 트래픽을 나눠 보낼 수 있습니다. 예를 들어 /images/*는 이미지 서버 풀로, /api/*는 API 서버 풀로, 나머지는 기본 풀로 라우팅합니다. Kong의 경로 기반 라우트 설정과 동일한 개념입니다.",
        when: "하나의 Application Gateway로 여러 마이크로서비스(예: /images, /api)를 경로별로 다른 백엔드에 연결하고 싶을 때",
        example: "az network application-gateway url-path-map create -g MyResourceGroup --gateway-name MyAppGateway -n MyUrlPathMap --rule-name MyUrlPathMapRule1 --paths /mypath1/* --address-pool MyAddressPool --default-address-pool MyAddressPool --http-settings MyHttpSettings --default-http-settings MyHttpSettings" },
      { name: "상태 프로브(Health Probe)와 백엔드 풀", code: "Health Probe / interval·threshold", url: "https://learn.microsoft.com/ko-kr/cli/azure/network/application-gateway/probe",
        easy: "헬스 프로브는 백엔드 서버에 주기적으로 요청을 보내 정상 응답 여부를 확인합니다. interval(기본 30초)마다 확인하고, threshold(기본 8회) 연속 실패하면 해당 백엔드를 비정상으로 표시해 트래픽에서 자동 제외시킵니다. Kong의 upstream health check와 같은 역할입니다.",
        when: "백엔드 서버 하나가 다운됐을 때 Application Gateway가 자동으로 그 서버를 트래픽에서 빼주길 원할 때",
        example: "az network application-gateway probe create -g MyResourceGroup --gateway-name MyAppGateway -n MyProbe --protocol Https --host 127.0.0.1 --path /health --interval 30 --threshold 3 --timeout 30" },
      { name: "오토스케일링", code: "Autoscaling (--min-capacity/--max-capacity)", url: "https://learn.microsoft.com/ko-kr/azure/application-gateway/application-gateway-autoscaling-zone-redundant",
        easy: "Standard_v2/WAF_v2 SKU는 최소·최대 인스턴스 수만 지정하면 트래픽에 따라 자동으로 늘고 줄어듭니다. 최소 인스턴스는 0부터, 최대는 최대 125까지 설정 가능하며, 트래픽이 없을 때 최소값 이하로는 내려가지 않습니다.",
        when: "고정 인스턴스 수 대신 트래픽 변동에 맞춰 게이트웨이가 자동으로 확장·축소되게 하고 싶을 때",
        example: "az network application-gateway update --name MyAppGateway --resource-group MyResourceGroup --min-capacity 2 --max-capacity 10" },
      { name: "TLS 정책", code: "SSL Policy / Predefined Policy", url: "https://learn.microsoft.com/ko-kr/cli/azure/network/application-gateway/ssl-policy",
        easy: "Application Gateway는 Azure가 미리 정의한 SSL/TLS 정책(예: AppGwSslPolicy20220101S) 하나를 지정하거나, 최소 TLS 버전과 암호화 스위트 목록을 직접 커스텀으로 지정할 수 있습니다. 20220101S 계열 정책은 TLS 1.2 이상만 허용하는 최신 강화 정책입니다.",
        when: "오래된 TLS 1.0/1.1 접속을 막고 최신 암호화 스위트만 허용하도록 강제해야 할 때",
        example: "az network application-gateway ssl-policy set -g MyResourceGroup --gateway-name MyAppGateway -n AppGwSslPolicy20220101S --policy-type Predefined" }
```

- [ ] **Step 2: 구문 검사** (Task 1과 동일 명령)

- [ ] **Step 3: 커밋**

```bash
git add index.html
git commit -m "docs: Application Gateway 레퍼런스에 WAF/경로라우팅/헬스프로브/오토스케일/TLS 항목 5개 추가"
```

---

### Task 4: SSL 인증서 / 방화벽 기초 카테고리에 6개 항목 추가

**Files:** Modify `index.html` — `DATA_INFRA`의 `cat: "SSL 인증서 / 방화벽 기초"` 항목

**Interfaces:** 없음.

- [ ] **Step 1: 6개 항목 추가**

SSL/방화벽 카테고리의 마지막 기존 항목(`NSG(네트워크 보안 그룹) 기초`) 뒤, 카테고리를 닫는 `] },` 직전에 추가:

```js
      ,
      { name: "NSG vs Azure Firewall vs WAF 계층 비교", code: "netlayer-compare", url: "https://learn.microsoft.com/en-us/azure/networking/security/network-security",
        easy: "셋 다 '방화벽'이라 불리지만 계층이 다릅니다. NSG는 서브넷/NIC 단위로 IP·포트를 허용/차단하는 L3/L4 ACL, Azure Firewall은 VNet(허브-스포크)이나 Virtual WAN 허브 전체를 대상으로 하는 중앙집중식 L3/L4 스테이트풀 방화벽(위협 인텔리전스 포함), WAF는 App Gateway나 Front Door 위에서 동작하며 HTTP 요청 내용(SQL 인젝션, XSS 등)을 OWASP 규칙 기준으로 검사하는 L7 서비스입니다.",
        when: "'트래픽이 왜 막히는지' 원인을 찾을 때 어느 계층부터 봐야 할지 헷갈릴 때 — 포트 단위 차단이면 NSG, VNet 간 아웃바운드 정책이면 Azure Firewall, HTTP 요청 내용 기반 차단이면 WAF",
        example: "-- NSG: 서브넷/NIC 단위, L3/L4, 무료\n-- Azure Firewall: VNet/허브 단위 중앙집중, L3/L4 + 위협 인텔리전스, 유료(시간+데이터 처리량)\n-- WAF: App Gateway/Front Door에 부착, L7(HTTP), OWASP CRS 기반, 유료" },
      { name: "Azure Firewall 기본 개념", code: "AzFirewall", url: "https://learn.microsoft.com/en-us/cli/azure/network/firewall",
        easy: "Azure Firewall은 허브 VNet(또는 Virtual WAN 허브)에 하나 배치해 여러 스포크 VNet의 인/아웃바운드·이스트웨스트 트래픽을 한 곳에서 통제하는 관리형 스테이트풀 방화벽입니다. threat-intel-mode를 Alert(경고만)나 Deny(차단)로 켜면 마이크로소프트 위협 인텔리전스 피드에 있는 악성 IP·도메인 통신을 자동으로 잡아냅니다.",
        when: "허브-스포크 구조에서 스포크마다 규칙을 반복 설정하지 않고 한 곳에서 아웃바운드 정책을 통제하고 싶을 때",
        example: "az network firewall create -g MyResourceGroup -n MyFirewall --sku AZFW_VNet --tier Standard --vnet-name MyVNet --conf-name MyIpConfig --public-ip MyPublicIp --threat-intel-mode Alert" },
      { name: "App Service 관리 인증서(무료)", code: "ASMC", url: "https://learn.microsoft.com/en-us/azure/app-service/configure-ssl-certificate",
        easy: "App Service 커스텀 도메인용 TLS 인증서를 무료로 발급받고 App Service가 자동으로 갱신까지 맡아줍니다. 단, App Service 플랜이 Basic 이상이어야 하고(Free/Shared 불가), 와일드카드 인증서·사설 DNS·App Service Environment는 지원하지 않으며 내보내기(export)도 불가능합니다. 현재 az CLI로 생성하는 명령은 미리보기(Preview) 상태입니다.",
        when: "커스텀 도메인에 HTTPS만 필요하고 인증서 구매·수동 갱신 관리를 피하고 싶을 때",
        example: "az webapp config ssl create --resource-group MyResourceGroup --name MyWebapp --hostname cname.mycustomdomain.com" },
      { name: "인증서 회전 자동화 검증", code: "cert-rotate-check", url: "https://learn.microsoft.com/en-us/azure/key-vault/certificates/tutorial-rotate-certificates",
        easy: "Key Vault 인증서는 파트너 CA(DigiCert, GlobalSign)로 발급받았을 때만 자동 회전(autorotation)이 동작하며, 기본 정책은 수명의 80% 지점에서 자동 갱신하도록 되어 있습니다. '설정했으니 되겠지'가 아니라 실제 만료일이 갱신되고 있는지 주기적으로 직접 확인해야 합니다.",
        when: "만료 임박 알림 이메일만 믿지 않고, 실제로 인증서가 자동 갱신되어 만료일이 밀리고 있는지 직접 검증하고 싶을 때",
        example: "az keyvault certificate show --vault-name MyVault --name MyCert --query \"attributes.expires\" -o tsv\n-- 갱신 시점(수명의 몇 %/만료 며칠 전) 조회 전용 CLI는 없어 포털의 인증서 > Issuance Policy 화면이나 PowerShell Set-AzKeyVaultCertificatePolicy로 확인" },
      { name: "프라이빗 엔드포인트(Private Endpoint)", code: "PrivateEndpoint", url: "https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview",
        easy: "Private Endpoint는 VNet 서브넷 안에 사설 IP를 가진 NIC를 만들어 Key Vault·Storage 같은 PaaS 서비스를 공용 인터넷을 거치지 않고 연결합니다. NSG/UDR 네트워크 정책을 적용할 수 있지만, 연결은 클라이언트 쪽에서만 시작 가능해 서비스가 먼저 트래픽을 보내는 구조가 아니므로 '아웃바운드 차단' 같은 규칙은 의미가 없습니다.",
        when: "DB(PostgreSQL)나 Key Vault를 공인 IP 노출 없이 VNet 내부 전용으로 열고 싶을 때, 또는 NSG 규칙이 프라이빗 엔드포인트에 왜 안 먹히는지 헷갈릴 때",
        example: "az network private-endpoint create -g MyResourceGroup -n MyPE --vnet-name MyVNet --subnet MySubnet --group-id vault --private-connection-resource-id \"/subscriptions/<sub-id>/resourceGroups/MyResourceGroup/providers/Microsoft.KeyVault/vaults/MyVault\" --connection-name MyConnection" },
      { name: "DDoS 보호 기본/표준", code: "DDoSProtection", url: "https://learn.microsoft.com/en-us/azure/ddos-protection/ddos-protection-overview",
        easy: "모든 Azure 공인 IP는 별도 설정 없이 무료 '인프라 수준 DDoS 보호'(과거 명칭 Basic)를 자동으로 받습니다. VNet 단위로 유료 플랜을 붙이면(현재 명칭 DDoS Network Protection, 과거 Standard) 공격 완화 정책 튜닝, 공격 리포트, 비용 보장(Cost Protection)까지 받을 수 있습니다.",
        when: "인터넷에 노출된 서비스가 대규모 트래픽 공격에 얼마나 버틸지, 기본 제공 보호만으로 충분한지 판단할 때",
        example: "az network ddos-protection create -g MyResourceGroup -n MyDdosPlan --vnets MyVnet" }
```

- [ ] **Step 2: 구문 검사** (Task 1과 동일 명령)

- [ ] **Step 3: 커밋**

```bash
git add index.html
git commit -m "docs: SSL/방화벽 레퍼런스에 계층비교/Azure Firewall/관리인증서/Private Endpoint/DDoS 항목 6개 추가"
```

---

### Task 5: Azure 기초 개념 카테고리에 6개 항목 추가

**Files:** Modify `index.html` — `DATA_INFRA`의 `cat: "Azure 기초 개념"` 항목

**Interfaces:** 없음.

- [ ] **Step 1: 6개 항목 추가**

Azure 기초 카테고리의 마지막 기존 항목(`관리 ID(Managed Identity)`) 뒤, 카테고리와 `DATA_INFRA` 배열을 닫는 `] }\n  ];` 직전에 추가:

```js
      ,
      { name: "관리 그룹(Management Group)", code: "Management Group", url: "https://learn.microsoft.com/ko-kr/azure/governance/management-groups/overview",
        easy: "관리 그룹은 여러 구독을 하나의 상위 계층으로 묶어서, 정책이나 RBAC 역할을 구독 여러 개에 한 번에 적용할 수 있게 해주는 컨테이너입니다. 예를 들어 '전사' 관리 그룹 아래에 '개발', '운영' 구독을 두면, 전사 수준에서 부여한 권한이나 정책이 두 구독에 모두 상속됩니다.",
        when: "구독이 여러 개라서 정책·권한을 구독 단위가 아니라 조직 전체 단위로 관리하고 싶을 때",
        example: "az account management-group create --name mg-migration --display-name \"Migration Group\" --parent mg-platform" },
      { name: "RBAC 스코프 4단계", code: "RBAC Scope Hierarchy", url: "https://learn.microsoft.com/ko-kr/azure/role-based-access-control/scope-overview",
        easy: "RBAC 역할은 관리 그룹 → 구독 → 리소스 그룹 → 리소스 순으로 점점 좁아지는 4단계 범위(scope) 중 원하는 위치에 할당할 수 있습니다. 상위 단계에서 부여한 역할은 하위 단계로 자동 상속되므로, 구독 수준에서 Reader를 주면 그 안의 모든 리소스 그룹·리소스에서도 읽기 권한이 생깁니다.",
        when: "권한을 조직 전체/구독 전체/특정 프로젝트 중 어느 범위까지 적용할지 결정할 때",
        example: "az role assignment create --assignee user@contoso.com --role Reader --scope \"/providers/Microsoft.Management/managementGroups/mg-migration\"\naz role assignment create --assignee user@contoso.com --role Reader --scope \"/subscriptions/{subscription-id}\"\naz role assignment create --assignee user@contoso.com --role Reader --scope \"/subscriptions/{subscription-id}/resourceGroups/rg-migration\"" },
      { name: "시스템 할당 vs 사용자 할당 관리 ID", code: "Managed Identity (System vs User Assigned)", url: "https://learn.microsoft.com/ko-kr/entra/identity/managed-identities-azure-resources/overview",
        easy: "시스템 할당 관리 ID는 VM 같은 리소스를 만들 때 함께 생성되어 그 리소스와 생명주기를 공유합니다 — 리소스를 삭제하면 ID도 자동으로 같이 삭제됩니다. 사용자 할당 관리 ID는 독립된 리소스로 먼저 만들어두고 여러 VM·App Service에 나눠서 붙일 수 있어 공유가 가능합니다.",
        when: "여러 리소스가 동일한 권한 세트를 재사용해야 하거나, 리소스 재활용이 잦아도 권한을 유지하고 싶을 때",
        example: "az identity create --name id-migration-shared --resource-group rg-migration" },
      { name: "Azure Policy 기초", code: "Azure Policy", url: "https://learn.microsoft.com/ko-kr/azure/governance/policy/overview",
        easy: "Azure Policy는 리소스가 조직의 규칙을 지키는지 강제하거나 감사하는 기능입니다(예: '특정 지역에만 리소스 생성 허용'). RBAC이 '누가 무엇을 할 수 있는가'를 정한다면, Policy는 '리소스 상태가 어떤 조건을 만족해야 하는가'를 정합니다 — 권한이 있어도 정책을 어기는 리소스는 생성이 거부될 수 있습니다.",
        when: "팀원 권한과 무관하게 위치·SKU·필수 태그 같은 규칙을 조직 전체에 강제하고 싶을 때",
        example: "az policy assignment create --name allow-korea-only --policy {policyName} --scope \"/subscriptions/{subscription-id}/resourceGroups/rg-migration\" --params \"{ 'allowedLocations': { 'value': ['koreacentral'] } }\"" },
      { name: "내장 역할 vs 사용자 지정 역할", code: "Built-in vs Custom Role", url: "https://learn.microsoft.com/ko-kr/azure/role-based-access-control/custom-roles",
        easy: "Contributor·Reader·Owner 같은 내장 역할은 Azure가 미리 만들어둔 표준 권한 세트로 대부분의 상황에 바로 쓸 수 있습니다. 내장 역할로 표현할 수 없는 세밀한 조합(예: 'VM 재시작은 되지만 삭제는 안 됨')이 필요하면 Actions/NotActions를 직접 조합한 사용자 지정 역할을 JSON으로 정의해서 만듭니다.",
        when: "내장 역할의 권한 조합이 너무 넓거나 좁아서 딱 맞는 역할이 없을 때",
        example: "az role definition create --role-definition '{ \"Name\": \"VM Operator\", \"Description\": \"Can monitor and restart virtual machines.\", \"Actions\": [\"Microsoft.Compute/*/read\", \"Microsoft.Compute/virtualMachines/restart/action\"], \"NotActions\": [], \"AssignableScopes\": [\"/subscriptions/{subscription-id}\"] }'" },
      { name: "태그(Tags)", code: "Tags", url: "https://learn.microsoft.com/ko-kr/azure/azure-resource-manager/management/tag-resources",
        easy: "태그는 리소스·리소스 그룹·구독에 key=value 형태의 메타데이터를 붙여서 분류하거나 비용을 추적하는 기능입니다(예: environment=prod, costcenter=teamA). 리소스 그룹에 붙인 태그가 그 안의 리소스에 자동으로 상속되지는 않으므로 리소스마다 따로 붙이거나 정책으로 강제해야 합니다.",
        when: "비용 보고서에서 팀/환경별로 리소스를 묶어서 보고 싶을 때",
        example: "az resource tag --tags environment=prod costcenter=teamA --resource-group rg-migration --name apim-migration --resource-type \"Microsoft.ApiManagement/service\"" }
```

- [ ] **Step 2: 구문 검사** (Task 1과 동일 명령)

- [ ] **Step 3: 커밋**

```bash
git add index.html
git commit -m "docs: Azure 기초 개념 레퍼런스에 관리그룹/RBAC스코프/관리ID종류/Policy/역할/태그 항목 6개 추가"
```

---

### Task 6: 통합 검증

**Files:** 코드 변경 없음(검증 전용).

- [ ] **Step 1: 항목 총 개수 확인**

Playwright로 `panel-data-ref` 패널을 열어 카테고리별 `.pcard` 개수를 세어, Cassandra 12·PostgreSQL 11·App Gateway 8·SSL/방화벽 10·Azure 기초 10 (총 50개)인지 확인.

- [ ] **Step 2: 퀴즈 자동 반영 확인**

`panel-data-quiz` 패널에서 "전체 항목 복습 (50개)" 문구가 나오는지 확인(기존 23개 → 50개로 자동 반영되는지가 이 Task의 핵심 확인 포인트).

- [ ] **Step 3: 콘솔 에러 확인**

전체 로드 및 두 패널 탐색 중 콘솔 에러 0건 확인.

- [ ] **Step 4: 완료 보고**

문제 없으면 사용자에게 최종 항목 수와 카테고리별 분포를 보고한다.

---

## Self-Review 체크리스트

- [x] 스펙 커버리지: 설계 문서의 5개 카테고리 27개 항목이 Task 1~5로 정확히 매핑됨(Cassandra 5·PostgreSQL 5·App Gateway 5·SSL/방화벽 6·Azure 기초 6 = 27).
- [x] 플레이스홀더 없음: 모든 항목이 실제 검증된 CLI 문법과 실제 문서 URL을 포함함.
- [x] 내부 일관성: 설계 문서의 "경량 트랜잭션(LWT) 미지원" 항목명은 실제 검증 결과(멀티 리전 쓰기 시에만 미지원) 반영해 "경량 트랜잭션(LWT) 지원 현황"으로 조정함 — 사실 정확성 우선.
- [x] 범위 검사: 카테고리 신설·기존 항목 수정 없이 순수 추가만, 단일 계획으로 구현 가능한 크기.
