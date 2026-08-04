# 로컬 테스트 랩 — 설계 문서

- 날짜: 2026-08-04
- 대상 파일: `azure-migration-tracker.html` (단일 HTML 학습 Artifact, 추가 방식)
- 참고: `sample.png` (동료가 만든 정적 대응표 페이지의 탭 구조 참고, 콘텐츠는 참고하지 않음)

## 배경 및 목적

기존 트래커는 Kong/Kibana/Cassandra/PostgreSQL → Azure 대응표와 능동회상 퀴즈(정책 XML, KQL, 데이터·인프라)를 다룬다. 그러나 "정책을 안다"와 "정책이 실제로 요청에 어떻게 작용하는지 본다"는 다른 학습이다. Azure 실 접속 권한이 없는 상태에서, 로컬에서 Kong mock과 Azure APIM mock을 각각 실행해 같은 요청에 대한 결과를 나란히 비교함으로써 정책 동작 이해를 강화하는 것이 이 기능의 목적이다.

## 범위

**새 네비게이션 그룹 "로컬 테스트 랩"**을 기존 `railNav`에 추가한다. 기존 8개 패널(구조 비교, APIM 정책 레퍼런스/퀴즈, KQL 레퍼런스/퀴즈, 데이터·인프라 레퍼런스/퀴즈, AIDD·Jira)과 그 데이터·함수는 전혀 수정하지 않는다 — 순수 추가(additive)만 한다.

새 패널 5개 (sample.png의 6탭 중 "전체 구조"/"Kong-Azure 비교"는 기존 정적 대응표와 중복되므로 "랩 개요" 1개로 통합):

1. **랩 개요** — 이 실습에서 쓸 샘플 Kong 서비스/라우트/플러그인 구성과 APIM 대응 리소스를 소개
2. **API 테스트** — 여러 정책이 조합된 엔드투엔드 요청에 대해 Kong mock ↔ APIM mock 응답을 좌우 비교
3. **APIM Policy** — 정책 XML을 직접 작성 → mock 엔진으로 즉시 실행 결과 확인 (레퍼런스/퀴즈 학습 검증용)
4. **로그 비교** — Kong/Nginx 로그 샘플 + 질문 → 사용자가 KQL 작성 → 정답과 비교 (기존 KQL 퀴즈 탭과 달리 실제 로그 라인을 입력 맥락으로 사용)
5. **마이그레이션** — 이관 시 놓치기 쉬운 함정 시나리오 카드 능동회상 퀴즈 (기존 AIDD·Jira 패널과 역할 분리: 여기는 기술적 함정, AIDD·Jira는 프로세스)

## 데이터 모델

### 샘플 Kong 구성 (탭 1~4 공유)

| 서비스 | 라우트 | Kong 플러그인 | 대응 APIM 정책 |
|---|---|---|---|
| `orders-api` | `GET/POST /orders` | `key-auth`(헤더 `apikey`), `rate-limiting`(5회/60초), `cors` | `check-header`, `rate-limit`, `cors` |
| `users-api` | `GET /users/{id}` | `request-transformer`(응답 헤더 추가, 경로 재작성), `ip-restriction` | `set-header`, `rewrite-uri`, `ip-filter` |
| `legacy-api` | `GET /legacy/report` | `request-termination`(즉시 차단) | `return-response` |

화이트리스트 정책 7종: `check-header`, `rate-limit`, `cors`, `set-header`, `rewrite-uri`, `ip-filter`, `return-response`. Kong 설정과 APIM 정책 XML은 실제 문법을 따른다.

### 로그 데이터셋 (탭 4)

- Kong/Nginx 쪽: 위 샘플 서비스 호출로 발생했을 access log 라인 10~15줄 (Kong `file-log`/`http-log` 실제 로그 스키마 기반)
- Azure 쪽: 동일 트래픽의 `ApiManagementGatewayLogs` 진단 로그 예시 — **정확한 컬럼명은 구현 단계에서 공식 문서로 재검증** (허구 스키마 방지, CLAUDE.md 규칙)
- 질문 3~4개(5xx 필터링, 느린 호출 top N, IP별 집계 등) + 정답 KQL + 정답 결과 테이블 사전 계산

### 마이그레이션 함정 카드 (탭 5, 6장 내외)

SSL 인증서 체인 순서, 헤더 대소문자 처리, rate-limit 윈도우 계산 방식 차이, 백엔드 타임아웃 기본값 차이, CORS preflight 처리, base path 라우팅 불일치. 각 카드는 "발생한다/안 한다" 예측 후 근거 공개. **기술적 근거는 구현 단계에서 공식 문서로 검증.**

## Mock 정책 실행 엔진

공통 결과 포맷 (Kong·APIM 엔진 모두 동일 셰이프 반환 → 비교 UI가 렌더링 함수 하나 재사용):

```js
{ statusCode, headers, body, trace: [{ policy: "rate-limiting", effect: "허용 (3/5회)" }, ...] }
```

**`runKongMock(request)`**
1. `path`+`method`로 샘플 라우트 매칭 (실패 시 404)
2. 플러그인을 고정 순서(인증 → IP 제한 → 트래픽 제어 → 변환 → 종료)로 순차 적용, 첫 차단에서 파이프라인 중단
   - `key-auth`: 헤더 불일치 시 401 + 중단
   - `ip-restriction`: 차단 목록 매칭 시 403 + 중단
   - `rate-limiting`: 세션 in-memory 카운터(`{routeId: [timestamp,...]}`)로 60초 슬라이딩 윈도우, 초과 시 429 + 중단
   - `request-transformer`: 헤더 추가/경로 재작성 실제 반영
   - `request-termination`: 조건 없이 지정 응답으로 즉시 종료
3. 끝까지 통과 시 더미 백엔드 200 응답

**`runApimMock(request, policyXml)`**
1. `DOMParser().parseFromString(policyXml, "text/xml")`로 실제 파싱 (파싱 에러 시 400 + 에러 위치 안내, 무시하지 않음)
2. `<inbound>` 자식을 문서 순서대로 순회, 화이트리스트 7종만 실행 로직 연결(카운터 등 계산 로직은 Kong 엔진과 공유하되 저장 키는 분리), 그 외 태그는 `trace`에 "⚠ 시뮬레이션 미지원" 배지만 남기고 통과
3. `<backend>`는 항상 더미 200으로 간주, `<outbound>`의 화이트리스트 정책까지 적용 후 최종 응답

**탭 간 재사용**: API 테스트 탭은 샘플 라우트의 고정 정책 조합을 그대로 실행(엔드투엔드), APIM Policy 탭은 사용자가 자유 편집한 XML을 같은 `runApimMock`에 전달(개별 정책 실습). 엔진 코드는 단일화.

## 에러 처리 / 엣지 케이스

- 라우트 미선택 시 실행 버튼 비활성화(에러 throw 없음)
- XML 파싱 에러: 400 + 명확한 위치 안내, 조용히 무시하지 않음
- 화이트리스트 밖 정책: 차단하지 않고 미지원 배지로 정직하게 노출(거짓 성공처럼 보이지 않게)
- rate-limit 경계값(5회째 허용/거부 기준)과 윈도우 자연 감소를 명확히 정의
- 복수 플러그인이 동시에 막을 조건이면 Kong 실행 순서상 첫 번째 사유만 표시
- localStorage 불가 환경: 기존 `storageBanner` 패턴 재사용(rate-limit 카운터는 in-memory라 무관, KQL 답안 저장 시에만 해당)
- KQL 채점은 자동 채점하지 않고 정답 공개 후 스스로 비교(능동회상 방식 유지)

## 테스트 방법

자동화 테스트 없음(정적 HTML+인라인 JS). 브라우저 수동 확인 시나리오:
1. `orders-api`에 apikey 없이 호출 → 401
2. 60초 내 6회 연속 호출 → 5회까지 허용, 6회째 429
3. `users-api` 응답 헤더에 `X-Backend` 실제 반영 확인
4. 화이트리스트 밖 정책(예: `<quota>`) 입력 시 미지원 배지 노출 확인
5. 닫히지 않은 태그 등 잘못된 XML 입력 시 파싱 에러 문구 확인
6. 기존 8개 패널 네비게이션·동작 회귀 확인

## 비고

이 디렉터리는 git 저장소가 아니므로(`git rev-parse` 확인 결과 미초기화) 이 설계 문서는 커밋 없이 파일로만 저장한다.
