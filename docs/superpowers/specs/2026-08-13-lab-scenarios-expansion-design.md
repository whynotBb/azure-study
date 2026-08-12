# 신규 랩 구조 보강 (3군) 설계

## 배경

1군(자기모순 버그 수정), 2군(데이터·인프라 콘텐츠)에 이어, 시니어 리뷰의 마지막 그룹인 3군(신규 랩 구조)을 다룬다. 세 항목 모두 "정책 문법을 아는 것"보다 "실전 운영 판단"에 가까운 공백이었다:

1. 타임아웃/재시도/백오프를 손으로 체험할 방법이 없음(모든 mock 요청이 즉시 응답)
2. 카나리·롤백 같은 컷오버 운영 판단 훈련이 없음
3. Jira "AI 작업 히스토리" 기록이 자유 텍스트라 부실한 답변도 걸러지지 않음

## 범위

3개 항목을 하나의 스펙·계획으로 묶어 진행한다.

**범위 제외**: 카나리 비율을 직접 조작하는 인터랙티브 슬라이더 UI(예측→정답 형식으로 대체), 실제 Azure 구독 연동.

## 설계

### A. 신규 탭 "백엔드 지연/타임아웃"

**위치**: "로컬 테스트 랩" 그룹에 새 nav 항목(`id: "lab-timeout"`, label "백엔드 지연/타임아웃") 추가, 이 그룹의 마지막 하위 항목(`lab-workbook` 뒤)에 배치. 새 패널 `panel-lab-timeout`.

**재사용 기반**: 1군에서 이미 만든 `billing-get` 라우트(`<retry count="3" interval="2">`+`<forward-request timeout="30">`+`<on-error><return-response>`)와 `runApimBackendSection`/`runApimOnErrorSection`/`runKongMock` 엔진을 그대로 재사용한다. 이 탭은 `billing-get` 라우트 전용(라우트 선택 UI 없음 — Kusto 실행기 탭처럼 단일 목적 랩).

**데이터 변경**: `billing-get` 라우트 객체에 `kongTimeoutMs: 3000` 필드를 추가한다(Kong의 `proxy_read_timeout` 기본값을 데모용으로 축소 표현한 것 — 실제 Kong 기본값은 60초이지만, ms 단위로 손쉽게 비교 체험하도록 3000ms로 명시하고 UI 설명에 "데모용 축소값"임을 명시한다).

**엔진 변경**:
- `runApimBackendSection(elements, input, trace)`에 `input.backendDelayMs`(양수)를 새로 받는다. `<retry>`의 자식 `<forward-request timeout="N">`의 `N`(초)을 ms로 환산해 `backendDelayMs`와 비교한다. 지연이 타임아웃을 초과하면 `forced = true`로 취급해 기존 재시도 로직을 그대로 태운다(트레이스 문구만 "백엔드 응답 지연 XXXms > 타임아웃 YYYYms — 타임아웃 발생"으로 분기). 기존 `input.forceBackendFailure` 경로는 그대로 유지(OR 조건 — 어느 쪽이든 실패로 취급).
- `runKongMock`에도 동일하게 `input.backendDelayMs`와 `route.kongTimeoutMs`를 비교하는 조건을 추가한다. 초과 시 즉시 503 실패(재시도 표시 없음 — 기존 "Kong은 재시도 시뮬레이션 안 함" 원칙 유지), 트레이스에 "백엔드 응답 지연 XXXms > 타임아웃 YYYYms(kongTimeoutMs) — 타임아웃 발생, 재시도 없음(Kong Service의 retries 필드가 별도 처리)" 명시.
- `input.forceBackendFailure`가 없는 이 새 탭에서는 항상 `backendDelayMs`만 사용하므로 기존 두 탭(API 테스트/APIM Policy)의 동작에는 영향이 없다(회귀 없음).

**UI**: "백엔드 지연(ms)" 숫자 입력 + "실행" 버튼. 기존 `renderLabResult`를 그대로 재사용해 Kong mock/APIM mock을 좌우로 비교 표시. 패널 설명에 "지연을 타임아웃(APIM 30초, Kong 3000ms 데모값)보다 크게/작게 넣어 재시도·타임아웃 동작 차이를 비교해보라"는 안내를 포함한다.

### B. 카나리·롤백 판단 시나리오

기존 `LAB_MIGRATION_PITFALLS` 배열(마이그레이션 탭, 예측→정답 형식)에 새 시나리오 4개를 추가한다. 코드 변경 없음 — 순수 데이터 추가, 기존 `renderLabMigration()` 함수가 자동으로 렌더링한다.

1. **표본 크기와 롤백 판단**: 카나리 5xx 비율이 2배 높지만 절대 건수가 적을 때 즉시 자동 롤백해야 하는가(정답: 아니요 — 최소 표본 크기·관찰 기간을 롤백 조건에 함께 명시해야 함)
2. **블루/그린 세션 어피니티**: 트래픽 100% 전환 직후 세션이 끊기는 게 APIM 버그인가(정답: 아니요 — 백엔드의 인메모리 세션 저장 방식 문제)
3. **수동 롤백의 구조적 취약점**: 알림 기반 수동 롤백에 담당자 응답 지연이 발생한 사례의 근본 원인(정답: 예 — Automation Runbook/Logic App 자동 트리거 구성 부재)
4. **API Revision과 트래픽 비율 분산의 오해**: APIM 리비전만으로 50:50 트래픽 분산이 되는가(정답: 아니요 — 리비전은 버전관리/스테이징용, 퍼센트 분산은 Front Door/Traffic Manager 가중치 라우팅 필요)

### C. Jira 템플릿 루브릭 보강

`panel-jira`에 코드 로직 변경 없이(기존 reveal 토글 패턴 재사용) 두 가지를 추가한다.

1. 오른쪽 카드(`AI 작업 히스토리 기록 절차`) 하단에 "최소 기준" 목록을 추가 — Dev Notes 각 필드가 충분히 구체적인지 판단하는 나쁜 예/좋은 예 대조(5개 필드 각각).
2. 왼쪽 카드(연습 시나리오) 하단에 "모범 예시 티켓 보기" 버튼 추가 — 클릭 시 기존 연습 시나리오(Kong rate-limiting → APIM rate-limit 이관)에 맞춰 완전히 작성된 예시 티켓 전문을 `.answer-reveal`로 펼쳐 보여준다.

## Global Constraints

- 새 CLI/정책 문법 없음(기존 1군 엔진 재사용) — 허구 문법 위험 없음.
- 회귀 없음: 기존 API 테스트/APIM Policy 탭은 `backendDelayMs`를 설정하지 않으므로 동작 변화 없어야 한다.
- 커밋 메시지는 한국어.

## 검증 방법

- Node 구문 검사(기존 패턴).
- Playwright: 새 탭에서 지연 300ms(타임아웃 이내) → 정상 응답, 지연 5000ms(타임아웃 초과) → APIM 재시도 3회+on-error, Kong 즉시 실패 확인. 기존 3개 라우트가 API 테스트/APIM Policy 탭에서 회귀 없는지 확인.
- 마이그레이션 탭에서 신규 시나리오 4개 렌더링·정답 토글 확인.
- Jira 탭에서 최소 기준 목록과 모범 예시 리빌 확인.

---

## Self-Review 체크리스트

- [x] 플레이스홀더 없음: A/B/C 모두 구체적 구현 방식과 실제 내용을 명시.
- [x] 내부 일관성: A는 1군의 `runApimBackendSection`/`runKongMock`/`billing-get`을 정확히 참조.
- [x] 범위 검사: 3개 항목이 서로 독립적이나 규모가 작아 하나의 계획으로 무리 없음.
- [x] 모호성 점검: 카나리 UI는 슬라이더가 아닌 기존 예측→정답 형식으로 확정(사용자 승인).
