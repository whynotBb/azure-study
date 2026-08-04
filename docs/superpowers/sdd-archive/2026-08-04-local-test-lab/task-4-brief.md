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

