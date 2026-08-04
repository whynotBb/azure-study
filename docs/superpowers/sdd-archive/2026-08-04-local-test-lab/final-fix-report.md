# 최종 리뷰 지적사항 수정 보고서

대상 파일: `azure-migration-tracker.html` (경로: `D:\99. S\20260730_Azure\azure-migration-tracker.html`)
검증 방법: `browse` 스킬(Playwright 기반 headless Chromium)로 파일을 직접 열어 실제 인터랙션 수행. 코드 트레이스가 아닌 실제 렌더링/실행 결과를 확인함.

---

## C1. 랩 개요 패널이 APIM 정책 XML을 화면에 표시하지 못함

**변경**
- `azure-migration-tracker.html` 약 1597줄: `escapeHtml(str)` 헬퍼 함수 신규 추가(`&`, `<`, `>`, `"` 치환).
- `renderLabOverview()`(약 1601-1603줄): `r.kongYaml`, `r.apimPolicyXml`를 코드 블록에 넣기 직전에만 `escapeHtml()`을 통과시키도록 수정. `LAB_ROUTES` 원본 데이터, Task 5 textarea 로드(`loadXml`), `runApimMock`의 `DOMParser` 파싱 로직은 그대로 둠(원문 필요).

**브라우저 검증 증거**
`$B html "#labOverviewBody"` 로 추출한 실제 DOM 내용(발췌):
```
<div class="code-block">&lt;policies&gt;
  &lt;inbound&gt;
    &lt;base /&gt;
    &lt;cors allow-credentials="false"&gt;
      ...
    &lt;/cors&gt;
    &lt;check-header name="apikey" ...&gt;
      &lt;value&gt;demo-key-123&lt;/value&gt;
    &lt;/check-header&gt;
    &lt;rate-limit-by-key calls="5" renewal-period="60" counter-key="@(context.Request.IpAddress)" /&gt;
  &lt;/inbound&gt;
  ...
&lt;/policies&gt;</div>
```
`&lt;`/`&gt;`로 이스케이프되어 있어 브라우저가 이를 실제 DOM 엘리먼트로 파싱하지 않고 텍스트로 그대로 렌더링함(화면에는 `<policies>` 등 리터럴 텍스트로 보임). 3개 카드 전부 동일하게 확인됨.

**결론**: 수정됨(FIXED).

---

## C2. 마이그레이션 함정 카드 #2가 사실과 다른 내용을 가르침

**변경**
- `LAB_MIGRATION_PITFALLS[1]`(azure-migration-tracker.html, 약 1938-1940줄): 옵션 (a) 채택 — 시나리오를 "헤더 **값**의 대소문자" 문제로 재작성. `ignore-case`가 헤더 **값** 비교에만 적용되고 기본값이 `false`라는 점, 헤더 **이름** 매칭과는 무관(HTTP 헤더 이름 자체는 원래 대소문자 무관)하다는 점을 근거 문장에 명시.

**브라우저 검증 증거**
마이그레이션 탭 → 시나리오 2 전체 텍스트(정답 공개 버튼 클릭 후 추출):
> "Kong 플러그인에서 `apikey` 헤더 값을 대소문자 관계없이 비교하도록 커스텀했는데(예: `DEMO-KEY-123`도 통과), 이를 APIM `check-header`로 그대로 옮기면서 `ignore-case` 속성을 설정하지 않았다(기본값 `false`). 클라이언트가 대소문자만 다른 값(`DEMO-KEY-123`)을 보내면 APIM에서도 Kong과 동일하게 통과할까?"
> 정답: "아니요, 통과하지 않을 수 있습니다."
> 근거: "`check-header` 정책의 `ignore-case` 속성은 헤더 **값**(value) 비교에만 적용되며 기본값은 `false`입니다 — 헤더 **이름** 매칭과는 무관합니다(HTTP 헤더 이름 자체는 대소문자 구분 없이 매칭됩니다). ..."

시나리오/정답/근거가 서로 모순 없이 일관됨을 확인.

**결론**: 수정됨(FIXED).

---

## I1. 대표 샘플의 `<rate-limit>`이 실제 APIM에서 동작하지 않는 조합

**변경**
- `LAB_ROUTES[0].apimPolicyXml`(orders-get, 약 1503줄): `<rate-limit calls="5" renewal-period="60" />` → `<rate-limit-by-key calls="5" renewal-period="60" counter-key="@(context.Request.IpAddress)" />`로 교체.
- `APIM_WHITELIST`(약 1666줄): `"rate-limit-by-key"` 추가(기존 `"rate-limit"`도 유지).
- `runApimSection`의 rate-limit 분기(약 1708-1720줄): 태그 조건을 `tag === "rate-limit" || tag === "rate-limit-by-key"`로 확장. 처리 로직(calls/renewal-period 읽기, `checkRateLimit` 호출)은 기존 그대로 재사용, `counter-key` 속성은 이 mock에서 무시.
- Kong YAML은 변경하지 않음(요구사항대로).

**브라우저 검증 증거(엔드투엔드)**
API 테스트 탭 → GET /orders 선택 → apikey="demo-key-123" 입력 → 실행. APIM mock 결과:
```
실행 결과 상태: 200
trace:
  cors — CORS 헤더 추가 (Access-Control-Allow-Origin 등)
  check-header — 통과 — apikey 값 일치
  rate-limit-by-key — 통과 — 남은 호출 4회
```
표시된 XML뿐 아니라 실제 실행 엔진에서도 `rate-limit-by-key` 태그가 인식되어 트레이스에 나타남을 확인.

**결론**: 수정됨(FIXED).

---

## I2. `<cors>` 배치 순서가 공식 가이드와 반대

**변경**
- `LAB_ROUTES[0].apimPolicyXml`(orders-get, 약 1497-1509줄): `<cors>...</cors>` 블록 전체를 `<base />` 바로 다음, `check-header` 앞으로 이동.

**브라우저 검증 증거**
랩 개요 탭에서 추출한 XML 순서: `<base />` → `<cors>` → `<check-header>` → `<rate-limit-by-key>`.
API 테스트 탭 실행 결과 trace 순서도 `cors` → `check-header` → `rate-limit-by-key`로 동일하게 나타남(위 I1 증거와 동일 실행 결과 재사용).

**결론**: 수정됨(FIXED).

---

## I3. 마이그레이션 카드 #5의 근거가 preflight와 simple request를 혼동

**변경**
- `LAB_MIGRATION_PITFALLS[4]`(약 1955줄): 근거 문장을 프리플라이트(OPTIONS) 시나리오에 맞게 재작성. "빈 200 OK로 즉시 종료(백엔드로 전달되지 않음)"이라는 정확한 메커니즘과 GET/HEAD 단순 요청과의 차이를 명시. 문서 자체의 기본값 표기 모호성에 대한 각주도 추가.

**브라우저 검증 증거**
마이그레이션 탭 → 시나리오 5 전체 텍스트:
> 근거: "`terminate-unmatched-request`의 기본값이 `false`이므로, Origin이 매칭되지 않는 프리플라이트(OPTIONS) 요청은 다른 cors 정책이 없으면 빈 200 OK로 즉시 종료되어(백엔드로 전달되지 않음), 커스텀 프리플라이트 응답을 흉내내던 Kong 설정과 결과적으로 비슷한 200이 나옵니다. 다만 GET/HEAD 같은 단순 요청에서는 이 옵션이 요청을 그대로 통과시키는(CORS 헤더만 안 붙는) 다른 의미로 작동하므로 두 경로를 혼동하면 안 됩니다. (참고: 공식 문서도 속성표와 'Common configuration issues' 절에서 기본값을 다르게 적고 있어 문서 자체에 모호함이 있습니다.)"

**결론**: 수정됨(FIXED).

---

## I4. 화이트리스트 정책이 잘못된 섹션에 있으면 아무 흔적 없이 무시됨

**변경**
- `runApimSection`(약 1679-1761줄): `check-header`/`rate-limit`(`-by-key`)/`ip-filter`/`rewrite-uri`/`cors` 각 분기를 `tag === "..." && isInbound` 형태에서 `tag === "..."` 로 태그만 먼저 매칭한 뒤, 내부에서 `if (!isInbound) { trace.push({policy: tag, effect: "이 섹션(" + (isInbound?"inbound":"outbound") + ")에서는 시뮬레이션 대상이 아님"}) } else { 기존 로직 }` 형태로 재구성.

**브라우저 검증 증거**
APIM Policy 탭에서 orders 라우트 기본 XML을 편집해 `<rate-limit-by-key .../>`를 `<outbound>` 블록으로 이동 후 실행. 결과 trace:
```
cors — CORS 헤더 추가 (Access-Control-Allow-Origin 등)
check-header — 통과 — apikey 값 일치
rate-limit-by-key — 이 섹션(outbound)에서는 시뮬레이션 대상이 아님
```
상태 200(더 이상 차단하지 않되, 정직하게 미시뮬레이션임을 노출).

**결론**: 수정됨(FIXED).

---

## I5. 엔진이 `check-header`의 `name`을 하드코딩하고 `ignore-case`를 무시

**변경**
- `runApimSection`의 check-header 분기(약 1688-1707줄):
  - `name`이 `"apikey"`가 아닌 경우, `trace.push({policy:"check-header", effect: "이 랩에서는 'apikey' 헤더 값만 입력받을 수 있어 '" + name + "' 헤더는 항상 빈 값으로 시뮬레이션됩니다"})`를 먼저 남기고 기존 401 처리로 진행.
  - `ignore-case="true"`이면 `values`와 `actual` 값을 모두 소문자로 변환해 비교(`actualCmp`/`valuesCmp`).

**브라우저 검증 증거**
- name 불일치 케이스: `name="apikey"` → `name="Authorization"`로 변경 후 실행(apikey 입력값 "demo-key-123"). 결과:
```
상태: 401
trace:
  cors — CORS 헤더 추가 (Access-Control-Allow-Origin 등)
  check-header — 이 랩에서는 'apikey' 헤더 값만 입력받을 수 있어 'Authorization' 헤더는 항상 빈 값으로 시뮬레이션됩니다
  check-header — 차단 — Authorization 값 불일치 (401)
```
- ignore-case 케이스: `ignore-case="false"` → `ignore-case="true"`로 변경, apikey 입력값을 대소문자 다른 "DEMO-KEY-123"으로 테스트. 결과:
```
상태: 200
trace:
  cors — CORS 헤더 추가 (Access-Control-Allow-Origin 등)
  check-header — 통과 — apikey 값 일치
  rate-limit-by-key — 통과 — 남은 호출 4회
```
(수정 전이었다면 대소문자 비교로 인해 401이 났을 상황이 이제 200으로 통과함을 확인.)

**결론**: 수정됨(FIXED).

---

## I6. API 테스트 탭과 APIM Policy 탭이 rate-limit 카운터를 공유하는데 초기화 수단이 없음

**변경**
- `renderLabPolicy()`(약 1840-1847줄): "카운터 초기화" 버튼(`#labPolicyResetCounters`)과 안내 문구("API 테스트 탭과 요율 제한(rate-limit) 카운터를 공유합니다. 429가 계속 뜨면 여기를 눌러 초기화하세요.")를 추가. 클릭 시 `apimRateCounters = {}; kongRateCounters = {};`로 리셋.

**브라우저 검증 증거**
1. APIM Policy 탭에서 "실행"을 6회 연속 클릭 → 결과 제목 `실행 결과 — 429` 확인(요율 제한 도달).
2. `#labPolicyResetCounters` 클릭.
3. "실행" 재클릭 → 결과 제목 `실행 결과 — 200` 확인(카운터가 초기화되어 다시 통과).

**결론**: 수정됨(FIXED).

---

## I7. 설계 문서가 요구한 Azure 측 로그 샘플이 "로그 비교" 탭에 누락

**변경**
- `LAB_AZURE_LOG_SAMPLE` 상수 신규 추가(`LAB_KONG_LOG_SAMPLE` 바로 아래, 약 1876-1883줄): 동일한 5개 요청이 `ApiManagementGatewayLogs`에 쌓였다고 가정한 예시(파이프 구분 텍스트), 실제 컬럼명(`TimeGenerated`, `ApiId`, `Method`, `Url`, `ResponseCode`, `BackendResponseCode`, `CallerIpAddress`, `TotalTime`, `BackendTime`) 사용. Kong 로그의 5xx/최대 지연/IP 집계 결과와 일관되도록 수치 설계(예: `/legacy/report`만 5xx, `/orders` 812ms가 최대 TotalTime, `203.0.113.10`이 3건으로 최다).
- `renderLabLogs()`(약 1906-1908줄): Kong 로그 블록 바로 아래에 새 섹션으로 렌더링. 기존 3개 질문의 정답 KQL/결과 설명은 변경하지 않음.

**브라우저 검증 증거**
로그 비교 탭 heading 목록: `Kong/Nginx 원본 로그 샘플 || Azure 쪽: ApiManagementGatewayLogs 진단 로그 샘플(동일 트래픽) || 질문`
두 번째 코드 블록 내용:
```
TimeGenerated | ApiId | Method | Url | ResponseCode | BackendResponseCode | CallerIpAddress | TotalTime | BackendTime
2024-12-30T20:26:40Z | orders-api | GET | /orders | 200 | 200 | 203.0.113.10 | 42 | 38
2024-12-30T20:26:45Z | orders-api | GET | /orders | 429 | 0 | 203.0.113.11 | 3 | 0
2024-12-30T20:26:50Z | users-api | GET | /users/1 | 403 | 0 | 203.0.113.10 | 5 | 0
2024-12-30T20:27:00Z | legacy-api | GET | /legacy/report | 503 | 0 | 198.51.100.20 | 1 | 0
2024-12-30T20:27:10Z | orders-api | GET | /orders | 200 | 200 | 203.0.113.10 | 812 | 790
```

**결론**: 수정됨(FIXED).

---

## 종합

- Critical 2건(C1, C2), Important 7건(I1-I7) 모두 코드 수정 완료 및 실제 브라우저 인터랙션으로 검증 완료.
- 브라우저 콘솔 에러 없음(`console --errors` → "no console errors", 전체 검증 과정 전후 확인).
- Minor 항목(M1-M8)은 지시대로 손대지 않음.
- 판단이 필요했던 지점: C2는 findings 문서가 제시한 두 옵션 중 (a)(헤더 값 대소문자 시나리오로 변경)를 채택함. 근거: 실제 학습 포인트("ignore-case는 값에만 적용된다")를 더 직접적으로 가르칠 수 있고, 시나리오/정답/근거 세 요소를 자연스럽게 일치시키기 쉬웠음.
