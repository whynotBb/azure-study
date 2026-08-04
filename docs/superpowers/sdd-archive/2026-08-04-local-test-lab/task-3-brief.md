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

