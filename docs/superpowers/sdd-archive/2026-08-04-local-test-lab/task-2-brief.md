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

