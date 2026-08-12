# 데이터·인프라 레퍼런스 콘텐츠 보강 (2군) 설계

## 배경

Azure 시니어 실무자 리뷰에서, `index.html`의 세 학습 축(Kong/APIM, Kibana/Azure Monitor, 데이터·인프라) 중 **데이터·인프라 축이 확연히 얇다**는 지적을 받았다.

| 축 | 카테고리 | 항목 수 |
|---|---|---|
| Kong → APIM | 11 | 67 |
| Kibana → Azure Monitor | 13 | 53 |
| **데이터·인프라(현재)** | **5** | **23** |

프로젝트 CLAUDE.md는 세 전환 대상(Kong/Kibana뿐 아니라 Cassandra/PostgreSQL 데이터 계층, Application Gateway·SSL·방화벽, Azure 기초 개념)을 균형 있게 다룰 것을 명시하고 있어, 이 격차를 좁히는 것이 이번 작업의 목표다.

## 범위

`DATA_INFRA` 배열(`index.html`)의 기존 5개 카테고리 구조는 그대로 유지하고, 시니어 리뷰에서 구체적으로 지적된 공백을 메우는 **신규 항목 27개**를 추가한다(23개 → 50개, 다른 두 축과 비슷한 비중).

**범위 제외**: 새 카테고리 신설, 기존 항목 수정(단, 오탈자/사실 오류 발견 시 별도로 보고), 3군(신규 랩 구조) 작업.

## 접근 방식

**초안 작성 → 타겟 검증(hybrid)** 방식을 사용한다:
1. 각 항목을 실제 Azure 지식으로 직접 작성한다(레퍼런스 항목 스키마: `name`/`code`/`url`/`easy`/`when`/`example` — 기존 항목과 동일한 형식).
2. CLI 명령어·설정값·플래그 등 "허구 문법 금지" 규칙에 걸리는 부분은 WebFetch로 공식 Microsoft Learn 문서와 대조 검증한다(이전 세션에서 `--zonal-resiliency`, `--enable-managed-identity`를 검증했던 방식과 동일).
3. `url` 필드는 실제 존재하는 Microsoft Learn 문서 경로를 사용한다(추측 금지 — 확인 안 되면 카테고리 베이스 URL 패턴을 따르거나, 검증 후 정확한 경로로 교체).

## 구현 방식 — 코드 변경 없음, 순수 데이터 추가

`DATA_INFRA` 배열은 이미 `renderRefPage("dataToc", "dataRefBody", DATA_INFRA, "")`(레퍼런스 페이지)와 `buildFullQuiz("dataQuizList", DATA_INFRA, "data-full", "")`(퀴즈)에서 공용 함수로 재사용되고 있다. 새 항목을 각 카테고리의 `items` 배열에 추가하기만 하면 레퍼런스 페이지와 퀴즈에 **자동으로 동시 반영**된다 — 프로젝트 CLAUDE.md의 "renderRefPage/buildFullQuiz 공용 함수를 그대로 재사용" 규칙을 그대로 따른다. 별도의 JS 로직/CSS 변경은 필요 없다.

## 신규 항목 목록 (카테고리별, 총 27개)

기존 항목과 겹치지 않도록 현재 `DATA_INFRA`의 정확한 항목명을 대조 확인했다(예: "일관성 수준 매핑", "지원되지 않는 기능 확인"은 이미 존재해 제외).

### Cassandra → Azure Cosmos DB for Apache Cassandra (7 → 12개, +5)
기존: 키스페이스/테이블 매핑, cqlsh로 연결하기, 처리량(RU) 모델, 일관성 수준 매핑, 권한 관리 방식 변화, 지원되지 않는 기능 확인

신규:
1. **TTL 지원 범위** — Cosmos DB Cassandra API의 TTL 지원 수준과 컬럼 단위 TTL 차이
2. **컬렉션 타입 제약** — list/set/map 지원 여부와 실무 제약사항
3. **경량 트랜잭션(LWT) 미지원** — `IF NOT EXISTS`/compare-and-set류 미지원, 애플리케이션 레벨 대안
4. **파티션 키 재설계** — 토큰 링 기반 파티셔닝과 Cosmos DB 논리적 파티션 키의 차이, 핫 파티션 회피
5. **드라이버 호환성** — DataStax 드라이버 버전 요구사항, CQL 프로토콜 버전 요구사항

### PostgreSQL → Azure Database for PostgreSQL (6 → 11개, +5)
기존: 배포 옵션(Flexible Server), 연결 문자열/호스트 이름, 네트워크 접근 방식, 확장 활성화, 고가용성(HA), NSG 트래픽 허용

신규:
1. **내장 커넥션 풀링** — PgBouncer 내장 기능, 포트 6432, transaction/session 모드
2. **읽기 복제본(Read Replica)** — 비동기 복제, 읽기 확장 전용, 승격(promote) 관련 제약
3. **자동 유지관리 및 마이너 버전 업그레이드** — 유지관리 창(maintenance window) 설정, 다운타임 특성
4. **백업 및 시점 복원(PITR)** — 자동 백업 보존 기간, 지역 중복 백업 옵션
5. **서버 매개변수 커스터마이징** — 파라미터 그룹 개념, 재시작 필요 여부에 따른 파라미터 분류

### Application Gateway vs API Management (3 → 8개, +5)
기존: 역할 차이, 함께 쓰는 아키텍처, 호스트 헤더 처리 주의점 (모두 "APIM과의 관계"에 집중)

신규(App Gateway 자체 운영 지식 보강):
1. **WAF_v2 티어와 OWASP 규칙 세트** — Detection/Prevention 모드, 커스텀 규칙
2. **경로 기반 라우팅** — URL 경로 맵으로 여러 백엔드 풀에 분기
3. **상태 프로브(Health Probe)와 백엔드 풀** — 비정상 백엔드 자동 제외 동작
4. **오토스케일링** — v2 SKU의 인스턴스 수 자동 조정(최소/최대 설정)
5. **TLS 정책** — 최소 TLS 버전, 사전 정의된 암호화 스위트 정책

### SSL 인증서 / 방화벽 기초 (4 → 10개, +6)
기존: Key Vault 인증서 연동, 관리 ID로 Key Vault 접근 권한 부여, 사용자 지정 도메인 DNS 설정, NSG(네트워크 보안 그룹) 기초

신규:
1. **NSG vs Azure Firewall vs WAF 계층 비교** — 역할이 다른 세 방화벽 개념을 표로 대조(가장 중요한 공백)
2. **Azure Firewall 기본 개념** — 중앙집중식 스테이트풀 방화벽, 위협 인텔리전스 기반 필터링
3. **App Service 관리 인증서(무료)** — 커스텀 도메인 무료 인증서, 자동 갱신과 제약사항
4. **인증서 회전 자동화 검증** — Key Vault 인증서 만료 임박 시 자동 갱신 동작 확인 방법
5. **프라이빗 엔드포인트(Private Endpoint)** — 공용 인터넷 우회, NSG/방화벽 규칙과의 관계
6. **DDoS 보호 기본/표준** — 기본 제공(Basic) vs 표준(Standard) 티어 차이

### Azure 기초 개념 (4 → 10개, +6)
기존: 구독(Subscription), 리소스 그룹(Resource Group), RBAC(역할 기반 액세스 제어), 관리 ID(Managed Identity)

신규:
1. **관리 그룹(Management Group)** — 구독 여러 개를 묶는 상위 계층, 정책 상속 구조
2. **RBAC 스코프 4단계** — 관리 그룹/구독/리소스 그룹/리소스 단위 스코프와 상속 방향
3. **시스템 할당 vs 사용자 할당 관리 ID** — 수명 주기 차이, 각각 언제 쓰는지
4. **Azure Policy 기초** — 규정 준수 강제 메커니즘, RBAC와의 차이
5. **내장 역할 vs 사용자 지정 역할** — Contributor/Reader 등 기본 제공 역할, 커스텀 역할 정의
6. **태그(Tags)** — 리소스 분류·비용 추적용 메타데이터

## Global Constraints

- 각 신규 항목은 기존 스키마(`name`/`code`/`url`/`easy`/`when`/`example`)를 정확히 따른다.
- `example` 필드에 CLI 명령/설정 문법이 포함되면 실제 문법이어야 한다(허구 금지) — WebFetch로 검증 후 작성.
- `url` 필드는 실제 존재하는 Microsoft Learn 문서 URL이어야 한다.
- 응답/커밋 메시지는 한국어.
- 기존 5개 카테고리 구조와 기존 23개 항목은 변경하지 않는다(순수 추가만).

## 검증 방법

- Node로 두 `<script>` 블록 구문 검사(기존 세션 패턴 재사용).
- Playwright로 데이터·인프라 레퍼런스 패널(`panel-data-ref`)과 퀴즈 패널(`panel-data-quiz`)에서 신규 항목 27개가 실제로 렌더링되는지 확인(항목 수, 카테고리별 개수).
- 신규 CLI 예시 중 검증이 필요한 것들의 WebFetch 검증 결과를 계획 문서에 근거로 남긴다.

---

## Self-Review 체크리스트

- [x] 플레이스홀더 없음: 27개 항목 모두 구체적 주제와 핵심 설명 포인트를 명시함.
- [x] 내부 일관성: 기존 항목과의 중복 여부를 실제 파일 대조로 확인함.
- [x] 범위 검사: 카테고리 신설이나 코드 로직 변경 없이 순수 데이터 추가로 국한 — 단일 계획으로 구현 가능한 크기.
- [x] 모호성 점검: 소싱/검증 방식(초안 후 타겟 검증)을 사용자 승인으로 확정함.
