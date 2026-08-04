## Task 1: Nav 그룹 + 패널 셸 + CSS 추가

**Files:**
- Modify: `azure-migration-tracker.html` (CSS는 `</style>` 직전, `NAV` 배열은 `jira` 항목 뒤, 패널 HTML은 `panel-jira` `</section>` 뒤, `state.navOpen` 기본값은 `defaultState()` 안)

**Interfaces:**
- Produces: 5개 새 `data-panel` id (`panel-lab-overview`, `panel-lab-api-test`, `panel-lab-policy`, `panel-lab-logs`, `panel-lab-migration`), 컨테이너 DOM id(`labOverviewBody`, `labApiTestBody`, `labPolicyBody`, `labLogsBody`, `labMigrationBody`) — Task 2·4·5·6·7이 이 id들에 렌더링한다.

- [ ] **Step 1: CSS 추가**

`azure-migration-tracker.html`에서 `.theme-toggle { position: fixed; ... }` 규칙 바로 다음, `</style>` 바로 앞에 추가:

```css
.lab-compare-grid { display: grid; grid-template-columns: 1fr 1fr; gap: var(--sp-4); }
@media (max-width: 720px) { .lab-compare-grid { grid-template-columns: 1fr; } }
.lab-result-title { font-size: var(--fs-md); margin-bottom: var(--sp-2); }
.lab-status-2xx { color: #1a7f37; font-weight: 700; }
.lab-status-4xx { color: #b45309; font-weight: 700; }
.lab-status-5xx { color: #b91c1c; font-weight: 700; }
.lab-headers { font-size: var(--fs-xs); color: var(--text-dim); margin: var(--sp-2) 0; font-family: var(--font-mono); }
.lab-trace { display: flex; flex-direction: column; gap: 4px; margin-top: var(--sp-2); }
.lab-trace-row { font-size: var(--fs-xs); color: var(--text-dim); }
.lab-trace-row code { color: var(--accent-target); }
.lab-input-row { display: flex; gap: var(--sp-3); flex-wrap: wrap; margin-bottom: var(--sp-3); }
.lab-input-row label { font-size: var(--fs-xs); color: var(--text-faint); display: flex; flex-direction: column; gap: 4px; }
.lab-input-row input { border: 1px solid var(--border); border-radius: var(--radius-md); padding: 6px 10px; background: var(--bg); color: var(--text); }
.lab-route-tabs { display: flex; gap: var(--sp-2); flex-wrap: wrap; margin-bottom: var(--sp-4); }
.lab-route-tabs button.active { border-color: var(--accent-target); color: var(--accent-target); }
```

- [ ] **Step 2: `NAV` 배열에 그룹 추가**

`azure-migration-tracker.html`에서 `{ id: "jira", label: "AIDD·Jira", ... }` 항목 뒤, 배열을 닫는 `];` 앞에 추가:

```js
,
    { id: "lab", label: "로컬 테스트 랩", icon: '<path d="M9 3h6M10 3v6l-5 9a2 2 0 0 0 2 3h10a2 2 0 0 0 2-3l-5-9V3"/>', children: [
        { id: "lab-overview", label: "랩 개요", icon: '<rect x="3" y="4" width="18" height="16" rx="1"/><path d="M3 9h18"/>' },
        { id: "lab-api-test", label: "API 테스트", icon: '<path d="M4 17l6-6-6-6M12 19h8"/>' },
        { id: "lab-policy", label: "APIM Policy", icon: '<path d="M6 4h9l5 5v11H6z"/><path d="M15 4v5h5"/><path d="M9 12h6M9 15.5h6"/>' },
        { id: "lab-logs", label: "로그 비교", icon: '<rect x="3" y="4" width="18" height="16" rx="1"/><path d="M7 9l3 3-3 3M13 15h4"/>' },
        { id: "lab-migration", label: "마이그레이션", icon: '<path d="M4 12h16M14 6l6 6-6 6"/>' }
      ] }
```

- [ ] **Step 3: `defaultState()`의 `navOpen`에 `lab: true` 추가**

`defaultState()` 안의 `navOpen: { apim: true, kqlgroup: true, datainfra: true }`를 다음으로 교체:

```js
navOpen: { apim: true, kqlgroup: true, datainfra: true, lab: true }
```

- [ ] **Step 4: 패널 셸 HTML 추가**

`<section class="panel" id="panel-jira" data-panel>...</section>` 닫는 태그 직후, `</div>`(main div 닫힘) 앞에 추가:

```html
<section class="panel" id="panel-lab-overview" data-panel>
  <div class="panel-head">
    <span class="eyebrow">09 · Local Test Lab</span>
    <h1>랩 개요</h1>
    <p class="desc">Azure 실 접속 권한 없이도, 샘플 Kong 설정과 그 APIM 대응 정책을 로컬 mock 엔진으로 실행해 결과를 비교합니다. 이 페이지의 샘플 데이터를 아래 4개 탭이 공유합니다.</p>
  </div>
  <div id="labOverviewBody"></div>
</section>

<section class="panel" id="panel-lab-api-test" data-panel>
  <div class="panel-head">
    <span class="eyebrow">10 · Local Test Lab</span>
    <h1>API 테스트</h1>
    <p class="desc">샘플 라우트를 선택하고 값을 입력해 실행하면, 같은 요청에 대해 Kong mock과 APIM mock이 각각 어떻게 응답하는지 나란히 비교합니다.</p>
  </div>
  <div id="labApiTestBody"></div>
</section>

<section class="panel" id="panel-lab-policy" data-panel>
  <div class="panel-head">
    <span class="eyebrow">11 · Local Test Lab</span>
    <h1>APIM Policy</h1>
    <p class="desc">정책 XML을 직접 편집해서 실행해 보세요. 화이트리스트 7개 정책(check-header, rate-limit, cors, set-header, rewrite-uri, ip-filter, return-response)만 실제 동작을 시뮬레이션하며, 그 외 태그는 구문은 인정하되 "미지원" 배지로 표시합니다.</p>
  </div>
  <div id="labPolicyBody"></div>
</section>

<section class="panel" id="panel-lab-logs" data-panel>
  <div class="panel-head">
    <span class="eyebrow">12 · Local Test Lab</span>
    <h1>로그 비교</h1>
    <p class="desc">Kong/Nginx 로그 샘플을 보고 질문에 맞는 KQL(Kusto)을 직접 작성한 뒤 정답과 비교하세요. 기존 KQL 퀴즈와 달리 실제 로그 라인을 입력 맥락으로 사용합니다.</p>
  </div>
  <div id="labLogsBody"></div>
</section>

<section class="panel" id="panel-lab-migration" data-panel>
  <div class="panel-head">
    <span class="eyebrow">13 · Local Test Lab</span>
    <h1>마이그레이션</h1>
    <p class="desc">Kong→APIM 이관 시 놓치기 쉬운 함정을 시나리오로 예측해 보세요. "발생한다/안 한다"를 먼저 판단한 뒤 근거를 확인합니다.</p>
  </div>
  <div id="labMigrationBody"></div>
</section>
```

- [ ] **Step 5: 브라우저에서 확인**

파일 탐색기에서 `azure-migration-tracker.html`을 더블클릭해 브라우저로 연다.
확인 사항:
1. 좌측 네비게이션에 "로컬 테스트 랩" 그룹이 새로 보이고 펼쳐져 있다(다른 그룹처럼 기본 열림).
2. 그 아래 "랩 개요/API 테스트/APIM Policy/로그 비교/마이그레이션" 5개 항목이 보인다.
3. 각 항목을 클릭하면 해당 패널(제목·설명만 있고 본문은 비어있음)로 전환된다.
4. 기존 "구조 비교", "APIM 정책" 등 8개 항목을 클릭해도 기존과 동일하게 동작한다(회귀 없음).

---

