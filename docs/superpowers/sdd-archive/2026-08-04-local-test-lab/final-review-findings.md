# 최종 전체 브랜치 리뷰 — Critical/Important findings (Fix 대상)

리뷰 범위: 90c3c03..8941701 (8개 태스크 전체). Minor 항목은 이번 fix wave에서 다루지 않는다(후속 정리로 남김).

## Critical

### C1. 랩 개요 패널이 APIM 정책 XML을 화면에 표시하지 못한다 (`azure-migration-tracker.html` `renderLabOverview`, 약 1600-1603줄)

```js
'<div class="code-block">' + r.apimPolicyXml + '</div>'
```

`apimPolicyXml`은 이스케이프되지 않은 원문(`<policies>`, `<check-header ...>`)인데 `innerHTML`로 주입되어, HTML 파서가 이를 DOM 엘리먼트로 파싱해버린다. 3개 카드의 "APIM 대응 정책" 블록이 사실상 빈 코드 블록으로 렌더된다.

**주의**: `apimPolicyXml` 필드는 ① `renderLabOverview`의 innerHTML, ② Task 5 textarea `.value`, ③ `DOMParser`(runApimMock) 세 곳에서 원문 그대로 소비된다. **데이터 자체를 사전 이스케이프하면 안 되고**, ①번 렌더링 지점에서만 `escapeHtml()`을 적용해야 한다. `kongYaml` 필드도 같은 문제이니 함께 이스케이프한다(`<`, `>`는 안 쓰지만 일관성을 위해 적용).

**수정**: 간단한 `escapeHtml(str)` 헬퍼(`&`, `<`, `>`, `"` 치환)를 추가하고, `renderLabOverview`에서 `r.kongYaml`/`r.apimPolicyXml`을 코드 블록에 넣기 직전에만 이 헬퍼를 통과시킨다. `LAB_ROUTES` 데이터, Task 5의 textarea 로드(`loadXml`), `runApimMock`의 파싱 로직은 전혀 건드리지 않는다(원문이 필요하다).

### C2. 마이그레이션 함정 카드 #2가 사실과 다른 내용을 가르친다 (`LAB_MIGRATION_PITFALLS[1]`, 두 번째 배열 항목)

현재 근거: "APIM의 check-header 정책은 `ignore-case` 속성이 필수이며 명시적으로 `true`로 설정하지 않으면 대소문자를 구분해 비교합니다."

공식 문서(check-header-policy) 원문: `ignore-case`는 헤더 **값(value)** 비교에만 적용되고, 헤더 **이름(name)** 매칭과는 무관하다. HTTP 헤더 이름은 APIM에서도 대소문자 구분 없이 매칭된다.

**수정**: 이 카드의 시나리오와 근거를 실제로 맞는 내용으로 다시 쓴다. 두 가지 중 하나를 선택:
- (a) 시나리오를 "헤더 **값**의 대소문자" 문제로 바꾼다 — 예: Kong 플러그인이 대소문자 무관하게 값을 비교했는데 APIM `check-header`는 `ignore-case`를 명시적으로 `true`로 켜지 않으면 값 비교 시 대소문자를 구분한다는 내용으로.
- (b) 헤더 **이름** 대소문자 시나리오는 유지하되, 정답을 "아니요, 동일하게 인식됩니다(HTTP 헤더 이름은 대소문자 무관)"로 바꾸고, 근거를 `ignore-case`가 아닌 실제 사실(HTTP 헤더 이름 자체는 원래 대소문자 무관이라는 표준 동작)로 다시 쓴다.
어느 쪽이든 근거 문장에 `ignore-case`가 "값 비교"에 적용된다는 정확한 설명을 포함시킨다.

## Important

### I1. 대표 샘플의 `<rate-limit>`은 실제 APIM에서 동작하지 않는 조합이다 (`LAB_ROUTES[0].apimPolicyXml`, orders-get 라우트)

공식 문서: `rate-limit` 정책은 **구독 키(subscription key)로 접근하는 API에만 적용된다**. 이 샘플은 커스텀 `apikey` 헤더 인증(`check-header`)만 쓰므로 `rate-limit`이 실제로는 적용되지 않는 조합이다. Kong의 consumer/IP 기준 rate-limiting에 대응하는 올바른 정책은 `rate-limit-by-key`(예: `counter-key="@(context.Request.IpAddress)"` 또는 커스텀 키 기준)이며, 이 트래커의 기존 APIM 정책 레퍼런스에 이미 실려 있다.

**수정**: `orders-get`의 `apimPolicyXml`에서 `<rate-limit calls="5" renewal-period="60" />`를 `<rate-limit-by-key calls="5" renewal-period="60" counter-key="@(context.Request.IpAddress)" />` 같은 실제 동작하는 형태로 교체한다. `runApimMock`의 `APIM_WHITELIST`와 `runApimSection`의 rate-limit 분기도 `rate-limit-by-key` 태그명을 인식하도록 갱신해야 한다(속성은 `rate-limit`과 동일하게 `calls`/`renewal-period`를 쓰므로 처리 로직 자체는 거의 그대로 재사용 가능 — `counter-key` 속성은 이 mock에서는 무시해도 되지만, 태그명 매칭만 정확히 해야 한다). `랩 개요`/`API 테스트`/`APIM Policy` 탭에 표시되는 XML 전부가 이 필드를 공유하므로 한 곳만 고치면 세 곳에 전파된다. Kong YAML은 그대로 두어도 된다(Kong rate-limiting은 이 시나리오에서 실제로 유효하다).

### I2. `<cors>` 배치 순서가 공식 가이드와 반대다 (`LAB_ROUTES[0].apimPolicyXml`, orders-get 라우트)

공식 문서: "cors 정책이 inbound 섹션의 **첫 번째** 정책이 아니면 예상치 못한 동작이 발생할 수 있다"고 명시한다. 현재 샘플은 `check-header` → `rate-limit`(또는 I1 수정 후 `rate-limit-by-key`) → `cors` 순인데, `cors`를 `<base />` 바로 다음(맨 앞)으로 옮겨야 한다.

**수정**: `orders-get`의 `apimPolicyXml`에서 `<cors>...</cors>` 블록 전체를 `<base />` 바로 다음, `check-header` 앞으로 옮긴다.

### I3. 마이그레이션 카드 #5의 근거가 preflight와 simple request를 혼동한다 (`LAB_MIGRATION_PITFALLS[4]`, 다섯 번째 배열 항목)

현재 근거: "`terminate-unmatched-request`의 기본값이 false이므로, Origin이 매칭되지 않아도 요청은 계속 진행되어 결과적으로 200이 반환됩니다."

공식 문서상 `terminate-unmatched-request=false`일 때 **preflight(OPTIONS) 요청**은 "다른 cors 정책을 찾고, 없으면 빈 200 OK로 **종료**"한다(백엔드로 진행하지 않음). "요청이 계속 진행되어 결과적으로 200"이라는 설명은 **GET/HEAD 같은 simple request**에만 해당하는 이야기다. 카드의 최종 결론(200)은 우연히 맞지만 메커니즘 설명이 틀렸다.

**수정**: 근거 문장을 시나리오(OPTIONS 프리플라이트)에 맞게 고친다 — 예: "`terminate-unmatched-request`의 기본값이 `false`이므로, Origin이 매칭되지 않는 프리플라이트(OPTIONS) 요청은 다른 cors 정책이 없으면 빈 200 OK로 즉시 종료되어(백엔드로 전달되지 않음), 커스텀 프리플라이트 응답을 흉내내던 Kong 설정과 결과적으로 비슷한 200이 나옵니다. 다만 GET/HEAD 같은 단순 요청에서는 이 옵션이 요청을 그대로 통과시키는(CORS 헤더만 안 붙는) 다른 의미로 작동하므로 두 경로를 혼동하면 안 됩니다." 문서 자체가 속성표(`false`)와 Common configuration issues 절(`true`)에서 기본값을 다르게 적고 있다는 점도 한 줄 각주로 남기면 좋다(선택사항, 시간 되면).

### I4. 화이트리스트 정책이 잘못된 섹션(outbound/backend)에 있으면 아무 흔적 없이 무시된다 (`runApimSection`, 약 1675-1730줄)

`check-header`/`rate-limit`/`ip-filter`/`rewrite-uri`/`cors` 분기가 전부 `&& isInbound`로 게이팅돼 있다. `<outbound>`에 이런 태그를 넣으면 화이트리스트에는 있으므로 "미지원" 배지도 안 붙고 실행도 안 돼서 trace에 아무 것도 안 남는다. 설계상 "차단하지 않고 미지원 배지로 정직하게 노출" 원칙 위반.

**수정**: 각 `&& isInbound` 조건이 실패하는 경우(즉 인바운드 전용 정책이 outbound/backend 섹션에 들어온 경우)를 잡는 `else` 분기를 추가해 `trace.push({policy: tag, effect: "이 섹션(" + (isInbound ? "inbound" : "outbound") + ")에서는 시뮬레이션 대상이 아님"})` 같은 안내를 남긴다. `<backend>` 섹션은 애초에 순회 대상이 아니므로(현재도 그렇다), 이 부분은 범위 밖으로 그대로 둬도 된다.

### I5. 엔진이 `check-header`의 `name`을 하드코딩하고 `ignore-case`를 완전히 무시한다 (`runApimSection`, check-header 분기, 약 1688줄)

```js
var actual = name.toLowerCase() === "apikey" ? input.apikeyHeader : "";
```

Task 5(자유 편집 탭)에서 학습자가 `name`을 다른 값으로 바꾸거나(`Authorization` 등) `ignore-case="true"`를 켜고 대소문자 다른 값을 입력하면, 시뮬레이터의 한계가 정책의 실제 동작인 것처럼 보인다(항상 401).

**수정**: 최소한의 개선으로, `name`이 `"apikey"`가 아닌 경우와 `ignore-case="true"`인 경우를 trace에 명시적으로 안내한다 — 예: `name`이 apikey가 아니면 `trace.push({policy:"check-header", effect: "이 랩에서는 'apikey' 헤더 값만 입력받을 수 있어 '" + name + "' 헤더는 항상 빈 값으로 시뮬레이션됩니다"})`를 남기고 기존 401 처리로 진행하며, `ignore-case`가 `"true"`이면 값 비교를 대소문자 무시로 바꾼다(`values`와 `actual`을 모두 소문자로 비교). 과도한 엔지니어링은 피하고, 이 두 가지 안내/보정만 추가한다.

### I6. API 테스트 탭과 APIM Policy 탭이 rate-limit(-by-key 포함) 카운터를 공유하는데 초기화 수단이 없다 (`apimRateCounters`, 약 1609/1697줄)

두 탭 모두 같은 `apimRateCounters[route.id]`를 쓴다. Task 4에서 6회 클릭 실습(429까지)을 마친 뒤 APIM Policy 탭으로 넘어가면 첫 실행부터 429가 뜨고, 60초를 기다리는 것 외에 초기화 방법이 없으며 UI 설명도 없다.

**수정**: 가장 간단한 해결책으로, `APIM Policy` 탭에 "카운터 초기화" 버튼을 하나 추가해 클릭 시 `apimRateCounters = {}`(및 `kongRateCounters = {}`)로 리셋한다. 버튼 옆에 "API 테스트 탭과 요율 제한 카운터를 공유합니다. 429가 계속 뜨면 여기를 눌러 초기화하세요" 같은 안내 문구를 붙인다.

### I7. 설계 문서가 요구한 Azure 측 로그 샘플이 "로그 비교" 탭에 누락됐다

설계 문서는 Kong 로그와 함께 "Azure 쪽: 동일 트래픽의 `ApiManagementGatewayLogs` 진단 로그 예시"도 요구했으나 구현에는 Kong 로그만 있다.

**수정**: `LAB_KONG_LOG_SAMPLE` 옆에 `LAB_AZURE_LOG_SAMPLE` 같은 상수를 추가해, 동일한 5개 요청이 `ApiManagementGatewayLogs`에 쌓였다고 가정한 예시 행을 보여준다(실제 컬럼명 사용: `TimeGenerated`, `ApiId`, `Method`, `Url`, `ResponseCode`, `BackendResponseCode`, `CallerIpAddress`, `TotalTime`, `BackendTime`). JSON 배열이나 파이프(`|`) 구분 텍스트 등 코드 블록으로 표시 가능한 형태면 되고, `renderLabLogs()`에서 Kong 로그 블록 바로 아래 새 섹션으로 추가한다. 기존 3개 질문의 정답 KQL/결과 설명은 이미 이 컬럼명을 쓰고 있으므로 그대로 둔다.

---

## 참고: 이번 fix wave에서 다루지 않는 항목 (Minor, 후속 정리로 남김)

M1(localStorage 얕은 병합으로 기존 사용자에게 lab 그룹이 접힌 채 시작), M2(마이그레이션 카드의 `data-guess` 속성 미사용 죽은 코드), M3(로그 비교 퀴즈가 기존 퀴즈와 힌트/버튼 라벨 동작이 다름), M4(Task4/5 라우트 탭 로직 중복), M5(필수 속성 누락 시 NaN 노출), M6(XSS 억제 범위가 body에도 해당하지만 위험도 낮음 판정은 유효), M7(Kong 샘플 YAML에 cors 플러그인이 있는데 kongPlugins 배열엔 없음), M8(랩 개요와 CLAUDE.md 레퍼런스→퀴즈 자동변환 규칙의 관계 미논의) — 이번 라운드에서는 수정하지 않는다.
