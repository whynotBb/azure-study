# 트래커 시뮬레이터 자기모순 수정 (1군) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `index.html`의 APIM 정책 랩·rate-limit 시뮬레이션·KQL 엔진 3곳에서, 문서/레퍼런스가 가르치는 내용과 실제 손으로 확인되는 동작이 서로 모순되는 5건을 고쳐 시뮬레이터의 신뢰도를 높인다.

**Architecture:** 기존 화이트리스트 기반 mock 엔진(`runApimMock`/`runApimSection`/`runKongMock`, `checkRateLimit`, `kustoApplyWhere`)을 확장하는 방식으로 진행하며, 새 파일/새 IIFE를 만들지 않는다. KQL 엔진은 두 벌(단일 테이블용/조인 랩용, 서로 다른 IIFE)이 있으므로 동일한 종류의 확장을 각각 따로 적용한다.

**Tech Stack:** 순수 HTML/CSS/Vanilla JS 단일 파일. 빌드 도구·테스트 프레임워크 없음 — 검증은 Node `new Function()` 구문 체크 + Playwright(`C:\Users\whynot\browsertest\test-tracker-merge.js` 확장)로 대체한다.

## Global Constraints

- `docs/superpowers/specs/2026-08-12-tracker-fidelity-fixes-design.md`의 설계를 그대로 따른다.
- 새 정책/함수 문법은 실제 Azure/Kusto 문법이어야 한다(허구 문법 금지).
- **회귀 없음이 최우선 성공 기준**: "백엔드 강제 실패" 토글이 꺼져 있을 때(기본값) 기존 모든 라우트·정책·쿼리는 지금과 100% 동일하게 동작해야 한다.
- 각 Task 완료 후 Node로 두 `<script>` 블록 구문 검사를 통과해야 다음 Task로 진행한다.
- 커밋 메시지는 한국어로 작성한다.

---

### Task 1: APIM 정책 랩 — 강제 실패 토글 + backend/on-error 엔진

**Files:**
- Modify: `index.html` — `LAB_ROUTES` 배열(신규 4번째 라우트 추가), `runKongMock`, `runApimSection`/`runApimMock` 근처(신규 함수 2개 추가), `renderLabApiTest`/`renderLabPolicy`(입력 폼에 체크박스 추가), `panel-lab-policy`의 `.desc` 문구

**Interfaces:**
- Consumes: 기존 `runApimSection(elements, headers, trace, route, input, isInbound)`, `parseApimPolicyXml(xmlString)`, `renderLabResult(container, label, result)` — 시그니처 변경 없음.
- Produces: `runApimBackendSection(elements, input, trace)` → `{ failed: boolean, statusCode?: number }`, `runApimOnErrorSection(elements, trace, failStatus)` → `{ statusCode, body } | null`. `input` 객체에 `forceBackendFailure: boolean` 필드 추가 — 이후 Task 2도 동일 `input` 객체를 사용.

- [ ] **Step 1: LAB_ROUTES에 4번째 라우트(백엔드 재시도 예제) 추가**

`index.html`에서 `LAB_ROUTES` 배열의 마지막 원소(`legacy-report`) 뒤, 배열을 닫는 `];`(1839행 부근) 직전에 추가:

```js
    ,
    {
      id: "billing-get",
      label: "GET /billing — 백엔드 재시도 + 커스텀 오류 응답",
      method: "GET", path: "/billing",
      inputs: [],
      kongYaml:
        '_format_version: "3.0"\n' +
        'services:\n' +
        '  - name: billing-service\n' +
        '    url: http://billing.internal\n' +
        '    retries: 3\n' +
        '    routes:\n' +
        '      - name: billing-route\n' +
        '        paths: ["/billing"]',
      kongPlugins: [],
      apimPolicyXml:
        '<policies>\n' +
        '  <inbound><base /></inbound>\n' +
        '  <backend>\n' +
        '    <retry condition="@(context.Response.StatusCode == 503)" count="3" interval="2" max-interval="10" delta="2" first-fast-retry="true">\n' +
        '      <forward-request timeout="30" />\n' +
        '    </retry>\n' +
        '  </backend>\n' +
        '  <on-error>\n' +
        '    <return-response>\n' +
        '      <set-status code="503" reason="Service Unavailable" />\n' +
        '      <set-body>{"error":"billing-service 응답 없음 - 잠시 후 다시 시도하세요"}</set-body>\n' +
        '    </return-response>\n' +
        '  </on-error>\n' +
        '  <outbound><base /></outbound>\n' +
        '</policies>',
      backendResponse: { invoices: [{ id: 501, amount: 89000 }] }
    }
```

- [ ] **Step 2: runApimBackendSection / runApimOnErrorSection 함수 추가**

`runApimMock` 함수(`function runApimMock(route, policyXml, input) {`) 바로 위에 두 함수를 새로 추가:

```js
  function runApimBackendSection(elements, input, trace) {
    var forced = !!input.forceBackendFailure;
    var failStatus = 503;
    if (!forced) return { failed: false };

    var retryEl = null;
    for (var i = 0; i < elements.length; i++) {
      if (elements[i].tagName === "retry") { retryEl = elements[i]; break; }
    }
    if (!retryEl) {
      trace.push({ policy: "backend", effect: "백엔드 호출 실패 (" + failStatus + ")" });
      return { failed: true, statusCode: failStatus };
    }

    var count = parseInt(retryEl.getAttribute("count"), 10) || 1;
    var interval = parseInt(retryEl.getAttribute("interval"), 10) || 0;
    var condition = retryEl.getAttribute("condition") || "";
    var condMatch = condition.match(/context\.Response\.StatusCode\s*==\s*(\d+)/);
    if (condMatch) {
      if (parseInt(condMatch[1], 10) !== failStatus) {
        trace.push({ policy: "retry", effect: "조건(StatusCode==" + condMatch[1] + ") 불일치 — 재시도 미실행" });
        return { failed: true, statusCode: failStatus };
      }
    } else {
      trace.push({ policy: "retry", effect: "이 조건식은 시뮬레이션 대상이 아니며 항상 재시도합니다" });
    }
    for (var r = 1; r <= count; r++) {
      trace.push({ policy: "retry", effect: "재시도 " + r + "/" + count + " (" + interval + "초 대기 후) — 여전히 " + failStatus });
    }
    trace.push({ policy: "retry", effect: count + "회 재시도 소진 — 최종 실패" });
    return { failed: true, statusCode: failStatus };
  }

  function runApimOnErrorSection(elements, trace, failStatus) {
    for (var i = 0; i < elements.length; i++) {
      var el = elements[i];
      var tag = el.tagName;
      if (tag === "base") continue;
      if (tag === "return-response") {
        var statusEl = el.querySelector("set-status");
        var bodyEl = el.querySelector("set-body");
        var code = statusEl ? parseInt(statusEl.getAttribute("code"), 10) : failStatus;
        trace.push({ policy: "on-error/return-response", effect: "커스텀 오류 응답 반환 (" + code + ")" });
        return { statusCode: code, body: { message: bodyEl ? bodyEl.textContent : "" } };
      }
      trace.push({ policy: "on-error/" + tag, effect: "⚠ 이 학습 도구에서는 시뮬레이션 미지원 (구문은 유효)" });
    }
    return null;
  }

```

- [ ] **Step 3: runApimMock에 backend/on-error 단계 연결**

`function runApimMock(route, policyXml, input) {` 내부, 기존:

```js
    var inResult = runApimSection(Array.prototype.slice.call(inbound.children), headers, trace, route, input, true);
    if (inResult.blocked) return { statusCode: inResult.statusCode, headers: headers, body: inResult.body, trace: trace };

    var outbound = parsed.doc.querySelector("policies > outbound");
```

를 다음으로 교체(중간에 backend 단계 삽입):

```js
    var inResult = runApimSection(Array.prototype.slice.call(inbound.children), headers, trace, route, input, true);
    if (inResult.blocked) return { statusCode: inResult.statusCode, headers: headers, body: inResult.body, trace: trace };

    var backend = parsed.doc.querySelector("policies > backend");
    var backendResult = runApimBackendSection(backend ? Array.prototype.slice.call(backend.children) : [], input, trace);
    if (backendResult.failed) {
      var onError = parsed.doc.querySelector("policies > on-error");
      var overridden = onError ? runApimOnErrorSection(Array.prototype.slice.call(onError.children), trace, backendResult.statusCode) : null;
      if (overridden) return { statusCode: overridden.statusCode, headers: headers, body: overridden.body, trace: trace };
      return { statusCode: backendResult.statusCode, headers: headers, body: { message: "Backend call failed" }, trace: trace };
    }

    var outbound = parsed.doc.querySelector("policies > outbound");
```

- [ ] **Step 4: runKongMock에도 강제 실패 반영**

`function runKongMock(route, input) {` 마지막 부분, 기존:

```js
    var headers = {};
    route.kongPlugins.forEach(function (p) {
      if (p.type === "response-transformer") headers[p.config.addHeader.name] = p.config.addHeader.value;
    });
    return { statusCode: 200, headers: headers, body: route.backendResponse, trace: trace };
  }
```

를 다음으로 교체:

```js
    var headers = {};
    route.kongPlugins.forEach(function (p) {
      if (p.type === "response-transformer") headers[p.config.addHeader.name] = p.config.addHeader.value;
    });
    if (input.forceBackendFailure) {
      trace.push({ policy: "backend", effect: "백엔드 강제 실패 시뮬레이션 (503) — Kong의 재시도는 Service의 retries 필드로 별도 설정되며, 이 랩은 그 재시도 동작 자체는 시뮬레이션하지 않습니다" });
      return { statusCode: 503, headers: headers, body: { message: "Service Unavailable" }, trace: trace };
    }
    return { statusCode: 200, headers: headers, body: route.backendResponse, trace: trace };
  }
```

- [ ] **Step 5: API 테스트 탭에 체크박스 추가**

`renderLabApiTest` 함수 내부 `renderInputs`/`currentInput`, 기존:

```js
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
```

를 다음으로 교체:

```js
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
        html += '<span class="pwhen">이 라우트는 apikey/IP 입력이 필요 없습니다.</span>';
      }
      html += '<label><input type="checkbox" id="labInputForceFail"> 백엔드 강제 실패(503)</label>';
      html += "</div>";
      inputsBody.innerHTML = html;
    }

    function currentInput() {
      var apikeyEl = document.getElementById("labInputApikey");
      var ipEl = document.getElementById("labInputIp");
      var forceFailEl = document.getElementById("labInputForceFail");
      return { apikeyHeader: apikeyEl ? apikeyEl.value : "", clientIp: ipEl ? ipEl.value : "", forceBackendFailure: forceFailEl ? forceFailEl.checked : false };
    }
```

- [ ] **Step 6: APIM Policy 탭에 체크박스 추가**

`renderLabPolicy` 함수 내부, 기존:

```js
      '<div class="lab-input-row" style="margin-top:var(--sp-3);">' +
      '<label>apikey 헤더 값<input type="text" id="labPolicyApikey" placeholder="예: demo-key-123"></label>' +
      '<label>요청자 IP<input type="text" id="labPolicyIp" placeholder="예: 10.0.0.99"></label>' +
      "</div>" +
```

를 다음으로 교체:

```js
      '<div class="lab-input-row" style="margin-top:var(--sp-3);">' +
      '<label>apikey 헤더 값<input type="text" id="labPolicyApikey" placeholder="예: demo-key-123"></label>' +
      '<label>요청자 IP<input type="text" id="labPolicyIp" placeholder="예: 10.0.0.99"></label>' +
      '<label><input type="checkbox" id="labPolicyForceFail"> 백엔드 강제 실패(503)</label>' +
      "</div>" +
```

그리고 같은 함수의 실행 핸들러, 기존:

```js
    document.getElementById("labPolicyRun").addEventListener("click", function () {
      var route = LAB_ROUTES[selectedIdx];
      var xml = document.getElementById("labPolicyXmlInput").value;
      var input = {
        apikeyHeader: document.getElementById("labPolicyApikey").value,
        clientIp: document.getElementById("labPolicyIp").value
      };
      renderLabResult(document.getElementById("labPolicyResult"), "실행 결과", runApimMock(route, xml, input));
    });
```

를 다음으로 교체:

```js
    document.getElementById("labPolicyRun").addEventListener("click", function () {
      var route = LAB_ROUTES[selectedIdx];
      var xml = document.getElementById("labPolicyXmlInput").value;
      var input = {
        apikeyHeader: document.getElementById("labPolicyApikey").value,
        clientIp: document.getElementById("labPolicyIp").value,
        forceBackendFailure: document.getElementById("labPolicyForceFail").checked
      };
      renderLabResult(document.getElementById("labPolicyResult"), "실행 결과", runApimMock(route, xml, input));
    });
```

- [ ] **Step 7: 패널 설명 문구 정정**

`panel-lab-policy`의 `.desc` 문구:

Before: `정책 XML을 직접 편집해서 실행해 보세요. 화이트리스트 7개 정책(check-header, rate-limit, cors, set-header, rewrite-uri, ip-filter, return-response)만 실제 동작을 시뮬레이션하며, 그 외 태그는 구문은 인정하되 "미지원" 배지로 표시합니다.`

After: `정책 XML을 직접 편집해서 실행해 보세요. 화이트리스트 8개 정책(check-header, rate-limit, rate-limit-by-key, cors, set-header, rewrite-uri, ip-filter, return-response)과 backend 섹션의 retry, on-error 섹션의 return-response를 시뮬레이션하며, 그 외 태그는 구문은 인정하되 "미지원" 배지로 표시합니다.`

- [ ] **Step 8: 구문 검사**

Run: 두 `<script>` 블록에 대해 Node `new Function(scriptText)` 실행(기존 세션에서 쓰던 방식 그대로).
Expected: `block 0: OK`, `block 1: OK`.

- [ ] **Step 9: 수동/Playwright 확인**

- API 테스트 탭에서 `billing-get` 라우트 선택, "백엔드 강제 실패" 체크 후 실행 → APIM mock 트레이스에 재시도 3회 + 최종 커스텀 응답(503, `billing-service 응답 없음...`)이 보이는지, Kong mock은 재시도 표시 없이 즉시 503인지 확인.
- 체크박스를 끈 상태로 기존 3개 라우트(`orders-get`/`users-get`/`legacy-report`)를 API 테스트 탭·APIM Policy 탭 양쪽에서 실행해 이전과 동일한 결과가 나오는지 확인(회귀 없음).

- [ ] **Step 10: 커밋**

```bash
git add index.html
git commit -m "feat: APIM 정책 랩에 backend/on-error 실행과 강제 실패 토글 추가"
```

---

### Task 2: Rate-limit 정확도 — counter-key 파티셔닝 + 고정 윈도우

**Files:**
- Modify: `index.html` — `checkRateLimit` 근처(신규 함수 추가), `runApimSection`의 rate-limit 분기

**Interfaces:**
- Consumes: Task 1에서 확정된 `input.forceBackendFailure`와 무관 — 이 Task는 `input.clientIp`만 사용(기존 필드, 변경 없음).
- Produces: `checkRateLimitFixedWindow(store, key, limit, windowSec)` → `{ allowed: boolean, remaining: number }` — Task 3·4와 무관, 이후 Task에서 참조되지 않음.

- [ ] **Step 1: checkRateLimitFixedWindow 함수 추가**

`function checkRateLimit(store, key, limit, windowSec) {` 함수 바로 뒤에 추가:

```js

  function checkRateLimitFixedWindow(store, key, limit, windowSec) {
    var windowMs = windowSec * 1000;
    var bucket = Math.floor(Date.now() / windowMs);
    var bucketKey = key + "#" + bucket;
    var count = store[bucketKey] || 0;
    if (count >= limit) {
      return { allowed: false, remaining: 0 };
    }
    store[bucketKey] = count + 1;
    return { allowed: true, remaining: limit - (count + 1) };
  }
```

- [ ] **Step 2: rate-limit/rate-limit-by-key 분기를 counter-key + 고정 윈도우로 교체**

`runApimSection` 내부, 기존:

```js
      } else if (tag === "rate-limit" || tag === "rate-limit-by-key") {
        if (!isInbound) {
          trace.push({ policy: tag, effect: "이 섹션(" + (isInbound ? "inbound" : "outbound") + ")에서는 시뮬레이션 대상이 아님" });
        } else {
          var calls = parseInt(el.getAttribute("calls"), 10);
          var renewal = parseInt(el.getAttribute("renewal-period"), 10);
          var rl = checkRateLimit(apimRateCounters, route.id, calls, renewal);
          if (!rl.allowed) {
            trace.push({ policy: tag, effect: "차단 — " + calls + "회/" + renewal + "초 초과 (429)" });
            return { blocked: true, statusCode: 429, body: { message: "Too Many Requests" } };
          }
          trace.push({ policy: tag, effect: "통과 — 남은 호출 " + rl.remaining + "회" });
        }
      } else if (tag === "ip-filter") {
```

를 다음으로 교체:

```js
      } else if (tag === "rate-limit" || tag === "rate-limit-by-key") {
        if (!isInbound) {
          trace.push({ policy: tag, effect: "이 섹션(" + (isInbound ? "inbound" : "outbound") + ")에서는 시뮬레이션 대상이 아님" });
        } else {
          var calls = parseInt(el.getAttribute("calls"), 10);
          var renewal = parseInt(el.getAttribute("renewal-period"), 10);
          var counterKey = route.id;
          if (tag === "rate-limit-by-key") {
            var counterKeyAttr = el.getAttribute("counter-key");
            if (counterKeyAttr === "@(context.Request.IpAddress)") {
              counterKey = route.id + "|" + (input.clientIp || "");
            } else {
              trace.push({ policy: tag, effect: "이 랩은 counter-key로 @(context.Request.IpAddress)만 지원합니다 — route 단위 카운터로 대체" });
            }
          }
          var rl = checkRateLimitFixedWindow(apimRateCounters, counterKey, calls, renewal);
          if (!rl.allowed) {
            trace.push({ policy: tag, effect: "차단 — " + calls + "회/" + renewal + "초 초과 (429, 고정 윈도우 — 실제 v2 토큰 버킷의 단순화)" });
            return { blocked: true, statusCode: 429, body: { message: "Too Many Requests" } };
          }
          trace.push({ policy: tag, effect: "통과 — 남은 호출 " + rl.remaining + "회 (고정 윈도우 — 실제 v2 토큰 버킷의 단순화)" });
        }
      } else if (tag === "ip-filter") {
```

- [ ] **Step 3: 구문 검사**

Run: Node `new Function()` 구문 체크.
Expected: `block 0: OK`, `block 1: OK`.

- [ ] **Step 4: 수동/Playwright 확인**

- API 테스트 탭에서 `orders-get`(`rate-limit-by-key counter-key="@(context.Request.IpAddress)"` 포함) 선택, 서로 다른 IP 두 개(예: `203.0.113.10`, `203.0.113.11`)로 각각 5회 초과 호출 → 각 IP의 카운터가 독립적으로 429가 뜨는지 확인(한쪽 IP가 이미 차단돼도 다른 IP는 통과).
- "카운터 초기화" 버튼이 여전히 정상 동작하는지 확인.

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "fix: rate-limit-by-key counter-key 반영 및 APIM 고정 윈도우 알고리즘 분리"
```

---

### Task 3: KQL where 절 파서 확장 — 단일 테이블 엔진

**Files:**
- Modify: `index.html` — 단일 테이블 Kusto 엔진의 `kustoParseLiteral`, `kustoApplyWhere` (약 2299-2330행대, `runKustoQuery` 함수보다 앞쪽)

**Interfaces:**
- Consumes: 없음(독립적인 파서 확장).
- Produces: `SIMULATED_NOW`(단일 테이블 엔진 전용, 값 `"2026-08-06T09:15:00Z"`), `kustoAgoValue(amount, unit)`, `KUSTO_CAST_FUNCS` — 전부 이 IIFE 스코프 안에서만 쓰이며 Task 4의 조인 랩 엔진과는 별개(이름은 같아도 다른 스코프).

- [ ] **Step 1: kustoParseLiteral에 ago()/now() 지원 추가**

단일 테이블 엔진의 기존:

```js
  function kustoParseLiteral(raw) {
    raw = raw.trim();
    if (raw === "true") return true;
    if (raw === "false") return false;
    if (/^-?\d+(\.\d+)?$/.test(raw)) return Number(raw);
    var strMatch = raw.match(/^"(.*)"$/);
    if (strMatch) return strMatch[1];
    throw new Error("문자열 값은 큰따옴표로 감싸야 합니다: " + raw);
  }
```

를 다음으로 교체(바로 앞에 상수·헬퍼 함수 추가):

```js
  var SIMULATED_NOW = "2026-08-06T09:15:00Z";

  function kustoAgoValue(amount, unit) {
    var ms = amount * (unit === "d" ? 86400000 : unit === "h" ? 3600000 : unit === "m" ? 60000 : 1000);
    return new Date(new Date(SIMULATED_NOW).getTime() - ms).toISOString();
  }

  function kustoParseLiteral(raw) {
    raw = raw.trim();
    if (raw === "true") return true;
    if (raw === "false") return false;
    if (raw === "now()") return SIMULATED_NOW;
    var agoMatch = raw.match(/^ago\(\s*(\d+)(s|m|h|d)\s*\)$/);
    if (agoMatch) return kustoAgoValue(Number(agoMatch[1]), agoMatch[2]);
    if (/^-?\d+(\.\d+)?$/.test(raw)) return Number(raw);
    var strMatch = raw.match(/^"(.*)"$/);
    if (strMatch) return strMatch[1];
    throw new Error("문자열 값은 큰따옴표로 감싸야 합니다: " + raw);
  }
```

- [ ] **Step 2: kustoApplyWhere에 함수 호출(toint 등) 지원 추가**

단일 테이블 엔진의 기존:

```js
  function kustoApplyWhere(rows, clause) {
    var conditions = clause.split(/\s+and\s+/i).map(function (c) {
      var m = c.match(/^(\w+)\s*(==|!=|>=|<=|>|<|!contains|contains)\s*(.+)$/);
      if (!m) throw new Error("조건을 해석할 수 없습니다: " + c);
      return { col: m[1], op: m[2], val: kustoParseLiteral(m[3]) };
    });
    return rows.filter(function (row) {
      return conditions.every(function (c) { return kustoCompare(row[c.col], c.op, c.val); });
    });
  }
```

를 다음으로 교체:

```js
  var KUSTO_CAST_FUNCS = { toint: Number, tostring: String, todouble: Number, tolong: Number };

  function kustoApplyWhere(rows, clause) {
    var conditions = clause.split(/\s+and\s+/i).map(function (c) {
      var m = c.match(/^(?:(\w+)\((\w+)\)|(\w+))\s*(==|!=|>=|<=|>|<|!contains|contains)\s*(.+)$/);
      if (!m) throw new Error("조건을 해석할 수 없습니다: " + c);
      var fn = m[1], col = m[1] ? m[2] : m[3], op = m[4], val = m[5];
      if (fn && !KUSTO_CAST_FUNCS[fn]) throw new Error("지원하지 않는 함수: " + fn + "()");
      return { col: col, fn: fn ? KUSTO_CAST_FUNCS[fn] : null, op: op, val: kustoParseLiteral(val) };
    });
    return rows.filter(function (row) {
      return conditions.every(function (c) {
        var actual = c.fn ? c.fn(row[c.col]) : row[c.col];
        return kustoCompare(actual, c.op, c.val);
      });
    });
  }
```

- [ ] **Step 3: 구문 검사**

Run: Node `new Function()` 구문 체크.
Expected: `block 0: OK`, `block 1: OK`.

- [ ] **Step 4: 기존 예제 회귀 확인**

"Kusto 실행기" 탭의 기존 프리셋/예제 쿼리를 전부 실행해 이전과 동일한 결과가 나오는지 확인.

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "feat: 단일 테이블 Kusto 엔진에 함수 호출과 ago()/now() 지원 추가"
```

---

### Task 4: KQL where 절 파서 확장 — 조인 랩 엔진

**Files:**
- Modify: `index.html` — KQL 조인 랩 엔진의 `kustoParseLiteral`, `kustoApplyWhere` (별도 IIFE, 약 3321행대)

**Interfaces:**
- Consumes: 없음.
- Produces: Task 3과 이름은 동일하지만 별개 스코프인 `SIMULATED_NOW`(값 `"2026-08-11T09:15:00Z"`), `kustoAgoValue`, `KUSTO_CAST_FUNCS`.

- [ ] **Step 1: kustoParseLiteral에 ago()/now() 지원 추가**

조인 랩 엔진의 기존(단일 테이블 엔진과 동일한 형태):

```js
  function kustoParseLiteral(raw) {
    raw = raw.trim();
    if (raw === "true") return true;
    if (raw === "false") return false;
    if (/^-?\d+(\.\d+)?$/.test(raw)) return Number(raw);
    var strMatch = raw.match(/^"(.*)"$/);
    if (strMatch) return strMatch[1];
    throw new Error("문자열 값은 큰따옴표로 감싸야 합니다: " + raw);
  }
```

를 다음으로 교체(이 IIFE의 `SIMULATED_NOW`는 조인 랩 fixture 날짜 2026-08-11에 맞춘 값 사용):

```js
  var SIMULATED_NOW = "2026-08-11T09:15:00Z";

  function kustoAgoValue(amount, unit) {
    var ms = amount * (unit === "d" ? 86400000 : unit === "h" ? 3600000 : unit === "m" ? 60000 : 1000);
    return new Date(new Date(SIMULATED_NOW).getTime() - ms).toISOString();
  }

  function kustoParseLiteral(raw) {
    raw = raw.trim();
    if (raw === "true") return true;
    if (raw === "false") return false;
    if (raw === "now()") return SIMULATED_NOW;
    var agoMatch = raw.match(/^ago\(\s*(\d+)(s|m|h|d)\s*\)$/);
    if (agoMatch) return kustoAgoValue(Number(agoMatch[1]), agoMatch[2]);
    if (/^-?\d+(\.\d+)?$/.test(raw)) return Number(raw);
    var strMatch = raw.match(/^"(.*)"$/);
    if (strMatch) return strMatch[1];
    throw new Error("문자열 값은 큰따옴표로 감싸야 합니다: " + raw);
  }
```

- [ ] **Step 2: kustoApplyWhere에 함수 호출 지원 추가 (기존 isnull/isnotempty·or 미지원은 유지)**

조인 랩 엔진의 기존:

```js
  function kustoApplyWhere(rows, clause) {
    if (/\bor\b/i.test(clause)) throw new Error("이 시뮬레이터는 where 절에서 'or'를 지원하지 않습니다(and만 지원). 여러 조건은 개별 where 스테이지로 나눠보세요.");
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

를 다음으로 교체(unary 마커와 cast 함수 마커를 분리해 기존 isnull/isnotempty 동작을 그대로 보존):

```js
  var KUSTO_CAST_FUNCS = { toint: Number, tostring: String, todouble: Number, tolong: Number };

  function kustoApplyWhere(rows, clause) {
    if (/\bor\b/i.test(clause)) throw new Error("이 시뮬레이터는 where 절에서 'or'를 지원하지 않습니다(and만 지원). 여러 조건은 개별 where 스테이지로 나눠보세요.");
    var conditions = clause.split(/\s+and\s+/i).map(function (c) {
      var unary = c.match(/^(isnull|isnotempty)\((\w+)\)$/);
      if (unary) return { unaryFn: unary[1], col: unary[2] };
      var m = c.match(/^(?:(\w+)\((\w+)\)|(\w+))\s*(==|!=|>=|<=|>|<|!contains|contains)\s*(.+)$/);
      if (!m) throw new Error("조건을 해석할 수 없습니다: " + c);
      var castFn = m[1], col = m[1] ? m[2] : m[3], op = m[4], val = m[5];
      if (castFn && !KUSTO_CAST_FUNCS[castFn]) throw new Error("지원하지 않는 함수: " + castFn + "()");
      return { col: col, castFn: castFn ? KUSTO_CAST_FUNCS[castFn] : null, op: op, val: kustoParseLiteral(val) };
    });
    return rows.filter(function (row) {
      return conditions.every(function (c) {
        if (c.unaryFn === "isnull") return row[c.col] === null || row[c.col] === undefined;
        if (c.unaryFn === "isnotempty") return row[c.col] !== null && row[c.col] !== undefined && row[c.col] !== "";
        var actual = c.castFn ? c.castFn(row[c.col]) : row[c.col];
        return kustoCompare(actual, c.op, c.val);
      });
    });
  }
```

- [ ] **Step 3: 구문 검사**

Run: Node `new Function()` 구문 체크.
Expected: `block 0: OK`, `block 1: OK`.

- [ ] **Step 4: 기존 회귀 + 신규 케이스 확인**

- KQL 조인 랩의 튜토리얼 6단계 + 예제 17개(A1~C2)를 전부 실행해 이전과 동일한 결과가 나오는지 확인(순수 확장이라 회귀 없어야 함).
- `KQL_QUIZ`(트래커의 "KQL 퀴즈" 패널) 1번 정답 `AppRequests\n| where toint(ResultCode) >= 500\n| where AppRoleName == "payment-api"`와 2번 정답 `| where TimeGenerated > ago(1h)`(앞에 `AppRequests`를 붙여서)를 조인 랩 편집기에 그대로 입력해 파싱 에러 없이 실행되는지 확인.

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "feat: KQL 조인 랩 엔진에 함수 호출과 ago()/now() 지원 추가"
```

---

### Task 5: 문구 정정

**Files:**
- Modify: `index.html` — `AZURE_MONITOR_REF`의 join 설명, `DATA_INFRA`의 관리 ID 항목

**Interfaces:**
- Consumes/Produces: 없음(순수 텍스트 수정, Task 1-4와 독립).

- [ ] **Step 1: join 미지원 문구를 조인 랩 존재에 맞게 정정**

`AZURE_MONITOR_REF`에서 기존:

```js
        easy: "OperationId 같은 공통 키로 두 테이블(예: AppRequests와 AppExceptions)을 연결합니다. 이 프로젝트의 로컬 Kusto 실행기는 join을 지원하지 않습니다 — 정확히 재현하기 어려워 화이트리스트에서 제외했습니다.",
```

를 다음으로 교체:

```js
        easy: "OperationId 같은 공통 키로 두 테이블(예: AppRequests와 AppExceptions)을 연결합니다. 이 페이지의 단일 테이블 Kusto 실행기(\"Kusto 실행기\" 탭)는 join을 지원하지 않지만, 여러 테이블 join은 \"KQL 조인 랩\" 탭에서 실습할 수 있습니다.",
```

- [ ] **Step 2: 관리 ID 항목에 Consumption 티어 캐비아트 추가**

`DATA_INFRA`의 "관리 ID(Managed Identity)" 항목에서 기존:

```js
      { name: "관리 ID(Managed Identity)", code: "Managed Identity", url: "https://learn.microsoft.com/ko-kr/azure/active-directory/managed-identities-azure-resources/overview",
        easy: "Azure 리소스끼리 비밀번호 없이 서로 인증할 수 있게 해주는 자동 발급 신원입니다. APIM이 Key Vault나 Storage에 접근할 때 별도 자격 증명을 코드에 넣지 않고 이 관리 ID로 인증합니다.",
        when: "APIM/App Gateway가 Key Vault, Storage 같은 다른 Azure 리소스에 접근해야 할 때",
        example: "az apim update --resource-group rg-migration --name my-apim --enable-managed-identity true" }
```

를 다음으로 교체:

```js
      { name: "관리 ID(Managed Identity)", code: "Managed Identity", url: "https://learn.microsoft.com/ko-kr/azure/active-directory/managed-identities-azure-resources/overview",
        easy: "Azure 리소스끼리 비밀번호 없이 서로 인증할 수 있게 해주는 자동 발급 신원입니다. APIM이 Key Vault나 Storage에 접근할 때 별도 자격 증명을 코드에 넣지 않고 이 관리 ID로 인증합니다. --enable-managed-identity 플래그는 Consumption 티어 전용이며, 다른 티어(Developer/Basic/Standard/Premium 등)에서는 az apim update ... --set identity.type=SystemAssigned를 사용합니다.",
        when: "APIM/App Gateway가 Key Vault, Storage 같은 다른 Azure 리소스에 접근해야 할 때",
        example: "az apim update --resource-group rg-migration --name my-apim --enable-managed-identity true" }
```

- [ ] **Step 3: 구문 검사**

Run: Node `new Function()` 구문 체크.
Expected: `block 0: OK`, `block 1: OK`.

- [ ] **Step 4: 커밋**

```bash
git add index.html
git commit -m "docs: join 미지원 문구 정정 및 관리 ID 티어 캐비아트 추가"
```

---

### Task 6: 통합 검증

**Files:**
- Modify: `C:\Users\whynot\browsertest\test-tracker-merge.js`(새 시나리오 추가) 또는 신규 스크립트 `test-fidelity-fixes.js`
- 코드 변경 없음(`index.html`은 이미 Task 1-5에서 완료)

**Interfaces:**
- Consumes: Task 1-5에서 완성된 모든 기능.
- Produces: 없음(검증 결과만 보고).

- [ ] **Step 1: 새 Playwright 시나리오 스크립트 작성**

`C:\Users\whynot\browsertest\test-fidelity-fixes.js` 생성:

```js
const { chromium } = require('playwright');
const path = require('path');

function fileUrl(p) {
  const abs = path.resolve(p).replace(/\\/g, '/');
  return 'file:///' + abs.split('/').map(encodeURIComponent).join('/');
}

(async () => {
  const url = fileUrl('D:/99. S/20260730_Azure/index.html');
  const browser = await chromium.launch({ channel: 'chrome', headless: true });
  const page = await browser.newPage();
  const consoleErrors = [];
  page.on('console', (msg) => { if (msg.type() === 'error') consoleErrors.push(msg.text()); });
  page.on('pageerror', (err) => consoleErrors.push('pageerror: ' + String(err)));

  await page.goto(url);
  await page.waitForTimeout(300);

  const results = {};

  // 1) 로컬 테스트 랩 그룹 펼치고 API 테스트 탭으로 이동
  await page.click('[data-group="lab"] .nav-group-head');
  await page.waitForTimeout(100);
  await page.click('button[data-tab="lab-api-test"]');
  await page.waitForTimeout(200);

  // billing-get 라우트 선택 (4번째 탭)
  await page.click('#labApiTestBody [data-route-idx="3"]');
  await page.waitForTimeout(100);
  await page.check('#labInputForceFail');
  await page.click('#labApiTestRun');
  await page.waitForTimeout(150);
  results.apimTraceAfterForcedFailure = await page.locator('#labApiTestApim .lab-trace-row').allInnerTexts();
  results.kongTraceAfterForcedFailure = await page.locator('#labApiTestKong .lab-trace-row').allInnerTexts();

  // 2) rate-limit-by-key 파티셔닝: orders-get 라우트(0번째)에서 서로 다른 IP로 반복 호출
  await page.click('#labApiTestBody [data-route-idx="0"]');
  await page.waitForTimeout(100);
  await page.uncheck('#labInputForceFail').catch(() => {});
  for (let i = 0; i < 6; i++) {
    await page.fill('#labInputApikey', 'demo-key-123');
    await page.fill('#labInputIp', '203.0.113.50');
    await page.click('#labApiTestRun');
    await page.waitForTimeout(30);
  }
  results.apimStatusIpA = await page.locator('#labApiTestApim .lab-result-title').innerText();
  await page.fill('#labInputIp', '203.0.113.51');
  await page.click('#labApiTestRun');
  await page.waitForTimeout(100);
  results.apimStatusIpB = await page.locator('#labApiTestApim .lab-result-title').innerText();

  // 3) KQL 조인 랩에서 KQL_QUIZ 정답 실행
  await page.click('button[data-tab="kql-join-lab"]');
  await page.waitForTimeout(200);
  await page.click('#kqlLabTabs button[data-tab="examples"]');
  await page.waitForTimeout(150);
  await page.locator('[data-example-id="A1"] [data-role="editor"]').fill('AppRequests\n| where toint(ResultCode) >= 500\n| where AppRoleName == "orders-api-backend"');
  await page.locator('[data-example-id="A1"] [data-action="run"]').click();
  await page.waitForTimeout(150);
  results.tointQueryError = await page.locator('[data-example-id="A1"] .kql-error').count();

  await page.locator('[data-example-id="A2"] [data-role="editor"]').fill('AppRequests\n| where TimeGenerated > ago(1h)');
  await page.locator('[data-example-id="A2"] [data-action="run"]').click();
  await page.waitForTimeout(150);
  results.agoQueryError = await page.locator('[data-example-id="A2"] .kql-error').count();
  results.agoQueryResultRows = await page.locator('[data-example-id="A2"] .kql-table tr').count();

  results.consoleErrors = consoleErrors;
  console.log(JSON.stringify(results, null, 2));
  await browser.close();
})().catch((e) => { console.error('FATAL', e); process.exit(1); });
```

- [ ] **Step 2: 신규 시나리오 실행**

Run: `cd "C:\Users\whynot\browsertest" && node test-fidelity-fixes.js`

Expected:
- `apimTraceAfterForcedFailure`에 "재시도 1/3", "재시도 2/3", "재시도 3/3", "3회 재시도 소진", "커스텀 오류 응답 반환 (503)" 계열 문구 포함.
- `kongTraceAfterForcedFailure`에는 재시도 문구 없이 "백엔드 강제 실패 시뮬레이션" 문구만 포함.
- `apimStatusIpA`에 429 포함(6회째 호출로 5회/60초 초과), `apimStatusIpB`는 200(다른 IP라 카운터 분리 확인).
- `tointQueryError`/`agoQueryError` 둘 다 0(파싱 에러 없음), `agoQueryResultRows` > 1(결과 행 존재).
- `consoleErrors` 빈 배열.

- [ ] **Step 3: 기존 회귀 스위트 재실행**

Run: `cd "C:\Users\whynot\browsertest" && node test-tracker-merge.js`

Expected: 이전 세션과 동일한 결과(사이드바 기본 접힘, 새 패널 전환, 튜토리얼/예제/리셋, 콘솔 에러 0건) — 전부 그대로 통과.

- [ ] **Step 4: 완료 보고**

문제가 없으면 사용자에게 5개 항목 수정 결과를 보고한다. 문제가 있으면 해당 Task로 돌아가 원인을 수정한 뒤 Step 1부터 재검증한다.

---

## Self-Review 체크리스트

- [x] 스펙 커버리지: 설계 문서의 A~E 5개 항목이 Task 1~5로 정확히 매핑됨, Task 6은 스펙의 검증 섹션(E)을 확장 구현.
- [x] 플레이스홀더 없음: 모든 Step이 실제 코드(before/after)와 실제 실행 명령을 포함함.
- [x] 타입/이름 일관성: `runApimBackendSection`/`runApimOnErrorSection`/`checkRateLimitFixedWindow`/`SIMULATED_NOW`/`kustoAgoValue`/`KUSTO_CAST_FUNCS` 명명이 Task 전체에서 동일하게 사용됨(단, Task 3·4의 동일 이름 상수는 서로 다른 IIFE 스코프로 명시).
