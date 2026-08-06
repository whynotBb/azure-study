# Azure Monitor 학습 콘텐츠 — 설계 문서

- 날짜: 2026-08-06
- 대상 파일: `azure-migration-tracker.html` (단일 HTML 학습 Artifact, 추가 방식)
- 선행 참고: `docs/superpowers/specs/2026-08-04-local-test-lab-design.md` (동일 문서 포맷), `docs/superpowers/plans/2026-08-04-local-test-lab.md` (동일 plan 포맷)

## 배경 및 목적

기존 트래커는 Kong→APIM 정책, KQL(Kusto), 데이터·인프라(Cassandra/PostgreSQL/App Gateway/SSL/Azure 기초) 세 축을 레퍼런스+퀴즈 쌍으로 다룬다. 그러나 프로젝트 배경(CLAUDE.md)이 명시한 전환 대상 중 **Azure Monitor**(Kibana/Elasticsearch의 전환 목표)는 지금까지 "구조 비교" 대응표 수준으로만 다뤄졌고, APIM 정책처럼 "레퍼런스 → 퀴즈 → 직접 실행해 검증"하는 3단 학습 경로가 없다. 이 기능은 Azure Monitor의 핵심 구성요소(Log Analytics/Kusto 쿼리, 알림 규칙, 액션 그룹, 워크북/대시보드, 진단 설정)에 대해 동일한 3단 학습 경로를 추가한다.

로컬 테스트 파트는 사용자 결정에 따라 **Kusto 쿼리 실행 시뮬레이터**와 **알림 규칙 평가 시뮬레이터** 두 패널로 나눈다. 전자는 로그 조회(Log search) 계열, 후자는 메트릭 알림(Metric alert) 계열을 각각 대표한다 — 둘 다 넣는 이유는 Azure Monitor의 "로그 기반"과 "메트릭 기반" 관찰 방식이 서로 다른 사고방식(쿼리 작성 vs 임계값/평가주기 설계)을 요구하기 때문에, 하나로 합치면 어느 한쪽 학습이 얕아진다.

## 범위

**새 레퍼런스/퀴즈 그룹 "Azure Monitor"**를 기존 `NAV` 배열에 추가하고(`datainfra` 그룹 다음, `jira` 그룹 앞), 기존 "로컬 테스트 랩" 그룹(`lab`)의 children에 새 패널 2개를 추가한다. 기존 8개 패널과 로컬 테스트 랩 5개 패널·데이터·함수(`renderRefPage`, `buildFullQuiz`, `refUrl`, `runKongMock`, `runApimMock`, `LAB_ROUTES` 등)는 전혀 수정하지 않는다 — 순수 추가(additive)만 한다.

새 패널 4개:

1. **Azure Monitor 레퍼런스** (`panel-azuremonitor-ref`) — `renderRefPage` 재사용. 6개 카테고리(아래 데이터 모델).
2. **Azure Monitor 퀴즈** (`panel-azuremonitor-quiz`) — `buildFullQuiz` 재사용. 레퍼런스 전 항목 자동 변환(CLAUDE.md 규칙).
3. **Kusto 실행기** (`panel-lab-kusto`, 기존 `lab` 그룹 children에 추가) — 사용자가 KQL(Kusto) 쿼리를 직접 작성 → 로컬 in-memory 로그 데이터셋(`ApiManagementGatewayLogs` 스키마, 기존 `orders-api`/`users-api`/`legacy-api` 트래픽 서사 재사용)에 대해 실제 실행 → 결과 테이블 확인.
4. **알림 시뮬레이터** (`panel-lab-alert`, 기존 `lab` 그룹 children에 추가) — 사용자가 임계값/평가 빈도/조회 기간(lookback)을 설정 → 샘플 메트릭 시계열에 슬라이딩 윈도우 평가를 적용 → 발동(Fired)/해제(Resolved) 타임라인 확인.

## 결정 사항 (아키텍처)

- **Kusto 실행기는 가상 테이블 1개(`ApiManagementGatewayLogs`)만 노출한다.** 레퍼런스 페이지는 `AppRequests`/`AppExceptions`/`AppDependencies` 스키마도 다루지만, 실행 엔진까지 다중 테이블을 지원하면 스코프가 과도해진다. 하나의 실제 스키마로 정확하게 동작하는 것이, 여러 테이블을 얕게 흉내 내는 것보다 학습 신뢰도가 높다는 판단이다.
- **`join` 연산자는 지원하지 않는다.** 레퍼런스/퀴즈에는 `join` 문법과 예시를 포함하지만, 로컬 실행기의 화이트리스트에는 넣지 않는다(정확한 재현이 어렵고 잘못 구현하면 "거짓 성공"이 됨). 화이트리스트 밖 연산자는 기존 APIM Policy 엔진의 "⚠ 시뮬레이션 미지원" 배지 패턴을 그대로 따른다.
- **알림 시뮬레이터는 정적 임계값(Static threshold) + 메트릭 시그널만 지원한다.** 동적 임계값(머신러닝 기반)은 로컬에서 의미 있게 재현할 수 없으므로 레퍼런스/퀴즈에서 개념만 다루고 실행기 범위에서 제외한다. 로그 검색 알림(Log search alert)은 Kusto 실행기와 개념이 겹치므로 별도 구현하지 않는다.

## 데이터 모델

### Azure Monitor 레퍼런스 (`AZURE_MONITOR_REF`, 6 카테고리 · `renderRefPage`/`buildFullQuiz` 공용 포맷)

기존 `APIM_POLICIES`/`KQL_OPERATORS`/`DATA_INFRA`와 동일한 셰이프: `{ cat, anchor, items: [{ name, code, url, easy, when, example }] }`.

1. **Log Analytics 테이블 스키마** — `AppRequests`, `AppExceptions`, `AppDependencies`, `ApiManagementGatewayLogs` 각 테이블의 실제 컬럼(리서치로 확인, 출처: `learn.microsoft.com/azure/azure-monitor/reference/tables/*`).
2. **KQL(Kusto) 연산자** — `let`, `where`, `project`, `summarize ... by bin()`, `top`, `join`(레퍼런스에만 존재, 실행기 미지원 명시), `render` (출처: `learn.microsoft.com/kusto/query/*`).
3. **알림 규칙(Alert Rules)** — 시그널 유형 5종(Metric/Log search/Activity log/Smart detection/Prometheus), "평가 빈도 vs 조회 기간" 차이, 정적/동적 임계값 (출처: `alerts-types`, `alerts-create-metric-alert-rule`).
4. **액션 그룹(Action Groups)** — Email/Email ARM Role/SMS/Push/Voice(알림) + Automation Runbook/Event Hubs/Function/ITSM/Logic Apps/Secure Webhook/Webhook(액션) (출처: `action-groups`).
5. **워크북 vs 대시보드** — 목적·공유 방식·상호작용 차이 (출처: `workbooks-overview`, `best-practices-visualize`).
6. **진단 설정(Diagnostic Settings)** — API Management(`ApiManagementGatewayLogs` 등 5개 카테고리) vs Key Vault(`AuditEvent` 단일 카테고리) 대조 — "리소스마다 카테고리 구성이 다르다"를 명확한 대비로 학습 (출처: `monitor-api-management-reference`; App Service/Key Vault 카테고리는 2차 출처 경유로 확인됐으므로 항목 작성 시 공식 레퍼런스 표로 재검증 필수).

각 항목의 `example`은 리서치에서 확인한 실제 문법만 사용한다(허구 금지, CLAUDE.md 규칙).

### Kusto 실행기 — 가상 테이블 데이터셋 (`AM_GATEWAY_LOGS`)

`ApiManagementGatewayLogs` 실제 컬럼 기반, 로컬 테스트 랩의 `orders-api`/`users-api`/`legacy-api` 트래픽 서사를 재사용해 15~20행 정적 배열로 생성(런타임에 `LAB_ROUTES`를 참조하지 않는 독립 데이터 — 기존 변수 미변경 원칙 유지):

| 컬럼 | 타입 | 설명 |
|---|---|---|
| `TimeGenerated` | string (ISO) | 요청 시각 |
| `ApiId` | string | `orders-api` / `users-api` / `legacy-api` |
| `Method` | string | `GET`/`POST` |
| `Url` | string | 요청 경로 |
| `ResponseCode` | int | 200/401/403/429/404 |
| `BackendResponseCode` | int | 백엔드 응답(대개 ResponseCode와 동일, `legacy-api`는 차단이라 0) |
| `IsRequestSuccess` | bool | `ResponseCode < 400` |
| `TotalTime` | long (ms) | 종단 처리 시간 |
| `CallerIpAddress` | string | 샘플 IP 4종 |

행 구성: 성공 200 다수, `key-auth` 실패로 인한 401 2~3건, `rate-limiting` 초과 429 2~3건, `legacy-api` 차단 유사 케이스 일부, `TotalTime`이 유난히 큰 느린 요청 2~3건(느린 요청 top N 퀴즈용) 포함.

### 알림 시뮬레이터 — 샘플 메트릭 시계열 (`AM_METRIC_SERIES`)

1분 간격 60포인트(1시간) 정적 배열 `{ t: 분 오프셋, value }`. 시나리오: "평균 응답 시간(ms)"이 평상시 80~150ms 유지되다가 20~35분 구간에서 400~900ms로 급등 후 회복 — 평가 빈도·조회 기간 조합에 따라 발동 시점이 달라지는 것을 보여주는 데 적합한 패턴.

## Kusto 실행기 — 미니 엔진 설계

**`runKustoQuery(queryText)`** — `AM_GATEWAY_LOGS`를 대상으로 파이프(`|`)로 분리된 스테이지를 순서대로 적용한다.

1. 첫 줄이 `let <name> = <value>;` 형태면 텍스트 치환 후 제거(여러 줄 허용).
2. 첫 파이프 이전 토큰은 테이블명 — `ApiManagementGatewayLogs`가 아니면 즉시 에러(`알 수 없는 테이블: ...`).
3. 화이트리스트 스테이지만 순차 적용, 각 스테이지 파싱 실패 또는 화이트리스트 밖 연산자는 그 자리에서 실행 중단 + 명확한 에러 메시지(`trace`에 어느 스테이지에서 멈췄는지 기록) — 조용히 건너뛰지 않는다.
   - `where <col> <op> <literal>` (`==`,`!=`,`>`,`<`,`>=`,`<=`,`contains`,`!contains`, `and`로 체이닝)
   - `project <col1>, <col2>, ...`
   - `summarize <agg>(<col>) by <col2>` 또는 `summarize count() by bin(TimeGenerated, <span>)` (agg: `count`,`avg`,`sum`,`max`,`min`)
   - `top N by <col> [asc|desc]`
   - `sort by <col> [asc|desc]` (`order by`는 동의어로 처리)
   - `take N` (`limit`은 동의어)
   - `render timechart` — 마지막 스테이지 전용, 결과 배열에 `chartHint: true`만 부여(실제 차트 라이브러리 없이 렌더러가 간단한 막대 시각화로 표현)
4. 최종 결과를 `{ columns: [...], rows: [...], trace: [{stage, resultCount}] }` 셰이프로 반환 → 기존 `lab-result` 스타일 재사용해 테이블 렌더링.

**정답 검증 방식**: 자동 채점하지 않는다(기존 KQL 퀴즈와 동일 철학). 문제마다 "정답 쿼리"를 실행한 결과와 "사용자 쿼리" 실행 결과를 나란히 비교해 사용자가 스스로 판단한다.

## 알림 시뮬레이터 — 평가 엔진 설계

**`evaluateAlertRule({ metricSeries, thresholdOp, thresholdValue, aggregation, evalFrequencyMin, lookbackMin })`**

1. `t = lookbackMin`부터 `evalFrequencyMin` 간격으로 시계열 끝까지 평가 시점을 이동.
2. 각 평가 시점에서 `[t - lookbackMin, t]` 구간 포인트를 `aggregation`(Average/Maximum/Minimum/Total/Count)으로 집계.
3. 집계값과 `thresholdOp`/`thresholdValue`를 비교해 조건 충족 시 `Fired`, 그 시점부터 미충족으로 전환되면 `Resolved`.
4. 반환: `[{ evalTime, aggregatedValue, breached, transition: "fired"|"resolved"|null }]` — 타임라인 렌더링에 사용.

**엣지 케이스**: `lookbackMin > evalFrequencyMin`이면 평가 구간이 겹친다는 안내 배지 표시(실제 Azure 동작과 동일하게 허용은 하되 UI로 설명). `lookbackMin`이 데이터 시작 이전이면 그 구간은 "데이터 부족"으로 표시하고 평가 자체를 건너뛴다(0으로 취급해 오탐 발생시키지 않음).

## 네비게이션 변경

`NAV` 배열에 `datainfra` 다음, `jira` 이전에 삽입:

```js
{ id: "azuremonitor", label: "Azure Monitor", icon: "...", children: [
    { id: "azuremonitor-ref", label: "Azure Monitor 레퍼런스", icon: "..." },
    { id: "azuremonitor-quiz", label: "Azure Monitor 퀴즈", icon: "..." }
  ] }
```

기존 `lab` 그룹의 `children` 배열 끝에 추가:

```js
{ id: "lab-kusto", label: "Kusto 실행기", icon: "..." },
{ id: "lab-alert", label: "알림 시뮬레이터", icon: "..." }
```

`defaultState().navOpen`에 `azuremonitor: true` 추가(다른 그룹과 동일 관례).

## 에러 처리 / 엣지 케이스

- Kusto 쿼리 파싱/실행 에러: 어느 스테이지에서 실패했는지 명시(예: "2번째 스테이지(`summarize`)에서 실패: 알 수 없는 집계 함수 `avgg`"), 조용히 무시하지 않는다.
- 화이트리스트 밖 연산자(`join`, `mv-expand` 등): 차단 메시지 + "이 시뮬레이터는 지원하지 않지만 실제 Azure Data Explorer/Log Analytics에서는 지원한다"는 안내(정확성 가장하지 않음).
- 빈 쿼리 실행 시 실행 버튼 비활성화(에러 throw 없음) — 기존 랩 패턴과 동일.
- 알림 시뮬레이터: 임계값 미입력 시 실행 비활성화. `lookbackMin`/`evalFrequencyMin`이 0 이하면 입력 검증 메시지.
- localStorage 불가 환경: 기존 `storageBanner` 패턴 재사용(두 실행기 모두 계산은 in-memory, 쿼리 텍스트/설정 입력값만 `state`에 저장해 `persist()` 대상).

## 테스트 방법

이 저장소는 git 저장소가 아니고 빌드/테스트 러너가 없다 — 모든 검증은 브라우저에서 `azure-migration-tracker.html`을 직접 열어 수동 확인한다.
