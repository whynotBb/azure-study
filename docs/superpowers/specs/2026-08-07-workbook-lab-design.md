# Workbooks 구축 실습 — 설계 문서

- 날짜: 2026-08-07
- 대상 파일: `azure-migration-tracker.html` (단일 HTML 학습 Artifact, 추가 방식)
- 선행 참고: `docs/superpowers/specs/2026-08-06-azure-monitor-design.md` (동일 문서 포맷, Kusto 실행기/알림 시뮬레이터 설계), `docs/superpowers/plans/2026-08-04-local-test-lab.md`

## 배경 및 목적

Azure Monitor 레퍼런스의 "워크북 vs 대시보드" 항목(`am-workbooks-dashboards`)은 Workbooks가 "텍스트·로그 쿼리(KQL)·메트릭·매개변수를 결합한 인터랙티브 분석 캔버스"라는 개념만 다루고, 실제로 셀을 조합해 보는 실습이 없다. 로컬 테스트 랩의 다른 6개 항목(Jira 티켓 작성, APIM Policy 시뮬레이터, 로그 비교, 마이그레이션 함정 예측, Kusto 실행기, 알림 시뮬레이터)은 모두 "직접 작성 → 정답/시뮬레이션 결과와 비교"하는 능동회상 패턴을 따르는데, Workbooks는 이 경로가 없다. 이 기능은 로컬 테스트 랩에 7번째 항목 "Workbooks 구축"을 추가해 이 공백을 채운다.

핵심 차별점은 **기존 Kusto 실행기(`lab-kusto`)와의 연동**이다. 워크북의 쿼리 셀은 별도 실행 엔진을 새로 만들지 않고 기존 `runKustoQuery()`를 그대로 재사용하며, Kusto 실행기 탭에서 작성한 쿼리를 워크북 쿼리 셀로 "가져오기"할 수 있다.

## 범위

기존 8개 패널·로컬 테스트 랩 6개 패널·데이터·함수(`renderRefPage`, `buildFullQuiz`, `runKustoQuery`, `runApimMock`, `evaluateAlertRule` 등)는 수정하지 않는다 — 순수 추가(additive)만 한다.

새 패널 1개:

- **Workbooks 구축** (`panel-lab-workbook`, 기존 `lab` 그룹 children 끝에 추가, nav id `lab-workbook`) — 텍스트/파라미터/쿼리/차트 4종 셀을 조합해 워크북을 직접 만들어 보는 캔버스 빌더. 미션 3개(난이도 순) + 자유 캔버스를 함께 제공한다.

**비목표** (이번 스코프에 포함하지 않음):

- 진단 설정(Diagnostic Settings) 활성화 실습 — 별도 향후 실습 후보로 식별되어 있으나 이번 스코프 밖.
- 실제 Azure Workbooks JSON 템플릿 import/export.
- 워크북 시각화를 대시보드에 "Pin" 고정하는 기능.
- 시간 범위(Time Range) 매개 변수 — 공식 문법상 특수 매크로 확장(`{TimeRange}` → `ago(1d)` 등)이라 이번 스코프의 일반 문자열 파라미터 치환과는 다른 별도 구현이 필요해 제외. 파라미터 종류는 드롭다운(문자열 값) 1종만 지원한다.

## 결정 사항 (아키텍처)

- **셀은 단일 정렬 배열로 관리하고 구조 변경 시에만 전체 재렌더링한다.** `state.lab["workbook-cells"]`에 셀 배열을 저장하고, 셀 추가/삭제/순서 변경 시 캔버스 전체를 `innerHTML`로 재구성한다. 텍스트/쿼리 입력 같은 키 입력 이벤트는 `state`만 갱신하고 DOM은 재구성하지 않는다(기존 `kustoQueryInput`의 `input` 리스너와 동일 패턴 — 포커스 손실 없음). 셀 유형별로 독립된 리스트를 두는 대안은 "위→아래로 자유롭게 셀을 쌓는다"는 Workbooks의 핵심 경험을 잃으므로 채택하지 않는다.
- **쿼리 셀은 재실행형이다.** 워크북 쿼리 셀은 KQL 텍스트를 저장만 하고, `[실행]`을 누를 때마다 파라미터 토큰을 치환한 뒤 기존 `runKustoQuery()`를 호출한다. 실행 결과를 "얼려서" 저장하지 않는다 — 실제 Azure Workbooks의 쿼리 셀이 매번 재실행되는 것과 동일한 개념이며, 결과 스냅샷을 저장하는 방식은 CLAUDE.md의 "허구 동작 금지" 원칙과 충돌할 소지가 있어 제외했다.
- **파라미터는 공식 문법 `{ParameterName}` 토큰 치환만 지원한다.** ([공식 문서](https://learn.microsoft.com/ko-kr/azure/azure-monitor/visualize/workbooks-parameters) "KQL을 사용하여 매개 변수 참조" 절 확인) 파라미터 종류는 드롭다운(고정 옵션 목록에서 문자열 값 선택) 1종만 지원하고, 쿼리 셀 실행 직전에 단순 문자열 대체를 수행한다. JSONPath 서식(`{param:$.x}`)이나 `:label`/`:base64` 같은 형식 지정자는 레퍼런스에 언급하지 않고 이번 스코프에서 다루지 않는다(과도한 확장).
- **차트 셀은 실제 SVG 라인 차트로 렌더링한다.** 기존 Kusto 실행기의 `render timechart`가 표로 대체 표시하는 것과 달리, 워크북의 차트 셀은 이 실습의 핵심 가치("시각적 분석 캔버스")를 체감시키기 위해 순수 SVG `<polyline>`을 새로 그린다. 차트 라이브러리는 추가하지 않는다(CSP·단일 파일 원칙 유지).
- **화이트리스트 원칙은 쿼리 셀에서 `runKustoQuery()` 재사용을 통해 자동으로 계승된다.** 별도 화이트리스트 목록을 워크북 쪽에 새로 두지 않는다.
- **채점은 자동 정답 비교가 아니라 구조적 체크리스트다.** 자유 텍스트/쿼리를 문자열로 정확히 비교할 수 없으므로(마이그레이션 함정 예측·알림 시뮬레이터와 동일 철학), 미션별로 "이런 셀이 있고 이렇게 연결됐는가"를 구조적으로 판정하고, 통과 시 모범 답안 구성을 보여준다.

## 데이터 모델

### 워크북 상태 (`state.lab["workbook-cells"]`, JSON 문자열로 저장 — 기존 `alert-config` 저장 방식과 동일)

```js
{
  missionId: "free" | "m1" | "m2" | "m3",
  cells: [
    { id: "c1", type: "text",  value: "..." },
    { id: "c2", type: "param", name: "ApiIdParam", options: ["orders-api","users-api","legacy-api"], value: "orders-api" },
    { id: "c3", type: "query", value: "ApiManagementGatewayLogs\n| where ApiId == \"{ApiIdParam}\"\n| take 10", lastRunOk: null },
    { id: "c4", type: "chart", title: "응답시간(ms)" }
  ]
}
```

- `type: "query"`의 `lastRunOk`는 `null`(미실행) / `true`(성공) / `false`(에러) 3가지 값을 가진다. 체크리스트가 "실행해봤는지"와 "성공했는지"를 구분하는 데 쓰인다.
- `type: "param"`의 `options`는 고정값 `["orders-api", "users-api", "legacy-api"]` — 기존 `AM_GATEWAY_LOGS`(Kusto 실행기 데이터셋)에 실제 존재하는 `ApiId` 값과 일치시켜, 파라미터를 참조하는 쿼리가 실제로 결과를 반환하도록 한다.
- `type: "chart"`는 별도 데이터 입력이 없다 — 항상 기존 `AM_METRIC_SERIES`(알림 시뮬레이터의 60포인트 샘플 시계열)를 그린다.

### 미션 정의 (`WORKBOOK_MISSIONS`, 코드 내 정적 배열)

| id | 제목 | 요구 셀 | 체크리스트 |
|---|---|---|---|
| `m1` | 느린 요청 Top 5 리포트 | 쿼리 셀 1개 | 쿼리 셀 ≥1 & 그중 하나 이상 `lastRunOk === true` |
| `m2` | API별 필터링 리포트 | 파라미터 셀 1개 + 참조 쿼리 셀 1개 | 파라미터 셀 ≥1 & 그 파라미터명을 `{...}` 형태로 포함하는 쿼리 셀 존재 & 그 쿼리의 `lastRunOk === true` |
| `m3` | 장애 조사용 종합 워크북 | 텍스트 1 + 파라미터 1 + 참조 쿼리 1 + 차트 1 | m2 조건 전부 + 텍스트 셀 ≥1 & 차트 셀 ≥1 |

각 미션은 `modelAnswer: { cells: [...] }` 필드로 모범 답안 셀 구성을 가지며, 체크리스트 통과 후 "모범 답안 구성 보기"를 누르면 사용자의 캔버스 옆에 나란히 렌더링한다(값을 자동으로 캔버스에 주입하지는 않음 — 비교만).

## Workbooks 미니 빌더 설계

**파라미터 토큰 치환** — `substituteWorkbookParams(queryText, cells)`:

1. 쿼리 텍스트에서 `{([A-Za-z0-9_]+)}` 패턴을 전부 찾는다.
2. 각 토큰명이 현재 캔버스의 `param` 셀 중 하나의 `name`과 일치하면 그 파라미터의 현재 `value`(예: `orders-api`)로 문자열 치환한다.
3. 일치하는 파라미터 셀이 없는 토큰이 남아 있으면 실행하지 않고 에러를 반환한다: `"정의되지 않은 매개 변수 참조: {Foo}"`.
4. 치환이 끝난 텍스트를 그대로 기존 `runKustoQuery()`에 넘긴다 — 이후의 화이트리스트 검사·컬럼 검증·에러 메시지는 전부 기존 로직을 그대로 통과한다.

**Kusto 실행기 연동**:

- 워크북 캔버스 상단에 `Kusto 실행기 탭 쿼리 가져오기` 버튼을 둔다.
- 클릭 시 `state.lab["kusto-editor"]`(Kusto 실행기 탭의 현재 편집 중인 쿼리 텍스트)를 그대로 값으로 삼아 새 쿼리 셀을 캔버스 끝에 추가한다.
- `state.lab["kusto-editor"]`가 빈 값이면 버튼을 `disabled` 처리하고 "Kusto 실행기 탭에서 먼저 쿼리를 작성하세요" 안내를 표시한다.

**차트 셀 렌더링** — `renderWorkbookChart()`:

- `AM_METRIC_SERIES`(60개 `{t, value}` 포인트)의 `value` 최소/최대를 구해 `viewBox="0 0 600 120"`에 선형 매핑한 `<polyline points="...">` 1개를 생성한다.
- x축 라벨은 `0`, `20`, `40`, `59`(분)만, y축 라벨은 최솟값/최댓값만 텍스트로 표기한다(차트 라이브러리 없이 순수 SVG 문자열).

**캔버스 렌더링** — `renderWorkbookCanvas()`:

- `state.lab["workbook-cells"].cells`를 순서대로 순회해 셀 유형별 HTML을 이어붙인다.
- 각 셀에 `↑`/`↓`(순서 변경)/`삭제` 버튼을 붙인다. 첫 셀의 `↑`, 마지막 셀의 `↓`는 비활성화.
- 상단에 `+ 텍스트`, `+ 파라미터`, `+ 쿼리`, `+ 차트`, `Kusto 실행기 탭 쿼리 가져오기` 5개 버튼.

## 네비게이션 변경

기존 `lab` 그룹의 `children` 배열 끝(`lab-alert` 다음)에 추가:

```js
{ id: "lab-workbook", label: "Workbooks 구축", icon: "..." }
```

`panel-lab-workbook` 섹션을 `panel-lab-alert` 다음에 추가(eyebrow `"18 · Local Test Lab"`).

## 에러 처리 / 엣지 케이스

- 정의되지 않은 파라미터 토큰 참조: 위 "파라미터 토큰 치환" 3항목 참고 — 실행 자체를 막고 명확한 에러 메시지.
- 화이트리스트 밖 연산자, 존재하지 않는 컬럼, 빈 쿼리, 알 수 없는 테이블명: 기존 `runKustoQuery()`의 에러 메시지를 그대로 재사용(중복 구현 없음).
- 셀이 0개인 상태에서 "체크리스트 확인"을 누르면 에러가 아니라 "셀을 먼저 추가하세요" 안내로 처리한다.
- 미션 전환(`m1` → `m2` 등)을 누르면 별도 확인 대화상자 없이 즉시 캔버스를 초기화한다 — 미션별로 다른 목표를 요구하므로 이전 미션의 셀이 남아 있으면 체크리스트 판정이 혼란스러워진다. 다만 예측 불가능한 데이터 손실로 느껴지지 않도록, 미션 전환 버튼 옆에 "전환 시 현재 캔버스가 초기화됩니다" 안내 문구를 상시 노출한다.
- localStorage 저장 불가 환경: 기존 `storageBanner` 패턴이 이미 전역으로 커버하므로 별도 처리 불필요.

## 테스트 방법

이 프로젝트는 빌드/테스트 러너가 없는 단일 HTML Artifact다. 구현 완료 후 브라우저에서 `azure-migration-tracker.html`을 직접 열어, 미션 1~3을 실제로 완주하며 체크리스트·모범 답안 비교·Kusto 실행기 연동("가져오기" 버튼)·차트 렌더링이 전부 동작하는지 수동으로 확인한다(다른 6개 랩과 동일한 검증 방식).
