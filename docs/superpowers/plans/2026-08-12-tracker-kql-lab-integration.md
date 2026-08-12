# 트래커·KQL 조인 랩 통합 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `azure-migration-tracker.html`의 사이드바를 기본 전체 접힘 상태로 바꾸고, 독립 파일이던 `azure-kql-join-lab.html`의 튜토리얼·예제 콘텐츠를 트래커와 **같은 탭·같은 사이드바**를 쓰는 새 패널로 이식한 뒤, title/rail-brand 문구를 통합 페이지에 맞게 갱신한다.

**Architecture:** `azure-kql-join-lab.html`은 별개 파일로 남겨두고(삭제하지 않음), 그 안의 CSS/HTML/JS를 `azure-migration-tracker.html` 내부에 새 패널(`#panel-kql-join-lab`)로 복사·이식한다. 이식 스크립트는 트래커의 기존 IIFE와 완전히 분리된 **자체 IIFE**로 감싸 전역 스코프 오염과 이름 충돌을 원천 차단한다. 두 파일의 CSS 변수(`--accent-target`, `--sp-*`, `--fs-*` 등)는 이미 동일한 디자인 시스템을 쓰므로 그대로 재사용한다.

**Tech Stack:** 순수 HTML/CSS/Vanilla JS 단일 파일(빌드 도구 없음). 테스트 프레임워크 없음 — 검증은 정적 grep 점검 + 실제 브라우저 수동/DevTools 확인으로 대체한다.

## Global Constraints

- 산출물은 빌드 도구 없이 즉시 열람 가능한 단일 HTML이어야 한다(프로젝트 CLAUDE.md).
- "KQL"이라는 약어가 Kibana Query Language와 Kusto Query Language 두 곳에서 쓰이므로, 새 패널에서도 두 언어를 명시적으로 구분하는 안내 문구(`.kql-note`)를 유지한다.
- 응답/커밋 메시지/주석은 한국어로 작성한다.
- 사용자가 확정한 결정: (1) 새 메뉴 항목은 **같은 탭에서 열리며 트래커와 동일한 사이드바 레이아웃을 공유**한다(별도 파일로 링크 연결하지 않음), (2) rail-brand mark는 `Learning Hub`, title은 `Azure 전환 학습 허브`로 변경한다.
- 기존 `azure-kql-join-lab.html` 파일은 삭제하지 않고 그대로 둔다(사용자가 삭제를 요청하지 않았음).

---

### Task 1: 사이드바 기본 접힘 상태 + title/rail-brand 문구 변경

**Files:**
- Modify: `azure-migration-tracker.html:3` (title 태그)
- Modify: `azure-migration-tracker.html:4` (meta description)
- Modify: `azure-migration-tracker.html:383` (`.rail-brand .mark` 텍스트)
- Modify: `azure-migration-tracker.html:649` (`defaultState()`의 `navOpen` 기본값)

**Interfaces:**
- Consumes: 없음(독립적인 텍스트/기본값 수정).
- Produces: `defaultState().navOpen`의 5개 그룹(`apim`, `kqlgroup`, `datainfra`, `azuremonitor`, `lab`)이 전부 `false`로 시작 — Task 2에서 추가하는 `kql-join-lab` 항목은 그룹이 아닌 최상위 항목이므로 `navOpen`에 키를 추가하지 않는다.

- [ ] **Step 1: title 태그 수정**

`azure-migration-tracker.html:3`을 다음과 같이 바꾼다.

```html
<title>Azure 전환 학습 허브</title>
```

- [ ] **Step 2: meta description 수정**

`azure-migration-tracker.html:4`를 다음과 같이 바꾼다.

```html
<meta name="description" content="Kong/Cassandra/PostgreSQL/Kibana → Azure APIM/Monitor 전환 학습용 인터랙티브 허브">
```

- [ ] **Step 3: rail-brand mark 텍스트 수정**

`azure-migration-tracker.html:383`을 다음과 같이 바꾼다(`.name`은 변경하지 않음).

```html
<span class="mark">Learning Hub</span>
```

- [ ] **Step 4: navOpen 기본값을 전부 false로 변경**

`azure-migration-tracker.html:649`의 `defaultState()` 반환값 중 `navOpen` 부분을 다음과 같이 바꾼다.

Before:
```js
navOpen: { apim: true, kqlgroup: true, datainfra: true, azuremonitor: true, lab: true }
```

After:
```js
navOpen: { apim: false, kqlgroup: false, datainfra: false, azuremonitor: false, lab: false }
```

- [ ] **Step 5: 정적 확인**

Grep으로 남아있는 `true` 잔재가 없는지 확인한다.

Run: `grep -n "navOpen:" "azure-migration-tracker.html"`
Expected: 5개 그룹 전부 `false`로 출력됨.

- [ ] **Step 6: 커밋**

```bash
git add "azure-migration-tracker.html"
git commit -m "feat: 트래커 사이드바 기본 접힘 + 학습 허브 브랜딩으로 변경"
```

---

### Task 2: KQL 조인 랩 패널 이식 (CSS + HTML + JS)

**Files:**
- Modify: `azure-migration-tracker.html` (CSS `</style>` 직전, `panel-lab-workbook` 섹션 뒤, `NAV` 배열, `</body>` 직전)
- Read (원본, 수정하지 않음): `azure-kql-join-lab.html`

**Interfaces:**
- Consumes: Task 1에서 만든 `navOpen` 기본값 구조(변경 없음, `kql-join-lab`은 최상위 항목이라 관여 없음).
- Produces: 새 패널 `#panel-kql-join-lab`, 새 nav 항목 `data-tab="kql-join-lab"`, 새 컨테이너 id `#kqlLabApp`/`#kqlLabTabs`/`#kqlLabResetBtn` — 이후 어떤 작업도 이 이름들을 다시 쓰지 않으므로 후속 태스크 없음(이 태스크가 마지막 기능 태스크).

**충돌 방지 규칙(정확히 이대로 적용):**

| 원본(`azure-kql-join-lab.html`) | 처리 |
|---|---|
| `.storage-banner`, `.storage-banner.show`, `<div id="storageBanner">` | **복사하지 않음.** 트래커에 이미 동일 id(`#storageBanner`)의 배너가 있으므로 그걸 재사용한다(스크립트의 `getElementById("storageBanner")` 호출은 그대로 둔다). |
| `.btn`, `.btn.primary`, `.btn:disabled` | **복사하지 않음.** 트래커에 이미 동일 목적의 `.btn` 규칙이 있어 그대로 재사용한다. |
| `.empty-hint` | **복사하지 않음.** 트래커의 기존 `.empty-hint` 규칙을 재사용한다(색상/크기가 약간 다르지만 통합 페이지 전체에서 더 일관됨). |
| `.workbook-model-answer` | **복사하지 않음.** 트래커에 완전히 동일한 규칙이 이미 존재한다(줄 369). |
| `.page-header`, `.page-header h1` | **복사하지 않음.** 대신 트래커의 기존 `.panel-head.apim-ref-head` 레이아웃을 재사용한다. |
| `.tabs`, `.tabs button`, `.tabs button.active` | `.kql-lab-tabs`로 이름 변경 후 복사(범용 이름이라 향후 충돌 방지 목적). |
| `.kql-note`, `.kql-editor`, `.example-card`, `.example-badges span`, `.kql-error`, `.kql-table`, `.kql-table th/td` | 이름 변경 없이 그대로 복사(트래커에 동일 클래스 없음). |
| `id="tabs"` | `id="kqlLabTabs"`로 변경 |
| `id="app"` | `id="kqlLabApp"`로 변경 |
| `id="resetBtn"` | `id="kqlLabResetBtn"`로 변경 |
| 스크립트 전체를 감싸는 `(function(){"use strict"; ... })();` | 원본에는 없음(전역 스코프) — **새로 추가**해서 전역 오염을 막는다. |
| `document.addEventListener("DOMContentLoaded", function () {...})` 래퍼 | 제거하고 내부 로직만 스크립트 최하단에서 직접 실행(트래커 자체 스크립트와 동일한 패턴 — DOM이 이미 파싱된 뒤 `</body>` 직전에서 실행되므로 불필요). |

- [ ] **Step 1: CSS 이식**

`azure-migration-tracker.html`의 `</style>` 태그(377번째 줄 부근, `.demo-highlight` 규칙들 뒤) 직전에 아래 CSS 블록을 추가한다.

```css
  .kql-note { background: var(--accent-target-bg); border: 1px solid var(--accent-target); border-radius: var(--radius-md); padding: var(--sp-3) var(--sp-4); font-size: var(--fs-sm); margin-bottom: var(--sp-5); }
  .kql-lab-tabs { display: flex; gap: var(--sp-2); margin-bottom: var(--sp-5); border-bottom: 1px solid var(--border); }
  .kql-lab-tabs button { background: none; border: none; padding: var(--sp-3) var(--sp-4); font-size: var(--fs-sm); color: var(--text-dim); cursor: pointer; border-bottom: 2px solid transparent; }
  .kql-lab-tabs button.active { color: var(--accent-target); border-bottom-color: var(--accent-target); font-weight: 600; }
  .kql-editor { width: 100%; min-height: 100px; font-size: var(--fs-sm); border: 1px solid var(--border); border-radius: var(--radius-md); padding: var(--sp-3); background: var(--bg-elevated); color: var(--text); resize: vertical; }
  .example-card { border: 1px solid var(--border); border-radius: var(--radius-md); padding: var(--sp-4); margin-bottom: var(--sp-5); background: var(--bg-elevated); }
  .example-badges span { display: inline-block; font-size: var(--fs-xs); border: 1px solid var(--border); border-radius: var(--radius-md); padding: 2px 8px; margin-right: var(--sp-2); color: var(--text-dim); }
  .kql-error { color: #b91c1c; font-size: var(--fs-sm); padding: var(--sp-2) 0; }
  .kql-table { border-collapse: collapse; width: 100%; font-size: var(--fs-xs); margin-top: var(--sp-2); }
  .kql-table th, .kql-table td { border: 1px solid var(--border); padding: 4px 8px; text-align: left; }
```

- [ ] **Step 2: 패널 HTML 삽입**

`azure-migration-tracker.html`의 `<section class="panel" id="panel-lab-workbook" data-panel>` 섹션이 끝나는 `</section>`(약 626번째 줄, `.main`의 닫는 `</div>` 직전) 바로 뒤에 새 섹션을 추가한다.

```html
    <section class="panel" id="panel-kql-join-lab" data-panel>
      <div class="panel-head apim-ref-head">
        <div class="htext">
          <span class="eyebrow">19 · Local Test Lab</span>
          <h1>KQL 멀티테이블 실습 랩</h1>
          <p class="desc">게이트웨이(ApiManagementGatewayLogs)·백엔드(AppRequests)·DB(AppDependencies) 3개 가상 테이블을 튜토리얼로 익히고, 17개 예제(단일 테이블/2테이블 join/3테이블 조합)를 직접 쿼리로 풀어본 뒤 정답과 비교하세요.</p>
        </div>
        <button class="btn" id="kqlLabResetBtn" title="튜토리얼 진행 단계와 예제 17개의 내 쿼리·정답 펼침 상태를 전부 초기화합니다">다시 시작</button>
      </div>
      <div class="kql-note">
        이 탭의 "KQL"은 <strong>Kusto Query Language</strong>(Azure Monitor/Log Analytics 쿼리 언어)를 의미합니다.
        Kibana의 <strong>Kibana Query Language</strong>(검색창 필터 문법)와는 문법·용도가 다른 별개의 언어입니다.
      </div>
      <div class="kql-lab-tabs" id="kqlLabTabs">
        <button data-tab="tutorial">튜토리얼</button>
        <button data-tab="examples">예제 17개</button>
      </div>
      <div id="kqlLabApp"></div>
    </section>
```

- [ ] **Step 3: NAV 배열에 최상위 메뉴 항목 추가**

`azure-migration-tracker.html`의 `NAV` 배열(약 692번째 줄) 마지막 원소(`{ id: "lab", ... children: [...] }`) 뒤, 배열을 닫는 `];` 앞에 아래 항목을 추가한다.

```js
    ,
    { id: "kql-join-lab", label: "KQL 조인 랩", icon: '<circle cx="9" cy="12" r="5"/><circle cx="15" cy="12" r="5"/>' }
```

(주의: children이 없는 최상위 항목이므로 `NAV.forEach`의 `else` 분기가 처리해 자동으로 group이 아닌 단일 nav-item 버튼으로 렌더링된다. `navOpen`에는 아무 키도 추가하지 않는다.)

- [ ] **Step 4: 이식 스크립트 작성**

`azure-kql-join-lab.html`의 69~682번째 줄(`var GATEWAY_LOGS = [...]`부터 `DOMContentLoaded` 리스너까지 전체)을 원본 그대로 복사한 뒤, 아래 표의 치환만 정확히 적용하고 나머지 코드(데이터 배열·`kustoApplyJoin`·`runKustoQuery2`·`KQL_EXAMPLES`·`KQL_TUTORIAL_STEPS`·`renderChart` 등 전부)는 한 글자도 바꾸지 않는다.

치환 목록:
1. `document.getElementById("tabs")` (585번째 줄) → `document.getElementById("kqlLabTabs")`
2. `document.getElementById("resetBtn")` (597번째 줄) → `document.getElementById("kqlLabResetBtn")`
3. `document.getElementById("app")` (634, 639, 669번째 줄, 총 3곳) → `document.getElementById("kqlLabApp")`
4. `document.querySelectorAll("#tabs button")` (582, 594, 679번째 줄, 총 3곳) → `document.querySelectorAll("#kqlLabTabs button")`
5. `document.getElementById("storageBanner")` (565번째 줄) → **변경하지 않음**(트래커의 기존 배너 재사용)
6. 파일 맨 끝의 아래 블록:

```js
  document.addEventListener("DOMContentLoaded", function () {
    document.querySelectorAll("#tabs button").forEach(function (b) { b.classList.toggle("active", b.getAttribute("data-tab") === state.activeTab); });
    render();
  });
```

→ 다음으로 교체(래퍼 제거, id는 위 4번 규칙 적용):

```js
  document.querySelectorAll("#kqlLabTabs button").forEach(function (b) { b.classList.toggle("active", b.getAttribute("data-tab") === state.activeTab); });
  render();
```

이렇게 치환된 전체 코드를 `(function () { "use strict";` 로 시작하고 `})();` 로 끝나는 IIFE로 감싼 뒤, `azure-migration-tracker.html`의 `</body>` 직전(테마 토글 버튼과 기존 트래커 `<script>...</script>` 블록 뒤)에 새로운 `<script>` 태그로 삽입한다.

- [ ] **Step 5: 정적 충돌 점검**

```bash
grep -c 'id="kqlLabApp"' "azure-migration-tracker.html"
grep -c 'id="kqlLabTabs"' "azure-migration-tracker.html"
grep -c 'id="kqlLabResetBtn"' "azure-migration-tracker.html"
grep -n 'id="storageBanner"' "azure-migration-tracker.html"
grep -n 'DOMContentLoaded' "azure-migration-tracker.html"
```

Expected:
- `kqlLabApp`: HTML 요소 1곳 + JS `getElementById` 참조 3곳 = 총 4건.
- `kqlLabTabs`: HTML 요소 1곳 + JS `getElementById`/`querySelectorAll` 참조 4곳 = 총 5건.
- `kqlLabResetBtn`: HTML 요소 1곳 + JS 참조 1곳 = 총 2건.
- `id="storageBanner"`는 트래커 원본 배너 1곳만 출력되어야 한다(중복 생성 안 됐음을 의미).
- `DOMContentLoaded`는 트래커 자체 코드에서 쓰지 않으므로 0건이어야 한다(이식 스크립트의 래퍼를 제거했는지 확인하는 회귀 점검).

- [ ] **Step 6: 커밋**

```bash
git add "azure-migration-tracker.html"
git commit -m "feat: KQL 멀티테이블 조인 랩을 트래커 사이드바 패널로 통합"
```

---

### Task 3: 브라우저 통합 검증

**Files:**
- 없음(코드 변경 없이 확인만 수행). 문제 발견 시 Task 1/2의 해당 파일로 돌아가 수정한다.

**Interfaces:**
- Consumes: Task 1의 브랜딩/기본 상태, Task 2의 새 패널 전체.
- Produces: 없음(검증 결과만 보고).

- [ ] **Step 1: 파일을 기본 브라우저로 연다**

Run (PowerShell): `Start-Process "D:\99. S\20260730_Azure\azure-migration-tracker.html"`

- [ ] **Step 2: 기본 접힘 상태 확인**

localStorage를 한 번도 저장한 적 없는 시크릿/프라이빗 창으로 열어 사이드바의 `APIM 정책`/`KQL(Kusto)`/`데이터·인프라`/`Azure Monitor`/`로컬 테스트 랩` 5개 그룹이 전부 접혀 있는지(하위 항목이 안 보이는지) 눈으로 확인한다.

Expected: 5개 그룹 전부 닫힘, 화살표(caret)가 오른쪽을 가리킴.

- [ ] **Step 3: 새 메뉴 항목 동작 확인**

사이드바 맨 아래 "KQL 조인 랩"을 클릭한다.

Expected: 같은 탭 안에서 사이드바는 그대로 유지된 채 오른쪽 본문만 "KQL 멀티테이블 실습 랩" 패널(튜토리얼/예제 17개 탭)로 바뀐다. 별도 파일로 이동하거나 새 탭이 열리지 않는다.

- [ ] **Step 4: 기능 회귀 확인**

- "튜토리얼" 탭에서 이전/다음 버튼으로 6단계를 넘기며 각 단계 쿼리 결과·차트가 렌더링되는지 확인.
- "예제 17개" 탭에서 아무 예제나 골라 "내 쿼리 실행"(빈 입력이어도 에러 메시지가 표시되는지), "정답 보기"(정답 쿼리와 결과가 펼쳐지는지) 클릭 확인.
- "다시 시작" 버튼 클릭 → confirm 대화상자 → 확인 시 튜토리얼 1단계로 리셋되고 예제 답안이 모두 지워지는지 확인.
- 트래커의 기존 다른 탭(예: "구조 비교", "Kusto 실행기") 몇 개를 클릭해 이전과 동일하게 동작하는지 확인(회귀 없음 확인).

- [ ] **Step 5: 콘솔 에러 확인**

브라우저 개발자 도구(F12) Console 탭을 열고 위 Step 2~4 조작 중 빨간 에러가 출력되지 않는지 확인한다.

Expected: 에러 0건.

- [ ] **Step 6: 완료 보고**

문제가 없으면 사용자에게 결과를 보고한다. 문제가 있으면 Task 1/2로 돌아가 원인이 되는 치환/삽입 지점을 수정한 뒤 Step 1부터 재검증한다.

---

## Self-Review 체크리스트 (계획 작성자용, 실행 시 참고만)

- [x] 스펙 커버리지: 사이드바 기본 접힘(Task 1) / 메뉴 마지막 추가·링크(Task 2 Step 3) / title·rail-brand 수정(Task 1) 세 요구사항 모두 태스크로 매핑됨.
- [x] 플레이스홀더 없음: 모든 CSS/HTML/치환 규칙이 정확한 원본 줄 번호와 정확한 대체 문자열로 명시됨.
- [x] 타입/이름 일관성: `kqlLabApp`/`kqlLabTabs`/`kqlLabResetBtn`/`kql-lab-tabs` 네이밍이 Task 2의 모든 Step에서 동일하게 사용됨.
