# 로컬 테스트 랩 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `azure-migration-tracker.html`에 "로컬 테스트 랩" nav 그룹(랩 개요/API 테스트/APIM Policy/로그 비교/마이그레이션 5개 패널)을 순수 추가하여, 샘플 Kong 설정과 그 APIM 대응 정책을 로컬 mock 엔진으로 실행하고 결과를 나란히 비교하며 학습할 수 있게 한다.

**Architecture:** 기존 `railNav`/`data-panel` 라우터와 `NAV` 배열에 새 그룹 하나를 추가하는 방식. Kong mock과 APIM mock은 각각 `runKongMock`/`runApimMock` 함수로 구현하고 둘 다 동일한 `{statusCode, headers, body, trace}` 셰이프를 반환해 `renderLabResult` 렌더러 하나를 공유한다. APIM 쪽은 브라우저 내장 `DOMParser`로 정책 XML을 실제 파싱하되, 7개 화이트리스트 정책만 실행 로직을 연결한다.

**Tech Stack:** 순수 HTML/CSS/바닐라 JS (빌드 도구 없음), 브라우저 내장 `DOMParser`, `localStorage`(폴백: 메모리 객체).

## Global Constraints

- 기존 8개 패널(`구조 비교`, `APIM 정책 레퍼런스/퀴즈`, `KQL 레퍼런스/퀴즈`, `데이터·인프라 레퍼런스/퀴즈`, `AIDD·Jira`)과 그 함수(`renderRefPage`, `buildFullQuiz`, `refUrl` 등)는 절대 수정하지 않는다 — 순수 추가만 한다.
- 이 프로젝트는 git 저장소가 아니고 빌드/테스트 러너가 없다 — 모든 "테스트" 단계는 브라우저에서 파일을 직접 열어 수동으로 확인하는 시나리오다. `git add`/`git commit` 스텝은 포함하지 않는다.
- Kong 설정 예시와 APIM 정책 XML은 반드시 실제 문법을 따른다(CLAUDE.md 규칙). 이 계획의 XML은 Microsoft Learn 공식 문서(check-header-policy, rate-limit-policy, ip-filter-policy, rewrite-uri-policy, set-header-policy, return-response-policy, cors-policy)로 검증된 문법이다.
- Kong 플러그인 실행 순서는 실제 Kong의 `priority` 기반 순서를 그대로 재현하지 않고, 학습 편의를 위한 고정 순서(인증 → IP 제한 → 트래픽 제어 → 변환 → 종료)를 쓴다는 점을 UI에 명시한다 — 정확성을 가장하지 않는다.
- 화이트리스트(`check-header`, `rate-limit`, `cors`, `set-header`, `rewrite-uri`, `ip-filter`, `return-response`) 밖의 정책 태그는 차단하지 않고 "시뮬레이션 미지원" 배지만 남긴다.

---

## Task 1: Nav 그룹 + 패널 셸 + CSS 추가

**Files:**
- Modify: `azure-migration-tracker.html` (CSS는 `</style>` 직전, `NAV` 배열은 `jira` 항목 뒤, 패널 HTML은 `panel-jira` `</section>` 뒤, `state.navOpen` 기본값은 `defaultState()` 안)

**Interfaces:**
- Produces: 5개 새 `data-panel` id (`panel-lab-overview`, `panel-lab-api-test`, `panel-lab-policy`, `panel-lab-logs`, `panel-lab-migration`), 컨테이너 DOM id(`labOverviewBody`, `labApiTestBody`, `labPolicyBody`, `labLogsBody`, `labMigrationBody`) — Task 2·4·5·6·7이 이 id들에 렌더링한다.

- [ ] **Step 1: CSS 추가**

`azure-migration-tracker.html`에서 `.theme-toggle { position: fixed; ... }` 규칙 바로 다음, `</style>` 바로 앞에 추가:

```css
.lab-compare-grid { display: grid; grid-template-columns: 1fr 1fr; gap: var(--sp-4); }
@media (max-width: 720px) { .lab-compare-grid { grid-template-columns: 1fr; } }
.lab-result-title { font-size: var(--fs-md); margin-bottom: var(--sp-2); }
.lab-status-2xx { color: #1a7f37; font-weight: 700; }
.lab-status-4xx { color: #b45309; font-weight: 700; }
.lab-status-5xx { color: #b91c1c; font-weight: 700; }
.lab-headers { font-size: var(--fs-xs); color: var(--text-dim); margin: var(--sp-2) 0; font-family: var(--font-mono); }
.lab-trace { display: flex; flex-direction: column; gap: 4px; margin-top: var(--sp-2); }
.lab-trace-row { font-size: var(--fs-xs); color: var(--text-dim); }
.lab-trace-row code { color: var(--accent-target); }
.lab-input-row { display: flex; gap: var(--sp-3); flex-wrap: wrap; margin-bottom: var(--sp-3); }
.lab-input-row label { font-size: var(--fs-xs); color: var(--text-faint); display: flex; flex-direction: column; gap: 4px; }
.lab-input-row input { border: 1px solid var(--border); border-radius: var(--radius-md); padding: 6px 10px; background: var(--bg); color: var(--text); }
.lab-route-tabs { display: flex; gap: var(--sp-2); flex-wrap: wrap; margin-bottom: var(--sp-4); }
.lab-route-tabs button.active { border-color: var(--accent-target); color: var(--accent-target); }
```

- [ ] **Step 2: `NAV` 배열에 그룹 추가**

`azure-migration-tracker.html`에서 `{ id: "jira", label: "AIDD·Jira", ... }` 항목 뒤, 배열을 닫는 `];` 앞에 추가:

```js
,
    { id: "lab", label: "로컬 테스트 랩", icon: '<path d="M9 3h6M10 3v6l-5 9a2 2 0 0 0 2 3h10a2 2 0 0 0 2-3l-5-9V3"/>', children: [
        { id: "lab-overview", label: "랩 개요", icon: '<rect x="3" y="4" width="18" height="16" rx="1"/><path d="M3 9h18"/>' },
        { id: "lab-api-test", label: "API 테스트", icon: '<path d="M4 17l6-6-6-6M12 19h8"/>' },
        { id: "lab-policy", label: "APIM Policy", icon: '<path d="M6 4h9l5 5v11H6z"/><path d="M15 4v5h5"/><path d="M9 12h6M9 15.5h6"/>' },
        { id: "lab-logs", label: "로그 비교", icon: '<rect x="3" y="4" width="18" height="16" rx="1"/><path d="M7 9l3 3-3 3M13 15h4"/>' },
        { id: "lab-migration", label: "마이그레이션", icon: '<path d="M4 12h16M14 6l6 6-6 6"/>' }
      ] }
```

- [ ] **Step 3: `defaultState()`의 `navOpen`에 `lab: true` 추가**

`defaultState()` 안의 `navOpen: { apim: true, kqlgroup: true, datainfra: true }`를 다음으로 교체:

```js
navOpen: { apim: true, kqlgroup: true, datainfra: true, lab: true }
```

- [ ] **Step 4: 패널 셸 HTML 추가**

`<section class="panel" id="panel-jira" data-panel>...</section>` 닫는 태그 직후, `</div>`(main div 닫힘) 앞에 추가:

```html
<section class="panel" id="panel-lab-overview" data-panel>
  <div class="panel-head">
    <span class="eyebrow">09 · Local Test Lab</span>
    <h1>랩 개요</h1>
    <p class="desc">Azure 실 접속 권한 없이도, 샘플 Kong 설정과 그 APIM 대응 정책을 로컬 mock 엔진으로 실행해 결과를 비교합니다. 이 페이지의 샘플 데이터를 아래 4개 탭이 공유합니다.</p>
  </div>
  <div id="labOverviewBody"></div>
</section>

<section class="panel" id="panel-lab-api-test" data-panel>
  <div class="panel-head">
    <span class="eyebrow">10 · Local Test Lab</span>
    <h1>API 테스트</h1>
    <p class="desc">샘플 라우트를 선택하고 값을 입력해 실행하면, 같은 요청에 대해 Kong mock과 APIM mock이 각각 어떻게 응답하는지 나란히 비교합니다.</p>
  </div>
  <div id="labApiTestBody"></div>
</section>

<section class="panel" id="panel-lab-policy" data-panel>
  <div class="panel-head">
    <span class="eyebrow">11 · Local Test Lab</span>
    <h1>APIM Policy</h1>
    <p class="desc">정책 XML을 직접 편집해서 실행해 보세요. 화이트리스트 7개 정책(check-header, rate-limit, cors, set-header, rewrite-uri, ip-filter, return-response)만 실제 동작을 시뮬레이션하며, 그 외 태그는 구문은 인정하되 "미지원" 배지로 표시합니다.</p>
  </div>
  <div id="labPolicyBody"></div>
</section>

<section class="panel" id="panel-lab-logs" data-panel>
  <div class="panel-head">
    <span class="eyebrow">12 · Local Test Lab</span>
    <h1>로그 비교</h1>
    <p class="desc">Kong/Nginx 로그 샘플을 보고 질문에 맞는 KQL(Kusto)을 직접 작성한 뒤 정답과 비교하세요. 기존 KQL 퀴즈와 달리 실제 로그 라인을 입력 맥락으로 사용합니다.</p>
  </div>
  <div id="labLogsBody"></div>
</section>

<section class="panel" id="panel-lab-migration" data-panel>
  <div class="panel-head">
    <span class="eyebrow">13 · Local Test Lab</span>
    <h1>마이그레이션</h1>
    <p class="desc">Kong→APIM 이관 시 놓치기 쉬운 함정을 시나리오로 예측해 보세요. "발생한다/안 한다"를 먼저 판단한 뒤 근거를 확인합니다.</p>
  </div>
  <div id="labMigrationBody"></div>
</section>
```

- [ ] **Step 5: 브라우저에서 확인**

파일 탐색기에서 `azure-migration-tracker.html`을 더블클릭해 브라우저로 연다.
확인 사항:
1. 좌측 네비게이션에 "로컬 테스트 랩" 그룹이 새로 보이고 펼쳐져 있다(다른 그룹처럼 기본 열림).
2. 그 아래 "랩 개요/API 테스트/APIM Policy/로그 비교/마이그레이션" 5개 항목이 보인다.
3. 각 항목을 클릭하면 해당 패널(제목·설명만 있고 본문은 비어있음)로 전환된다.
4. 기존 "구조 비교", "APIM 정책" 등 8개 항목을 클릭해도 기존과 동일하게 동작한다(회귀 없음).

---

## Task 2: 샘플 데이터 모델 + 랩 개요 패널

**Files:**
- Modify: `azure-migration-tracker.html` (JS, `buildFullQuiz("dataQuizList", ...)` 호출 다음 줄에 새 블록 추가)

**Interfaces:**
- Produces: `LAB_ROUTES` (배열), `renderLabOverview()` — Task 3·4·5가 `LAB_ROUTES`를, Task 4·5가 이 섹션에서 정의하는 카드 스타일을 재사용한다.

- [ ] **Step 1: `LAB_ROUTES` 데이터와 개요 렌더 함수 작성**

`buildFullQuiz("dataQuizList", DATA_INFRA, "data-full", "");` 다음 줄에 추가:

```js
  /* ---------------- 로컬 테스트 랩: 샘플 데이터 ---------------- */
  var LAB_ROUTES = [
    {
      id: "orders-get",
      label: "GET /orders — 조회 + 인증 + 요율 제한 + CORS",
      method: "GET", path: "/orders",
      inputs: ["apikey"],
      kongYaml:
        '_format_version: "3.0"\n' +
        'services:\n' +
        '  - name: orders-api\n' +
        '    url: http://orders-backend.internal\n' +
        '    routes:\n' +
        '      - name: orders-route\n' +
        '        paths: ["/orders"]\n' +
        '        methods: ["GET", "POST"]\n' +
        '    plugins:\n' +
        '      - name: key-auth\n' +
        '        config:\n' +
        '          key_names: ["apikey"]\n' +
        '      - name: rate-limiting\n' +
        '        config:\n' +
        '          minute: 5\n' +
        '          policy: local\n' +
        '      - name: cors\n' +
        '        config:\n' +
        '          origins: ["*"]\n' +
        '          methods: ["GET", "POST"]',
      kongPlugins: [
        { type: "key-auth", config: { validKey: "demo-key-123" } },
        { type: "rate-limiting", config: { limit: 5, windowSec: 60 } }
      ],
      apimPolicyXml:
        '<policies>\n' +
        '  <inbound>\n' +
        '    <base />\n' +
        '    <check-header name="apikey" failed-check-httpcode="401" failed-check-error-message="Unauthorized" ignore-case="false">\n' +
        '      <value>demo-key-123</value>\n' +
        '    </check-header>\n' +
        '    <rate-limit calls="5" renewal-period="60" />\n' +
        '    <cors allow-credentials="false">\n' +
        '      <allowed-origins><origin>*</origin></allowed-origins>\n' +
        '      <allowed-methods><method>GET</method><method>POST</method></allowed-methods>\n' +
        '      <allowed-headers><header>*</header></allowed-headers>\n' +
        '    </cors>\n' +
        '  </inbound>\n' +
        '  <backend><base /></backend>\n' +
        '  <outbound><base /></outbound>\n' +
        '</policies>',
      backendResponse: { orders: [{ id: 1, total: 42000 }, { id: 2, total: 15000 }] }
    },
    {
      id: "users-get",
      label: "GET /users/{id} — 헤더 추가 + IP 차단",
      method: "GET", path: "/users/{id}",
      inputs: ["clientIp"],
      kongYaml:
        '_format_version: "3.0"\n' +
        'services:\n' +
        '  - name: users-api\n' +
        '    url: http://users-backend.internal/v2/users\n' +
        '    routes:\n' +
        '      - name: users-route\n' +
        '        paths: ["/users"]\n' +
        '        strip_path: true\n' +
        '    plugins:\n' +
        '      - name: response-transformer\n' +
        '        config:\n' +
        '          add:\n' +
        '            headers: ["X-Backend:users-svc-v2"]\n' +
        '      - name: ip-restriction\n' +
        '        config:\n' +
        '          deny: ["10.0.0.99"]',
      kongPlugins: [
        { type: "ip-restriction", config: { deny: ["10.0.0.99"] } },
        { type: "response-transformer", config: { addHeader: { name: "X-Backend", value: "users-svc-v2" } } }
      ],
      apimPolicyXml:
        '<policies>\n' +
        '  <inbound>\n' +
        '    <base />\n' +
        '    <ip-filter action="forbid">\n' +
        '      <address>10.0.0.99</address>\n' +
        '    </ip-filter>\n' +
        '    <rewrite-uri template="/v2/users/{id}" />\n' +
        '  </inbound>\n' +
        '  <backend><base /></backend>\n' +
        '  <outbound>\n' +
        '    <base />\n' +
        '    <set-header name="X-Backend" exists-action="override">\n' +
        '      <value>users-svc-v2</value>\n' +
        '    </set-header>\n' +
        '  </outbound>\n' +
        '</policies>',
      backendResponse: { id: 1, name: "샘플 사용자" }
    },
    {
      id: "legacy-report",
      label: "GET /legacy/report — 즉시 차단(점검 중)",
      method: "GET", path: "/legacy/report",
      inputs: [],
      kongYaml:
        '_format_version: "3.0"\n' +
        'services:\n' +
        '  - name: legacy-api\n' +
        '    url: http://legacy-backend.internal/report\n' +
        '    routes:\n' +
        '      - name: legacy-route\n' +
        '        paths: ["/legacy/report"]\n' +
        '    plugins:\n' +
        '      - name: request-termination\n' +
        '        config:\n' +
        '          status_code: 503\n' +
        '          message: "이 엔드포인트는 점검 중입니다"',
      kongPlugins: [
        { type: "request-termination", config: { statusCode: 503, message: "이 엔드포인트는 점검 중입니다" } }
      ],
      apimPolicyXml:
        '<policies>\n' +
        '  <inbound>\n' +
        '    <base />\n' +
        '    <return-response>\n' +
        '      <set-status code="503" reason="Service Unavailable" />\n' +
        '      <set-body>이 엔드포인트는 점검 중입니다</set-body>\n' +
        '    </return-response>\n' +
        '  </inbound>\n' +
        '  <backend><base /></backend>\n' +
        '  <outbound><base /></outbound>\n' +
        '</policies>',
      backendResponse: { report: "ok" }
    }
  ];

  function renderLabOverview() {
    var body = document.getElementById("labOverviewBody");
    body.innerHTML = LAB_ROUTES.map(function (r) {
      return '<div class="pcard"><div class="pcard-head"><span class="pname"><code>' + r.method + " " + r.path + "</code> — " + r.label + '</span></div>' +
        '<p class="pwhen"><strong>Kong 설정(요약)</strong></p><div class="code-block">' + r.kongYaml + '</div>' +
        '<p class="pwhen" style="margin-top:var(--sp-3);"><strong>APIM 대응 정책</strong></p><div class="code-block">' + r.apimPolicyXml + '</div></div>';
    }).join("");
  }
  renderLabOverview();
```

- [ ] **Step 2: 브라우저에서 확인**

파일을 새로고침하고 "랩 개요" 탭을 연다.
확인: 3개 카드(`orders-api`, `users-api`, `legacy-api`)가 보이고, 각 카드에 Kong YAML과 APIM XML이 `<div class="code-block">`로 나란히 표시된다. 콘솔 에러가 없어야 한다(F12 개발자 도구 Console 탭 확인).

---

## Task 3: Mock 정책 실행 엔진

**Files:**
- Modify: `azure-migration-tracker.html` (JS, Task 2에서 추가한 `renderLabOverview();` 호출 다음 줄에 추가)

**Interfaces:**
- Consumes: `LAB_ROUTES` (Task 2)
- Produces: `runKongMock(route, input)`, `runApimMock(route, policyXml, input)`, `renderLabResult(container, label, result)` — 각각 `{statusCode:number, headers:object, body:object, trace:Array<{policy:string, effect:string}>}`를 반환/렌더링. `input` 셰이프: `{apikeyHeader:string, clientIp:string}`. Task 4·5가 이 세 함수를 호출한다.

- [ ] **Step 1: 카운터 저장소 + 슬라이딩 윈도우 함수**

```js
  /* ---------------- 로컬 테스트 랩: mock 엔진 ---------------- */
  var kongRateCounters = {};
  var apimRateCounters = {};

  function checkRateLimit(store, key, limit, windowSec) {
    var now = Date.now();
    var windowMs = windowSec * 1000;
    var timestamps = (store[key] || []).filter(function (t) { return now - t < windowMs; });
    if (timestamps.length >= limit) {
      store[key] = timestamps;
      return { allowed: false, remaining: 0 };
    }
    timestamps.push(now);
    store[key] = timestamps;
    return { allowed: true, remaining: limit - timestamps.length };
  }
```

- [ ] **Step 2: 브라우저 콘솔로 슬라이딩 윈도우 확인**

브라우저에서 파일을 열고 F12 콘솔에 아래를 붙여넣어 실행:

```js
var s = {};
for (var i = 0; i < 6; i++) console.log(i, checkRateLimit(s, "k", 5, 60));
```

기대 결과: 0~4번째 호출은 `allowed: true`, 5번째(6회째) 호출은 `allowed: false`.

- [ ] **Step 3: `runKongMock` 작성**

```js
  var KONG_PLUGIN_ORDER = ["key-auth", "ip-restriction", "rate-limiting", "response-transformer", "request-termination"];

  function runKongMock(route, input) {
    var trace = [];
    var plugins = route.kongPlugins.slice().sort(function (a, b) {
      return KONG_PLUGIN_ORDER.indexOf(a.type) - KONG_PLUGIN_ORDER.indexOf(b.type);
    });
    for (var i = 0; i < plugins.length; i++) {
      var p = plugins[i];
      if (p.type === "key-auth") {
        if ((input.apikeyHeader || "") !== p.config.validKey) {
          trace.push({ policy: "key-auth", effect: "차단 — apikey 헤더 없음/불일치 (401)" });
          return { statusCode: 401, headers: {}, body: { message: "Unauthorized" }, trace: trace };
        }
        trace.push({ policy: "key-auth", effect: "통과 — apikey 일치" });
      } else if (p.type === "ip-restriction") {
        if (p.config.deny.indexOf(input.clientIp) !== -1) {
          trace.push({ policy: "ip-restriction", effect: "차단 — " + input.clientIp + "는 차단 목록 (403)" });
          return { statusCode: 403, headers: {}, body: { message: "Forbidden" }, trace: trace };
        }
        trace.push({ policy: "ip-restriction", effect: "통과 — 차단 목록에 없음" });
      } else if (p.type === "rate-limiting") {
        var rl = checkRateLimit(kongRateCounters, route.id, p.config.limit, p.config.windowSec);
        if (!rl.allowed) {
          trace.push({ policy: "rate-limiting", effect: "차단 — " + p.config.limit + "회/" + p.config.windowSec + "초 초과 (429)" });
          return { statusCode: 429, headers: {}, body: { message: "Too Many Requests" }, trace: trace };
        }
        trace.push({ policy: "rate-limiting", effect: "통과 — 남은 호출 " + rl.remaining + "회" });
      } else if (p.type === "response-transformer") {
        trace.push({ policy: "response-transformer", effect: "응답 헤더 추가: " + p.config.addHeader.name + "=" + p.config.addHeader.value });
      } else if (p.type === "request-termination") {
        trace.push({ policy: "request-termination", effect: "즉시 종료 (" + p.config.statusCode + ")" });
        return { statusCode: p.config.statusCode, headers: {}, body: { message: p.config.message }, trace: trace };
      }
    }
    var headers = {};
    route.kongPlugins.forEach(function (p) {
      if (p.type === "response-transformer") headers[p.config.addHeader.name] = p.config.addHeader.value;
    });
    return { statusCode: 200, headers: headers, body: route.backendResponse, trace: trace };
  }
```

- [ ] **Step 4: 브라우저 콘솔로 `runKongMock` 확인**

```js
var orders = LAB_ROUTES[0];
console.log(runKongMock(orders, { apikeyHeader: "wrong", clientIp: "" }));      // 401 기대
console.log(runKongMock(orders, { apikeyHeader: "demo-key-123", clientIp: "" })); // 200 기대, trace에 rate-limiting 통과 기록
var users = LAB_ROUTES[1];
console.log(runKongMock(users, { apikeyHeader: "", clientIp: "10.0.0.99" }));   // 403 기대
console.log(runKongMock(users, { apikeyHeader: "", clientIp: "1.2.3.4" }));     // 200 + X-Backend 헤더 기대
```

- [ ] **Step 5: `parseApimPolicyXml`과 `runApimMock` 작성**

```js
  var APIM_WHITELIST = ["check-header", "rate-limit", "cors", "set-header", "rewrite-uri", "ip-filter", "return-response"];

  function parseApimPolicyXml(xmlString) {
    var doc = new DOMParser().parseFromString(xmlString, "text/xml");
    var errorNode = doc.querySelector("parsererror");
    if (errorNode) return { ok: false, error: errorNode.textContent.trim() };
    return { ok: true, doc: doc };
  }

  function runApimSection(elements, headers, trace, route, input, isInbound) {
    for (var i = 0; i < elements.length; i++) {
      var el = elements[i];
      var tag = el.tagName;
      if (tag === "base") continue;
      if (APIM_WHITELIST.indexOf(tag) === -1) {
        trace.push({ policy: tag, effect: "⚠ 이 학습 도구에서는 시뮬레이션 미지원 (구문은 유효)" });
        continue;
      }
      if (tag === "check-header" && isInbound) {
        var name = el.getAttribute("name");
        var httpcode = parseInt(el.getAttribute("failed-check-httpcode"), 10);
        var values = Array.prototype.map.call(el.querySelectorAll("value"), function (v) { return v.textContent; });
        var actual = name.toLowerCase() === "apikey" ? input.apikeyHeader : "";
        if (values.indexOf(actual) === -1) {
          trace.push({ policy: "check-header", effect: "차단 — " + name + " 값 불일치 (" + httpcode + ")" });
          return { blocked: true, statusCode: httpcode, body: { message: el.getAttribute("failed-check-error-message") } };
        }
        trace.push({ policy: "check-header", effect: "통과 — " + name + " 값 일치" });
      } else if (tag === "rate-limit" && isInbound) {
        var calls = parseInt(el.getAttribute("calls"), 10);
        var renewal = parseInt(el.getAttribute("renewal-period"), 10);
        var rl = checkRateLimit(apimRateCounters, route.id, calls, renewal);
        if (!rl.allowed) {
          trace.push({ policy: "rate-limit", effect: "차단 — " + calls + "회/" + renewal + "초 초과 (429)" });
          return { blocked: true, statusCode: 429, body: { message: "Too Many Requests" } };
        }
        trace.push({ policy: "rate-limit", effect: "통과 — 남은 호출 " + rl.remaining + "회" });
      } else if (tag === "ip-filter" && isInbound) {
        var action = el.getAttribute("action");
        var addresses = Array.prototype.map.call(el.querySelectorAll("address"), function (a) { return a.textContent; });
        var matched = addresses.indexOf(input.clientIp) !== -1;
        var isBlocked = action === "forbid" ? matched : !matched;
        if (isBlocked) {
          trace.push({ policy: "ip-filter", effect: "차단 — " + input.clientIp + " (action=" + action + ") (403)" });
          return { blocked: true, statusCode: 403, body: { message: "Forbidden" } };
        }
        trace.push({ policy: "ip-filter", effect: "통과 — action=" + action });
      } else if (tag === "rewrite-uri" && isInbound) {
        trace.push({ policy: "rewrite-uri", effect: "경로를 " + el.getAttribute("template") + "로 재작성" });
      } else if (tag === "cors" && isInbound) {
        trace.push({ policy: "cors", effect: "CORS 헤더 추가 (Access-Control-Allow-Origin 등)" });
      } else if (tag === "set-header") {
        var hName = el.getAttribute("name");
        var valueEl = el.querySelector("value");
        headers[hName] = valueEl ? valueEl.textContent : "";
        trace.push({ policy: "set-header", effect: hName + " = " + headers[hName] + (isInbound ? "" : " (outbound)") });
      } else if (tag === "return-response") {
        var statusEl = el.querySelector("set-status");
        var bodyEl = el.querySelector("set-body");
        var code = statusEl ? parseInt(statusEl.getAttribute("code"), 10) : 200;
        trace.push({ policy: "return-response", effect: "즉시 응답 반환 (" + code + ")" });
        return { blocked: true, statusCode: code, body: { message: bodyEl ? bodyEl.textContent : "" } };
      }
    }
    return { blocked: false };
  }

  function runApimMock(route, policyXml, input) {
    var trace = [];
    var headers = {};
    var parsed = parseApimPolicyXml(policyXml);
    if (!parsed.ok) {
      return { statusCode: 400, headers: {}, body: { message: "정책 XML 구문 오류: " + parsed.error }, trace: trace };
    }
    var inbound = parsed.doc.querySelector("policies > inbound");
    if (!inbound) {
      return { statusCode: 400, headers: {}, body: { message: "<policies><inbound> 요소를 찾을 수 없습니다" }, trace: trace };
    }
    var inResult = runApimSection(Array.prototype.slice.call(inbound.children), headers, trace, route, input, true);
    if (inResult.blocked) return { statusCode: inResult.statusCode, headers: headers, body: inResult.body, trace: trace };

    var outbound = parsed.doc.querySelector("policies > outbound");
    if (outbound) {
      var outResult = runApimSection(Array.prototype.slice.call(outbound.children), headers, trace, route, input, false);
      if (outResult.blocked) return { statusCode: outResult.statusCode, headers: headers, body: outResult.body, trace: trace };
    }
    return { statusCode: 200, headers: headers, body: route.backendResponse, trace: trace };
  }
```

- [ ] **Step 6: 브라우저 콘솔로 `runApimMock` 확인**

```js
console.log(runApimMock(orders, orders.apimPolicyXml, { apikeyHeader: "wrong", clientIp: "" }));         // 401 기대
console.log(runApimMock(orders, orders.apimPolicyXml, { apikeyHeader: "demo-key-123", clientIp: "" }));  // 200 기대
console.log(runApimMock(orders, "<policies><inbound><foo></inbound></policies>", {}));                   // 400 파싱 에러 기대(태그 안 닫힘)
console.log(runApimMock(users, users.apimPolicyXml, { apikeyHeader: "", clientIp: "10.0.0.99" }));       // 403 기대
console.log(runApimMock(users, users.apimPolicyXml, { apikeyHeader: "", clientIp: "1.2.3.4" }));         // 200 + X-Backend 헤더 기대
```

- [ ] **Step 7: 공유 렌더러 `renderLabResult` 작성**

```js
  function renderLabResult(container, label, result) {
    var statusClass = "lab-status-" + Math.floor(result.statusCode / 100) + "xx";
    var headerRows = Object.keys(result.headers).map(function (k) {
      return "<div>" + k + ": " + result.headers[k] + "</div>";
    }).join("");
    var traceRows = result.trace.map(function (t) {
      return '<div class="lab-trace-row"><code>' + t.policy + "</code> — " + t.effect + "</div>";
    }).join("");
    container.innerHTML =
      '<h3 class="lab-result-title">' + label + ' — <span class="' + statusClass + '">' + result.statusCode + "</span></h3>" +
      '<div class="code-block">' + JSON.stringify(result.body, null, 2) + "</div>" +
      (headerRows ? '<div class="lab-headers">' + headerRows + "</div>" : "") +
      '<div class="lab-trace">' + traceRows + "</div>";
  }
```

- [ ] **Step 8: 브라우저에서 최종 확인**

새로고침 후 F12 콘솔에서 Step 4, 6의 명령을 다시 실행해 전부 기대값과 일치하는지 확인하고, 콘솔에 에러가 없는지 확인한다. 아직 UI에는 연결하지 않았으므로 화면 변화는 없다.

---

## Task 4: "API 테스트" 탭 UI

**Files:**
- Modify: `azure-migration-tracker.html` (JS, Task 3의 `renderLabResult` 정의 다음 줄에 추가)

**Interfaces:**
- Consumes: `LAB_ROUTES`(Task 2), `runKongMock`/`runApimMock`/`renderLabResult`(Task 3)

- [ ] **Step 1: 라우트 선택 + 입력 + 실행 UI 작성**

```js
  /* ---------------- 로컬 테스트 랩: API 테스트 탭 ---------------- */
  function renderLabApiTest() {
    var body = document.getElementById("labApiTestBody");
    var tabsHtml = LAB_ROUTES.map(function (r, idx) {
      return '<button type="button" class="btn' + (idx === 0 ? " active" : "") + '" data-route-idx="' + idx + '">' + r.method + " " + r.path + "</button>";
    }).join("");
    body.innerHTML =
      '<div class="lab-route-tabs">' + tabsHtml + "</div>" +
      '<div id="labApiTestInputs"></div>' +
      '<button class="btn primary" type="button" id="labApiTestRun">실행</button>' +
      '<p class="pwhen" style="margin-top:var(--sp-2);"><strong>안내</strong>이 랩은 학습 편의를 위해 Kong 플러그인을 고정 순서(인증→IP제한→트래픽제어→변환→종료)로 실행합니다. 실제 Kong의 실행 순서는 각 플러그인의 priority 값에 따라 달라질 수 있습니다.</p>' +
      '<div class="lab-compare-grid" style="margin-top:var(--sp-4);"><div id="labApiTestKong" class="pcard"></div><div id="labApiTestApim" class="pcard"></div></div>';

    var selectedIdx = 0;

    function renderInputs() {
      var route = LAB_ROUTES[selectedIdx];
      var inputsBody = document.getElementById("labApiTestInputs");
      var html = '<div class="lab-input-row">';
      if (route.inputs.indexOf("apikey") !== -1) {
        html += '<label>apikey 헤더 값<input type="text" id="labInputApikey" placeholder="예: demo-key-123"></label>';
      }
      if (route.inputs.indexOf("clientIp") !== -1) {
        html += '<label>요청자 IP<input type="text" id="labInputIp" placeholder="예: 10.0.0.99"></label>';
      }
      if (route.inputs.length === 0) {
        html += '<span class="pwhen">이 라우트는 별도 입력 없이 항상 동일하게 동작합니다.</span>';
      }
      html += "</div>";
      inputsBody.innerHTML = html;
    }

    function currentInput() {
      var apikeyEl = document.getElementById("labInputApikey");
      var ipEl = document.getElementById("labInputIp");
      return { apikeyHeader: apikeyEl ? apikeyEl.value : "", clientIp: ipEl ? ipEl.value : "" };
    }

    body.querySelectorAll("[data-route-idx]").forEach(function (btn) {
      btn.addEventListener("click", function () {
        selectedIdx = parseInt(btn.dataset.routeIdx, 10);
        body.querySelectorAll("[data-route-idx]").forEach(function (b) { b.classList.toggle("active", b === btn); });
        renderInputs();
      });
    });

    document.getElementById("labApiTestRun").addEventListener("click", function () {
      var route = LAB_ROUTES[selectedIdx];
      var input = currentInput();
      renderLabResult(document.getElementById("labApiTestKong"), "Kong mock", runKongMock(route, input));
      renderLabResult(document.getElementById("labApiTestApim"), "APIM mock", runApimMock(route, route.apimPolicyXml, input));
    });

    renderInputs();
  }
  renderLabApiTest();
```

- [ ] **Step 2: 브라우저에서 확인 (시나리오 1 — key-auth 실패)**

"API 테스트" 탭 → `GET /orders` 선택(기본 선택) → apikey 입력란을 비운 채 "실행" 클릭.
기대 결과: Kong mock·APIM mock 모두 `401`, trace에 각각 `key-auth`/`check-header` 차단 사유 표시.

- [ ] **Step 3: 브라우저에서 확인 (시나리오 2 — rate-limit 429)**

apikey에 `demo-key-123` 입력 후 "실행"을 6번 연속 클릭(1분 이내).
기대 결과: 1~5번째 클릭은 Kong·APIM 모두 `200`, 6번째 클릭은 둘 다 `429`.

- [ ] **Step 4: 브라우저에서 확인 (시나리오 3 — IP 차단 + 헤더 반영)**

`GET /users/{id}` 라우트로 전환 → IP 입력란에 `10.0.0.99` 입력 → 실행(둘 다 `403` 기대) → IP를 `1.2.3.4`로 바꿔 다시 실행(둘 다 `200` + 응답에 `X-Backend: users-svc-v2` 헤더 표시 기대).

---

## Task 5: "APIM Policy" 탭 UI

**Files:**
- Modify: `azure-migration-tracker.html` (JS, Task 4의 `renderLabApiTest();` 호출 다음 줄에 추가)

**Interfaces:**
- Consumes: `LAB_ROUTES`(Task 2), `runApimMock`/`renderLabResult`(Task 3)

- [ ] **Step 1: 자유편집 XML 실행 UI 작성**

```js
  /* ---------------- 로컬 테스트 랩: APIM Policy 탭 ---------------- */
  function renderLabPolicy() {
    var body = document.getElementById("labPolicyBody");
    var tabsHtml = LAB_ROUTES.map(function (r, idx) {
      return '<button type="button" class="btn' + (idx === 0 ? " active" : "") + '" data-policy-route-idx="' + idx + '">' + r.method + " " + r.path + "</button>";
    }).join("");
    body.innerHTML =
      '<div class="lab-route-tabs">' + tabsHtml + "</div>" +
      '<textarea class="answer-in" id="labPolicyXmlInput" style="min-height:260px;font-family:var(--font-mono);"></textarea>' +
      '<div class="lab-input-row" style="margin-top:var(--sp-3);">' +
      '<label>apikey 헤더 값<input type="text" id="labPolicyApikey" placeholder="예: demo-key-123"></label>' +
      '<label>요청자 IP<input type="text" id="labPolicyIp" placeholder="예: 10.0.0.99"></label>' +
      "</div>" +
      '<button class="btn primary" type="button" id="labPolicyRun">실행</button>' +
      '<div id="labPolicyResult" class="pcard" style="margin-top:var(--sp-4);"></div>';

    var selectedIdx = 0;
    function loadXml() { document.getElementById("labPolicyXmlInput").value = LAB_ROUTES[selectedIdx].apimPolicyXml; }

    body.querySelectorAll("[data-policy-route-idx]").forEach(function (btn) {
      btn.addEventListener("click", function () {
        selectedIdx = parseInt(btn.dataset.policyRouteIdx, 10);
        body.querySelectorAll("[data-policy-route-idx]").forEach(function (b) { b.classList.toggle("active", b === btn); });
        loadXml();
      });
    });

    document.getElementById("labPolicyRun").addEventListener("click", function () {
      var route = LAB_ROUTES[selectedIdx];
      var xml = document.getElementById("labPolicyXmlInput").value;
      var input = {
        apikeyHeader: document.getElementById("labPolicyApikey").value,
        clientIp: document.getElementById("labPolicyIp").value
      };
      renderLabResult(document.getElementById("labPolicyResult"), "실행 결과", runApimMock(route, xml, input));
    });

    loadXml();
  }
  renderLabPolicy();
```

- [ ] **Step 2: 브라우저에서 확인 (정상 실행)**

"APIM Policy" 탭 → 기본 로드된 `orders-api` XML 그대로 두고 apikey에 `demo-key-123` 입력 → 실행.
기대 결과: `200` + trace에 `check-header`/`rate-limit`/`cors` 통과 기록.

- [ ] **Step 3: 브라우저에서 확인 (화이트리스트 밖 정책)**

텍스트 영역의 `<rate-limit .../>` 줄 위에 `<quota calls="1000" renewal-period="3600" />`를 추가해 실행.
기대 결과: 에러 없이 실행되고, trace에 `quota — ⚠ 이 학습 도구에서는 시뮬레이션 미지원 (구문은 유효)`가 표시된다.

- [ ] **Step 4: 브라우저에서 확인 (잘못된 XML)**

`</check-header>` 닫는 태그를 지우고 실행.
기대 결과: `400` + "정책 XML 구문 오류: ..." 메시지가 표시된다(조용히 무시되지 않음).

---

## Task 6: 로그 데이터셋 + "로그 비교" 탭 UI

**Files:**
- Modify: `azure-migration-tracker.html` (JS, Task 5의 `renderLabPolicy();` 호출 다음 줄에 추가)

**Interfaces:**
- Produces: `LAB_LOG_QUESTIONS` — Task 6 내부에서만 쓰임. 기존 `quiz-card`/`answer-in`/`answer-reveal` CSS 클래스를 재사용(새 CSS 불필요).

- [ ] **Step 1: 로그 샘플 + 질문 데이터**

```js
  /* ---------------- 로컬 테스트 랩: 로그 비교 탭 ---------------- */
  var LAB_KONG_LOG_SAMPLE =
    '{"client_ip":"203.0.113.10","method":"GET","request_uri":"/orders","status":200,"latencies":{"request":42},"started_at":1735600000}\n' +
    '{"client_ip":"203.0.113.11","method":"GET","request_uri":"/orders","status":429,"latencies":{"request":3},"started_at":1735600005}\n' +
    '{"client_ip":"203.0.113.10","method":"GET","request_uri":"/users/1","status":403,"latencies":{"request":5},"started_at":1735600010}\n' +
    '{"client_ip":"198.51.100.20","method":"GET","request_uri":"/legacy/report","status":503,"latencies":{"request":1},"started_at":1735600020}\n' +
    '{"client_ip":"203.0.113.10","method":"GET","request_uri":"/orders","status":200,"latencies":{"request":812},"started_at":1735600030}';

  var LAB_LOG_QUESTIONS = [
    {
      q: "Azure Monitor의 ApiManagementGatewayLogs 테이블에서, 최근 1일 동안 5xx 응답만 필터링하려면?",
      answerKql:
        "ApiManagementGatewayLogs\n" +
        "| where TimeGenerated > ago(1d)\n" +
        "| where ResponseCode >= 500\n" +
        "| project TimeGenerated, ApiId, Method, Url, ResponseCode, BackendResponseCode, CallerIpAddress",
      answerResult: "위 샘플 로그 기준: /legacy/report (503) 1건이 해당됩니다."
    },
    {
      q: "가장 느린 호출 top 3를 총 소요시간(TotalTime) 기준으로 뽑으려면?",
      answerKql:
        "ApiManagementGatewayLogs\n" +
        "| top 3 by TotalTime desc\n" +
        "| project TimeGenerated, ApiId, Url, TotalTime, BackendTime",
      answerResult: "위 샘플 로그 기준: /orders (812ms) 요청이 가장 느린 호출로 1위에 나와야 합니다."
    },
    {
      q: "호출자 IP(CallerIpAddress)별 요청 수를 집계해 많은 순으로 정렬하려면?",
      answerKql:
        "ApiManagementGatewayLogs\n" +
        "| summarize RequestCount = count() by CallerIpAddress\n" +
        "| sort by RequestCount desc",
      answerResult: "위 샘플 로그 기준: 203.0.113.10이 3건으로 1위입니다."
    }
  ];

  function renderLabLogs() {
    var body = document.getElementById("labLogsBody");
    var html = '<h2 class="section-title">Kong/Nginx 원본 로그 샘플</h2><div class="code-block">' + LAB_KONG_LOG_SAMPLE + "</div>";
    html += '<h2 class="section-title" style="margin-top:var(--sp-6);">질문</h2>';
    LAB_LOG_QUESTIONS.forEach(function (item, idx) {
      html += '<div class="quiz-card"><div class="quiz-head"><h3>질문 ' + (idx + 1) + '</h3></div><div class="quiz-body">' +
        '<p class="quiz-scenario">' + item.q + '</p>' +
        '<textarea class="answer-in" data-log-q="' + idx + '" placeholder="KQL을 직접 작성해 보세요"></textarea>' +
        '<div class="quiz-actions"><button class="btn primary" type="button" data-log-reveal="' + idx + '">정답 보기</button></div>' +
        '<div class="answer-reveal" id="labLogReveal' + idx + '"><span class="tag">정답 예시</span><div class="code-block">' + item.answerKql + '</div><p class="pwhen" style="margin-top:var(--sp-2);">' + item.answerResult + '</p></div>' +
        "</div></div>";
    });
    body.innerHTML = html;

    LAB_LOG_QUESTIONS.forEach(function (item, idx) {
      var ta = body.querySelector('[data-log-q="' + idx + '"]');
      var uid = "lab-log-" + idx;
      ta.value = state.quiz[uid] || "";
      ta.addEventListener("input", function () { state.quiz[uid] = ta.value; persist(); });
      body.querySelector('[data-log-reveal="' + idx + '"]').addEventListener("click", function () {
        document.getElementById("labLogReveal" + idx).classList.add("show");
      });
    });
  }
  renderLabLogs();
```

- [ ] **Step 2: 브라우저에서 확인**

"로그 비교" 탭을 연다. 원본 로그 샘플(5줄 JSON)이 code-block으로 보이고, 그 아래 질문 3개가 기존 퀴즈 카드와 동일한 스타일로 보인다. 텍스트 영역에 아무 텍스트나 입력 후 새로고침해 값이 유지되는지 확인(저장 가능 환경 기준), "정답 보기"를 누르면 정답 KQL과 해설이 펼쳐지는지 확인한다.

---

## Task 7: 마이그레이션 함정 카드 + "마이그레이션" 탭 UI

**Files:**
- Modify: `azure-migration-tracker.html` (JS, Task 6의 `renderLabLogs();` 호출 다음 줄에 추가)

**Interfaces:**
- Produces: `LAB_MIGRATION_PITFALLS` — Task 7 내부에서만 쓰임.

- [ ] **Step 1: 함정 시나리오 데이터 + 렌더**

```js
  /* ---------------- 로컬 테스트 랩: 마이그레이션 탭 ---------------- */
  var LAB_MIGRATION_PITFALLS = [
    {
      scenario: "Kong에서 쓰던 중간 인증서(intermediate certificate)를 그대로 Key Vault에 인증서로 등록하지 않고 리프 인증서만 올렸다. APIM 사용자 지정 도메인 연결이 실패할까?",
      answer: "예, 실패합니다.",
      reason: "Key Vault에는 리프 인증서뿐 아니라 전체 인증서 체인(중간 인증서 포함)이 포함된 .pfx를 등록해야 합니다. 체인이 빠지면 클라이언트가 인증서 신뢰 경로를 완성하지 못해 TLS 핸드셰이크가 실패할 수 있습니다."
    },
    {
      scenario: "Kong 플러그인에서 요청 헤더 이름을 `X-Request-ID`로 검사하도록 만들었는데, 클라이언트가 `x-request-id`(소문자)로 보냈다. Kong에서 정상 인식됐다면 APIM의 `check-header`에서도 그대로 정상 인식될까?",
      answer: "설정에 따라 다릅니다 — 기본값이 아닙니다.",
      reason: "HTTP 헤더 이름은 원래 대소문자 구분이 없지만, APIM의 check-header 정책은 `ignore-case` 속성이 필수이며 명시적으로 `true`로 설정하지 않으면 대소문자를 구분해 비교합니다. Kong 쪽 동작만 보고 그대로 옮기면 놓치기 쉬운 지점입니다."
    },
    {
      scenario: "Kong의 rate-limiting을 '분당 60회'로 쓰다가 APIM `rate-limit` 정책에 `calls=60, renewal-period=60`으로 그대로 옮겼다. 두 시스템의 카운팅 창(윈도우) 계산 방식이 완전히 동일할까?",
      answer: "아니요, 동일하지 않을 수 있습니다.",
      reason: "APIM classic 티어의 rate-limit은 슬라이딩 윈도우 방식이지만 분산 아키텍처 특성상 완전히 정확하지 않을 수 있다고 공식 문서에 명시되어 있고, v2 티어는 토큰 버킷 알고리즘을 사용해 classic과 계산 방식 자체가 다릅니다. Kong의 카운팅 정책(local/cluster/redis)과도 정확히 1:1로 일치한다고 가정하면 안 됩니다."
    },
    {
      scenario: "Application Gateway 뒤에 내부 APIM을 두었는데, App Gateway의 '호스트 이름 재정의'를 기본값(재정의함)으로 그대로 두고 이관했다. APIM이 요청을 거부할까?",
      answer: "예, 거부할 가능성이 높습니다.",
      reason: "App Gateway가 호스트 헤더를 재작성하면 APIM은 클라이언트가 원래 요청한 사용자 지정 도메인과 헤더가 다르다고 판단해 요청을 거부할 수 있습니다. '호스트 이름 재정의'를 '아니요'로 설정해 원래 호스트 헤더를 유지해야 합니다."
    },
    {
      scenario: "Kong의 CORS 플러그인은 프리플라이트(OPTIONS) 요청을 항상 200으로 응답하도록 커스텀했는데, APIM `cors` 정책을 기본값(`terminate-unmatched-request` 미설정)으로 이관했다. Origin이 허용 목록에 없는 프리플라이트 요청에서 Kong과 동일하게 동작할까?",
      answer: "예, 동일하게 동작합니다(우연히).",
      reason: "`terminate-unmatched-request`의 기본값이 false이므로, Origin이 매칭되지 않아도 요청은 계속 진행되어 결과적으로 200이 반환됩니다. 다만 이는 '의도한 동작'이 아니라 기본값이 우연히 비슷하게 보이는 것이라, 다른 시나리오(GET/HEAD 단순 요청)에서는 CORS 헤더 자체가 붙지 않는 차이가 생길 수 있습니다."
    },
    {
      scenario: "Kong 라우트가 `strip_path: true`로 `/api/v1` 접두사를 벗겨내고 백엔드로 전달했는데, APIM에서는 API의 '기본 경로(base path)' 설정을 그대로 유지한 채 백엔드 URL만 바꿔치기했다. 요청 경로가 깨질까?",
      answer: "예, 깨질 수 있습니다.",
      reason: "APIM은 API의 base path와 백엔드 URL의 경로를 조합해 최종 백엔드 요청 경로를 구성합니다. Kong의 strip_path 동작(접두사 제거)과 APIM의 경로 조합 규칙이 다르므로, `rewrite-uri` 정책으로 명시적으로 맞춰주지 않으면 백엔드가 기대하지 않는 경로로 요청이 전달될 수 있습니다."
    }
  ];

  function renderLabMigration() {
    var body = document.getElementById("labMigrationBody");
    body.innerHTML = LAB_MIGRATION_PITFALLS.map(function (item, idx) {
      return '<div class="quiz-card"><div class="quiz-head"><h3>시나리오 ' + (idx + 1) + '</h3></div><div class="quiz-body">' +
        '<p class="quiz-scenario">' + item.scenario + '</p>' +
        '<div class="quiz-actions">' +
        '<button class="btn" type="button" data-mig-guess="' + idx + '" data-guess="true">발생한다</button>' +
        '<button class="btn" type="button" data-mig-guess="' + idx + '" data-guess="false">발생하지 않는다</button>' +
        "</div>" +
        '<div class="answer-reveal" id="labMigReveal' + idx + '"><span class="tag">정답</span><p class="pwhen" style="margin-top:var(--sp-2);"><strong>' + item.answer + '</strong></p><p class="quiz-scenario">' + item.reason + "</p></div>" +
        "</div></div>";
    }).join("");

    LAB_MIGRATION_PITFALLS.forEach(function (item, idx) {
      body.querySelectorAll('[data-mig-guess="' + idx + '"]').forEach(function (btn) {
        btn.addEventListener("click", function () {
          document.getElementById("labMigReveal" + idx).classList.add("show");
        });
      });
    });
  }
  renderLabMigration();
```

- [ ] **Step 2: 브라우저에서 확인**

"마이그레이션" 탭을 연다. 6개 시나리오 카드가 보이고, "발생한다"/"발생하지 않는다" 버튼 중 아무거나 누르면 정답과 근거가 펼쳐지는지 확인한다.

---

## Task 8: 전체 회귀 확인

**Files:**
- 없음(코드 변경 없음, 최종 수동 QA)

- [ ] **Step 1: 신규 기능 전체 시나리오 재확인**

브라우저에서 `azure-migration-tracker.html`을 새로고침 후:
1. `orders-api`에 apikey 없이 호출 → API 테스트 탭에서 401 (Task 4 Step 2)
2. 60초 내 6회 연속 호출 → 5회까지 허용, 6회째 429 (Task 4 Step 3)
3. `users-api` 응답 헤더에 `X-Backend` 반영 확인 (Task 4 Step 4)
4. 화이트리스트 밖 정책(`<quota>`) 입력 시 미지원 배지 노출 확인 (Task 5 Step 3)
5. 잘못된 XML 입력 시 파싱 에러 문구 확인 (Task 5 Step 4)
6. 로그 비교 탭 질문 3개, 마이그레이션 탭 시나리오 6개가 모두 정상 표시·동작하는지 확인

- [ ] **Step 2: 기존 8개 패널 회귀 확인**

"구조 비교", "APIM 정책 레퍼런스", "정책 퀴즈", "KQL 레퍼런스", "KQL 퀴즈", "데이터·인프라 레퍼런스", "데이터·인프라 퀴즈", "AIDD·Jira" 8개 탭을 전부 한 번씩 클릭해 이전과 동일하게 렌더링되고 동작하는지 확인한다(특히 "정책 퀴즈"의 "정답 보기", "AIDD·Jira"의 티켓 초안 저장, 테마 토글).

- [ ] **Step 3: 콘솔 에러 최종 확인**

F12 개발자 도구 Console 탭을 열고 페이지를 새로고침 → 모든 탭을 한 번씩 순회하는 동안 에러(빨간색 로그)가 없는지 확인한다.
