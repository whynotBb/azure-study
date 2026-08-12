# 트래커 시뮬레이터 자기모순 수정 (1군) 설계

## 배경

Azure 시니어 실무자 관점으로 `index.html`(Azure 전환 학습 허브)을 검토한 결과, 콘텐츠 자체는 넓고 정확하지만 **레퍼런스/문서가 가르치는 내용과 실제 실습 랩이 손으로 보여주는 동작이 서로 모순되는 지점**이 여러 건 발견됐다. Active recall(먼저 써보고 정답과 비교) 방식은 손으로 확인한 결과가 더 강하게 각인되므로, 이 모순은 콘텐츠 누락보다 우선순위가 높은 신뢰 훼손 요인이다.

전체 개선 후보는 성격이 다른 세 그룹(1군: 자기모순 버그 수정 / 2군: 데이터·인프라 콘텐츠 보강 / 3군: 신규 랩 구조)으로 나뉘며, 이번 스펙은 **1군만** 다룬다. 2·3군은 별도 스펙·계획 사이클로 진행한다.

## 범위 (1군 — 이번 스펙에서 다루는 5개 항목)

1. APIM 정책 랩이 `<backend>`/`<on-error>` 섹션을 조용히 무시하는 문제
2. Kong·APIM rate-limit 시뮬레이션이 동일 알고리즘·동일 카운터 키를 써서, 문서가 설명하는 차이를 손으로 확인할 수 없는 문제
3. KQL 퀴즈의 "정답"이 실제 로컬 Kusto 엔진에서 파싱되지 않는 문제 (`toint()`, `ago()` 등)
4. Azure Monitor 레퍼런스의 "join 미지원" 문구가 실제로 join을 지원하는 KQL 조인 랩과 모순되는 문제
5. 데이터·인프라 레퍼런스의 CLI 예시 2건에 대한 문법 재검증

**범위 제외(2·3군, 이번에 다루지 않음)**: `<base/>` 상속/정책 스코프 계층, Data Collection Rules/비용·샘플링 레퍼런스 카드, 데이터·인프라 축 콘텐츠 보강(PostgreSQL 운영/Cassandra 마이그레이션 함정/App Gateway 운영/방화벽 계층/RBAC·관리ID), 타임아웃/재시도/백오프 체험 레인, 카나리·롤백 시나리오, Jira 루브릭.

## Global Constraints

- 빌드 도구 없는 단일 HTML 파일(`index.html`) 그대로 유지 — 별도 파일 분리 없음.
- 새로 추가되는 정책/함수 문법은 반드시 실제 Azure/Kusto 문법을 따른다(프로젝트 CLAUDE.md "허구 문법 금지" 원칙).
- 기존 화이트리스트 철학(지원하지 않는 것은 조용히 무시하지 않고 "미지원" 트레이스로 명시) 그대로 유지·확장.
- KQL 엔진은 파일 안에 **두 벌이 중복 구현**되어 있다(단일 테이블용 IIFE, KQL 조인 랩용 별도 IIFE). 두 IIFE를 병합하는 리팩터링은 이번 스펙 범위 밖 — 동일한 수정을 양쪽에 각각 적용한다.
- 테스트 프레임워크 없음 — 검증은 Node 구문 체크 + Playwright(`C:\Users\whynot\browsertest\test-tracker-merge.js` 확장) + 수동 확인으로 대체.

---

## 설계

### A. APIM 정책 랩 — `<backend>`/`<on-error>` 실행 + 강제 실패 토글

**현재 상태(코드 근거)**
- `runApimMock`(index.html:2007)은 `policies > inbound`, `policies > outbound`만 조회하고 `<backend>`/`<on-error>`는 아예 쿼리하지 않는다.
- `route.backendResponse`(예: index.html:1757, 1802, 1837)는 항상 고정 성공 객체라, 애초에 백엔드가 실패하는 상황을 재현할 방법이 없다.
- 샘플 XML(index.html:1751-1754)에는 이미 `<backend><base /></backend>`가 있고, `POLICY_QUIZ`의 "Backend Retry" 항목 노트(931행)는 "retry는 policies/backend 섹션에 배치"라고 명시한다.

**변경 사항**
1. `panel-lab-api-test`(2093-2094행 호출부)와 `panel-lab-policy`(2139-2142행 호출부) 양쪽의 입력 폼에 **"백엔드 강제 실패(503)" 체크박스**를 추가한다. 체크 시 `input.forceBackendFailure = true`가 `runKongMock`/`runApimMock`에 전달된다.
2. `runApimMock`에 `<backend>` 단계를 추가한다:
   - `policies > backend`를 조회. 자식으로 `<retry>`가 있으면 그 `condition`/`count`/`interval` 속성을 읽는다.
   - `condition` 값이 정확히 `@(context.Response.StatusCode == <N>)` 패턴이고 강제 실패 상태 코드와 일치할 때만 재시도를 수행(기존 check-header/ip-filter와 동일하게 "이 패턴만 지원"). 그 외 패턴은 트레이스에 "이 조건식은 시뮬레이션 대상이 아니며 항상 재시도합니다" 노트를 남기고 무조건 재시도.
   - `count`회만큼 트레이스에 `재시도 N/COUNT (interval초 대기 후) — 여전히 <status>` 기록(실제 대기는 하지 않음). `forceBackendFailure`가 켜져 있는 한 전부 실패로 끝난다.
   - `<retry>`가 없으면 backend 호출 1회로 취급(강제 실패면 실패, 아니면 통과).
3. `runApimMock`에 `<on-error>` 단계를 추가한다:
   - backend 단계에서 에러(강제 실패 또는 재시도 소진)가 발생했을 때만 실행.
   - 자식으로 `<return-response>`가 있으면 기존 핸들러를 재사용해 최종 응답을 커스텀 상태/바디로 교체.
   - `<choose>` 등 그 외 태그는 기존 "미지원" 트레이스 패턴을 그대로 적용(범위 확장 안 함 — `<choose>/<when>` 조건부 분기 자체는 구현하지 않는다).
   - `<on-error>`가 없으면 backend 에러를 그대로 최종 응답으로 반환(기존 동작과 동일, 회귀 없음).
4. `APIM_WHITELIST`(1914행)는 변경하지 않는다(정책 태그 자체는 이미 8개 그대로이며, 이번 변경은 "어느 섹션에서 실행되는가"를 늘리는 것).
5. `panel-lab-policy`의 설명 문구(556행)를 "화이트리스트 7개 정책" → "8개 정책, backend/on-error 섹션도 시뮬레이션"으로 정정한다.

**에러 처리/엣지 케이스**
- `forceBackendFailure`가 꺼져 있으면 A 항목 전체가 기존 동작과 100% 동일해야 한다(회귀 없음이 핵심 성공 기준).
- `<backend>`/`<on-error>` 섹션 자체가 XML에 없는 기존 샘플 라우트는 지금처럼 동작해야 한다.

### B. Rate-limit 정확도 — counter-key 파티셔닝 + Kong/APIM 알고리즘 분리

**현재 상태(코드 근거)**
- `checkRateLimit`(1859행)은 슬라이딩 윈도우 로그 방식이며 Kong(1894행)·APIM(1958행) 양쪽이 동일 함수·동일 키(`route.id`)로 호출된다.
- 샘플 XML(1752행)에 이미 `<rate-limit-by-key calls="5" renewal-period="60" counter-key="@(context.Request.IpAddress)" />`가 있지만 `counter-key` 값은 실제로 평가되지 않는다.
- `LAB_MIGRATION_PITFALLS`(2233-2236행)는 이미 "APIM classic은 슬라이딩 윈도우(분산 환경이라 완전 정확하지 않을 수 있음), v2는 토큰 버킷 — Kong과 1:1로 같다고 가정하면 안 됨"이라고 정확히 설명하고 있다.

**변경 사항**
1. `rate-limit-by-key` 처리 시(`runApimSection`의 1952-1964행 분기), `counter-key` 속성 값이 정확히 `@(context.Request.IpAddress)`이면 카운터 키를 `route.id + "|" + input.clientIp`로 파티셔닝한다. 그 외 표현식이면 기존 check-header 패턴과 동일하게 "이 랩은 counter-key로 IP 주소만 지원합니다" 트레이스 노트를 남기고 `route.id` 단일 카운터로 폴백한다.
2. Kong 쪽 `checkRateLimit` 호출(1894행)은 변경하지 않는다(기존 슬라이딩 윈도우 로그 유지).
3. APIM 전용 `checkRateLimitFixedWindow(store, key, limit, windowSec)`를 새로 추가한다: `bucket = Math.floor(Date.now() / (windowSec*1000))`을 키의 일부로 사용해, 윈도우 경계에서 정확히 리셋되는 고정 윈도우 카운터로 동작. `runApimSection`의 rate-limit/rate-limit-by-key 분기(1958행)에서 `checkRateLimit` 대신 이 함수를 호출하도록 교체.
4. 트레이스 문구에 "이 랩은 실제 v2 토큰 버킷이 아니라 개념 설명용 고정 윈도우로 단순화했습니다"를 명시해, 이번 수정 자체가 또 다른 "가짜 정밀함"이 되지 않도록 한다.

**에러 처리/엣지 케이스**
- 기존 `rate-limit`(by-key 아님) 정책은 지금처럼 `route.id` 단일 카운터를 그대로 사용(파티셔닝 대상 아님).
- 동일 라우트에 서로 다른 `clientIp`로 반복 호출 시, `rate-limit-by-key` 카운터가 IP별로 분리되어야 한다(검증 항목).

### C. KQL where 절 파서 확장 — 함수 호출 + `ago()`/`now()`

**현재 상태(코드 근거)**
- `kustoApplyWhere`가 두 곳에 중복 구현되어 있다: 단일 테이블 엔진(`kustoParseLiteral` 2299행, `kustoApplyWhere` 2321행, `runKustoQuery` 2412행 — fixture 날짜 2026-08-06)과 KQL 조인 랩 엔진(각각 3321행대 — fixture 날짜 2026-08-11), 서로 별개의 IIFE.
- where 조건 정규식 `^(\w+)\s*(op)\s*(.+)$`은 왼쪽에 바레 컬럼명만 허용, `kustoParseLiteral`은 오른쪽에 true/false/숫자/따옴표 문자열만 허용 — 함수 호출 전혀 미지원.
- `KQL_QUIZ`(943행)의 정답 예시가 `toint(ResultCode) >= 500`(949행)과 `ago(1h)`(957행)를 실제로 사용한다. 이 두 표현은 `AppRequests`/`TimeGenerated` 등 KQL 조인 랩의 실제 테이블·컬럼명과 일치해, 학습자가 그대로 조인 랩에 입력해볼 개연성이 높다.

**변경 사항** — 아래를 **두 엔진(단일 테이블용, 조인 랩용) 모두에 동일하게 적용**한다.
1. where 조건 정규식을 확장해 왼쪽에 `함수명(컬럼명)` 또는 바레 `컬럼명` 둘 다 허용한다. 지원 함수는 `toint`/`tostring`/`todouble`/`tolong` 4개로 제한(레퍼런스·퀴즈에 실제로 등장하는 것만 — 범용 표현식 엔진으로 확장하지 않음). 화이트리스트 밖 함수는 명확한 에러 메시지("지원하지 않는 함수: xxx()")를 던진다.
2. `kustoParseLiteral`을 확장해 오른쪽에 `ago(<숫자><단위>)`와 `now()`를 새 리터럴 형태로 인식한다. 단위는 기존 `bin()`과 동일하게 `s`/`m`/`h`/`d`를 지원.
3. 각 엔진에 **고정된 "시뮬레이션 기준 시각" 상수**를 도입해 `ago()`/`now()`가 이 값을 기준으로 계산하도록 한다(절대 실제 `Date.now()`를 쓰지 않는다 — 그러면 실제 날짜가 fixture 날짜를 지날 때마다 결과가 깨진다):
   - 단일 테이블 엔진(fixture 마지막 이벤트 09:09:25, 2026-08-06): `SIMULATED_NOW = "2026-08-06T09:15:00Z"`
   - 조인 랩 엔진(fixture 마지막 이벤트 09:11:5x, 2026-08-11): `SIMULATED_NOW = "2026-08-11T09:15:00Z"`

**에러 처리/엣지 케이스**
- 기존에 통과하던 모든 예제/정답 쿼리(단일 테이블 예제, 조인 랩 17개 예제)는 이번 확장 후에도 동일한 결과를 내야 한다(순수 확장, 기존 문법 제거 없음).
- `KQL_QUIZ` 1번(`toint(ResultCode) >= 500`)과 2번(`ago(1h)`)을 조인 랩의 텍스트 입력창에 그대로 붙여넣었을 때 파싱 에러 없이 실행되어야 한다(검증 항목).

### D. 문구 정정

1. `AZURE_MONITOR_REF`에서 "이 프로젝트의 로컬 Kusto 실행기는 join을 지원하지 않습니다"라고 쓴 부분을 "이 페이지의 단일 테이블 실행기(Kusto 실행기 탭)는 join을 지원하지 않지만, 여러 테이블 join은 'KQL 조인 랩' 탭에서 지원합니다"로 정정한다.
2. 데이터·인프라 레퍼런스의 CLI 예시 2건 재검증 결과: **`--zonal-resiliency`/`--zone`/`--standby-zone`(1488행)과 `--enable-managed-identity`(1542행) 모두 실제 `az` CLI에 존재하는 정확한 문법으로 확인됨**(공식 문서 기준). 허구 문법이 아니었으므로 예시 자체는 그대로 두되, `--enable-managed-identity` 항목에 "Consumption 티어 전용이며, 다른 티어(Developer/Basic/Standard/Premium 등)에서는 `az apim update ... --set identity.type=SystemAssigned`를 사용합니다" 캐비아트를 한 줄 추가한다.

### E. 검증 방법

1. 각 파일 수정 후 Node로 두 `<script>` 블록 구문 검사(기존 세션에서 쓰던 `new Function(scriptText)` 패턴 재사용).
2. `C:\Users\whynot\browsertest\test-tracker-merge.js`(Playwright)를 재실행해 기존 회귀 확인(사이드바 기본 접힘, 새 패널 전환, 튜토리얼/예제/리셋, 콘솔 에러 0건).
3. 새 Playwright 시나리오 추가(같은 파일에 스텝 추가 또는 신규 스크립트):
   - "백엔드 강제 실패" 토글 켠 뒤 backend에 `<retry>`+`<on-error>`가 있는 라우트 실행 → 트레이스에 재시도 횟수와 최종 커스텀 응답이 보이는지 확인.
   - 동일 `rate-limit-by-key` 라우트에 서로 다른 `clientIp` 두 개로 반복 호출 → 카운터가 IP별로 분리되는지 확인(한쪽은 429, 다른 쪽은 통과).
   - `KQL_QUIZ` 1·2번 "정답 예시" 텍스트를 조인 랩 편집기에 그대로 입력해 정상 실행되는지 확인.
4. 강제 실패 토글을 끈 상태에서 기존 라우트들이 이전과 동일한 결과를 내는지 확인(회귀 없음).

---

## Self-Review 체크리스트

- [x] 플레이스홀더 없음: 모든 변경 사항이 정확한 파일 내 위치(줄 번호)와 정확한 동작 정의를 포함함.
- [x] 내부 일관성: A~E 항목이 서로 참조하는 함수명(`checkRateLimit`, `checkRateLimitFixedWindow`, `runApimMock`, `runApimSection`)이 전부 동일하게 사용됨.
- [x] 범위 검사: 1군 5개 항목에 집중, `<base/>` 상속·DCR/비용 레퍼런스·데이터인프라 콘텐츠 보강·신규 랩 구조는 명시적으로 범위 밖으로 분리함 — 단일 계획으로 구현 가능한 크기.
- [x] 모호성 점검: rate-limit 알고리즘 정밀도(고정 윈도우 vs 토큰 버킷 vs 미구현)는 사용자 승인을 거쳐 "고정 윈도우 + 트레이스 문구로 단순화 명시"로 확정함.
