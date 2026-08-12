# 신규 랩 구조 보강 (3군) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 시니어 리뷰 3군(신규 랩 구조) 3개 항목 — 백엔드 지연/타임아웃 체험 탭, 카나리·롤백 시나리오, Jira 루브릭 보강 — 을 구현한다.

**Architecture:** 1군에서 만든 `runApimBackendSection`/`runApimOnErrorSection`/`runKongMock`/`billing-get` 라우트를 확장 재사용(신규 UI+`backendDelayMs` 파라미터만 추가). 카나리 시나리오는 기존 `LAB_MIGRATION_PITFALLS` 배열에 순수 데이터 추가. Jira 루브릭은 기존 `.answer-reveal` 토글 패턴 재사용.

**Tech Stack:** 순수 HTML/CSS/Vanilla JS 단일 파일. 테스트 프레임워크 없음 — Node 구문 체크 + Playwright.

## Global Constraints

- `docs/superpowers/specs/2026-08-13-lab-scenarios-expansion-design.md`의 설계를 따른다.
- **회귀 없음**: 기존 API 테스트/APIM Policy 탭은 `input.backendDelayMs`를 설정하지 않으므로 동작이 지금과 100% 동일해야 한다.
- 새 CLI/정책 문법 없음(1군 엔진 재사용).
- 커밋 메시지는 한국어.

---

### Task 1: 백엔드 지연/타임아웃 체험 탭

**Files:** Modify `index.html` — `NAV` 배열, `billing-get` 라우트, `runApimBackendSection`, `runKongMock`, 새 패널 HTML, 새 `renderLabTimeout` 함수.

**Interfaces:** `input.backendDelayMs`(number, ms) — Task 2·3·4와 무관, `runApimBackendSection(elements, input, trace)`/`runKongMock(route, input)`의 기존 시그니처 그대로 재사용(새 필드만 추가로 읽음).

- [ ] **Step 1: billing-get 라우트에 kongTimeoutMs 필드 추가**

기존:
```js
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
  ];
```

교체:
```js
      kongPlugins: [],
      kongTimeoutMs: 3000,
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
  ];
```

- [ ] **Step 2: runApimBackendSection에 지연 기반 타임아웃 감지 추가**

기존:
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
```

교체:
```js
  function runApimBackendSection(elements, input, trace) {
    var failStatus = 503;
    var retryEl = null;
    for (var i = 0; i < elements.length; i++) {
      if (elements[i].tagName === "retry") { retryEl = elements[i]; break; }
    }

    var timedOut = false;
    if (typeof input.backendDelayMs === "number" && input.backendDelayMs > 0 && retryEl) {
      var fwdEl = retryEl.querySelector("forward-request");
      var timeoutSec = fwdEl ? parseInt(fwdEl.getAttribute("timeout"), 10) : null;
      if (timeoutSec && input.backendDelayMs > timeoutSec * 1000) {
        timedOut = true;
        trace.push({ policy: "backend", effect: "백엔드 응답 지연 " + input.backendDelayMs + "ms > 타임아웃 " + (timeoutSec * 1000) + "ms — 타임아웃 발생" });
      }
    }

    var forced = !!input.forceBackendFailure || timedOut;
    if (!forced) return { failed: false };

    if (!retryEl) {
```

(이후 `if (!retryEl) { ... }`부터 함수 끝까지는 변경 없음 — 그대로 유지)

- [ ] **Step 3: runKongMock에 지연 기반 타임아웃 감지 추가**

기존:
```js
    if (input.forceBackendFailure) {
      trace.push({ policy: "backend", effect: "백엔드 강제 실패 시뮬레이션 (503) — Kong의 재시도는 Service의 retries 필드로 별도 설정되며, 이 랩은 그 재시도 동작 자체는 시뮬레이션하지 않습니다" });
      return { statusCode: 503, headers: headers, body: { message: "Service Unavailable" }, trace: trace };
    }
    return { statusCode: 200, headers: headers, body: route.backendResponse, trace: trace };
```

교체:
```js
    if (typeof input.backendDelayMs === "number" && input.backendDelayMs > 0 && route.kongTimeoutMs && input.backendDelayMs > route.kongTimeoutMs) {
      trace.push({ policy: "backend", effect: "백엔드 응답 지연 " + input.backendDelayMs + "ms > 타임아웃 " + route.kongTimeoutMs + "ms(kongTimeoutMs) — 타임아웃 발생, 재시도 없음(Kong Service의 retries 필드가 별도 처리)" });
      return { statusCode: 503, headers: headers, body: { message: "Service Unavailable" }, trace: trace };
    }
    if (input.forceBackendFailure) {
      trace.push({ policy: "backend", effect: "백엔드 강제 실패 시뮬레이션 (503) — Kong의 재시도는 Service의 retries 필드로 별도 설정되며, 이 랩은 그 재시도 동작 자체는 시뮬레이션하지 않습니다" });
      return { statusCode: 503, headers: headers, body: { message: "Service Unavailable" }, trace: trace };
    }
    return { statusCode: 200, headers: headers, body: route.backendResponse, trace: trace };
```

- [ ] **Step 4: NAV 배열에 새 항목 추가**

기존:
```js
        { id: "lab-workbook", label: "Workbooks 구축", icon: '<rect x="3" y="4" width="18" height="16" rx="1"/><path d="M7 9h10M7 13h6M7 17h4"/>' }
      ] },
    { id: "kql-join-lab", label: "KQL 조인 랩", icon: '<circle cx="9" cy="12" r="5"/><circle cx="15" cy="12" r="5"/>' }
  ];
```

교체:
```js
        { id: "lab-workbook", label: "Workbooks 구축", icon: '<rect x="3" y="4" width="18" height="16" rx="1"/><path d="M7 9h10M7 13h6M7 17h4"/>' },
        { id: "lab-timeout", label: "백엔드 지연/타임아웃", icon: '<circle cx="12" cy="12" r="9"/><path d="M12 8v4l3 3"/>' }
      ] },
    { id: "kql-join-lab", label: "KQL 조인 랩", icon: '<circle cx="9" cy="12" r="5"/><circle cx="15" cy="12" r="5"/>' }
  ];
```

- [ ] **Step 5: 새 패널 HTML 삽입**

`panel-lab-workbook` 섹션의 닫는 `</section>` 직후(`panel-kql-join-lab` 시작 전)에 추가:

```html
    <section class="panel" id="panel-lab-timeout" data-panel>
      <div class="panel-head">
        <span class="eyebrow">20 · Local Test Lab</span>
        <h1>백엔드 지연/타임아웃</h1>
        <p class="desc">billing-get 라우트(백엔드 재시도+커스텀 오류 응답)를 대상으로, 백엔드 응답 지연(ms)을 직접 입력해 타임아웃·재시도 동작을 체험합니다. APIM 타임아웃은 30초(30000ms), Kong은 데모용 축소값 3000ms입니다 — 지연 값을 이 사이(예: 5000ms)로 넣으면 APIM은 재시도 후 커스텀 오류로 응답하고 Kong은 즉시 실패(재시도 없음)하는 차이를 비교할 수 있습니다.</p>
      </div>
      <div id="labTimeoutBody"></div>
    </section>

```

- [ ] **Step 6: renderLabTimeout 함수 추가 및 호출**

`renderLabPolicy();` 호출 직후에 추가:

```js

  function renderLabTimeout() {
    var route = LAB_ROUTES.filter(function (r) { return r.id === "billing-get"; })[0];
    var body = document.getElementById("labTimeoutBody");
    body.innerHTML =
      '<div class="lab-input-row">' +
      '<label>백엔드 지연(ms)<input type="number" id="labTimeoutDelay" placeholder="예: 300 또는 5000" min="0"></label>' +
      '</div>' +
      '<button class="btn primary" type="button" id="labTimeoutRun">실행</button>' +
      '<div class="lab-compare-grid" style="margin-top:var(--sp-4);"><div id="labTimeoutKong" class="pcard"></div><div id="labTimeoutApim" class="pcard"></div></div>';

    document.getElementById("labTimeoutRun").addEventListener("click", function () {
      var delayEl = document.getElementById("labTimeoutDelay");
      var delayMs = parseInt(delayEl.value, 10);
      var input = { backendDelayMs: isNaN(delayMs) ? 0 : delayMs };
      renderLabResult(document.getElementById("labTimeoutKong"), "Kong mock", runKongMock(route, input));
      renderLabResult(document.getElementById("labTimeoutApim"), "APIM mock", runApimMock(route, route.apimPolicyXml, input));
    });
  }
  renderLabTimeout();
```

- [ ] **Step 7: 구문 검사**

Run: `node -e "const fs=require('fs');const html=fs.readFileSync('D:/99. S/20260730_Azure/index.html','utf8');const scripts=[...html.matchAll(/<script>([\s\S]*?)<\/script>/g)].map(m=>m[1]);scripts.forEach((s,i)=>{try{new Function(s);console.log('block '+i+': OK');}catch(e){console.log('block '+i+': SYNTAX ERROR -> '+e.message);}});"`
Expected: `block 0: OK`, `block 1: OK`.

- [ ] **Step 8: 커밋**

```bash
git add index.html
git commit -m "feat: 백엔드 지연/타임아웃 체험 탭 추가 (billing-get 재사용)"
```

---

### Task 2: 카나리·롤백 판단 시나리오 4개 추가

**Files:** Modify `index.html` — `LAB_MIGRATION_PITFALLS` 배열.

**Interfaces:** 없음(순수 데이터, 기존 `renderLabMigration()`이 자동 렌더링).

- [ ] **Step 1: 4개 시나리오 추가**

`LAB_MIGRATION_PITFALLS` 배열의 마지막 기존 항목(strip_path 시나리오) 뒤, 배열을 닫는 `];` 직전에 추가:

```js
    ,
    {
      scenario: "APIM API Revision으로 새 버전을 5% 트래픽에 카나리 배포했다. 새 리비전의 5xx 비율이 기존 리비전보다 2배 높게 나왔지만, 절대 요청 수는 하루 10건 미만이라 통계적으로 유의미하지 않다. 지금 바로 자동 롤백해야 할까?",
      answer: "아니요, 바로 자동 롤백할 필요는 없습니다(다만 계속 관찰은 필요).",
      reason: "표본이 너무 작으면(하루 10건 미만) 우연에 의한 변동일 가능성이 높아, 절대 건수·신뢰구간을 고려하지 않고 비율만으로 판단하면 정상 배포도 롤백시키는 false positive가 잦아집니다. 실무에서는 최소 표본 크기(예: N건 이상)와 관찰 기간을 롤백 조건에 함께 명시해야 합니다."
    },
    {
      scenario: "APIM Backend를 기존 백엔드에서 새 백엔드로 트래픽을 100% 전환(블루/그린 스위치)했다. 전환 직후 이미 로그인된 사용자들의 세션이 끊기거나 장바구니가 사라지는 문제가 보고됐다. 원인이 APIM 자체의 버그일 가능성이 높을까?",
      answer: "아니요, APIM 자체 버그일 가능성은 낮습니다.",
      reason: "APIM Backend 엔티티 전환은 요청 라우팅 대상만 바꾸는 것으로, 세션 상태나 캐시는 APIM이 관리하지 않습니다. 세션이 인메모리(sticky session)로 백엔드 서버 하나에 저장되고 있었다면, 새 백엔드로 라우팅이 바뀌는 순간 그 세션 데이터는 존재하지 않는 서버를 가리키게 됩니다. 원인은 대개 백엔드 애플리케이션의 세션 저장 방식(외부 세션 스토어 미사용)에 있습니다."
    },
    {
      scenario: "Azure Monitor 로그 검색 알림으로 '신규 리비전의 5xx 비율이 5분간 5% 초과' 시 알림을 받도록 설정하고, 알림이 오면 담당자가 수동으로 API Revision을 이전 버전으로 되돌리기로 했다. 그런데 알림이 새벽 3시에 왔고 담당자가 30분 뒤에야 확인했다. 이 구성의 근본적인 취약점은?",
      answer: "예, 근본적인 취약점이 있습니다 — 수동 대응에 의존하는 구조.",
      reason: "알림 기반 수동 롤백은 담당자 응답 시간만큼 장애가 지속됩니다. 카나리 배포에서는 임계치 초과 시 Action Group의 Azure Automation Runbook이나 Logic App을 트리거해 API Revision을 자동으로 되돌리는 구성(또는 최소한 카나리 트래픽 비율을 자동으로 0%로 낮추는 구성)이 사람의 대응 시간을 제거합니다."
    },
    {
      scenario: "API Revision 2개(v1 현재 운영, v2 신규)를 두고 API Management의 '리비전' 기능만으로 트래픽을 50:50 자동 분산하려 했다. 실제로 그렇게 동작할까?",
      answer: "아니요, 그렇게 동작하지 않습니다.",
      reason: "APIM 리비전(Revision)은 버전 관리와 스테이징(비운영 URL로 미리보기) 용도이며, 하나의 리비전만 'current'로 지정되어 실제 프로덕션 트래픽을 받습니다. 트래픽을 퍼센트 기반으로 분산하려면 리비전이 아니라 Azure Front Door의 가중치 기반 라우팅이나, App Gateway/Traffic Manager의 가중치 라우팅을 앞단에 둬야 합니다."
    }
```

- [ ] **Step 2: 구문 검사** (Task 1과 동일 명령)

- [ ] **Step 3: 커밋**

```bash
git add index.html
git commit -m "docs: 마이그레이션 탭에 카나리·롤백 판단 시나리오 4개 추가"
```

---

### Task 3: Jira 템플릿 루브릭 보강

**Files:** Modify `index.html` — `panel-jira` HTML, Jira 관련 JS(이벤트 리스너 추가).

**Interfaces:** 없음(기존 `.answer-reveal`/`.show` 토글 패턴 재사용, 새 CSS 없음).

- [ ] **Step 1: 오른쪽 카드에 최소 기준 목록 추가**

기존:
```html
        <div class="card">
          <h3 style="font-size:var(--fs-md);margin-bottom:var(--sp-1);">AI 작업 히스토리 기록 절차</h3>
          <p class="quiz-scenario" style="margin-bottom:var(--sp-3);">AIDD 방법론에서 요구하는 "AI 활용 이력 남기기" 체크리스트입니다. 티켓 작성 시 습관화하세요.</p>
          <div class="proc-list" id="procList"></div>
        </div>
```

교체:
```html
        <div class="card">
          <h3 style="font-size:var(--fs-md);margin-bottom:var(--sp-1);">AI 작업 히스토리 기록 절차</h3>
          <p class="quiz-scenario" style="margin-bottom:var(--sp-3);">AIDD 방법론에서 요구하는 "AI 활용 이력 남기기" 체크리스트입니다. 티켓 작성 시 습관화하세요.</p>
          <div class="proc-list" id="procList"></div>
          <div style="margin-top:var(--sp-4);padding-top:var(--sp-4);border-top:1px dashed var(--border);">
            <h3 style="font-size:var(--fs-md);margin-bottom:var(--sp-2);">최소 기준(이 정도는 있어야 "기록했다"고 봄)</h3>
            <ul style="margin:0;padding-left:var(--sp-4);font-size:var(--fs-xs);color:var(--text-dim);line-height:1.8;">
              <li><strong>사용 AI/도구</strong>: "AI 사용함"(X) → "Claude Code CLI (Sonnet 5)"처럼 도구+모델명(O)</li>
              <li><strong>프롬프트 요약</strong>: "물어봤음"(X) → 실제 지시문 핵심을 재구성 가능한 수준(O)</li>
              <li><strong>채택/거부 근거</strong>: "그대로 씀"(X) → 채택/수정한 기술적 이유 1문장 이상(O)</li>
              <li><strong>변경 파일</strong>: "일부 수정"(X) → 실제 경로 나열(O)</li>
              <li><strong>테스트 증적</strong>: "테스트함"(X) → 실행 명령어 + 실제 출력/스크린샷 링크(O)</li>
            </ul>
          </div>
        </div>
```

- [ ] **Step 2: 왼쪽 카드에 모범 예시 버튼 추가**

기존:
```html
          <textarea class="answer-in" id="jiraDraft" style="min-height:220px;" placeholder="위 템플릿을 참고해 직접 티켓 초안을 작성해 보세요..."></textarea>
          <p class="empty-hint" id="jiraSaved" style="margin-top:var(--sp-2);">&nbsp;</p>
        </div>
```

교체:
```html
          <textarea class="answer-in" id="jiraDraft" style="min-height:220px;" placeholder="위 템플릿을 참고해 직접 티켓 초안을 작성해 보세요..."></textarea>
          <p class="empty-hint" id="jiraSaved" style="margin-top:var(--sp-2);">&nbsp;</p>
          <button class="btn" type="button" id="jiraExampleToggle" style="margin-top:var(--sp-3);">모범 예시 티켓 보기</button>
          <div class="answer-reveal" id="jiraExampleReveal">
            <span class="tag">모범 예시</span>
            <div class="code-block">[Story]
As a API 게이트웨이 운영자, I want Kong의 분당 100회 rate-limiting을 APIM rate-limit 정책으로 동일하게 이관하고 싶다, so that Kong 폐기 후에도 기존 클라이언트의 호출 제한 정책이 끊김 없이 유지된다.

[Acceptance Criteria]
- Given 클라이언트가 60초 내 100회를 초과 호출하면, When 101번째 요청을 보내면, Then APIM이 429 Too Many Requests를 반환한다.
- Given 60초 윈도우가 지나면, When 다음 요청을 보내면, Then 카운터가 리셋되어 정상 응답한다.

[Dev Notes - AI 협업 기록]
- 사용 AI/도구: Claude Code CLI (Sonnet 5)
- 프롬프트 요약: "Kong rate-limiting(분당 100회)을 APIM rate-limit 정책 XML로 변환해줘. renewal-period는 초 단위로 변환 필요"
- AI 제안 채택/거부 및 근거: AI가 처음 제안한 renewal-period=100을 60으로 직접 수정(분당 제한이므로 60초가 맞음, 100은 오타로 추정). 나머지 XML 구조는 그대로 채택.
- 변경 파일: apim-policies/orders-api.inbound.xml

[Test Evidence]
- 테스트 방법: Postman으로 60초 내 101회 연속 호출 스크립트 실행
- 결과: 100번째까지 200, 101번째부터 429 확인(스크린샷: jira-evidence-01.png)</div>
          </div>
        </div>
```

- [ ] **Step 3: JS 이벤트 리스너 추가**

`var jiraTa = document.getElementById("jiraDraft");` 로 시작하는 기존 블록 근처(Jira 관련 스크립트가 있는 곳, `var saveTimer = null;` 다음 줄)에 추가:

```js
  document.getElementById("jiraExampleToggle").addEventListener("click", function () {
    var reveal = document.getElementById("jiraExampleReveal");
    var isShown = reveal.classList.toggle("show");
    this.textContent = isShown ? "모범 예시 접기" : "모범 예시 티켓 보기";
  });
```

- [ ] **Step 4: 구문 검사** (Task 1과 동일 명령)

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "feat: Jira 템플릿에 최소 기준 체크리스트와 모범 예시 티켓 추가"
```

---

### Task 4: 통합 검증

**Files:** 코드 변경 없음(검증 전용).

- [ ] **Step 1: 백엔드 지연/타임아웃 탭 검증**

Playwright로 "로컬 테스트 랩" 그룹 펼치고 "백엔드 지연/타임아웃" 탭 이동. 지연 300ms 입력·실행 → APIM/Kong 모두 정상 200 응답 확인. 지연 5000ms 입력·실행 → APIM 트레이스에 재시도 3회+on-error 커스텀 응답(503), Kong은 즉시 503(재시도 문구 없음) 확인.

- [ ] **Step 2: 기존 탭 회귀 확인**

API 테스트 탭에서 billing-get을 "백엔드 강제 실패" 토글로 실행해 기존 동작(재시도 3회+on-error)이 그대로인지 확인. orders-get/users-get/legacy-report도 정상 확인.

- [ ] **Step 3: 카나리 시나리오 확인**

마이그레이션 탭에서 신규 시나리오 4개가 렌더링되고 "발생한다"/"발생하지 않는다" 클릭 시 정답이 펼쳐지는지 확인(총 10개 시나리오).

- [ ] **Step 4: Jira 루브릭 확인**

Jira 탭에서 최소 기준 목록이 보이고, "모범 예시 티켓 보기" 클릭 시 예시 티켓이 펼쳐지는지 확인.

- [ ] **Step 5: 콘솔 에러 확인 및 완료 보고**

전체 탐색 중 콘솔 에러 0건 확인 후 사용자에게 보고.

---

## Self-Review 체크리스트

- [x] 스펙 커버리지: 설계 A/B/C가 Task 1/2/3으로 정확히 매핑됨, Task 4는 검증.
- [x] 플레이스홀더 없음: 모든 Step이 실제 코드와 실제 시나리오 텍스트를 포함.
- [x] 내부 일관성: `input.backendDelayMs` 명명이 Task 1 전체에서 동일하게 사용됨.
- [x] 범위 검사: 3개 항목이 서로 독립적 파일 위치를 건드리므로 순서 무관하게 안전.
