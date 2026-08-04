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

