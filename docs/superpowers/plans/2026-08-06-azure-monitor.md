# Azure Monitor 학습 콘텐츠 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `azure-migration-tracker.html`에 "Azure Monitor" 레퍼런스/퀴즈 그룹과 "로컬 테스트 랩" 그룹에 "Kusto 실행기"/"알림 시뮬레이터" 패널 2개를 순수 추가하여, Log Analytics/Kusto/알림 규칙/액션 그룹/워크북·대시보드/진단 설정을 학습하고 로컬에서 직접 실행해 검증할 수 있게 한다.

**Architecture:** 레퍼런스/퀴즈는 기존 `renderRefPage`/`buildFullQuiz` 공용 함수를 그대로 재사용(새 데이터 배열만 추가). Kusto 실행기는 파이프(`|`) 스테이지를 순차 적용하는 미니 인터프리터(`runKustoQuery`)가 정적 가상 테이블(`AM_GATEWAY_LOGS`)에 대해 동작한다. 알림 시뮬레이터는 슬라이딩 윈도우 평가 함수(`evaluateAlertRule`)가 정적 메트릭 시계열(`AM_METRIC_SERIES`)을 평가해 Fired/Resolved 타임라인을 만든다.

**Tech Stack:** 순수 HTML/CSS/바닐라 JS(빌드 도구 없음), 정규식 기반 파서, `localStorage`(폴백: 메모리 객체).

## Global Constraints

- 기존 8개 패널 + 로컬 테스트 랩 5개 패널과 그 함수(`renderRefPage`, `buildFullQuiz`, `refUrl`, `runKongMock`, `runApimMock`, `LAB_ROUTES`, `NAV`의 기존 항목 등)는 절대 수정하지 않는다 — 순수 추가(additive)만 한다. 기존 패널의 `eyebrow`(일련번호) 텍스트도 변경하지 않는다 — 새 패널은 이어지는 번호(14~17)를 새로 붙인다.
- 이 저장소는 git 저장소이며(현재 `master` 브랜치, 별도 워크트리 없이 직접 작업) 빌드/테스트 러너는 없다 — 모든 "테스트" 단계는 브라우저에서 파일을 직접 열어 콘솔/UI로 수동 확인하는 시나리오다. 각 Task 완료 시 `git add`/`git commit`으로 커밋을 남긴다(과거 로컬 테스트 랩 SDD와 동일 관례).
- Kusto 예시(레퍼런스·퀴즈·실행기 전부)는 `docs/superpowers/specs/2026-08-06-azure-monitor-design.md`에 정리된, Microsoft Learn 공식 문서로 검증된 문법만 사용한다.
- `join` 연산자는 레퍼런스/퀴즈에만 등장하고, `runKustoQuery`의 화이트리스트에는 포함하지 않는다. 화이트리스트 밖 연산자를 만나면 조용히 무시하지 않고 몇 번째 스테이지에서 왜 실패했는지 명시한다.
- 알림 시뮬레이터는 정적 임계값 + 메트릭 시그널만 지원한다(동적 임계값/로그 검색 알림은 레퍼런스에서만 개념으로 다룸).
- "KQL"이라는 약어가 기존 KQL(Kibana Query Language) 퀴즈 패널과 혼동되지 않도록, 새 패널 라벨·헤더에 "Kusto(Azure Monitor)"를 명시한다.

---

## Task 1: Nav 그룹 + 패널 셸 + CSS 추가

**Files:**
- Modify: `azure-migration-tracker.html` (CSS는 `.lab-route-tabs button.active { ... }` 규칙 다음·`</style>` 직전, `NAV` 배열은 `datainfra` 그룹 다음·`jira` 항목 앞에 새 그룹 삽입 + `lab` 그룹의 `children` 배열 끝에 항목 2개 추가, `defaultState()`의 `navOpen`에 `azuremonitor: true` 추가, 패널 HTML은 `panel-lab-migration`의 `</section>` 뒤·`.main` 닫는 `</div>` 앞에 4개 패널 추가)

**Interfaces:**
- Produces: 패널 4개 (`panel-azuremonitor-ref`, `panel-azuremonitor-quiz`, `panel-lab-kusto`, `panel-lab-alert`), 컨테이너 DOM id(`azuremonitorToc`, `azuremonitorRefBody`, `azuremonitorQuizList`, `labKustoBody`, `labAlertBody`) — Task 2·4·6이 이 id들에 렌더링한다.

- [ ] **Step 1: CSS 추가**

`azure-migration-tracker.html`에서 `.lab-route-tabs button.active { border-color: var(--accent-target); color: var(--accent-target); }` 바로 다음, `</style>` 바로 앞에 추가:

```css
.kusto-editor { width: 100%; min-height: 120px; font-family: var(--font-mono); font-size: var(--fs-sm); border: 1px solid var(--border); border-radius: var(--radius-md); padding: var(--sp-3); background: var(--bg); color: var(--text); resize: vertical; }
.kusto-table-scroll { overflow-x: auto; margin-top: var(--sp-3); }
.kusto-table { border-collapse: collapse; width: 100%; font-size: var(--fs-xs); }
.kusto-table th, .kusto-table td { border: 1px solid var(--border); padding: 6px 10px; text-align: left; white-space: nowrap; }
.kusto-table th { background: var(--bg-alt, var(--bg)); font-weight: 700; }
.kusto-error { color: #b91c1c; font-size: var(--fs-sm); margin-top: var(--sp-2); }
.kusto-preset-grid { display: grid; gap: var(--sp-3); margin-bottom: var(--sp-4); }
.alert-form-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(160px, 1fr)); gap: var(--sp-3); margin-bottom: var(--sp-4); }
.alert-form-grid label { font-size: var(--fs-xs); color: var(--text-faint); display: flex; flex-direction: column; gap: 4px; }
.alert-form-grid input, .alert-form-grid select { border: 1px solid var(--border); border-radius: var(--radius-md); padding: 6px 10px; background: var(--bg); color: var(--text); }
.alert-timeline { display: flex; flex-wrap: wrap; gap: 6px; margin-top: var(--sp-3); }
.alert-point { font-size: var(--fs-xs); border: 1px solid var(--border); border-radius: var(--radius-md); padding: 4px 8px; font-family: var(--font-mono); }
.alert-point.breached { border-color: #b91c1c; color: #b91c1c; }
.alert-point.fired { background: #b91c1c; color: #fff; }
.alert-point.resolved { background: #1a7f37; color: #fff; }
.alert-overlap-badge { font-size: var(--fs-xs); color: var(--text-dim); margin-bottom: var(--sp-2); }
```

- [ ] **Step 2: NAV 배열에 Azure Monitor 그룹 삽입**

`NAV` 배열에서 `datainfra` 그룹의 닫는 `] },` 바로 다음, `{ id: "jira", ... }` 바로 앞에 삽입:

```js
    { id: "azuremonitor", label: "Azure Monitor", icon: '<rect x="3" y="4" width="18" height="12" rx="1"/><path d="M8 20h8M12 16v4"/>', children: [
        { id: "azuremonitor-ref", label: "Azure Monitor 레퍼런스", icon: '<path d="M6 4h9l5 5v11H6z"/><path d="M15 4v5h5"/><path d="M9 12h6M9 15.5h6"/>' },
        { id: "azuremonitor-quiz", label: "Azure Monitor 퀴즈", icon: '<path d="M9 9a3 3 0 1 1 3 3v2"/><circle cx="12" cy="17" r=".6" fill="currentColor" stroke="none"/>' }
      ] },
```

- [ ] **Step 3: `lab` 그룹 children에 새 패널 2개 추가**

`NAV`의 `lab` 그룹 `children` 배열에서 `{ id: "lab-migration", ... }` 바로 다음, 그 배열을 닫는 `] }` 바로 앞에 추가:

```js
        , { id: "lab-kusto", label: "Kusto 실행기", icon: '<rect x="3" y="4" width="18" height="16" rx="1"/><path d="M7 9l3 3-3 3M13 15h4"/>' },
        { id: "lab-alert", label: "알림 시뮬레이터", icon: '<path d="M6 8a6 6 0 1 1 12 0c0 7 3 9 3 9H3s3-2 3-9"/><path d="M10 21a2 2 0 0 0 4 0"/>' }
```

- [ ] **Step 4: `defaultState()`에 navOpen 항목 추가**

`defaultState()`의 `navOpen: { apim: true, kqlgroup: true, datainfra: true, lab: true }`를 다음으로 교체:

```js
navOpen: { apim: true, kqlgroup: true, datainfra: true, azuremonitor: true, lab: true }
```

- [ ] **Step 5: 패널 셸 HTML 추가**

`panel-lab-migration`의 `</section>` 바로 다음, `.main`을 닫는 `</div>` 바로 앞에 추가:

```html
    <section class="panel" id="panel-azuremonitor-ref" data-panel>
      <div class="panel-head apim-ref-head">
        <div class="htext">
          <span class="eyebrow">14 · Reference</span>
          <h1>Azure Monitor 레퍼런스</h1>
          <p class="desc">Log Analytics 테이블 스키마, KQL(Kusto) 연산자, 알림 규칙, 액션 그룹, 워크북/대시보드, 진단 설정을 공식 문서 기준으로 정리했습니다. <strong>주의:</strong> 여기서 다루는 KQL은 앞선 "KQL 레퍼런스"의 Kusto Query Language와 같은 언어입니다(Kibana Query Language 아님) — Azure Monitor 데이터에 실제로 적용하는 예시로 재구성했습니다.</p>
        </div>
        <a class="top-doc-link" href="https://learn.microsoft.com/ko-kr/azure/azure-monitor/overview" target="_blank" rel="noopener">공식 문서 전체 보기 ↗</a>
      </div>
      <nav class="apim-toc" id="azuremonitorToc" aria-label="카테고리 바로가기"></nav>
      <div id="azuremonitorRefBody"></div>
    </section>

    <section class="panel" id="panel-azuremonitor-quiz" data-panel>
      <div class="panel-head">
        <span class="eyebrow">15 · Active Recall</span>
        <h1>Azure Monitor 퀴즈</h1>
        <p class="desc">레퍼런스에 실린 항목을 설명만 보고 먼저 떠올려 본 뒤 정답과 비교하세요.</p>
      </div>
      <button class="btn goto-ref-btn" type="button" id="gotoAzureMonitorRef">← Azure Monitor 레퍼런스에서 전체 목록 먼저 보기</button>
      <div id="azuremonitorQuizList"></div>
    </section>

    <section class="panel" id="panel-lab-kusto" data-panel>
      <div class="panel-head">
        <span class="eyebrow">16 · Local Test Lab</span>
        <h1>Kusto 실행기</h1>
        <p class="desc">Kusto(Azure Monitor) 쿼리를 직접 작성해 로컬 가상 <code>ApiManagementGatewayLogs</code> 테이블에 실행합니다. 화이트리스트 연산자(where/project/summarize/top/sort/take/render)만 지원하며, <code>join</code> 등 그 외 연산자는 실행하지 않고 명확히 안내합니다.</p>
      </div>
      <div id="labKustoBody"></div>
    </section>

    <section class="panel" id="panel-lab-alert" data-panel>
      <div class="panel-head">
        <span class="eyebrow">17 · Local Test Lab</span>
        <h1>알림 시뮬레이터</h1>
        <p class="desc">샘플 응답 시간 메트릭 시계열에 임계값·평가 빈도·조회 기간(lookback)을 설정해 실제 Azure Monitor 메트릭 알림처럼 평가해 봅니다. 정적 임계값만 지원합니다.</p>
      </div>
      <div id="labAlertBody"></div>
    </section>
```

- [ ] **Step 6: 브라우저에서 셸 확인**

`azure-migration-tracker.html`을 브라우저로 열어 왼쪽 nav에 "Azure Monitor"(레퍼런스/퀴즈 2개 하위 항목) 그룹과 "로컬 테스트 랩" 그룹 아래 "Kusto 실행기"/"알림 시뮬레이터"가 보이는지 확인. 4개 탭 모두 클릭했을 때 빈 패널(제목/설명만 있고 본문은 비어 있음)로 전환되는지 확인. 기존 8개 탭과 기존 5개 로컬 테스트 랩 탭이 그대로 동작하는지 확인.
예상 결과: 콘솔 에러 없음, 새 탭 4개가 빈 본문으로 정상 전환됨.

---

## Task 2: Azure Monitor 레퍼런스 데이터 + 레퍼런스/퀴즈 렌더링 연결

**Files:**
- Modify: `azure-migration-tracker.html` (`DATA_INFRA` 배열 정의 뒤, `renderRefPage("apimToc", ...)` 등 기존 호출부 근처에 데이터 배열 정의 + 호출 3줄 추가)

**Interfaces:**
- Consumes: `renderRefPage(tocId, bodyId, groups, docBase)`, `buildFullQuiz(containerId, groups, storeKey, docBase)`, `refUrl(it, docBase)` — 기존 함수, 시그니처 변경 없음.
- Produces: `AZURE_MONITOR_REF` (groups 셰이프: `{ cat, anchor, items: [{ name, code, url, easy, when, example }] }`) — Task 3의 `AM_GATEWAY_LOGS` 서술 내용과 정합성 유지(같은 테이블/컬럼명 사용).

- [ ] **Step 1: 데이터 배열 추가**

`DATA_INFRA` 배열을 닫는 `];` 바로 다음(즉 `renderRefPage("apimToc", ...)` 호출부 바로 앞)에 추가:

```js
  var AZURE_MONITOR_REF = [
    { cat: "Log Analytics 테이블 스키마", anchor: "am-tables", items: [
      { name: "AppRequests 테이블", code: "AppRequests", url: "https://learn.microsoft.com/ko-kr/azure/azure-monitor/reference/tables/apprequests",
        easy: "Application Insights가 수집하는 HTTP 요청 로그입니다. 요청 이름(Name), 처리 시간(DurationMs), 성공 여부(Success), 결과 코드(ResultCode), 작업 ID(OperationId) 등을 담습니다.",
        when: "특정 앱의 요청 성공률/응답 시간을 조회하고 싶을 때",
        example: "AppRequests\n| where TimeGenerated > ago(1h)\n| where Success == false\n| project Name, ResultCode, DurationMs" },
      { name: "AppExceptions 테이블", code: "AppExceptions", url: "https://learn.microsoft.com/ko-kr/azure/azure-monitor/reference/tables/appexceptions",
        easy: "앱에서 발생한 예외(Exception)를 기록합니다. 예외 타입(ExceptionType), 메시지(Message), 심각도(SeverityLevel), 관련 작업 ID(OperationId)를 담습니다.",
        when: "에러 로그를 원인별로 집계하거나 특정 요청과 연결된 예외를 추적할 때",
        example: 'AppExceptions\n| where TimeGenerated > ago(1h)\n| summarize count() by ExceptionType' },
      { name: "AppDependencies 테이블", code: "AppDependencies", url: "https://learn.microsoft.com/ko-kr/azure/azure-monitor/reference/tables/appdependencies",
        easy: "앱이 호출한 외부 의존성(DB, 다른 API 등) 호출 결과를 기록합니다. 의존성 종류(DependencyType), 대상(Target), 처리 시간(DurationMs), 성공 여부(Success)를 담습니다.",
        when: "백엔드/DB 호출 지연이 전체 응답 지연의 원인인지 확인할 때",
        example: "AppDependencies\n| where Success == false\n| project DependencyType, Target, ResultCode, DurationMs" },
      { name: "ApiManagementGatewayLogs 테이블", code: "ApiManagementGatewayLogs", url: "https://learn.microsoft.com/ko-kr/azure/azure-monitor/reference/tables/apimanagementgatewaylogs",
        easy: "APIM 게이트웨이를 통과한 모든 요청의 진단 로그입니다. API ID(ApiId), 응답 코드(ResponseCode), 요청 성공 여부(IsRequestSuccess), 총 처리 시간(TotalTime, ms) 등을 담습니다. 이 프로젝트의 Kusto 실행기가 사용하는 테이블입니다.",
        when: "APIM을 통과하는 트래픽의 성공률/지연을 직접 조회하고 싶을 때",
        example: "ApiManagementGatewayLogs\n| where IsRequestSuccess == false\n| top 100 by TimeGenerated desc" }
    ] },
    { cat: "KQL(Kusto) 연산자", anchor: "am-kusto-ops", items: [
      { name: "where — 조건 필터링", code: "where", url: "https://learn.microsoft.com/ko-kr/kusto/query/where-operator",
        easy: "조건에 맞는 행만 남깁니다. Kibana의 검색창 필터와 비슷한 역할이지만 문법은 완전히 다릅니다.",
        when: "특정 조건(실패 요청, 특정 API 등)만 골라낼 때",
        example: "ApiManagementGatewayLogs\n| where ResponseCode >= 500" },
      { name: "summarize + bin() — 시간대별 집계", code: "summarize ... by bin()", url: "https://learn.microsoft.com/ko-kr/kusto/query/summarize-operator",
        easy: "그룹별 집계(평균/합계/개수 등)를 구합니다. bin()과 함께 쓰면 시간을 일정 간격(예: 10분)으로 묶어 시계열 집계를 만들 수 있습니다.",
        when: "시간대별 트래픽 추이나 평균 응답 시간 그래프를 만들 때",
        example: "ApiManagementGatewayLogs\n| summarize avg(TotalTime) by bin(TimeGenerated, 10m)\n| render timechart" },
      { name: "top N — 상위 N건", code: "top", url: "https://learn.microsoft.com/ko-kr/kusto/query/top-operator",
        easy: "지정한 컬럼 기준으로 정렬해 상위 N건만 가져옵니다. sort by + take를 한 번에 하는 것과 비슷합니다.",
        when: "가장 느린 요청 Top 10처럼 순위가 필요한 조회를 할 때",
        example: "ApiManagementGatewayLogs\n| top 10 by TotalTime desc" },
      { name: "join — 테이블 결합", code: "join", url: "https://learn.microsoft.com/ko-kr/kusto/query/join-operator",
        easy: "OperationId 같은 공통 키로 두 테이블(예: AppRequests와 AppExceptions)을 연결합니다. 이 프로젝트의 로컬 Kusto 실행기는 join을 지원하지 않습니다 — 정확히 재현하기 어려워 화이트리스트에서 제외했습니다.",
        when: "실패한 요청과 그 원인 예외를 함께 보고 싶을 때 (실제 Azure Monitor에서만 사용 가능, 이 프로젝트의 로컬 실행기에서는 미지원)",
        example: 'AppRequests\n| where Success == false\n| join kind=inner (AppExceptions) on OperationId\n| project Name, ExceptionType' }
    ] },
    { cat: "알림 규칙(Alert Rules)", anchor: "am-alerts", items: [
      { name: "알림 시그널 유형", code: "Signal type", url: "https://learn.microsoft.com/ko-kr/azure/azure-monitor/alerts/alerts-types",
        easy: "알림은 메트릭(Metric), 로그 검색(Log search), 활동 로그(Activity log), 스마트 감지(Smart detection), Prometheus 5가지 시그널 유형 중 하나로 만듭니다. 이 프로젝트의 알림 시뮬레이터는 메트릭 알림만 다룹니다.",
        when: "어떤 종류의 알림을 만들지 처음 결정할 때",
        example: "-- 메트릭 알림: CPU %, 응답 시간 등 수치형 시계열\n-- 로그 검색 알림: Kusto 쿼리 결과 건수 기준" },
      { name: "평가 빈도 vs 조회 기간", code: "Check every / Lookback period", url: "https://learn.microsoft.com/ko-kr/azure/azure-monitor/alerts/alerts-create-metric-alert-rule",
        easy: "'평가 빈도(Check every)'는 조건을 확인하는 주기이고, '조회 기간(Lookback period, 집계 기간)'은 매번 확인할 때 얼마나 과거까지 데이터를 모아 집계할지입니다. 예: 1분마다 확인하되, 최근 5분치를 집계.",
        when: "알림이 왜 예상보다 늦게(또는 빨리) 발동하는지 이해할 때",
        example: "-- Check every: 1 minute\n-- Lookback period: 5 minutes\n-- → 매분 최근 5분 평균을 계산해 임계값과 비교" },
      { name: "정적 임계값 vs 동적 임계값", code: "Static / Dynamic threshold", url: "https://learn.microsoft.com/ko-kr/azure/azure-monitor/alerts/alerts-types",
        easy: "정적 임계값은 사용자가 직접 숫자를 지정합니다. 동적 임계값은 머신러닝이 과거 패턴을 학습해 임계값을 자동으로 산출합니다(민감도 상/중/하 설정 가능). 이 프로젝트의 시뮬레이터는 정적 임계값만 지원합니다.",
        when: "트래픽 패턴이 계절성/주기성을 띠어 고정 숫자로는 오탐이 많을 때 동적 임계값을 고려",
        example: "-- 정적: TotalTime average > 500 ms\n-- 동적: 민감도 Medium, 과거 패턴 대비 이상치 자동 판정" }
    ] },
    { cat: "액션 그룹(Action Groups)", anchor: "am-action-groups", items: [
      { name: "알림 액션(Notification)", code: "Notifications", url: "https://learn.microsoft.com/ko-kr/azure/azure-monitor/alerts/action-groups",
        easy: "사람에게 직접 알리는 채널입니다: Email, Email Azure Resource Manager Role, SMS, Azure 앱 푸시 알림(Azure app Push notifications), Voice(음성 전화).",
        when: "담당자에게 즉시 알림이 필요할 때",
        example: "-- Action Group 'oncall-notify': Email(team@contoso.com) + SMS(+82-10-...)" },
      { name: "자동화 액션(Action)", code: "Actions", url: "https://learn.microsoft.com/ko-kr/azure/azure-monitor/alerts/action-groups",
        easy: "시스템이 자동으로 처리하게 하는 채널입니다: Automation Runbook, Event Hubs, Function(Azure Function), ITSM, Logic Apps, Secure Webhook, Webhook.",
        when: "알림 발생 시 자동 재시작 스크립트를 트리거하거나 ITSM 티켓을 자동 생성하고 싶을 때",
        example: "-- Action Group 'auto-remediate': Automation Runbook(Restart-AppService) + Webhook(https://ops.contoso.com/hook)" }
    ] },
    { cat: "워크북 vs 대시보드", anchor: "am-workbooks-dashboards", items: [
      { name: "워크북(Workbooks)", code: "Workbooks", url: "https://learn.microsoft.com/ko-kr/azure/azure-monitor/visualize/workbooks-overview",
        easy: "텍스트·로그 쿼리(KQL)·메트릭·매개변수(parameter)를 결합한 인터랙티브 분석 캔버스입니다. 템플릿 갤러리로 공유할 수 있고, 여러 데이터 소스를 자유롭게 조합해 탐색하기 좋습니다.",
        when: "장애 조사처럼 자유롭게 파고들며 분석하는 보고서를 만들 때",
        example: "-- Workbook: 파라미터(리소스 선택) + KQL 쿼리 셀 + 메트릭 차트를 한 페이지에 결합" },
      { name: "대시보드(Dashboards)", code: "Dashboards", url: "https://learn.microsoft.com/ko-kr/azure/azure-monitor/visualize/best-practices-visualize",
        easy: "Azure 인프라 상태를 한 화면(single pane of glass)에 모아 보여주지만, 필터 기능은 없고 상단 시간 범위 선택만 가능해 워크북보다 기능이 제한적입니다. 워크북 시각화를 대시보드에 핀 고정(pin)해서 함께 쓸 수 있습니다.",
        when: "운영팀이 상시 모니터하는 고정된 요약 화면이 필요할 때",
        example: "-- Dashboard: 워크북에서 만든 timechart를 pin → 대시보드 타일로 고정 배치" }
    ] },
    { cat: "진단 설정(Diagnostic Settings)", anchor: "am-diagnostic-settings", items: [
      { name: "API Management 진단 카테고리", code: "Microsoft.ApiManagement/service", url: "https://learn.microsoft.com/ko-kr/azure/api-management/monitor-api-management-reference",
        easy: "API Management는 진단 로그 카테고리가 여러 개입니다: Gateway 로그(ApiManagementGatewayLogs), Developer Portal 사용 로그, WebSocket 연결 로그 등. 리소스 종류마다 노출되는 카테고리 구성이 다릅니다.",
        when: "APIM에서 어떤 로그를 Log Analytics로 보낼지 진단 설정을 구성할 때",
        example: "-- 진단 설정에서 'Logs related to ApiManagement Gateway' 카테고리를 활성화 → ApiManagementGatewayLogs 테이블에 적재" },
      { name: "Key Vault 진단 카테고리", code: "Microsoft.KeyVault/vaults", url: "https://learn.microsoft.com/ko-kr/azure/key-vault/general/logging",
        easy: "Key Vault는 진단 카테고리가 AuditEvent(감사 이벤트) 하나뿐입니다. API Management처럼 카테고리가 여러 개인 리소스와 대조하면, '리소스마다 진단 카테고리 구성이 다르다'는 점이 뚜렷하게 드러납니다.",
        when: "새 리소스 종류를 모니터링에 추가할 때마다 그 리소스의 진단 카테고리 목록을 먼저 확인해야 하는 이유를 이해할 때",
        example: "-- Key Vault 진단 설정: 'AuditEvent' 카테고리만 존재, 활성화하면 AZKVAuditLogs 테이블에 적재" }
    ] }
  ];

```

- [ ] **Step 2: 렌더링 호출 추가**

`renderRefPage("dataToc", "dataRefBody", DATA_INFRA, "");` 바로 다음에 추가:

```js
  renderRefPage("azuremonitorToc", "azuremonitorRefBody", AZURE_MONITOR_REF, "");
```

`var gotoDataRefBtn = ...` 블록 바로 다음에 추가:

```js
  var gotoAzureMonitorRefBtn = document.getElementById("gotoAzureMonitorRef");
  if (gotoAzureMonitorRefBtn) gotoAzureMonitorRefBtn.addEventListener("click", function () { switchTab("azuremonitor-ref"); });
```

`buildFullQuiz("dataQuizList", DATA_INFRA, "data-full", "");` 바로 다음에 추가:

```js
  buildFullQuiz("azuremonitorQuizList", AZURE_MONITOR_REF, "azuremonitor-full", "");
```

- [ ] **Step 3: 브라우저에서 확인**

브라우저에서 "Azure Monitor 레퍼런스" 탭을 열어 6개 카테고리, 총 17개 항목이 카드로 렌더링되는지 확인. 상단 카테고리 링크 클릭 시 해당 섹션으로 스크롤되는지 확인. "Azure Monitor 퀴즈" 탭에서 "← Azure Monitor 레퍼런스에서 전체 목록 먼저 보기" 버튼을 누르면 레퍼런스 탭으로 이동하는지, 퀴즈 카드 17개가 순서대로 나오고 "정답 보기"가 정상 동작하는지 확인.
예상 결과: 콘솔 에러 없음, 카드/퀴즈 개수 = 17개(4+4+3+2+2+2).

---

## Task 3: Kusto 실행기 — 데이터셋 + 엔진 함수

**Files:**
- Modify: `azure-migration-tracker.html` (`renderLabMigration();` 호출 뒤, `/* ---------------- jira draft ---------------- */` 주석 앞에 추가)

**Interfaces:**
- Produces: `AM_GATEWAY_LOGS`(배열), `runKustoQuery(queryText)` → `{ error, stageIndex?, message?, columns?, rows?, trace?, chartHint? }` — Task 4가 이 함수를 호출.

- [ ] **Step 1: 가상 테이블 데이터셋 추가**

```js
  /* ---------------- Kusto 실행기: 가상 테이블 + 엔진 ---------------- */
  var AM_GATEWAY_LOGS = [
    { TimeGenerated: "2026-08-06T09:00:05Z", ApiId: "orders-api", Method: "GET", Url: "/orders", ResponseCode: 200, BackendResponseCode: 200, IsRequestSuccess: true, TotalTime: 82, CallerIpAddress: "203.0.113.10" },
    { TimeGenerated: "2026-08-06T09:00:41Z", ApiId: "orders-api", Method: "POST", Url: "/orders", ResponseCode: 200, BackendResponseCode: 200, IsRequestSuccess: true, TotalTime: 118, CallerIpAddress: "203.0.113.11" },
    { TimeGenerated: "2026-08-06T09:01:12Z", ApiId: "orders-api", Method: "GET", Url: "/orders", ResponseCode: 401, BackendResponseCode: 0, IsRequestSuccess: false, TotalTime: 9, CallerIpAddress: "203.0.113.12" },
    { TimeGenerated: "2026-08-06T09:01:47Z", ApiId: "orders-api", Method: "GET", Url: "/orders", ResponseCode: 429, BackendResponseCode: 0, IsRequestSuccess: false, TotalTime: 4, CallerIpAddress: "203.0.113.10" },
    { TimeGenerated: "2026-08-06T09:02:03Z", ApiId: "orders-api", Method: "GET", Url: "/orders", ResponseCode: 429, BackendResponseCode: 0, IsRequestSuccess: false, TotalTime: 5, CallerIpAddress: "203.0.113.10" },
    { TimeGenerated: "2026-08-06T09:02:30Z", ApiId: "users-api", Method: "GET", Url: "/users/1", ResponseCode: 200, BackendResponseCode: 200, IsRequestSuccess: true, TotalTime: 640, CallerIpAddress: "203.0.113.20" },
    { TimeGenerated: "2026-08-06T09:03:01Z", ApiId: "users-api", Method: "GET", Url: "/users/2", ResponseCode: 200, BackendResponseCode: 200, IsRequestSuccess: true, TotalTime: 95, CallerIpAddress: "203.0.113.21" },
    { TimeGenerated: "2026-08-06T09:03:44Z", ApiId: "users-api", Method: "GET", Url: "/users/3", ResponseCode: 403, BackendResponseCode: 0, IsRequestSuccess: false, TotalTime: 6, CallerIpAddress: "203.0.113.22" },
    { TimeGenerated: "2026-08-06T09:04:12Z", ApiId: "users-api", Method: "GET", Url: "/users/4", ResponseCode: 200, BackendResponseCode: 200, IsRequestSuccess: true, TotalTime: 88, CallerIpAddress: "203.0.113.20" },
    { TimeGenerated: "2026-08-06T09:04:55Z", ApiId: "orders-api", Method: "GET", Url: "/orders", ResponseCode: 200, BackendResponseCode: 200, IsRequestSuccess: true, TotalTime: 730, CallerIpAddress: "203.0.113.11" },
    { TimeGenerated: "2026-08-06T09:05:20Z", ApiId: "orders-api", Method: "POST", Url: "/orders", ResponseCode: 200, BackendResponseCode: 200, IsRequestSuccess: true, TotalTime: 101, CallerIpAddress: "203.0.113.12" },
    { TimeGenerated: "2026-08-06T09:05:58Z", ApiId: "legacy-api", Method: "GET", Url: "/legacy/report", ResponseCode: 403, BackendResponseCode: 0, IsRequestSuccess: false, TotalTime: 3, CallerIpAddress: "203.0.113.30" },
    { TimeGenerated: "2026-08-06T09:06:33Z", ApiId: "legacy-api", Method: "GET", Url: "/legacy/report", ResponseCode: 403, BackendResponseCode: 0, IsRequestSuccess: false, TotalTime: 3, CallerIpAddress: "203.0.113.30" },
    { TimeGenerated: "2026-08-06T09:07:02Z", ApiId: "users-api", Method: "GET", Url: "/users/5", ResponseCode: 200, BackendResponseCode: 200, IsRequestSuccess: true, TotalTime: 92, CallerIpAddress: "203.0.113.21" },
    { TimeGenerated: "2026-08-06T09:07:40Z", ApiId: "orders-api", Method: "GET", Url: "/orders", ResponseCode: 200, BackendResponseCode: 200, IsRequestSuccess: true, TotalTime: 79, CallerIpAddress: "203.0.113.10" },
    { TimeGenerated: "2026-08-06T09:08:15Z", ApiId: "orders-api", Method: "GET", Url: "/orders", ResponseCode: 200, BackendResponseCode: 200, IsRequestSuccess: true, TotalTime: 850, CallerIpAddress: "203.0.113.11" },
    { TimeGenerated: "2026-08-06T09:08:50Z", ApiId: "users-api", Method: "GET", Url: "/users/6", ResponseCode: 200, BackendResponseCode: 200, IsRequestSuccess: true, TotalTime: 100, CallerIpAddress: "203.0.113.22" },
    { TimeGenerated: "2026-08-06T09:09:25Z", ApiId: "orders-api", Method: "POST", Url: "/orders", ResponseCode: 200, BackendResponseCode: 200, IsRequestSuccess: true, TotalTime: 113, CallerIpAddress: "203.0.113.12" }
  ];

  function kustoParseLiteral(raw) {
    raw = raw.trim();
    if (raw === "true") return true;
    if (raw === "false") return false;
    if (/^-?\d+(\.\d+)?$/.test(raw)) return Number(raw);
    return raw.replace(/^"(.*)"$/, "$1");
  }

  function kustoCompare(a, op, b) {
    if (op === "==") return a == b;
    if (op === "!=") return a != b;
    if (op === ">") return a > b;
    if (op === "<") return a < b;
    if (op === ">=") return a >= b;
    if (op === "<=") return a <= b;
    if (op === "contains") return String(a).indexOf(String(b)) !== -1;
    if (op === "!contains") return String(a).indexOf(String(b)) === -1;
    throw new Error("지원하지 않는 비교 연산자: " + op);
  }

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

  function kustoApplyProject(rows, clause) {
    var cols = clause.split(",").map(function (s) { return s.trim(); });
    return rows.map(function (row) {
      var out = {};
      cols.forEach(function (c) { out[c] = row[c]; });
      return out;
    });
  }

  function kustoBinValue(iso, col, amount, unit) {
    var ms = amount * (unit === "h" ? 3600000 : unit === "m" ? 60000 : 1000);
    var t = new Date(iso).getTime();
    return new Date(Math.floor(t / ms) * ms).toISOString();
  }

  function kustoApplySummarize(rows, clause) {
    var m = clause.match(/^(count\(\)|avg|sum|max|min)\(?(\w+)?\)?\s+by\s+(.+)$/);
    if (!m) throw new Error("summarize 문법을 해석할 수 없습니다: " + clause);
    var aggToken = m[1], aggCol = m[2], byClause = m[3].trim();
    var byBin = byClause.match(/^bin\(\s*(\w+)\s*,\s*(\d+)(m|h|s)\s*\)$/);
    var groups = {};
    rows.forEach(function (row) {
      var key = byBin ? kustoBinValue(row[byBin[1]], byBin[1], Number(byBin[2]), byBin[3]) : row[byClause];
      if (!groups[key]) groups[key] = [];
      groups[key].push(row);
    });
    return Object.keys(groups).sort().map(function (key) {
      var group = groups[key];
      var value;
      if (aggToken === "count()") value = group.length;
      else if (aggToken === "avg") value = group.reduce(function (s, r) { return s + Number(r[aggCol]); }, 0) / group.length;
      else if (aggToken === "sum") value = group.reduce(function (s, r) { return s + Number(r[aggCol]); }, 0);
      else if (aggToken === "max") value = Math.max.apply(null, group.map(function (r) { return Number(r[aggCol]); }));
      else if (aggToken === "min") value = Math.min.apply(null, group.map(function (r) { return Number(r[aggCol]); }));
      var out = {};
      out[byBin ? byBin[1] : byClause] = key;
      out[aggToken === "count()" ? "count_" : aggToken + "_" + aggCol] = value;
      return out;
    });
  }

  function kustoApplyTopOrSort(rows, clause, isTop) {
    var m = isTop
      ? clause.match(/^(\d+)\s+by\s+(\w+)(\s+asc|\s+desc)?$/)
      : clause.match(/^(\w+)(\s+asc|\s+desc)?$/);
    if (!m) throw new Error((isTop ? "top" : "sort by") + " 문법을 해석할 수 없습니다: " + clause);
    var col = isTop ? m[2] : m[1];
    var dir = (isTop ? m[3] : m[2]) || " desc";
    var sorted = rows.slice().sort(function (a, b) {
      if (a[col] < b[col]) return dir.indexOf("asc") !== -1 ? -1 : 1;
      if (a[col] > b[col]) return dir.indexOf("asc") !== -1 ? 1 : -1;
      return 0;
    });
    return isTop ? sorted.slice(0, Number(m[1])) : sorted;
  }

  var KUSTO_STAGE_HANDLERS = {
    where: function (rows, clause) { return kustoApplyWhere(rows, clause); },
    project: function (rows, clause) { return kustoApplyProject(rows, clause); },
    summarize: function (rows, clause) { return kustoApplySummarize(rows, clause); },
    top: function (rows, clause) { return kustoApplyTopOrSort(rows, clause, true); },
    sort: function (rows, clause) { return kustoApplyTopOrSort(rows, clause.replace(/^by\s+/, ""), false); },
    order: function (rows, clause) { return kustoApplyTopOrSort(rows, clause.replace(/^by\s+/, ""), false); },
    take: function (rows, clause) { return rows.slice(0, Number(clause.trim())); },
    limit: function (rows, clause) { return rows.slice(0, Number(clause.trim())); }
  };

  function runKustoQuery(queryText) {
    var text = (queryText || "").trim();
    if (!text) return { error: true, message: "쿼리가 비어 있습니다." };

    var letRe = /^let\s+(\w+)\s*=\s*(.+?);\s*/;
    var letMatch, bindings = {};
    while ((letMatch = text.match(letRe))) {
      bindings[letMatch[1]] = letMatch[2].trim();
      text = text.slice(letMatch[0].length);
    }
    Object.keys(bindings).forEach(function (name) {
      text = text.split(new RegExp("\\b" + name + "\\b", "g")).join(bindings[name]);
    });

    var stages = text.split("\n").join(" ").split("|").map(function (s) { return s.trim(); }).filter(Boolean);
    var tableName = stages.shift();
    if (tableName !== "ApiManagementGatewayLogs") {
      return { error: true, stageIndex: 0, message: "알 수 없는 테이블: " + tableName + " (이 실행기는 ApiManagementGatewayLogs만 지원합니다)" };
    }

    var rows = AM_GATEWAY_LOGS.slice();
    var trace = [{ stage: "ApiManagementGatewayLogs", resultCount: rows.length }];
    var chartHint = false;

    for (var i = 0; i < stages.length; i++) {
      var stage = stages[i];
      var opMatch = stage.match(/^(\w+)\s*([\s\S]*)$/);
      var op = opMatch ? opMatch[1] : "";
      var clause = opMatch ? opMatch[2] : "";
      if (op === "render") { chartHint = clause.trim() === "timechart"; continue; }
      var handler = KUSTO_STAGE_HANDLERS[op];
      if (!handler) {
        return { error: true, stageIndex: i + 1, message: (i + 1) + "번째 스테이지에서 실패: 지원하지 않는 연산자 `" + op + "` (이 시뮬레이터의 화이트리스트 밖입니다. 실제 Kusto에서는 지원될 수 있습니다)", trace: trace };
      }
      try {
        rows = handler(rows, clause);
      } catch (e) {
        return { error: true, stageIndex: i + 1, message: (i + 1) + "번째 스테이지(`" + op + "`)에서 실패: " + e.message, trace: trace };
      }
      trace.push({ stage: op, resultCount: rows.length });
    }

    var columns = rows.length ? Object.keys(rows[0]) : [];
    return { error: false, columns: columns, rows: rows, trace: trace, chartHint: chartHint };
  }
```

- [ ] **Step 2: 브라우저 콘솔에서 수동 검증**

`azure-migration-tracker.html`을 열고 개발자 도구 콘솔에서 실행:

```js
runKustoQuery('ApiManagementGatewayLogs\n| where IsRequestSuccess == false\n| sort by TimeGenerated desc\n| take 20')
```
예상 결과: `error: false`, `rows.length`가 데이터셋의 `IsRequestSuccess: false` 행 전체 건수(6건)와 일치, 최신순 정렬.

```js
runKustoQuery('ApiManagementGatewayLogs\n| summarize avg(TotalTime) by ApiId\n| sort by avg_TotalTime desc')
```
예상 결과: `error: false`, 3개 그룹(`orders-api`/`users-api`/`legacy-api`)에 대한 평균값, `legacy-api`가 가장 낮음(전부 차단이라 TotalTime이 작음).

```js
runKustoQuery('ApiManagementGatewayLogs\n| joinxyz kind=inner (AppExceptions) on OperationId')
```
예상 결과: `error: true`, `message`에 "지원하지 않는 연산자"와 스테이지 번호(1) 포함.

---

## Task 4: Kusto 실행기 UI 연결

**Files:**
- Modify: `azure-migration-tracker.html` (Task 3의 `runKustoQuery` 정의 뒤에 UI 초기화 코드 추가)

**Interfaces:**
- Consumes: `runKustoQuery(queryText)`(Task 3), `state`/`persist()`(기존 상태 계층), `labKustoBody`(Task 1의 컨테이너 id).

- [ ] **Step 1: 예제 문제 세트 + 렌더링 함수 추가**

Task 3에서 추가한 코드 바로 다음에 추가:

```js
  var KUSTO_PRACTICE_QUESTIONS = [
    { id: "kq1", prompt: "실패(IsRequestSuccess=false)한 요청만 최신순으로 최대 20건 조회하는 쿼리를 작성하세요.",
      answer: "ApiManagementGatewayLogs\n| where IsRequestSuccess == false\n| sort by TimeGenerated desc\n| take 20" },
    { id: "kq2", prompt: "API별 평균 처리 시간(TotalTime)을 구해 느린 순서로 정렬하세요.",
      answer: "ApiManagementGatewayLogs\n| summarize avg(TotalTime) by ApiId\n| sort by avg_TotalTime desc" },
    { id: "kq3", prompt: "orders-api 요청만 골라 ApiId/ResponseCode/TotalTime만 남기고, 처리 시간이 가장 긴 5건을 보여주세요.",
      answer: 'ApiManagementGatewayLogs\n| where ApiId == "orders-api"\n| project ApiId, ResponseCode, TotalTime\n| top 5 by TotalTime desc' }
  ];

  function renderKustoResult(result) {
    if (result.error) {
      return '<div class="kusto-error">' + result.message + '</div>';
    }
    if (!result.rows.length) {
      return '<p class="empty-hint">결과 0건.</p>';
    }
    var thead = "<tr>" + result.columns.map(function (c) { return "<th>" + c + "</th>"; }).join("") + "</tr>";
    var tbody = result.rows.map(function (row) {
      return "<tr>" + result.columns.map(function (c) { return "<td>" + row[c] + "</td>"; }).join("") + "</tr>";
    }).join("");
    var chartNote = result.chartHint ? '<p class="empty-hint">render timechart 감지됨 — 이 로컬 실행기는 실제 차트 대신 표로 표시합니다.</p>' : "";
    return chartNote + '<div class="kusto-table-scroll"><table class="kusto-table">' + thead + tbody + "</table></div>";
  }

  function initKustoLab() {
    var body = document.getElementById("labKustoBody");
    var presetHtml = KUSTO_PRACTICE_QUESTIONS.map(function (q) {
      return '<div class="quiz-card"><div class="quiz-body"><p class="quiz-scenario">' + q.prompt + '</p>' +
        '<div class="quiz-actions"><button class="btn" type="button" data-fill="' + q.id + '">이 쿼리를 편집창에 채우기</button>' +
        '<button class="btn" type="button" data-reveal="' + q.id + '">정답 쿼리 실행 결과 보기</button></div>' +
        '<div class="answer-reveal" id="kustoAnswer-' + q.id + '"></div></div></div>';
    }).join("");

    body.innerHTML =
      '<div class="kusto-preset-grid">' + presetHtml + '</div>' +
      '<textarea class="kusto-editor" id="kustoQueryInput" placeholder="ApiManagementGatewayLogs\n| where ...">' + (state.quiz["kusto-editor"] || "") + '</textarea>' +
      '<div class="quiz-actions" style="margin-top:var(--sp-3);"><button class="btn primary" type="button" id="kustoRunBtn">실행</button></div>' +
      '<div id="kustoResultArea"></div>';

    var input = document.getElementById("kustoQueryInput");
    input.addEventListener("input", function () { state.quiz["kusto-editor"] = input.value; persist(); });

    var resultArea = document.getElementById("kustoResultArea");
    document.getElementById("kustoRunBtn").addEventListener("click", function () {
      resultArea.innerHTML = renderKustoResult(runKustoQuery(input.value));
    });

    KUSTO_PRACTICE_QUESTIONS.forEach(function (q) {
      body.querySelector('[data-fill="' + q.id + '"]').addEventListener("click", function () {
        input.value = q.answer;
        state.quiz["kusto-editor"] = input.value;
        persist();
      });
      body.querySelector('[data-reveal="' + q.id + '"]').addEventListener("click", function () {
        var answerResult = runKustoQuery(q.answer);
        document.getElementById("kustoAnswer-" + q.id).innerHTML =
          '<div class="code-block">' + q.answer + "</div>" + renderKustoResult(answerResult);
        document.getElementById("kustoAnswer-" + q.id).classList.add("show");
      });
    });
  }
  initKustoLab();
```

- [ ] **Step 2: 브라우저에서 확인**

"Kusto 실행기" 탭을 열어 예제 문제 3개 카드가 보이는지 확인. 각 카드의 "이 쿼리를 편집창에 채우기"를 누르면 아래 편집창에 정답 쿼리가 채워지는지, "실행"을 누르면 표가 렌더링되는지 확인. "정답 쿼리 실행 결과 보기"를 누르면 카드 안에 정답 쿼리와 실행 결과가 함께 나타나는지 확인. 편집창에 `ApiManagementGatewayLogs | joinxyz on X`처럼 잘못된 연산자를 입력하고 실행하면 빨간 에러 메시지가 뜨는지 확인. 새로고침 후 편집창 내용이 유지되는지 확인(localStorage).
예상 결과: 콘솔 에러 없음, 정답 실행 결과 표와 에러 메시지 모두 정상 렌더링.

---

## Task 5: 알림 시뮬레이터 — 데이터셋 + 평가 엔진

**Files:**
- Modify: `azure-migration-tracker.html` (Task 4의 `initKustoLab();` 호출 뒤에 추가)

**Interfaces:**
- Produces: `AM_METRIC_SERIES`(배열), `evaluateAlertRule(config)` → `[{ evalTime, aggregatedValue, breached, transition, note }]` — Task 6이 이 함수를 호출.

- [ ] **Step 1: 샘플 메트릭 시계열 + 평가 엔진 추가**

```js
  /* ---------------- 알림 시뮬레이터: 샘플 메트릭 + 평가 엔진 ---------------- */
  var AM_METRIC_SERIES = (function () {
    var points = [];
    for (var t = 0; t < 60; t++) {
      var value;
      if (t >= 20 && t <= 35) {
        value = 400 + Math.round(500 * Math.sin((t - 20) / 15 * Math.PI));
      } else {
        value = 90 + (t % 5) * 12;
      }
      points.push({ t: t, value: value });
    }
    return points;
  })();

  function alertAggregateWindow(points, aggregation) {
    var values = points.map(function (p) { return p.value; });
    if (aggregation === "Average") return values.reduce(function (s, v) { return s + v; }, 0) / values.length;
    if (aggregation === "Maximum") return Math.max.apply(null, values);
    if (aggregation === "Minimum") return Math.min.apply(null, values);
    if (aggregation === "Total") return values.reduce(function (s, v) { return s + v; }, 0);
    if (aggregation === "Count") return values.length;
    throw new Error("지원하지 않는 집계 방식: " + aggregation);
  }

  function alertCompareThreshold(value, op, threshold) {
    if (op === ">") return value > threshold;
    if (op === "<") return value < threshold;
    if (op === ">=") return value >= threshold;
    if (op === "<=") return value <= threshold;
    throw new Error("지원하지 않는 임계값 연산자: " + op);
  }

  function evaluateAlertRule(config) {
    var series = config.metricSeries;
    var lastMinute = series[series.length - 1].t;
    var results = [];
    var wasBreached = false;

    for (var t = config.lookbackMin; t <= lastMinute; t += config.evalFrequencyMin) {
      var windowPoints = series.filter(function (p) { return p.t > t - config.lookbackMin && p.t <= t; });
      if (!windowPoints.length) {
        results.push({ evalTime: t, aggregatedValue: null, breached: false, transition: null, note: "데이터 부족" });
        continue;
      }
      var aggregatedValue = alertAggregateWindow(windowPoints, config.aggregation);
      var breached = alertCompareThreshold(aggregatedValue, config.thresholdOp, config.thresholdValue);
      var transition = null;
      if (breached && !wasBreached) transition = "fired";
      else if (!breached && wasBreached) transition = "resolved";
      wasBreached = breached;
      results.push({ evalTime: t, aggregatedValue: aggregatedValue, breached: breached, transition: transition, note: null });
    }
    return results;
  }
```

- [ ] **Step 2: 브라우저 콘솔에서 수동 검증**

```js
evaluateAlertRule({ metricSeries: AM_METRIC_SERIES, thresholdOp: ">", thresholdValue: 300, aggregation: "Average", evalFrequencyMin: 1, lookbackMin: 5 })
```
예상 결과: `t`가 20대 중반 근처에서 `breached: true`이면서 `transition: "fired"`인 항목이 처음 나타나고, 스파이크 종료 후(35~40분대) `transition: "resolved"` 항목이 한 번 나타남.

```js
evaluateAlertRule({ metricSeries: AM_METRIC_SERIES, thresholdOp: ">", thresholdValue: 300, aggregation: "Average", evalFrequencyMin: 10, lookbackMin: 5 })
```
예상 결과: 평가 시점 간격이 넓어(10분) fired 시점이 위 결과보다 늦게 잡히거나 스파이크를 완전히 놓칠 수 있음 — "평가 빈도가 넓으면 짧은 스파이크를 놓칠 수 있다"는 학습 포인트를 실제로 확인.

---

## Task 6: 알림 시뮬레이터 UI 연결

**Files:**
- Modify: `azure-migration-tracker.html` (Task 5의 `evaluateAlertRule` 정의 뒤, 파일 끝의 `})();` 직전 — `renderLabMigration();`/AI 체크리스트 블록과 같은 레벨)

**Interfaces:**
- Consumes: `evaluateAlertRule(config)`(Task 5), `AM_METRIC_SERIES`(Task 5), `state`/`persist()`(기존 상태 계층), `labAlertBody`(Task 1의 컨테이너 id).

- [ ] **Step 1: 설정 폼 + 타임라인 렌더러 추가**

Task 5에서 추가한 코드 바로 다음에 추가:

```js
  function renderAlertTimeline(results) {
    if (!results.length) return '<p class="empty-hint">평가 결과가 없습니다. 조회 기간이 전체 구간보다 깁니다.</p>';
    return '<div class="alert-timeline">' + results.map(function (r) {
      var cls = "alert-point";
      if (r.transition === "fired") cls += " fired";
      else if (r.transition === "resolved") cls += " resolved";
      else if (r.breached) cls += " breached";
      var label = r.aggregatedValue === null ? r.note : Math.round(r.aggregatedValue) + "ms";
      var badge = r.transition ? " · " + (r.transition === "fired" ? "발동" : "해제") : "";
      return '<span class="' + cls + '" title="t=' + r.evalTime + "분">t+" + r.evalTime + "분: " + label + badge + "</span>";
    }).join("") + "</div>";
  }

  function initAlertLab() {
    var body = document.getElementById("labAlertBody");
    body.innerHTML =
      '<div class="alert-form-grid">' +
      '<label>집계 방식<select id="alertAggregation"><option>Average</option><option>Maximum</option><option>Minimum</option><option>Total</option><option>Count</option></select></label>' +
      '<label>임계값 연산자<select id="alertOp"><option value=">">&gt;</option><option value="<">&lt;</option><option value=">=">&gt;=</option><option value="<=">&lt;=</option></select></label>' +
      '<label>임계값(ms)<input type="number" id="alertThreshold" value="300"></label>' +
      '<label>평가 빈도(분)<input type="number" id="alertEvalFreq" value="1" min="1"></label>' +
      '<label>조회 기간(분)<input type="number" id="alertLookback" value="5" min="1"></label>' +
      "</div>" +
      '<div class="quiz-actions"><button class="btn primary" type="button" id="alertRunBtn">평가 실행</button></div>' +
      '<div id="alertOverlapNote"></div>' +
      '<div id="alertTimelineArea"></div>';

    var aggregationEl = document.getElementById("alertAggregation");
    var opEl = document.getElementById("alertOp");
    var thresholdEl = document.getElementById("alertThreshold");
    var freqEl = document.getElementById("alertEvalFreq");
    var lookbackEl = document.getElementById("alertLookback");
    var overlapNote = document.getElementById("alertOverlapNote");
    var timelineArea = document.getElementById("alertTimelineArea");

    var saved = state.quiz["alert-config"];
    if (saved) {
      try {
        var cfg = JSON.parse(saved);
        aggregationEl.value = cfg.aggregation; opEl.value = cfg.thresholdOp;
        thresholdEl.value = cfg.thresholdValue; freqEl.value = cfg.evalFrequencyMin; lookbackEl.value = cfg.lookbackMin;
      } catch (e) { /* 저장값 손상 시 기본값 유지 */ }
    }

    document.getElementById("alertRunBtn").addEventListener("click", function () {
      var evalFrequencyMin = Number(freqEl.value);
      var lookbackMin = Number(lookbackEl.value);
      var thresholdValue = Number(thresholdEl.value);
      if (!evalFrequencyMin || evalFrequencyMin <= 0 || !lookbackMin || lookbackMin <= 0 || isNaN(thresholdValue)) {
        timelineArea.innerHTML = '<p class="empty-hint">평가 빈도·조회 기간은 1 이상, 임계값은 숫자로 입력하세요.</p>';
        return;
      }
      overlapNote.innerHTML = lookbackMin > evalFrequencyMin
        ? '<p class="alert-overlap-badge">⚠ 조회 기간(' + lookbackMin + '분)이 평가 빈도(' + evalFrequencyMin + '분)보다 길어 평가 구간이 서로 겹칩니다. 실제 Azure Monitor에서도 허용되는 설정입니다.</p>'
        : "";
      var config = { metricSeries: AM_METRIC_SERIES, thresholdOp: opEl.value, thresholdValue: thresholdValue, aggregation: aggregationEl.value, evalFrequencyMin: evalFrequencyMin, lookbackMin: lookbackMin };
      state.quiz["alert-config"] = JSON.stringify({ aggregation: config.aggregation, thresholdOp: config.thresholdOp, thresholdValue: config.thresholdValue, evalFrequencyMin: config.evalFrequencyMin, lookbackMin: config.lookbackMin });
      persist();
      timelineArea.innerHTML = renderAlertTimeline(evaluateAlertRule(config));
    });
  }
  initAlertLab();
```

- [ ] **Step 2: 브라우저에서 확인**

"알림 시뮬레이터" 탭을 열어 기본값(Average, `>`, 300ms, 평가 빈도 1분, 조회 기간 5분)으로 "평가 실행"을 누르면 타임라인이 렌더링되고, 20~35분 구간 근처에서 빨간(breached)/진한 빨강(fired) 포인트가, 그 직후 초록(resolved) 포인트가 나타나는지 확인. 조회 기간을 10, 평가 빈도를 1로 바꿔 다시 실행하면 조회 기간이 평가 빈도보다 길다는 경고 배지가 뜨는지 확인. 평가 빈도를 0으로 입력하고 실행하면 검증 메시지가 뜨는지 확인. 새로고침 후 마지막 설정값이 복원되는지 확인.
예상 결과: 콘솔 에러 없음, fired/resolved가 각 1회 이상 나타남, 겹침 경고와 입력 검증 메시지가 각각 올바른 조건에서만 표시됨.

---

## Self-Review 결과

- **Spec 커버리지**: 배경/목적(레퍼런스+퀴즈+로컬테스트 3단 경로) → Task 1·2, Kusto 실행기 → Task 3·4, 알림 시뮬레이터 → Task 5·6, 네비게이션 변경 → Task 1, 에러 처리/엣지 케이스(스테이지별 에러 메시지, 화이트리스트 배지, 겹침 배지, 데이터 부족 처리, localStorage 폴백) → 각 Task의 엔진/UI 스텝에 반영. 모두 커버됨.
- **Placeholder 스캔**: "TBD"/"적절히"/"필요시" 패턴 없음 — 모든 코드 블록이 실행 가능한 완성 코드.
- **타입/시그니처 일관성**: `runKustoQuery`가 반환하는 `{error, stageIndex, message, columns, rows, trace, chartHint}` 키를 Task 4의 `renderKustoResult`가 그대로 사용. `evaluateAlertRule`이 반환하는 `{evalTime, aggregatedValue, breached, transition, note}` 키를 Task 6의 `renderAlertTimeline`이 그대로 사용. `AM_GATEWAY_LOGS`의 컬럼명(`ApiId`, `ResponseCode`, `IsRequestSuccess`, `TotalTime`, `TimeGenerated`)이 Task 2 레퍼런스 항목의 예시 및 Task 4 예제 문제의 쿼리와 일치.
