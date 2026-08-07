# Workbooks 구축 실습 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `azure-migration-tracker.html`의 "로컬 테스트 랩" 그룹에 "Workbooks 구축" 패널을 순수 추가하여, 텍스트/파라미터/쿼리/차트 4종 셀을 조합해 Azure Monitor 워크북을 직접 만들어보고, 미션 3개(난이도 순)의 구조적 체크리스트로 스스로 채점할 수 있게 한다.

**Architecture:** 워크북 셀 배열을 `state.lab["workbook-cells"]`에 저장하고, 구조 변경(셀 추가/삭제/순서변경/미션전환) 시에만 캔버스 전체를 `innerHTML`로 재렌더링한다(입력 이벤트는 상태만 갱신, 재렌더링 없음 — 포커스 유지). 쿼리 셀은 별도 실행 엔진 없이 기존 `runKustoQuery()`를 파라미터 토큰 치환 후 그대로 호출하는 재실행형이다. 차트 셀은 기존 `AM_METRIC_SERIES`를 순수 SVG `<polyline>`으로 그린다. 채점은 문자열 완전일치가 아니라 "이런 셀이 있고 이렇게 연결됐는가"를 판정하는 구조적 체크리스트다.

**Tech Stack:** 순수 HTML/CSS/바닐라 JS(빌드 도구 없음), 이벤트 위임(delegated click/input/change), `localStorage`(폴백: 메모리 객체, 기존 `persist()` 재사용).

## Global Constraints

- 기존 8개 패널·로컬 테스트 랩 6개 패널과 그 함수(`renderRefPage`, `buildFullQuiz`, `runKustoQuery`, `renderKustoResult`, `runApimMock`, `evaluateAlertRule`, `AM_GATEWAY_LOGS`, `AM_METRIC_SERIES`, `escapeHtml` 등)는 절대 수정하지 않는다 — 순수 추가(additive)만 한다. 기존 패널의 `eyebrow`(일련번호) 텍스트도 변경하지 않는다 — 새 패널은 이어지는 번호(18)를 새로 붙인다.
- 이 저장소는 git 저장소이며(현재 `master` 브랜치, 워크트리 없이 직접 작업) 빌드/테스트 러너는 없다 — 모든 "테스트" 단계는 브라우저에서 파일을 직접 열어 콘솔/UI로 수동 확인하는 시나리오다. 각 Task 완료 시 `git add`/`git commit`으로 커밋을 남긴다(기존 로컬 테스트 랩 SDD와 동일 관례).
- 파라미터 참조 문법은 공식 문서(`https://learn.microsoft.com/ko-kr/azure/azure-monitor/visualize/workbooks-parameters`)로 확인된 `{ParameterName}` 토큰 치환만 사용한다. 파라미터 종류는 드롭다운(문자열 값) 1종만 지원하고, Time Range 매개변수·JSONPath 서식(`{param:$.x}`)·형식 지정자(`:label`, `:base64` 등)는 이번 스코프에서 다루지 않는다.
- 워크북 쿼리 셀은 재실행형이다 — 실행 결과를 상태에 스냅샷으로 저장하지 않는다(`lastRunOk` 불리언만 저장). 화이트리스트는 신설하지 않고 기존 `runKustoQuery()`의 화이트리스트를 그대로 상속한다.
- 진단 설정(Diagnostic Settings) 활성화 실습, 실제 Azure Workbooks JSON 템플릿 import/export, 대시보드 Pin 기능은 이번 스코프에 포함하지 않는다(비목표).
- 채점은 자동 정답 문자열 비교가 아니라 구조적 체크리스트(셀 개수·파라미터 참조 여부·실행 성공 여부)로 판정한다.
- 파라미터 드롭다운의 고정 옵션은 `["orders-api", "users-api", "legacy-api"]` — 기존 `AM_GATEWAY_LOGS`에 실제 존재하는 `ApiId` 값과 일치시켜, 파라미터를 참조하는 쿼리가 실제로 결과를 반환하게 한다.

---

## Task 1: Nav 항목 + 패널 셸 + CSS 추가

**Files:**
- Modify: `azure-migration-tracker.html` (CSS는 `.alert-overlap-badge { ... }` 규칙 다음·`</style>` 직전, `NAV` 배열은 `lab` 그룹의 `children` 배열에서 `lab-alert` 항목 다음에 추가, 패널 HTML은 `panel-lab-alert`의 `</section>` 뒤·`.main`을 닫는 `</div>` 앞에 추가)

**Interfaces:**
- Produces: 패널 1개(`panel-lab-workbook`), 컨테이너 DOM id `labWorkbookBody` — Task 4가 이 id에 렌더링한다.

- [ ] **Step 1: CSS 추가**

`azure-migration-tracker.html`에서 `.alert-overlap-badge { font-size: var(--fs-xs); color: var(--text-dim); margin-bottom: var(--sp-2); }` 바로 다음, `</style>` 바로 앞에 추가:

```css
.workbook-toolbar { display: flex; flex-wrap: wrap; gap: var(--sp-2); margin-bottom: var(--sp-3); }
.workbook-mission-tabs { display: flex; flex-wrap: wrap; gap: var(--sp-2); margin-bottom: var(--sp-2); }
.workbook-mission-tabs button.active { border-color: var(--accent-target); color: var(--accent-target); }
.workbook-mission-card { border: 1px solid var(--border); border-radius: var(--radius-md); padding: var(--sp-3); margin: var(--sp-2) 0 var(--sp-4); background: var(--bg); }
.workbook-cell { border: 1px solid var(--border); border-radius: var(--radius-md); padding: var(--sp-3); margin-bottom: var(--sp-3); }
.workbook-cell-head { display: flex; justify-content: space-between; align-items: center; margin-bottom: var(--sp-2); font-size: var(--fs-xs); color: var(--text-dim); }
.workbook-cell-head .cell-actions button { margin-left: 4px; }
.workbook-text-input, .workbook-query-input { width: 100%; border: 1px solid var(--border); border-radius: var(--radius-md); padding: var(--sp-3); background: var(--bg); color: var(--text); font-family: inherit; }
.workbook-query-input { font-family: var(--font-mono); font-size: var(--fs-sm); min-height: 90px; resize: vertical; }
.workbook-param-select { border: 1px solid var(--border); border-radius: var(--radius-md); padding: 6px 10px; background: var(--bg); color: var(--text); margin-left: var(--sp-2); }
.workbook-chart-svg { width: 100%; height: auto; color: var(--text); }
.workbook-checklist-item { font-size: var(--fs-sm); padding: 4px 0; }
.workbook-checklist-item.pass { color: #1a7f37; }
.workbook-checklist-item.fail { color: #b91c1c; }
.workbook-model-answer { border: 1px dashed var(--border); border-radius: var(--radius-md); padding: var(--sp-3); margin-top: var(--sp-3); }
```

- [ ] **Step 2: `lab` 그룹 children에 새 항목 추가**

`NAV`의 `lab` 그룹 `children` 배열에서 다음 줄:

```js
        { id: "lab-alert", label: "알림 시뮬레이터", icon: '<path d="M6 8a6 6 0 1 1 12 0c0 7 3 9 3 9H3s3-2 3-9"/><path d="M10 21a2 2 0 0 0 4 0"/>' }
      ] }
```

을 다음으로 교체(쉼표 추가 + 새 항목):

```js
        { id: "lab-alert", label: "알림 시뮬레이터", icon: '<path d="M6 8a6 6 0 1 1 12 0c0 7 3 9 3 9H3s3-2 3-9"/><path d="M10 21a2 2 0 0 0 4 0"/>' },
        { id: "lab-workbook", label: "Workbooks 구축", icon: '<rect x="3" y="4" width="18" height="16" rx="1"/><path d="M7 9h10M7 13h6M7 17h4"/>' }
      ] }
```

- [ ] **Step 3: 패널 셸 HTML 추가**

`panel-lab-alert`의 `</section>` 바로 다음(즉 `</div>\n</div>\n\n<button class="btn theme-toggle"` 앞), 다음 HTML을 추가:

```html
    <section class="panel" id="panel-lab-workbook" data-panel>
      <div class="panel-head">
        <span class="eyebrow">18 · Local Test Lab</span>
        <h1>Workbooks 구축</h1>
        <p class="desc">텍스트·파라미터·쿼리·차트 셀을 직접 조합해 Azure Monitor 워크북을 만들어 봅니다. 쿼리 셀은 Kusto 실행기와 동일한 엔진을 재사용하며, 실행기 탭의 쿼리를 그대로 가져올 수 있습니다.</p>
      </div>
      <div id="labWorkbookBody"></div>
    </section>
```

- [ ] **Step 4: 브라우저에서 셸 확인**

`azure-migration-tracker.html`을 브라우저로 열어 왼쪽 nav의 "로컬 테스트 랩" 그룹 아래 "Workbooks 구축" 항목이 보이는지 확인. 클릭 시 빈 패널(제목/설명만 있고 본문은 비어 있음)로 전환되는지 확인. 기존 8개 탭과 기존 6개 로컬 테스트 랩 탭이 그대로 동작하는지 확인.
예상 결과: 콘솔 에러 없음, 새 탭이 빈 본문으로 정상 전환됨.

---

## Task 2: 데이터 모델 — 상태 저장/로드 + 파라미터 토큰 치환

**Files:**
- Modify: `azure-migration-tracker.html` (`initAlertLab();` 호출 뒤, `/* ---------------- jira draft ---------------- */` 주석 앞에 추가)

**Interfaces:**
- Consumes: `state`(기존 전역, `state.lab` 네임스페이스 객체), `persist()`(기존 함수) — 둘 다 시그니처 변경 없음.
- Produces: `WORKBOOK_PARAM_OPTIONS`(문자열 배열), `loadWorkbookState()` → `{ missionId: string, cells: Array<Cell> }`, `saveWorkbookState(wbState)`, `substituteWorkbookParams(queryText, cells)` → `{ error: false, text: string } | { error: true, message: string }` — Task 3·4가 이 함수들을 사용한다. `Cell` 셰이프: `{ id: string, type: "text"|"param"|"query"|"chart", value?: string, name?: string, lastRunOk?: boolean|null }`.

- [ ] **Step 1: 상태 저장/로드 + 파라미터 치환 함수 추가**

`initAlertLab();` 호출 바로 다음에 추가:

```js
  /* ---------------- Workbooks 구축: 상태 + 파라미터 치환 ---------------- */
  var WORKBOOK_PARAM_OPTIONS = ["orders-api", "users-api", "legacy-api"];
  var workbookCellSeq = 0;

  function loadWorkbookState() {
    var raw = state.lab["workbook-cells"];
    if (raw) {
      try { return JSON.parse(raw); } catch (e) { /* 저장값 손상 시 기본값 사용 */ }
    }
    return { missionId: "free", cells: [] };
  }

  function saveWorkbookState(wbState) {
    state.lab["workbook-cells"] = JSON.stringify(wbState);
    persist();
  }

  function substituteWorkbookParams(queryText, cells) {
    var paramCells = cells.filter(function (c) { return c.type === "param"; });
    var tokens = queryText.match(/\{[A-Za-z0-9_]+\}/g) || [];
    var result = queryText;
    for (var i = 0; i < tokens.length; i++) {
      var token = tokens[i];
      var name = token.slice(1, -1);
      var matches = paramCells.filter(function (c) { return c.name === name; });
      if (!matches.length) {
        return { error: true, message: "정의되지 않은 매개 변수 참조: " + token };
      }
      result = result.split(token).join(matches[0].value);
    }
    return { error: false, text: result };
  }
```

- [ ] **Step 2: 브라우저 콘솔에서 수동 검증**

`azure-migration-tracker.html`을 열고 개발자 도구 콘솔에서 실행:

```js
substituteWorkbookParams('ApiManagementGatewayLogs\n| where ApiId == "{ApiIdParam}"', [{ id: "c1", type: "param", name: "ApiIdParam", value: "orders-api" }])
```
예상 결과: `{ error: false, text: 'ApiManagementGatewayLogs\n| where ApiId == "orders-api"' }`

```js
substituteWorkbookParams('ApiManagementGatewayLogs\n| where ApiId == "{Foo}"', [{ id: "c1", type: "param", name: "ApiIdParam", value: "orders-api" }])
```
예상 결과: `{ error: true, message: "정의되지 않은 매개 변수 참조: {Foo}" }`

```js
saveWorkbookState({ missionId: "free", cells: [{ id: "c1", type: "text", value: "hi" }] });
loadWorkbookState();
```
예상 결과: `loadWorkbookState()`가 `{ missionId: "free", cells: [{ id: "c1", type: "text", value: "hi" }] }`를 반환.

---

## Task 3: 미션 정의(체크리스트·모범답안) + 차트 SVG 렌더러

**Files:**
- Modify: `azure-migration-tracker.html` (Task 2에서 추가한 `substituteWorkbookParams` 함수 뒤에 추가)

**Interfaces:**
- Consumes: `AM_METRIC_SERIES`(기존 전역, `[{t, value}, ...]` 60개), `escapeHtml`(기존 함수) — 둘 다 시그니처 변경 없음.
- Produces: `renderWorkbookChartSvg()` → SVG 문자열, `WORKBOOK_MISSIONS`(배열, 각 원소 `{ id, title, desc, check(cells) → { pass: boolean, items: [{label, pass}] }, modelAnswer: Array<{type, value?, name?}> }`) — Task 4가 렌더링·채점에 사용한다.

- [ ] **Step 1: 차트 SVG 렌더러 추가**

```js
  /* ---------------- Workbooks 구축: 차트 렌더러 + 미션 정의 ---------------- */
  function renderWorkbookChartSvg() {
    var values = AM_METRIC_SERIES.map(function (p) { return p.value; });
    var min = Math.min.apply(null, values);
    var max = Math.max.apply(null, values);
    var range = max - min || 1;
    var points = AM_METRIC_SERIES.map(function (p) {
      var x = (p.t / 59) * 600;
      var y = 110 - ((p.value - min) / range) * 105;
      return x.toFixed(1) + "," + y.toFixed(1);
    }).join(" ");
    return '<svg class="workbook-chart-svg" viewBox="0 0 600 136" xmlns="http://www.w3.org/2000/svg">' +
      '<polyline points="' + points + '" fill="none" stroke="currentColor" stroke-width="2"/>' +
      '<text x="2" y="12" font-size="10" fill="currentColor">' + max + 'ms</text>' +
      '<text x="2" y="108" font-size="10" fill="currentColor">' + min + 'ms</text>' +
      '<text x="0" y="128" font-size="10" fill="currentColor">0분</text>' +
      '<text x="560" y="128" font-size="10" fill="currentColor">59분</text>' +
      '</svg>';
  }
```

- [ ] **Step 2: 미션 3개 정의 추가**

Step 1의 `renderWorkbookChartSvg` 바로 다음에 추가:

```js
  var WORKBOOK_MISSIONS = [
    { id: "m1", title: "느린 요청 Top 5 리포트",
      desc: "쿼리 셀 1개로 orders-api의 가장 느린 요청 5건을 조회하는 워크북을 만드세요.",
      check: function (cells) {
        var queries = cells.filter(function (c) { return c.type === "query"; });
        var hasSuccess = queries.some(function (c) { return c.lastRunOk === true; });
        return { pass: queries.length >= 1 && hasSuccess,
          items: [
            { label: "쿼리 셀이 1개 이상 있다", pass: queries.length >= 1 },
            { label: "쿼리 셀 중 하나 이상 에러 없이 실행됐다", pass: hasSuccess }
          ] };
      },
      modelAnswer: [
        { type: "query", value: 'ApiManagementGatewayLogs\n| where ApiId == "orders-api"\n| top 5 by TotalTime desc' }
      ] },
    { id: "m2", title: "API별 필터링 리포트",
      desc: "파라미터 셀 1개(이름: ApiIdParam)와 그 값을 참조하는 쿼리 셀 1개로, 선택한 API의 요청만 보여주는 워크북을 만드세요.",
      check: function (cells) {
        var params = cells.filter(function (c) { return c.type === "param"; });
        var refQueries = cells.filter(function (c) {
          if (c.type !== "query") return false;
          return params.some(function (p) { return c.value.indexOf("{" + p.name + "}") !== -1; });
        });
        var hasSuccess = refQueries.some(function (c) { return c.lastRunOk === true; });
        return { pass: params.length >= 1 && refQueries.length >= 1 && hasSuccess,
          items: [
            { label: "파라미터 셀이 1개 이상 있다", pass: params.length >= 1 },
            { label: "쿼리 셀이 파라미터를 {파라미터명} 형태로 참조한다", pass: refQueries.length >= 1 },
            { label: "그 쿼리가 에러 없이 실행됐다", pass: hasSuccess }
          ] };
      },
      modelAnswer: [
        { type: "param", name: "ApiIdParam", value: "orders-api" },
        { type: "query", value: 'ApiManagementGatewayLogs\n| where ApiId == "{ApiIdParam}"\n| take 10' }
      ] },
    { id: "m3", title: "장애 조사용 종합 워크북",
      desc: "텍스트 설명 1개, 파라미터 셀 1개, 그 파라미터를 참조하는 쿼리 셀 1개, 차트 셀 1개를 모두 갖춘 워크북을 만드세요.",
      check: function (cells) {
        var m2Result = WORKBOOK_MISSIONS[1].check(cells);
        var texts = cells.filter(function (c) { return c.type === "text"; });
        var charts = cells.filter(function (c) { return c.type === "chart"; });
        return { pass: m2Result.pass && texts.length >= 1 && charts.length >= 1,
          items: m2Result.items.concat([
            { label: "텍스트 셀이 1개 이상 있다", pass: texts.length >= 1 },
            { label: "차트 셀이 1개 이상 있다", pass: charts.length >= 1 }
          ]) };
      },
      modelAnswer: [
        { type: "text", value: "응답시간 급증 구간을 조사하는 워크북입니다. API를 선택하면 아래 쿼리와 차트가 해당 데이터를 보여줍니다." },
        { type: "param", name: "ApiIdParam", value: "orders-api" },
        { type: "query", value: 'ApiManagementGatewayLogs\n| where ApiId == "{ApiIdParam}"\n| top 10 by TotalTime desc' },
        { type: "chart" }
      ] }
  ];
```

- [ ] **Step 3: 브라우저 콘솔에서 수동 검증**

```js
renderWorkbookChartSvg().indexOf("<polyline") !== -1
```
예상 결과: `true`

```js
WORKBOOK_MISSIONS[0].check([{ id: "c1", type: "query", value: "...", lastRunOk: true }])
```
예상 결과: `{ pass: true, items: [{label: "...", pass: true}, {label: "...", pass: true}] }`

```js
WORKBOOK_MISSIONS[1].check([{ id: "c1", type: "param", name: "ApiIdParam", value: "orders-api" }, { id: "c2", type: "query", value: 'where ApiId == "{ApiIdParam}"', lastRunOk: false }])
```
예상 결과: `pass: false`(파라미터 참조는 있지만 실행 성공이 아직 없음), `items` 3개 중 세 번째(실행 성공)만 `pass: false`.

```js
WORKBOOK_MISSIONS[2].check([])
```
예상 결과: `pass: false`, `items` 5개 전부 `pass: false`(빈 배열이므로 m2 조건도 전부 불충족).

---

## Task 4: 캔버스 렌더링 + 셀 CRUD + Kusto 실행기 연동 + 체크리스트 UI

**Files:**
- Modify: `azure-migration-tracker.html` (Task 3의 `WORKBOOK_MISSIONS` 정의 뒤에 추가)

**Interfaces:**
- Consumes: `loadWorkbookState`/`saveWorkbookState`/`substituteWorkbookParams`/`WORKBOOK_PARAM_OPTIONS`(Task 2), `renderWorkbookChartSvg`/`WORKBOOK_MISSIONS`(Task 3), `runKustoQuery(queryText)`/`renderKustoResult(result)`(기존, `azure-migration-tracker.html` 기존 Kusto 실행기 코드), `escapeHtml`(기존), `state.lab["kusto-editor"]`(기존, Kusto 실행기 탭의 현재 쿼리 텍스트), `labWorkbookBody`(Task 1의 컨테이너 id).

- [ ] **Step 1: 셀 렌더링 함수 + 캔버스 렌더링 함수 추가**

Task 3에서 추가한 `WORKBOOK_MISSIONS` 정의 바로 다음에 추가:

```js
  /* ---------------- Workbooks 구축: 캔버스 렌더링 ---------------- */
  var WORKBOOK_CELL_LABELS = { text: "텍스트", param: "파라미터", query: "쿼리", chart: "차트" };
  var workbookRunResults = {};

  function renderWorkbookCellHtml(cell, idx, total) {
    var head = '<div class="workbook-cell-head"><span>' + WORKBOOK_CELL_LABELS[cell.type] + '</span><span class="cell-actions">' +
      '<button class="btn" type="button" data-action="up" data-cell-id="' + cell.id + '"' + (idx === 0 ? " disabled" : "") + '>↑</button>' +
      '<button class="btn" type="button" data-action="down" data-cell-id="' + cell.id + '"' + (idx === total - 1 ? " disabled" : "") + '>↓</button>' +
      '<button class="btn" type="button" data-action="delete" data-cell-id="' + cell.id + '">삭제</button>' +
      '</span></div>';
    var body = "";
    if (cell.type === "text") {
      body = '<textarea class="workbook-text-input" data-field="value" data-cell-id="' + cell.id + '" rows="2">' + escapeHtml(cell.value || "") + '</textarea>';
    } else if (cell.type === "param") {
      var opts = WORKBOOK_PARAM_OPTIONS.map(function (o) {
        return '<option value="' + escapeHtml(o) + '"' + (o === cell.value ? " selected" : "") + '>' + escapeHtml(o) + '</option>';
      }).join("");
      body = '<label class="pwhen">파라미터명 <input type="text" class="workbook-text-input" style="display:inline-block;width:auto;" data-field="name" data-cell-id="' + cell.id + '" value="' + escapeHtml(cell.name || "") + '"></label>' +
        '<select class="workbook-param-select" data-field="value" data-cell-id="' + cell.id + '">' + opts + '</select>';
    } else if (cell.type === "query") {
      var resultHtml = workbookRunResults[cell.id] || "";
      body = '<textarea class="workbook-query-input" data-field="value" data-cell-id="' + cell.id + '" placeholder="ApiManagementGatewayLogs\n| where ...">' + escapeHtml(cell.value || "") + '</textarea>' +
        '<div class="quiz-actions" style="margin-top:var(--sp-2);"><button class="btn primary" type="button" data-action="run" data-cell-id="' + cell.id + '">실행</button></div>' +
        '<div id="workbookQueryResult-' + cell.id + '">' + resultHtml + '</div>';
    } else if (cell.type === "chart") {
      body = renderWorkbookChartSvg();
    }
    return '<div class="workbook-cell" data-cell="' + cell.id + '">' + head + body + '</div>';
  }

  function renderWorkbookMissionBar(wbState) {
    var tabs = [{ id: "free", title: "자유 캔버스" }].concat(WORKBOOK_MISSIONS.map(function (m) { return { id: m.id, title: m.title }; }));
    return '<div class="workbook-mission-tabs">' + tabs.map(function (t) {
      return '<button class="btn' + (t.id === wbState.missionId ? " active" : "") + '" type="button" data-action="switch-mission" data-mission-id="' + t.id + '">' + escapeHtml(t.title) + '</button>';
    }).join("") + '</div><p class="empty-hint">탭을 전환하면 현재 캔버스가 초기화됩니다.</p>';
  }

  function renderWorkbookMissionCard(mission) {
    if (!mission) return "";
    return '<div class="workbook-mission-card"><h3 style="font-size:var(--fs-md);margin-bottom:var(--sp-2);">' + escapeHtml(mission.title) + '</h3>' +
      '<p class="quiz-scenario">' + escapeHtml(mission.desc) + '</p>' +
      '<div class="quiz-actions"><button class="btn" type="button" data-action="check-mission">체크리스트 확인</button></div>' +
      '<div id="workbookChecklistArea"></div></div>';
  }

  function renderWorkbookToolbar() {
    return '<div class="workbook-toolbar">' +
      '<button class="btn" type="button" data-action="add-cell" data-cell-type="text">+ 텍스트</button>' +
      '<button class="btn" type="button" data-action="add-cell" data-cell-type="param">+ 파라미터</button>' +
      '<button class="btn" type="button" data-action="add-cell" data-cell-type="query">+ 쿼리</button>' +
      '<button class="btn" type="button" data-action="add-cell" data-cell-type="chart">+ 차트</button>' +
      '<button class="btn" type="button" data-action="import-kusto">Kusto 실행기 탭 쿼리 가져오기</button>' +
      '</div><div id="workbookImportNote"></div>';
  }

  function renderWorkbookCanvas() {
    var wbState = loadWorkbookState();
    var missionMatches = WORKBOOK_MISSIONS.filter(function (m) { return m.id === wbState.missionId; });
    var mission = missionMatches.length ? missionMatches[0] : null;
    var cellsHtml = wbState.cells.length
      ? wbState.cells.map(function (cell, idx) { return renderWorkbookCellHtml(cell, idx, wbState.cells.length); }).join("")
      : '<p class="empty-hint">아직 셀이 없습니다. 위 버튼으로 셀을 추가하세요.</p>';
    document.getElementById("labWorkbookBody").innerHTML =
      renderWorkbookMissionBar(wbState) +
      renderWorkbookMissionCard(mission) +
      renderWorkbookToolbar() +
      '<div id="workbookCanvas">' + cellsHtml + '</div>';
  }
```

- [ ] **Step 2: 체크리스트/모범답안 렌더러 + 쿼리 실행 핸들러 추가**

Step 1의 `renderWorkbookCanvas` 바로 다음에 추가:

```js
  function renderWorkbookChecklist(mission, result) {
    var itemsHtml = result.items.map(function (it) {
      return '<div class="workbook-checklist-item ' + (it.pass ? "pass" : "fail") + '">' + (it.pass ? "✅ " : "❌ ") + escapeHtml(it.label) + '</div>';
    }).join("");
    var passHtml = result.pass
      ? '<div class="quiz-actions" style="margin-top:var(--sp-2);"><button class="btn primary" type="button" data-action="show-model-answer">모범 답안 구성 보기</button></div>'
      : "";
    document.getElementById("workbookChecklistArea").innerHTML = itemsHtml + passHtml + '<div id="workbookModelAnswerArea"></div>';
  }

  function renderWorkbookModelAnswer(mission) {
    var html = mission.modelAnswer.map(function (cell) {
      if (cell.type === "text") return '<div class="workbook-cell"><div class="workbook-cell-head"><span>텍스트</span></div><p>' + escapeHtml(cell.value) + '</p></div>';
      if (cell.type === "param") return '<div class="workbook-cell"><div class="workbook-cell-head"><span>파라미터</span></div><p>이름: <code>' + escapeHtml(cell.name) + '</code>, 값: <code>' + escapeHtml(cell.value) + '</code></p></div>';
      if (cell.type === "query") return '<div class="workbook-cell"><div class="workbook-cell-head"><span>쿼리</span></div><div class="code-block">' + escapeHtml(cell.value) + '</div></div>';
      if (cell.type === "chart") return '<div class="workbook-cell"><div class="workbook-cell-head"><span>차트</span></div>' + renderWorkbookChartSvg() + '</div>';
      return "";
    }).join("");
    var area = document.getElementById("workbookModelAnswerArea");
    if (area) area.innerHTML = '<div class="workbook-model-answer">' + html + '</div>';
  }

  function runWorkbookQueryCell(cellId) {
    var wbState = loadWorkbookState();
    var matches = wbState.cells.filter(function (c) { return c.id === cellId; });
    if (!matches.length) return;
    var cell = matches[0];
    var resultArea = document.getElementById("workbookQueryResult-" + cellId);
    var sub = substituteWorkbookParams(cell.value || "", wbState.cells);
    var html;
    if (sub.error) {
      cell.lastRunOk = false;
      html = '<div class="kusto-error">' + escapeHtml(sub.message) + '</div>';
    } else {
      var result = runKustoQuery(sub.text);
      cell.lastRunOk = !result.error;
      html = renderKustoResult(result);
    }
    workbookRunResults[cellId] = html;
    if (resultArea) resultArea.innerHTML = html;
    saveWorkbookState(wbState);
  }
```

- [ ] **Step 3: 이벤트 위임 + 초기화 함수 추가**

Step 2의 `runWorkbookQueryCell` 바로 다음에 추가:

```js
  function handleWorkbookFieldChange(e) {
    var field = e.target.getAttribute("data-field");
    if (!field) return;
    var cellId = e.target.getAttribute("data-cell-id");
    var wbState = loadWorkbookState();
    var matches = wbState.cells.filter(function (c) { return c.id === cellId; });
    if (!matches.length) return;
    matches[0][field] = e.target.value;
    saveWorkbookState(wbState);
  }

  function initWorkbookLab() {
    renderWorkbookCanvas();
    var body = document.getElementById("labWorkbookBody");

    body.addEventListener("input", handleWorkbookFieldChange);
    body.addEventListener("change", handleWorkbookFieldChange);

    body.addEventListener("click", function (e) {
      var btn = e.target.closest("button[data-action]");
      if (!btn) return;
      var action = btn.getAttribute("data-action");
      var wbState = loadWorkbookState();

      if (action === "switch-mission") {
        wbState.missionId = btn.getAttribute("data-mission-id");
        wbState.cells = [];
        workbookRunResults = {};
        saveWorkbookState(wbState);
        renderWorkbookCanvas();
        return;
      }
      if (action === "add-cell") {
        var type = btn.getAttribute("data-cell-type");
        workbookCellSeq++;
        var cell = { id: "wc" + workbookCellSeq, type: type };
        if (type === "text") cell.value = "";
        if (type === "param") { cell.name = "ApiIdParam"; cell.value = WORKBOOK_PARAM_OPTIONS[0]; }
        if (type === "query") { cell.value = ""; cell.lastRunOk = null; }
        wbState.cells.push(cell);
        saveWorkbookState(wbState);
        renderWorkbookCanvas();
        return;
      }
      if (action === "import-kusto") {
        var kustoText = state.lab["kusto-editor"];
        var note = document.getElementById("workbookImportNote");
        if (!kustoText || !kustoText.trim()) {
          if (note) note.innerHTML = '<p class="empty-hint">Kusto 실행기 탭에서 먼저 쿼리를 작성하세요.</p>';
          return;
        }
        if (note) note.innerHTML = "";
        workbookCellSeq++;
        wbState.cells.push({ id: "wc" + workbookCellSeq, type: "query", value: kustoText, lastRunOk: null });
        saveWorkbookState(wbState);
        renderWorkbookCanvas();
        return;
      }
      if (action === "check-mission") {
        var missionForCheck = WORKBOOK_MISSIONS.filter(function (m) { return m.id === wbState.missionId; })[0];
        if (!missionForCheck) return;
        if (!wbState.cells.length) {
          document.getElementById("workbookChecklistArea").innerHTML = '<p class="empty-hint">셀을 먼저 추가하세요.</p>';
          return;
        }
        renderWorkbookChecklist(missionForCheck, missionForCheck.check(wbState.cells));
        return;
      }
      if (action === "show-model-answer") {
        var missionForAnswer = WORKBOOK_MISSIONS.filter(function (m) { return m.id === wbState.missionId; })[0];
        if (missionForAnswer) renderWorkbookModelAnswer(missionForAnswer);
        return;
      }

      var cellId = btn.getAttribute("data-cell-id");
      if (!cellId) return;
      if (action === "delete") {
        wbState.cells = wbState.cells.filter(function (c) { return c.id !== cellId; });
        delete workbookRunResults[cellId];
        saveWorkbookState(wbState);
        renderWorkbookCanvas();
        return;
      }
      if (action === "up" || action === "down") {
        var idx = -1;
        for (var i = 0; i < wbState.cells.length; i++) { if (wbState.cells[i].id === cellId) { idx = i; break; } }
        var swapWith = action === "up" ? idx - 1 : idx + 1;
        if (idx === -1 || swapWith < 0 || swapWith >= wbState.cells.length) return;
        var tmp = wbState.cells[idx];
        wbState.cells[idx] = wbState.cells[swapWith];
        wbState.cells[swapWith] = tmp;
        saveWorkbookState(wbState);
        renderWorkbookCanvas();
        return;
      }
      if (action === "run") {
        runWorkbookQueryCell(cellId);
      }
    });
  }
  initWorkbookLab();
```

- [ ] **Step 4: 브라우저에서 전체 시나리오 확인**

`azure-migration-tracker.html`을 열고 "Workbooks 구축" 탭에서 아래를 순서대로 확인:

1. **미션 1**: "느린 요청 Top 5 리포트" 탭 선택 → `+ 쿼리` 클릭 → 편집창에 `ApiManagementGatewayLogs\n| where ApiId == "orders-api"\n| top 5 by TotalTime desc` 입력 → `실행` 클릭 → 표 5행 렌더링 확인 → `체크리스트 확인` 클릭 → 두 항목 모두 ✅ → `모범 답안 구성 보기` 클릭 → 모범 답안 카드 표시 확인.
2. **미션 2**: "API별 필터링 리포트" 탭 선택(캔버스가 비워지는지 확인) → `+ 파라미터` 클릭 → 드롭다운에서 `users-api` 선택 → `+ 쿼리` 클릭 → `ApiManagementGatewayLogs\n| where ApiId == "{ApiIdParam}"\n| take 10` 입력 → `실행` → 결과가 `users-api` 행만 나오는지 확인 → `체크리스트 확인` → 3항목 모두 ✅.
3. **오류 케이스**: 쿼리 셀에 `{Foo}`처럼 존재하지 않는 파라미터를 참조하는 쿼리를 넣고 `실행` → "정의되지 않은 매개 변수 참조: {Foo}" 에러가 표시되는지 확인.
4. **Kusto 가져오기**: "Kusto 실행기" 탭으로 이동해 아무 쿼리나 입력 → "Workbooks 구축" 탭으로 돌아와 `Kusto 실행기 탭 쿼리 가져오기` 클릭 → 새 쿼리 셀에 그 텍스트가 그대로 채워지는지 확인. Kusto 실행기 편집창을 비운 뒤 다시 가져오기를 누르면 "먼저 쿼리를 작성하세요" 안내가 뜨는지 확인.
5. **미션 3**: 텍스트+파라미터+쿼리(파라미터 참조, 실행 성공)+차트 4개 셀을 모두 추가 → 차트 셀에 실제 꺾은선 SVG가 렌더링되는지 확인 → `체크리스트 확인` → 5항목 모두 ✅.
6. **순서 변경/삭제**: 셀의 `↑`/`↓`/`삭제` 버튼이 정상 동작하는지, 첫 셀의 `↑`와 마지막 셀의 `↓`가 비활성화되는지 확인.
7. **빈 상태 체크리스트**: 자유 캔버스에서 미션 탭으로 전환 직후(셀 0개) `체크리스트 확인`을 누르면 "셀을 먼저 추가하세요" 안내가 뜨는지 확인(에러 아님).
8. 새로고침 후 마지막 미션/셀 구성이 유지되는지 확인(localStorage).

예상 결과: 콘솔 에러 없음, 위 8개 시나리오 전부 설계대로 동작.

---

## Self-Review 결과

- **Spec 커버리지**: 배경/목적(로컬 테스트 랩 7번째 항목, Kusto 실행기 연동) → Task 1·4, 셀 4종(텍스트/파라미터/쿼리/차트) → Task 4 Step 1, 파라미터 `{ParameterName}` 토큰 치환(공식 문법) → Task 2, 재실행형 쿼리 셀 → Task 4 `runWorkbookQueryCell`, SVG 차트 → Task 3 Step 1, 미션 3개(난이도순) + 구조적 체크리스트 + 모범답안 → Task 3 Step 2 + Task 4 Step 2, Kusto 실행기 연동("가져오기") → Task 4 Step 3의 `import-kusto` 액션, 네비게이션 변경 → Task 1, 에러 처리(미정의 파라미터/빈 캔버스 체크리스트/localStorage 폴백) → Task 2·4에 반영. 비목표(진단 설정, JSON import/export, 대시보드 Pin, Time Range 파라미터)는 Global Constraints에 명시해 어떤 Task에도 포함하지 않음. 모두 커버됨.
- **Placeholder 스캔**: "TBD"/"적절히"/"필요시" 패턴 없음 — 모든 코드 블록이 실행 가능한 완성 코드.
- **타입/시그니처 일관성**: `Cell` 셰이프(`id, type, value?, name?, lastRunOk?`)를 Task 2(정의)·Task 3(`check`/`modelAnswer`가 소비)·Task 4(렌더링·CRUD)가 동일하게 사용. `substituteWorkbookParams`의 반환 `{error, text|message}`를 Task 4의 `runWorkbookQueryCell`이 그대로 분기 처리. `WORKBOOK_MISSIONS[i].check(cells)`의 반환 `{pass, items:[{label,pass}]}`를 Task 4의 `renderWorkbookChecklist`가 그대로 사용. 기존 `runKustoQuery`/`renderKustoResult`/`escapeHtml`/`state.lab`/`persist` 시그니처는 그대로 소비만 하고 변경하지 않음(Global Constraints 준수).
- **잠재 이슈 수정**: 초안에서는 Kusto 가져오기 버튼을 렌더링 시점에 `disabled` 처리하려 했으나, 캔버스 재렌더링이 구조 변경 시에만 일어나는 아키텍처상 "Kusto 탭에 방금 입력했지만 워크북 캔버스는 아직 그 사실을 모르는" 상태 불일치가 생길 수 있었다. 이를 클릭 시점 검사(`import-kusto` 액션 안에서 빈 값 여부를 그때 확인)로 바꿔 상태 불일치 가능성을 제거했다.

---

**Plan complete and saved to `docs/superpowers/plans/2026-08-07-workbook-lab.md`.** Two execution options:

**1. Subagent-Driven (recommended)** - 태스크별로 새 서브에이전트를 띄우고 태스크 사이마다 검토, 빠른 반복

**2. Inline Execution** - 이 세션에서 executing-plans로 배치 실행, 체크포인트마다 검토

**Which approach?**
