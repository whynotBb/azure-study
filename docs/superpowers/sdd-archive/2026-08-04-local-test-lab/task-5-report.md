# Task 5: APIM Policy 탭 UI — 구현 완료 보고

## 요약
APIM Policy 탭의 자유편집 XML 실행 UI를 성공적으로 구현하고, 세 가지 검증 시나리오 모두 정상 동작을 확인했습니다.

## 변경 사항

**파일**: `azure-migration-tracker.html`
**라인 범위**: 1826–1871 (45줄 추가)
**커밋**: 1443519

### 추가된 코드
- `renderLabPolicy()` 함수: LAB_ROUTES 기반 탭 생성, XML 편집 UI, apikey/clientIp 입력 필드, 실행 버튼, 결과 렌더링
- `renderLabPolicy();` 호출: Task 4의 `renderLabApiTest();` 직후에 삽입
- 기존 `.answer-in` 클래스 재사용 (새 정의 없음)

## 검증 결과

### Scenario 1: 기본 orders-api XML + 올바른 apikey
✅ **PASS** (실제 브라우저 테스트)

**입력**:
- XML: 기본값 (check-header, rate-limit, cors 포함)
- apikey: `demo-key-123`
- clientIp: (입력 안 함)

**예상 결과**: `200` + 세 정책 trace
**실제 결과**:
```
상태: 200
check-header — 통과 — apikey 값 일치
rate-limit — 통과 — 남은 호출 4회
cors — CORS 헤더 추가 (Access-Control-Allow-Origin 등)
```

### Scenario 2: 화이트리스트 밖 정책 (`<quota>` 태그 추가)
✅ **PASS** (실제 브라우저 테스트)

**입력**:
- rate-limit 위에 `<quota calls="1000" renewal-period="3600" />` 추가
- apikey: `demo-key-123`

**예상 결과**: 에러 없이 실행 + trace에 "미지원" 배지

**실제 UI 결과** (브라우저 캡처):
```
실행 결과 — 200
{
  "orders": [
    {
      "id": 1,
      "total": 42000
    },
    {
      "id": 2,
      "total": 15000
    }
  ]
}

check-header — 통과 — apikey 값 일치
quota — ⚠ 이 학습 도구에서는 시뮬레이션 미지원 (구문은 유효)
rate-limit — 통과 — 남은 호출 4회
cors — CORS 헤더 추가 (Access-Control-Allow-Origin 등)
```

✅ **검증 완료**: quota 태그가 "미지원" 배지와 함께 정확하게 표시됨. 다른 정책들(check-header, rate-limit, cors)은 정상 동작하고 trace에 순서대로 나타남.

### Scenario 3: 잘못된 XML (닫는 태그 생략)
✅ **PASS** (실제 브라우저 테스트)

**입력**:
- `</check-header>` 닫는 태그 제거
- apikey: `demo-key-123`

**예상 결과**: `400` + 명확한 XML 구문 오류 메시지

**실제 UI 결과** (브라우저에서 직접 추출):
```html
<h3 class="lab-result-title">실행 결과 — <span class="lab-status-4xx">400</span></h3>
<div class="code-block">{
  "message": "정책 XML 구문 오류: This page contains the following errors:error on line 12 at column 13: Opening and ending tag mismatch: check-header line 4 and inbound\nBelow is a rendering of the page up to the first error."
}</div>
```

✅ **검증 완료**: 
- 상태 코드 400 표시됨 (lab-status-4xx 클래스로 빨간색)
- 에러 메시지가 명확하고 유용함 ("Opening and ending tag mismatch")
- 조용한 실패(silent failure)가 아니라 명확한 에러 응답을 받음

## 검증 방법

**모든 시나리오 실제 브라우저 테스트** (Playwright 브라우저 자동화 도구 사용):

1. **Scenario 1**: 파일 열기 → APIM Policy 탭 클릭 → 기본 XML 유지 → apikey 입력 → 실행 → 결과 UI 확인
2. **Scenario 2**: 기본 XML 편집 (quota 태그 추가) → apikey 입력 → 실행 → 결과 UI에서 "미지원" 배지 확인
3. **Scenario 3**: 기본 XML 편집 (닫는 태그 제거) → apikey 입력 → 실행 → 결과 UI에서 400 상태 및 구문 오류 메시지 확인

**테스트 도구**: gstack browse (Playwright 기반 자동화 도구, $B 별칭)

## 코드 품질 검토

✅ **기능 완성도**
- LAB_ROUTES 배열 순회 (Task 2 데이터 활용)
- 탭 버튼 활성화 상태 관리
- 텍스트영역 포커스 및 콘텐츠 로드 (loadXml 함수)
- runApimMock 호출 및 결과 렌더링 (Task 3 함수 재사용)

✅ **기존 코드와의 호환성**
- `.answer-in` CSS 클래스 재사용 (새 정의 없음)
- 기존 변수명/함수명 컨벤션 준수
- Task 4의 renderLabApiTest() 코드 패턴과 일치

✅ **에러 처리**
- 구문 오류: parseApimPolicyXml에서 감지 → 400 응답
- 알 수 없는 정책: 화이트리스트 체크 → trace에 "미지원" 표시

## 우려 사항
없음. 세 가지 검증 시나리오 모두 실제 브라우저에서 정상 동작을 확인했습니다.
- Scenario 1: ✓ 200 응답 + 3개 정책 trace
- Scenario 2: ✓ 200 응답 + quota에 "미지원" 배지 표시
- Scenario 3: ✓ 400 응답 + 명확한 XML 구문 오류 메시지

## 커밋 정보
```
commit 1443519
Author: Claude Code
Date:   <timestamp>

    로컬 테스트 랩 APIM Policy 탭 UI 추가
```

## 다음 단계
- Task 6: 로그 비교 탭 UI
- Task 7: 마이그레이션 탭 UI
- Task 8: 전체 회귀 확인
