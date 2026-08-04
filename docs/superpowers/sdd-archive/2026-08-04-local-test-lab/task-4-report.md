# Task 4: API 테스트 탭 UI - 구현 보고서

## 구현 내용

### 변경된 범위
- **파일**: `azure-migration-tracker.html`
- **라인**: 1770-1825 (54줄 추가)
- **위치**: Task 3의 `renderLabResult()` 함수 정의 직후에 `renderLabApiTest()` 함수 추가

### 구현된 기능

1. **라우트 탭 UI** (라인 1772-1774)
   - LAB_ROUTES 배열의 각 라우트마다 탭 버튼 생성
   - 첫 번째 탭(`orders-get`)이 기본 활성화
   - 각 탭은 `data-route-idx` 속성으로 라우트 인덱스 저장

2. **동적 입력 필드** (라인 1785-1800)
   - Route.inputs 배열에 따라 입력 필드 동적 생성
   - "apikey" 입력 필드 (라우트 0, 1에서 필요)
   - "clientIp" 입력 필드 (라우트 1, 2에서 필요)
   - 입력 불필요한 라우트는 안내 메시지 표시

3. **입력값 수집** (라인 1802-1806)
   - currentInput() 함수가 현재 입력 필드값을 수집
   - 입력 필드가 없으면 빈 문자열 반환

4. **탭 전환 및 실행 핸들링** (라인 1808-1821)
   - 탭 클릭 시 selectedIdx 업데이트 및 UI 재렌더링
   - "실행" 버튼 클릭 시:
     - LAB_ROUTES[selectedIdx]의 라우트 정보 조회
     - currentInput()으로 사용자 입력값 수집
     - runKongMock(route, input) 호출로 Kong 시뮬레이션 실행
     - runApimMock(route, route.apimPolicyXml, input) 호출로 APIM 시뮬레이션 실행
     - 각 결과를 renderLabResult()로 화면에 표시

## 커밋 정보
- **커밋 해시**: `5179d28`
- **커밋 메시지**: "로컬 테스트 랩 API 테스트 탭 UI 추가"
- **파일 변경**: 57줄 추가

## 수동 검증 시나리오 - 실제 브라우저 테스트 결과

### Step 2: 키 인증 실패 (401 응답) ✓ PASSED
**실행 결과**:
- apikey 입력란을 비운 상태 → "실행" 클릭
- **Kong mock 응답**:
  - 상태코드: 401
  - 응답: `{"message": "Unauthorized"}`
  - trace: `key-auth — 차단 — apikey 헤더 없음/불일치 (401)`
- **APIM mock 응답**:
  - 상태코드: 401
  - 응답: `{"message": "Unauthorized"}`
  - trace: `check-header — 차단 — apikey 값 불일치 (401)`

### Step 3: 레이트 리미팅 (429 응답) ✓ PASSED
**실행 결과**:
- apikey: "demo-key-123" 입력 후 6번 연속 클릭

| Click | Kong 상태코드 | Kong Trace | APIM 상태코드 | APIM Trace |
|-------|-------------|-----------|-------------|----------|
| 1 | 200 | rate-limiting — 통과 — 남은 호출 4회 | 200 | rate-limit — 통과 |
| 2 | 200 | rate-limiting — 통과 — 남은 호출 3회 | 200 | rate-limit — 통과 |
| 3 | 200 | rate-limiting — 통과 — 남은 호출 2회 | 200 | rate-limit — 통과 |
| 4 | 200 | rate-limiting — 통과 — 남은 호출 1회 | 200 | rate-limit — 통과 |
| 5 | 200 | rate-limiting — 통과 — 남은 호출 0회 | 200 | rate-limit — 통과 |
| 6 | 429 | rate-limiting — 차단 — 5회/60초 초과 (429) | 429 | rate-limit — 차단 — 5회/60초 초과 (429) |

### Step 4: IP 차단 및 헤더 반영 ✓ PASSED

#### 4-1) IP 10.0.0.99 (차단) ✓
- 라우트: GET /orders → GET /users/{id} (전환 성공, 입력 필드: apikey → 요청자 IP)
- IP: "10.0.0.99" 입력 후 실행
- **Kong mock 응답**:
  - 상태코드: 403
  - 응답: `{"message": "Forbidden"}`
  - trace: `ip-restriction — 차단 — 10.0.0.99는 차단 목록 (403)`
- **APIM mock 응답**:
  - 상태코드: 403
  - 응답: `{"message": "Forbidden"}`
  - trace: `ip-filter — 차단 — 10.0.0.99 (action=forbid) (403)`

#### 4-2) IP 1.2.3.4 (허용 + 헤더) ✓
- IP: "1.2.3.4"로 변경 후 실행
- **Kong mock 응답**:
  - 상태코드: 200
  - 응답 헤더: `X-Backend: users-svc-v2` (확인됨)
  - trace: 
    - `ip-restriction — 통과 — 차단 목록에 없음`
    - `response-transformer — 응답 헤더 추가: X-Backend=users-svc-v2`
- **APIM mock 응답**:
  - 상태코드: 200
  - 응답 헤더: `X-Backend: users-svc-v2` (확인됨)
  - trace:
    - `ip-filter — 통과 — action=forbid`
    - `set-header — X-Backend = users-svc-v2 (outbound)`

## 구현 검증 결과

### 코드 검증 ✓
- 코드가 task-4-brief.md의 명세와 정확히 일치
- renderLabApiTest() 함수가 정확한 위치(renderLabResult 직후)에 삽입됨
- renderLabApiTest() 호출이 함수 정의 직후에 즉시 실행됨
- Task 3의 runKongMock(), runApimMock(), renderLabResult() 함수를 올바르게 호출

### 코드 구조 검증 ✓
- LAB_ROUTES 배열 참조 정상
- 라우트 인덱스 기반 탭 선택 로직 정상
- 입력 필드 동적 생성 로직 정상
- 이벤트 핸들러 바인딩 정상
- 결과 표시를 위한 renderLabResult() 호출 정상

### 호출 흐름 검증 ✓
- 탭 선택 → renderInputs() → 입력 필드 표시
- 실행 버튼 클릭 → currentInput() → mock 함수 호출 → 결과 표시

## 주의사항 및 설계 결정

1. **HTML 이스케이프 미처리** (Task 3 코드 리뷰에서 지적, 수정 불요)
   - trace/header 문자열이 innerHTML을 통해 렌더링되므로 원본 HTML이 그대로 표시됨
   - 이것은 로컬 단일 파일 도구이고 서버/멀티유저 환경이 아니므로 허용된 저위험 트레이드오프
   - brief에서 요구하지 않으므로 스코프 크리프 방지를 위해 수정하지 않음

2. **Rate limit 상태 관리**
   - Kong/APIM mock 함수에서 rate limit 상태를 내부적으로 관리
   - 호출할 때마다 mock 함수가 상태를 추적하여 6번째 요청에 429 응답

3. **입력값 재사용**
   - 라우트 전환 시에도 입력값이 유지되어 사용자가 다시 입력하지 않아도 됨
   - 실제 실행 시점에만 입력값을 수집하므로 중간에 변경 가능

## 확인 사항

- [x] 코드 구현 완료
- [x] 코드 위치 검증 완료 (라인 1770-1825)
- [x] Brief 명세 준수 확인
- [x] Git 커밋 완료 (5179d28)
- [x] **브라우저 자동화 테스트 완료** (gstack browse 사용)

## 테스트 환경 및 방법

- **도구**: gstack browse (Chromium 헤드리스 자동화, 대약 100ms/command)
- **테스트 시간**: 2026-08-04
- **테스트 모드**: 전체 자동화 (스냅샷, 클릭, 텍스트 추출)
- **검증 항목**: Step 2, 3, 4 모든 시나리오 통과 (3/3 ✓)

## 테스트 요약

| 시나리오 | 항목 | 결과 | 비고 |
|---------|------|------|------|
| Step 2 | Missing apikey (401) | ✓ PASS | Kong·APIM 모두 정확한 상태코드 및 trace |
| Step 3 | Rate limit (429) | ✓ PASS | 1-5회 통과, 6회 차단 완벽 동작 |
| Step 4a | IP 차단 (403) | ✓ PASS | 10.0.0.99 양쪽 모두 403 반환 |
| Step 4b | IP 허용 + 헤더 (200) | ✓ PASS | 1.2.3.4 통과, X-Backend 헤더 양쪽 표시 |

**최종 상태**: ✓ 모든 검증 완료, 품질 검증됨
