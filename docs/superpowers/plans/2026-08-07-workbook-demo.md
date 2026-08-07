# Workbooks 구축 실습 — 자동 재생 데모 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `azure-migration-tracker.html`의 "Workbooks 구축" 패널에 "▶ 데모 보기" 버튼을 추가해, 미션 2(API별 필터링 리포트)의 전 과정을 실제 UI 버튼을 그대로 눌러가며 자동 재생하는 시연 모드를 제공한다.

**Architecture:** 데모는 별도의 가짜 화면을 그리지 않고 실제 `[data-action="..."]` 버튼에 `.click()`을 호출해 기존 이벤트 위임 핸들러를 그대로 통과시킨다. 진입 버튼과 캡션 배너는 `renderWorkbookCanvas()`가 매번 통째로 다시 그리는 `#labWorkbookBody`의 바깥(형제 엘리먼트)에 정적으로 배치해, 데모 도중 발생하는 재렌더링에도 배너가 사라지지 않게 한다. 시작 전 상태를 백업했다가 종료(닫기/완료 후 닫기/`beforeunload`/다른 탭 이동) 시 되돌리는 3중 안전장치로 사용자의 실제 진행 상황을 보호한다.

**Tech Stack:** 순수 HTML/CSS/바닐라 JS(빌드 도구 없음), `setTimeout` 기반 스텝 시퀀서, `MutationObserver`, `beforeunload`.

## Global Constraints

- 기존 8개 패널·로컬 테스트 랩 6개 패널의 함수는 절대 수정하지 않는다.
- **Workbooks 구축 실습 자체의 기존 함수도 수정하지 않는다**: `loadWorkbookState`, `saveWorkbookState`, `substituteWorkbookParams`, `renderWorkbookCanvas`, `renderWorkbookCellHtml`, `renderWorkbookMissionBar`, `renderWorkbookMissionCard`, `renderWorkbookToolbar`, `renderWorkbookChecklist`, `renderWorkbookModelAnswer`, `runWorkbookQueryCell`, `handleWorkbookFieldChange`, `initWorkbookLab`, `renderWorkbookChartSvg`, `WORKBOOK_MISSIONS`, `WORKBOOK_PARAM_OPTIONS` — 전부 그대로 두고 **호출만** 한다. 이를 지키기 위해 진입 버튼(`workbookDemoBar`)과 캡션 배너(`workbookDemoBanner`)는 `renderWorkbookMissionBar`가 그리는 영역이 아니라 패널의 정적 HTML에 형제 엘리먼트로 추가하고, 그 클릭 이벤트도 `initWorkbookLab`의 기존 델리게이트 리스너가 아니라 새 리스너로 별도 처리한다(스펙 초안은 "미션 탭 줄 오른쪽에 버튼"이라고 적었지만, 기존 함수 무수정 원칙을 완전히 지키기 위해 미션 탭 줄 **위쪽**의 정적 바로 배치를 조정했다 — 기능은 동일).
- 이 저장소는 git 저장소이며 빌드/테스트 러너는 없다 — 모든 "테스트"는 브라우저 수동 확인이다. 각 Task 완료 시 커밋한다.
- 데모 스크립트는 정확히 6단계(`click`/`select`/`type` 3가지 액션만 지원)이며, 미션 2 시나리오 하나만 재생한다 — 다른 미션용 데모나 일반화된 스크립트 엔진은 만들지 않는다.
- **일시정지는 스텝 경계에서만 적용된다**: 이미 시작된 스텝(하이라이트 표시→액션 실행)은 끝까지 완료된 뒤에만 정지하고, 다음 스텝으로의 자동 진행만 멈춘다. 액션 도중(특히 타이핑 애니메이션 도중)에 즉시 얼어붙는 정밀한 중간 정지는 구현하지 않는다(스펙 대비 단순화 — YAGNI: 데모用 애니메이션의 200ms 내외 오차는 학습 경험에 영향이 없다).
- 파라미터 드롭다운 고정 옵션(`orders-api`/`users-api`/`legacy-api`)이나 미션 정의 문구를 바꾸지 않는다 — 이 데모는 현재의 `WORKBOOK_MISSIONS[1]`과 `AM_GATEWAY_LOGS`에 결합되어 있으므로, 나중에 그 둘을 바꾸면 데모도 수동으로 재검증해야 한다(코드 대응 없음, 문서화만).

---

## Task 1: 정적 HTML 셸(진입 버튼 + 배너 컨테이너) + CSS

**Files:**
- Modify: `azure-migration-tracker.html` (HTML은 `panel-lab-workbook`의 `<div id="labWorkbookBody"></div>` 바로 앞, CSS는 `.workbook-model-answer { ... }` 규칙 다음·`</style>` 직전)

**Interfaces:**
- Produces: 컨테이너 DOM id `workbookDemoBar`(진입 버튼), `workbookDemoBanner`(빈 배너 컨테이너, 초기엔 내용 없음) — Task 2가 이 id들에 렌더링한다.

- [ ] **Step 1: CSS 추가**

`azure-migration-tracker.html`에서 `.workbook-model-answer { border: 1px dashed var(--border); border-radius: var(--radius-md); padding: var(--sp-3); margin-top: var(--sp-3); }` 바로 다음, `</style>` 바로 앞에 추가:

```css
.workbook-demo-bar { display: flex; justify-content: flex-end; margin-bottom: var(--sp-2); }
.workbook-demo-banner { position: sticky; top: 0; z-index: 5; display: flex; align-items: center; gap: var(--sp-3); flex-wrap: wrap; background: var(--bg); border: 1px solid var(--border); border-radius: var(--radius-md); padding: var(--sp-2) var(--sp-3); margin-bottom: var(--sp-3); }
.workbook-demo-progress { font-family: var(--font-mono); font-size: var(--fs-xs); color: var(--text-dim); white-space: nowrap; }
.workbook-demo-caption { flex: 1; font-size: var(--fs-sm); }
.workbook-demo-controls { display: flex; gap: var(--sp-2); flex-wrap: wrap; }
.demo-highlight { outline: 2px solid var(--accent-target); outline-offset: 2px; animation: demo-pulse 1s ease-in-out infinite; }
@keyframes demo-pulse { 0%, 100% { outline-color: var(--accent-target); } 50% { outline-color: transparent; } }
```

- [ ] **Step 2: 패널 HTML에 정적 컨테이너 추가**

`panel-lab-workbook` 섹션 안에서 다음 줄:

```html
      <div id="labWorkbookBody"></div>
```

을 다음으로 교체:

```html
      <div class="workbook-demo-bar" id="workbookDemoBar">
        <button class="btn primary" type="button" data-action="start-demo">▶ 데모 보기</button>
      </div>
      <div id="workbookDemoBanner"></div>
      <div id="labWorkbookBody"></div>
```

- [ ] **Step 3: 브라우저에서 셸 확인**

`azure-migration-tracker.html`을 브라우저로 열어 "Workbooks 구축" 탭에서 미션 탭 줄 위에 "▶ 데모 보기" 버튼이 보이는지 확인. 클릭해도(아직 JS가 없으므로) 아무 일도 일어나지 않는지, 콘솔 에러가 없는지, 캔버스와 미션 탭이 기존과 동일하게 동작하는지 확인.
예상 결과: 콘솔 에러 없음, 버튼만 추가로 보임.

---

## Task 2: 데모 스크립트 · 재생 엔진 · 안전장치 · UI 와이어링

**Files:**
- Modify: `azure-migration-tracker.html` (`initWorkbookLab();` 호출 뒤, `/* ---------------- jira draft ---------------- */` 주석 앞에 추가)

**Interfaces:**
- Consumes: `loadWorkbookState`/`saveWorkbookState`(Workbooks 구축 실습, 시그니처 변경 없음), `renderWorkbookCanvas`/`escapeHtml`/`persist`/`state`(동일), `workbookRunResults`(Workbooks 구축 실습의 모듈 스코프 변수, 재사용), `workbookDemoBar`/`workbookDemoBanner`/`labWorkbookBody`(Task 1의 컨테이너 id).

- [ ] **Step 1: 데모 스크립트 데이터 + 상태 객체 + 배너 렌더러 추가**

`initWorkbookLab();` 호출 바로 다음에 추가:

```js
  /* ---------------- Workbooks 구축: 자동 재생 데모 ---------------- */
  var WORKBOOK_DEMO_STEPS = [
    { caption: "먼저 API를 고를 파라미터 셀을 만들어요.",
      targetSelector: '[data-action="add-cell"][data-cell-type="param"]', act: "click" },
    { caption: "orders-api를 선택해봐요.",
      targetSelector: ".workbook-param-select", act: "select", value: "orders-api" },
    { caption: "이 파라미터를 참조하는 쿼리 셀을 추가해요.",
      targetSelector: '[data-action="add-cell"][data-cell-type="query"]', act: "click" },
    { caption: "파라미터를 참조하는 쿼리를 입력해요.",
      targetSelector: ".workbook-query-input", act: "type",
      value: 'ApiManagementGatewayLogs\n| where ApiId == "{ApiIdParam}"\n| take 10' },
    { caption: "실행해서 결과를 확인해요.",
      targetSelector: '[data-action="run"]', act: "click" },
    { caption: "체크리스트로 확인해요 — 3개 모두 통과!",
      targetSelector: '[data-action="check-mission"]', act: "click" }
  ];

  var workbookDemo = { running: false, paused: false, stepIndex: 0, backupRaw: null, timerId: null };

  function renderWorkbookDemoBanner() {
    var banner = document.getElementById("workbookDemoBanner");
    if (!workbookDemo.running) { banner.innerHTML = ""; return; }

    var total = WORKBOOK_DEMO_STEPS.length;
    var done = workbookDemo.stepIndex >= total;
    var caption = done ? "데모 완료! 이 흐름 그대로 직접 만들어보세요." : WORKBOOK_DEMO_STEPS[workbookDemo.stepIndex].caption;
    var progress = (done ? total : workbookDemo.stepIndex + 1) + " / " + total;

    var buttons;
    if (done) {
      buttons = '<button class="btn" type="button" data-action="close-demo">닫기</button>';
    } else if (workbookDemo.paused) {
      buttons =
        '<button class="btn" type="button" data-action="resume-demo">재생</button>' +
        '<button class="btn" type="button" data-action="next-demo">다음</button>' +
        '<button class="btn" type="button" data-action="close-demo">닫기</button>';
    } else {
      buttons =
        '<button class="btn" type="button" data-action="pause-demo">일시정지</button>' +
        '<button class="btn" type="button" data-action="close-demo">닫기</button>';
    }

    banner.innerHTML = '<div class="workbook-demo-banner">' +
      '<span class="workbook-demo-progress">' + progress + '</span>' +
      '<span class="workbook-demo-caption">' + escapeHtml(caption) + '</span>' +
      '<span class="workbook-demo-controls">' + buttons + '</span>' +
      '</div>';
  }

  function clearWorkbookDemoHighlight() {
    var highlighted = document.querySelectorAll(".demo-highlight");
    for (var i = 0; i < highlighted.length; i++) highlighted[i].classList.remove("demo-highlight");
  }
```

- [ ] **Step 2: 시작/스텝 실행/액션 함수 추가**

Step 1의 `clearWorkbookDemoHighlight` 바로 다음에 추가:

```js
  function startWorkbookDemo() {
    if (workbookDemo.running) return;
    workbookDemo.backupRaw = state.lab["workbook-cells"];
    workbookDemo.running = true;
    workbookDemo.paused = false;
    workbookDemo.stepIndex = 0;

    saveWorkbookState({ missionId: "m2", cells: [] });
    workbookRunResults = {};
    renderWorkbookCanvas();
    renderWorkbookDemoBanner();

    window.addEventListener("beforeunload", restoreWorkbookDemoSync);
    runWorkbookDemoStep(0);
  }

  function runWorkbookDemoStep(index) {
    if (!workbookDemo.running) return;
    workbookDemo.stepIndex = index;

    if (index >= WORKBOOK_DEMO_STEPS.length) {
      clearWorkbookDemoHighlight();
      renderWorkbookDemoBanner();
      return;
    }

    renderWorkbookDemoBanner();
    var step = WORKBOOK_DEMO_STEPS[index];
    var target = document.querySelector(step.targetSelector);
    if (!target) {
      abortWorkbookDemo("데모 재생 중 문제가 발생했습니다. 다시 시도해 주세요.");
      return;
    }
    clearWorkbookDemoHighlight();
    target.classList.add("demo-highlight");

    workbookDemo.timerId = setTimeout(function () {
      performWorkbookDemoAction(step, target, function () {
        workbookDemo.timerId = setTimeout(function () {
          if (!workbookDemo.paused) runWorkbookDemoStep(index + 1);
        }, 800);
      });
    }, 1200);
  }

  function performWorkbookDemoAction(step, target, done) {
    if (step.act === "click") {
      target.click();
      done();
      return;
    }
    if (step.act === "select") {
      target.value = step.value;
      target.dispatchEvent(new Event("change", { bubbles: true }));
      target.dispatchEvent(new Event("input", { bubbles: true }));
      done();
      return;
    }
    if (step.act === "type") {
      var text = step.value;
      var i = 0;
      (function typeChar() {
        target.value = text.slice(0, i + 1);
        target.dispatchEvent(new Event("input", { bubbles: true }));
        i++;
        if (i < text.length) {
          workbookDemo.timerId = setTimeout(typeChar, 30);
        } else {
          done();
        }
      })();
    }
  }
```

- [ ] **Step 3: 컨트롤 함수(일시정지/재생/다음/종료/중단) + 안전장치 + init 추가**

Step 2의 `performWorkbookDemoAction` 바로 다음에 추가:

```js
  function pauseWorkbookDemo() {
    workbookDemo.paused = true;
    renderWorkbookDemoBanner();
  }

  function resumeWorkbookDemo() {
    workbookDemo.paused = false;
    renderWorkbookDemoBanner();
  }

  function nextWorkbookDemoStep() {
    clearTimeout(workbookDemo.timerId);
    runWorkbookDemoStep(workbookDemo.stepIndex + 1);
  }

  function restoreWorkbookDemoSync() {
    if (!workbookDemo.running) return;
    state.lab["workbook-cells"] = workbookDemo.backupRaw;
    persist();
  }

  function endWorkbookDemo() {
    if (!workbookDemo.running) return;
    clearTimeout(workbookDemo.timerId);
    window.removeEventListener("beforeunload", restoreWorkbookDemoSync);
    clearWorkbookDemoHighlight();
    state.lab["workbook-cells"] = workbookDemo.backupRaw;
    persist();
    workbookRunResults = {};
    workbookDemo.running = false;
    workbookDemo.paused = false;
    workbookDemo.stepIndex = 0;
    renderWorkbookCanvas();
    renderWorkbookDemoBanner();
  }

  function abortWorkbookDemo(message) {
    endWorkbookDemo();
    var bar = document.getElementById("workbookDemoBar");
    var note = document.createElement("p");
    note.className = "empty-hint";
    note.textContent = message;
    bar.appendChild(note);
    setTimeout(function () { if (note.parentNode) note.parentNode.removeChild(note); }, 4000);
  }

  function watchWorkbookDemoPanel() {
    var panel = document.getElementById("panel-lab-workbook");
    var observer = new MutationObserver(function () {
      if (workbookDemo.running && !panel.classList.contains("active")) endWorkbookDemo();
    });
    observer.observe(panel, { attributes: true, attributeFilter: ["class"] });
  }

  function initWorkbookDemo() {
    document.getElementById("workbookDemoBar").addEventListener("click", function (e) {
      if (e.target.closest('[data-action="start-demo"]')) startWorkbookDemo();
    });
    document.getElementById("workbookDemoBanner").addEventListener("click", function (e) {
      var btn = e.target.closest("button[data-action]");
      if (!btn) return;
      var action = btn.getAttribute("data-action");
      if (action === "pause-demo") pauseWorkbookDemo();
      else if (action === "resume-demo") resumeWorkbookDemo();
      else if (action === "next-demo") nextWorkbookDemoStep();
      else if (action === "close-demo") endWorkbookDemo();
    });
    watchWorkbookDemoPanel();
  }
  initWorkbookDemo();
```

- [ ] **Step 4: 브라우저에서 전체 시나리오 확인**

`azure-migration-tracker.html`을 열고 "Workbooks 구축" 탭에서 순서대로 확인:

1. 자유 캔버스에서 셀 2~3개를 직접 만들어 둔다(복원 확인용 기준선).
2. "▶ 데모 보기" 클릭 → 캔버스가 미션 2로 전환되며 비워지고, 배너에 "1 / 6" + 첫 캡션이 뜨는지 확인.
3. 6단계가 자동으로 순서대로 재생되며, 파라미터 셀 추가 → `orders-api` 선택 → 쿼리 셀 추가 → 쿼리 텍스트가 한 글자씩 타이핑되듯 채워짐 → 실행(결과 표 등장) → 체크리스트 확인(3개 모두 ✅)까지 실제로 진행되는지 확인.
4. 마지막 단계 후 배너가 "6 / 6, 데모 완료!" + "닫기" 버튼만 남은 상태로 바뀌는지, 체크리스트 결과가 화면에 그대로 남아 있는지 확인.
5. "닫기" 클릭 → 캔버스가 1번에서 만들어 둔 원래 셀 구성(자유 캔버스, 셀 2~3개)으로 정확히 복원되는지 확인.
6. 다시 "데모 보기"를 눌러 재생 중 "일시정지" → 배너가 "재생/다음/닫기"로 바뀌는지 → "다음"을 눌러 한 스텝씩 수동으로 진행되는지 → "재생"으로 자동 진행이 재개되는지 확인.
7. 데모 재생 중간에 "닫기"를 눌러 중단 → 원래 상태로 복원되는지 확인.
8. 데모 재생 중 왼쪽 nav에서 다른 탭(예: "Kusto 실행기")을 클릭 → 데모가 자동으로 멈추고 워크북 탭으로 돌아왔을 때 원래 상태로 복원되어 있는지 확인.
9. 데모 재생 중 브라우저 새로고침을 시도(또는 실제로 새로고침) → 새로고침 후 워크북 상태가 데모 시작 전 값으로 복원되어 있는지 확인(`beforeunload` 동작 확인).

예상 결과: 콘솔 에러 없음, 위 9개 시나리오 전부 설계대로 동작.

---

## Self-Review 결과

- **Spec 커버리지**: 진입 버튼+배너 셸 → Task 1, 데모 스크립트 6단계 → Task 2 Step 1, 재생 엔진(하이라이트 재조회·타이핑 애니메이션·일시정지/다음) → Task 2 Step 2·3, 3중 복원 안전장치(닫기/`beforeunload`/`MutationObserver`) → Task 2 Step 3, 완료 상태에서 즉시 복원하지 않음 → `runWorkbookDemoStep`의 `index >= length` 분기, 에러 처리(대상 못 찾음/중복 실행) → `startWorkbookDemo`의 가드 + `abortWorkbookDemo`. 모두 커버됨.
- **Placeholder 스캔**: "TBD"/"적절히"/"필요시" 패턴 없음 — 모든 코드 블록이 실행 가능한 완성 코드.
- **타입/시그니처 일관성**: `WORKBOOK_DEMO_STEPS`의 `{caption, targetSelector, act, value?}` 셰이프를 `runWorkbookDemoStep`/`performWorkbookDemoAction`이 동일하게 사용. `workbookDemo` 상태 객체의 필드명(`running`/`paused`/`stepIndex`/`backupRaw`/`timerId`)이 모든 함수에서 일관됨. 기존 `loadWorkbookState`/`saveWorkbookState`/`renderWorkbookCanvas`/`escapeHtml`/`persist`/`state`/`workbookRunResults`는 시그니처 변경 없이 호출만 함(Global Constraints 준수).
- **스펙과의 의도적 차이 반영**: (1) 진입 버튼 위치를 "미션 탭 줄 오른쪽"에서 "미션 탭 줄 위 정적 바"로 조정 — 기존 `renderWorkbookMissionBar` 무수정 원칙을 완전히 지키기 위함. (2) 일시정지를 스텝 경계에서만 적용되도록 단순화 — 타이핑 도중 정밀 중단은 구현 복잡도 대비 학습 가치가 낮아 제외. 두 차이 모두 Global Constraints에 명시했고 기능적 목표(자동 재생 데모로 전체 흐름을 보여준다)는 그대로 달성함.

---

**Plan complete and saved to `docs/superpowers/plans/2026-08-07-workbook-demo.md`.** Two execution options:

**1. Subagent-Driven (recommended)** - 태스크별로 새 서브에이전트를 띄우고 태스크 사이마다 검토, 빠른 반복

**2. Inline Execution** - 이 세션에서 executing-plans로 배치 실행, 체크포인트마다 검토

**Which approach?**
