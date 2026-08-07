# Workbooks 구축 실습 — 자동 재생 데모 설계 문서

- 날짜: 2026-08-07
- 대상 파일: `azure-migration-tracker.html` (단일 HTML 학습 Artifact, 추가 방식)
- 선행 참고: `docs/superpowers/specs/2026-08-07-workbook-lab-design.md`(Workbooks 구축 실습 본체), `docs/superpowers/plans/2026-08-07-workbook-lab.md`

## 배경 및 목적

Workbooks 구축 실습을 실제로 사용해 본 사용자가 "어떻게 사용하는지 모르겠다"는 피드백을 줬다. 후속 확인 결과 특정 버튼 하나가 아니라 전체 흐름(미션→셀 조합→파라미터 참조→실행→체크리스트) 자체가 처음 보면 낯설다는 것이 확인됐다. 이 기능은 "▶ 데모 보기" 버튼으로 미션 2(API별 필터링 리포트)의 전 과정을 실제 UI에서 자동으로 재생해 보여주는 시연 모드를 추가해, 사용자가 손을 대기 전에 전체 그림을 한 번 보게 한다.

## 범위

기존 Workbooks 구축 실습의 함수(`loadWorkbookState`, `saveWorkbookState`, `substituteWorkbookParams`, `renderWorkbookCanvas`, `renderWorkbookCellHtml`, `runWorkbookQueryCell`, `initWorkbookLab` 등)와 그 외 기존 8개 패널·로컬 테스트 랩 6개 패널의 함수는 수정하지 않는다 — 순수 추가만 한다. 새 코드는 기존 버튼(`data-action` 셀렉터)에 실제 `.click()`을 호출하고 기존 렌더 함수가 다시 그린 결과를 그대로 이용하는 방식으로 동작한다.

**비목표**:
- 미션 1·3이나 자유 캔버스용 데모는 만들지 않는다 — 미션 2 하나로 핵심 개념(파라미터·쿼리·실행·체크리스트)을 전부 보여주는 것으로 충분하다.
- 데모 스크립트를 "미션 정의에서 자동 생성"하는 일반화된 엔진은 만들지 않는다 — 시나리오가 하나뿐이라 추상화가 이르다(YAGNI). 손으로 쓴 고정 배열로 표현한다.
- 데모 속도(스텝당 딜레이)를 사용자가 설정하는 옵션은 만들지 않는다 — 고정값 사용.

## 결정 사항 (아키텍처)

- **실제 렌더링 함수와 버튼을 그대로 재사용한다.** 별도의 가짜 데모 UI를 새로 그리지 않고, 스크립트가 실제 `[data-action="..."]` 버튼에 `.click()`을 호출해 기존 이벤트 위임 핸들러를 그대로 통과시킨다. 화면이 실제 사용자 조작 결과와 100% 동일하게 보장되고, 훗날 UI가 바뀌어도 데모가 별도로 낡지 않는다는 장점이 있다.
- **재렌더링으로 DOM 참조가 무효화되므로, 스텝마다 대상 엘리먼트를 새로 조회한다.** 클릭 액션은 `renderWorkbookCanvas()`의 전체 재렌더링을 유발해 이전 DOM 노드를 제거하므로, 절대 엘리먼트 참조를 스텝 간에 캐싱하지 않는다.
- **복원 안전장치는 3중이다**: ① 데모 종료(닫기/완료 후 닫기) 시 정상 복원, ② `beforeunload`로 탭을 닫거나 새로고침해도 동기적으로 복원, ③ `MutationObserver`로 사용자가 다른 nav 탭으로 이동해 `panel-lab-workbook`이 `active` 클래스를 잃으면 자동 중단+복원. 세 경로 모두 하나의 `endWorkbookDemo()` 함수로 모은다.
- **데모는 기존 `state.lab["workbook-cells"]`/`persist()`를 그대로 통과시킨다.** 별도의 "데모 모드 플래그"를 `persist()`에 추가하지 않는다(기존 공용 함수 무수정 원칙 유지) — 대신 데모 시작 전 원본 문자열을 백업해뒀다가 종료 시 그 문자열로 되돌리는 방식으로 안전성을 확보한다.
- **완료 시점에는 즉시 복원하지 않는다.** 마지막 스텝(체크리스트 확인) 실행 직후 컨트롤을 "닫기" 하나만 남은 "완료" 상태로 바꾸고, 사용자가 체크리스트 결과를 본 뒤 스스로 닫을 때 복원한다.

## 데이터 모델

### 데모 스크립트 (`WORKBOOK_DEMO_STEPS`, 정적 배열)

```js
var WORKBOOK_DEMO_STEPS = [
  { caption: "먼저 API를 고를 파라미터 셀을 만들어요.",
    targetSelector: '[data-action="add-cell"][data-cell-type="param"]', act: "click" },
  { caption: "orders-api를 선택해봐요.",
    targetSelector: '.workbook-param-select', act: "select", value: "orders-api" },
  { caption: "이 파라미터를 참조하는 쿼리 셀을 추가해요.",
    targetSelector: '[data-action="add-cell"][data-cell-type="query"]', act: "click" },
  { caption: "파라미터를 참조하는 쿼리를 입력해요.",
    targetSelector: '.workbook-query-input', act: "type",
    value: 'ApiManagementGatewayLogs\n| where ApiId == "{ApiIdParam}"\n| take 10' },
  { caption: "실행해서 결과를 확인해요.",
    targetSelector: '[data-action="run"]', act: "click" },
  { caption: "체크리스트로 확인해요 — 3개 모두 통과!",
    targetSelector: '[data-action="check-mission"]', act: "click" }
];
```

`act`는 `click`/`select`/`type` 3종만 지원한다 — 이 6단계를 표현하는 데 필요한 최소 집합이다.

`select` 액션은 `<select>` 엘리먼트의 `value`를 지정값으로 설정한 뒤 `change`와 `input` 이벤트를 순서대로 디스패치해 기존 `handleWorkbookFieldChange`가 그대로 반응하게 한다(실제 사용자가 드롭다운을 바꾼 것과 동일한 코드 경로).

`type` 액션은 대상 `<textarea>`에 `value.length` 문자를 한 글자씩 채워 넣으며 매 글자마다 `input` 이벤트를 디스패치하는 타이핑 애니메이션(약 30ms/자)이다. 일시정지 중 "다음"을 누르면 남은 글자를 즉시 전부 채운 뒤 그 스텝을 완료 처리한다.

### 데모 실행 상태 (모듈 스코프 변수, `state.lab`에 저장하지 않음 — 재생 중임을 나타내는 휘발성 상태)

```js
var workbookDemo = {
  running: false,
  paused: false,
  stepIndex: 0,
  backupRaw: null,          // 데모 시작 전 state.lab["workbook-cells"] 원본
  timerId: null,
  mutationObserver: null
};
```

## 데모 엔진 설계

**`startWorkbookDemo()`**:
1. `workbookDemo.running`이 이미 `true`면 즉시 반환(중복 실행 방지).
2. `workbookDemo.backupRaw = state.lab["workbook-cells"]`로 백업.
3. `window.addEventListener("beforeunload", restoreWorkbookDemoSync)` 등록.
4. `panel-lab-workbook`에 `MutationObserver`를 걸어 `class` 변경 시 `active`를 잃으면 `endWorkbookDemo()` 호출.
5. 미션 전환 버튼(`[data-action="switch-mission"][data-mission-id="m2"]`)에 실제 `.click()`을 호출해 미션 2로 전환(캔버스가 실제 핸들러에 의해 비워짐).
6. `workbookDemo.running = true`, `stepIndex = 0`으로 두고 캡션 배너를 렌더링한 뒤 `runWorkbookDemoStep(0)` 호출.

**`runWorkbookDemoStep(index)`**:
- `index >= WORKBOOK_DEMO_STEPS.length`이면 배너를 "완료" 상태(버튼: 닫기만)로 전환하고 반환한다(복원하지 않음).
- 대상 엘리먼트를 `document.querySelector(step.targetSelector)`로 조회한다. 못 찾으면 `abortWorkbookDemo("데모 재생 중 문제가 발생했습니다. 다시 시도해 주세요.")`를 호출하고 반환한다.
- 대상에 `demo-highlight` 클래스를 부여하고 배너 캡션·진행률(`(index+1) / 6`)을 갱신한다.
- 1.2초 대기 후 `act`에 따라 실행: `click`→`target.click()`, `select`→`value` 설정 후 `change`/`input` 디스패치, `type`→ 타이핑 애니메이션.
- 액션 완료 후 0.8초 대기하고, `workbookDemo.paused`가 아니면 `runWorkbookDemoStep(index + 1)`을 호출한다. 일시정지 상태면 다음 스텝 실행을 대기(“다음” 버튼 클릭 시 수동 호출).
- 모든 타이머 id는 `workbookDemo.timerId`에 저장해 일시정지/닫기 시 `clearTimeout`으로 즉시 취소할 수 있게 한다.

**`endWorkbookDemo()`** (닫기·완료 후 닫기·탭 이탈 공통 경로):
1. 진행 중인 타이머를 전부 취소한다.
2. `state.lab["workbook-cells"] = workbookDemo.backupRaw` 복원 후 `persist()` 호출, `renderWorkbookCanvas()`로 화면 갱신.
3. `beforeunload` 리스너와 `MutationObserver`를 해제한다.
4. 캡션 배너를 제거하고 `workbookDemo`를 초기 상태로 리셋한다.

**`restoreWorkbookDemoSync()`** (`beforeunload` 전용): `endWorkbookDemo()`와 동일한 상태 복원(2번 단계)만 동기적으로 수행한다 — 언로드 중에는 타이머 취소나 DOM 갱신이 의미 없으므로 상태 복원만 한다.

**`abortWorkbookDemo(message)`**: `endWorkbookDemo()`를 호출한 뒤 캡션 배너 자리에 에러 메시지를 잠시 표시한다.

## UI

**진입점**: `renderWorkbookMissionBar()`가 만드는 미션 탭 줄 오른쪽에 `▶ 데모 보기` 버튼(`data-action="start-demo"`) 추가.

**캡션 배너**: `panel-lab-workbook` 내부, 미션 카드 위에 `position: sticky; top: 0`로 붙는 바(`workbook-demo-banner`). 진행률(`3 / 6`) + 캡션 텍스트 + 상태별 버튼을 한 줄에 표시한다.

| 상태 | 보이는 버튼 |
|---|---|
| 재생 중 | 일시정지 / 닫기 |
| 일시정지 | 재생 / 다음 / 닫기 |
| 완료 | 닫기 |

**하이라이트**: 대상 엘리먼트에 `demo-highlight` 클래스(`outline: 2px solid var(--accent-target)` + 펄스 애니메이션).

## 에러 처리 / 엣지 케이스

- **"데모 보기" 중복 클릭**: `workbookDemo.running`이 `true`면 무시.
- **대상 엘리먼트를 찾지 못함**: 즉시 `abortWorkbookDemo(...)`로 중단+복원+에러 메시지. 조용히 멈추지 않는다.
- **"다음" 연타**: 스텝 실행 중임을 나타내는 잠금 플래그로 중복 호출을 막아, 한 번에 정확히 한 스텝만 전진하게 한다.
- **localStorage 사용 불가 환경**: 기존 `state.lab`/`persist()`의 메모리 폴백을 그대로 통과하므로 데모 백업/복원에 별도 분기가 필요 없다.
- **데이터·미션 결합 위험(문서화만, 코드 대응 없음)**: 이 데모 스크립트는 `AM_GATEWAY_LOGS` 데이터와 `WORKBOOK_MISSIONS[1]`(미션 2)의 문구·로직에 고정 결합되어 있다. 훗날 그 데이터나 미션 정의를 바꾸면 데모가 깨질 수 있으므로, 그럴 때는 데모를 수동으로 재검증해야 한다.

## 테스트 방법

이 프로젝트는 빌드/테스트 러너가 없는 단일 HTML Artifact다. 구현 후 브라우저에서 직접 다음을 확인한다:

1. 기존 셀이 있는 상태에서 "데모 보기" 클릭 → 6단계가 순서대로 자동 재생되고 마지막에 체크리스트 3개 모두 ✅로 끝나는지 확인.
2. "닫기" 클릭 → 캔버스가 데모 시작 전 원래 셀 구성으로 정확히 복원되는지 확인.
3. 일시정지 → 다음 → 재생 재개가 정상 동작하는지 확인.
4. 데모 진행 중간에 "닫기"를 눌러 중단해도 원래 상태로 복원되는지 확인.
5. 데모 진행 중 다른 nav 탭(예: "Kusto 실행기")을 클릭 → 데모가 자동으로 중단되고 워크북 상태가 복원되는지 확인.
6. 새로고침(또는 탭 닫기 시도) 중간에 걸어 `beforeunload`가 실제로 발동해 저장값이 복원되는지 확인.
