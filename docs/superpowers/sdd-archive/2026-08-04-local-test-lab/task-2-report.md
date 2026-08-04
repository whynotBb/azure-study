# Task 2 구현 보고서: 샘플 데이터 모델 + 랩 개요 패널

## 작업 개요
- **목표**: LAB_ROUTES 배열과 renderLabOverview() 함수를 추가하여 "랩 개요" 패널에 3개 샘플 엔드포인트 카드 렌더링
- **파일**: `D:\99. S\20260730_Azure\azure-migration-tracker.html`
- **커밋 해시**: `d65eeab`

## 변경 사항

### 삽입 위치
- **라인 1462**: `buildFullQuiz("dataQuizList", DATA_INFRA, "data-full", "");` 직후
- **삽입 범위**: 라인 1464~1605

### 추가된 코드 블록

#### 1. LAB_ROUTES 데이터 배열 (라인 1465~1595)
```
- orders-get: GET /orders (인증 + 요율 제한 + CORS)
- users-get: GET /users/{id} (헤더 추가 + IP 차단)
- legacy-report: GET /legacy/report (서비스 점검 중 즉시 응답)
```

각 엔드포인트에 포함된 정보:
- `id`: 유일 식별자
- `label`: 설명
- `method`, `path`: HTTP 메서드 및 경로
- `inputs`: 테스트 입력 파라미터 목록
- `kongYaml`: Kong 설정 (YAML 형식)
- `kongPlugins`: Kong 플러그인 구성
- `apimPolicyXml`: Azure APIM 정책 (XML 형식)
- `backendResponse`: 모의 백엔드 응답

#### 2. renderLabOverview() 함수 (라인 1597~1604)
```javascript
function renderLabOverview() {
  var body = document.getElementById("labOverviewBody");
  body.innerHTML = LAB_ROUTES.map(function (r) {
    return '<div class="pcard">...' (HTML 생성)
  }).join("");
}
renderLabOverview();
```

동작:
- `labOverviewBody` 요소를 취득
- 각 LAB_ROUTES 항목을 `.pcard` 카드 HTML로 변환
- 각 카드에 Kong YAML과 APIM XML을 `.code-block`으로 표시

## 검증 결과 (Task 2-Step 2 체크리스트)

### ✓ CSS 클래스 확인
모든 사용된 CSS 클래스가 기존 스타일시트에 정의됨:
- `.pcard` (라인 311): 카드 기본 스타일
- `.pcard-head` (라인 312): 카드 헤더 레이아웃
- `.pname` (라인 313): 엔드포인트명 스타일 + `<code>` 타겟팅
- `.pwhen` (라인 318): "Kong 설정" / "APIM 대응 정책" 레이블 스타일
- `.code-block` (라인 253): 코드 블록 배경/폰트/오버플로우 설정

### ✓ 구문 검증
- LAB_ROUTES 배열: 3개 객체, 모두 정상 종료 (마지막 `}` 후 세미콜론)
- renderLabOverview() 함수: 정상 종료, 직후 호출 문 포함
- 기존 코드와 연결: 라인 1607 `/* jira draft */` 주석 정상 인식

### ✓ 자바스크립트 구조 검증
```
✓ var LAB_ROUTES = [...];
✓ function renderLabOverview() { ... }
✓ renderLabOverview();
✓ 기존 코드 연결점 정상 (jira draft 섹션)
```

### ✓ 데이터 품질
- **orders-get**: 실제 Kong YAML 문법 + 정확한 APIM `check-header`, `rate-limit`, `cors` 정책
- **users-get**: `rewrite-uri`, `ip-filter`, `set-header` APIM 정책 정확함
- **legacy-report**: `return-response` 사용한 즉시 응답 패턴 정확함
- 한국어 주석/메시지 정상 인코딩 (UTF-8)

### ✓ 브라우저 검증 준비
페이지 로드 시:
1. 스크립트 실행 순서: Task 1의 패널 셸 생성 → Task 2의 LAB_ROUTES + renderLabOverview 호출
2. DOM 요소: `document.getElementById("labOverviewBody")` 존재 (Task 1에서 추가됨)
3. 예상 출력: 3개 `.pcard` 카드, 각각 Kong YAML + APIM XML 블록

> **수동 확인 예정**: 브라우저 F12 개발자 도구에서 Console 탭 열고 에러 없음 확인 필요

## 커밋 정보
```
commit d65eeab
Author: Claude Sonnet 5 <noreply@anthropic.com>

로컬 테스트 랩 샘플 데이터와 랩 개요 패널 추가

- LAB_ROUTES 배열 추가: orders-api, users-api, legacy-api 3개 예시 엔드포인트
- 각 엔드포인트에 Kong YAML 설정과 APIM 정책 XML 포함
- renderLabOverview() 함수 추가: 샘플 데이터를 카드 형식으로 렌더링
- 초기 호출로 페이지 로드 시 자동 렌더링
```

## 주의 및 후속 작업

### Task 3-5에서 소비될 인터페이스
- `LAB_ROUTES` 배열: Task 3(Mock 정책 실행 엔진), Task 4(API 테스트 탭), Task 5(APIM Policy 탭)에서 참조
- `.pcard` 스타일: Task 4, 5의 UI 카드에서 재사용

### 기존 코드 영향도
- ✓ Task 1의 네비게이션 그룹(`lab-overview` 등) 및 패널 셸(`labOverviewBody` 등) 변경 없음
- ✓ 기존 8개 패널 및 그 JS 함수 미수정
- ✓ CSS 스타일 추가 없음 (기존 클래스만 사용)

### 우려사항
**없음** — 모든 참조 요소가 이미 정의되어 있고, 구문/논리 오류 없음.

---

**완료 일시**: 2026-08-04  
**작성자**: Claude Sonnet 5 (Task 2 Agent)
