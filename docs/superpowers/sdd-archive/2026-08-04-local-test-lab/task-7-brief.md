## Task 7: 마이그레이션 함정 카드 + "마이그레이션" 탭 UI

**Files:**
- Modify: `azure-migration-tracker.html` (JS, Task 6의 `renderLabLogs();` 호출 다음 줄에 추가)

**Interfaces:**
- Produces: `LAB_MIGRATION_PITFALLS` — Task 7 내부에서만 쓰임.

- [ ] **Step 1: 함정 시나리오 데이터 + 렌더**

```js
  /* ---------------- 로컬 테스트 랩: 마이그레이션 탭 ---------------- */
  var LAB_MIGRATION_PITFALLS = [
    {
      scenario: "Kong에서 쓰던 중간 인증서(intermediate certificate)를 그대로 Key Vault에 인증서로 등록하지 않고 리프 인증서만 올렸다. APIM 사용자 지정 도메인 연결이 실패할까?",
      answer: "예, 실패합니다.",
      reason: "Key Vault에는 리프 인증서뿐 아니라 전체 인증서 체인(중간 인증서 포함)이 포함된 .pfx를 등록해야 합니다. 체인이 빠지면 클라이언트가 인증서 신뢰 경로를 완성하지 못해 TLS 핸드셰이크가 실패할 수 있습니다."
    },
    {
      scenario: "Kong 플러그인에서 요청 헤더 이름을 `X-Request-ID`로 검사하도록 만들었는데, 클라이언트가 `x-request-id`(소문자)로 보냈다. Kong에서 정상 인식됐다면 APIM의 `check-header`에서도 그대로 정상 인식될까?",
      answer: "설정에 따라 다릅니다 — 기본값이 아닙니다.",
      reason: "HTTP 헤더 이름은 원래 대소문자 구분이 없지만, APIM의 check-header 정책은 `ignore-case` 속성이 필수이며 명시적으로 `true`로 설정하지 않으면 대소문자를 구분해 비교합니다. Kong 쪽 동작만 보고 그대로 옮기면 놓치기 쉬운 지점입니다."
    },
    {
      scenario: "Kong의 rate-limiting을 '분당 60회'로 쓰다가 APIM `rate-limit` 정책에 `calls=60, renewal-period=60`으로 그대로 옮겼다. 두 시스템의 카운팅 창(윈도우) 계산 방식이 완전히 동일할까?",
      answer: "아니요, 동일하지 않을 수 있습니다.",
      reason: "APIM classic 티어의 rate-limit은 슬라이딩 윈도우 방식이지만 분산 아키텍처 특성상 완전히 정확하지 않을 수 있다고 공식 문서에 명시되어 있고, v2 티어는 토큰 버킷 알고리즘을 사용해 classic과 계산 방식 자체가 다릅니다. Kong의 카운팅 정책(local/cluster/redis)과도 정확히 1:1로 일치한다고 가정하면 안 됩니다."
    },
    {
      scenario: "Application Gateway 뒤에 내부 APIM을 두었는데, App Gateway의 '호스트 이름 재정의'를 기본값(재정의함)으로 그대로 두고 이관했다. APIM이 요청을 거부할까?",
      answer: "예, 거부할 가능성이 높습니다.",
      reason: "App Gateway가 호스트 헤더를 재작성하면 APIM은 클라이언트가 원래 요청한 사용자 지정 도메인과 헤더가 다르다고 판단해 요청을 거부할 수 있습니다. '호스트 이름 재정의'를 '아니요'로 설정해 원래 호스트 헤더를 유지해야 합니다."
    },
    {
      scenario: "Kong의 CORS 플러그인은 프리플라이트(OPTIONS) 요청을 항상 200으로 응답하도록 커스텀했는데, APIM `cors` 정책을 기본값(`terminate-unmatched-request` 미설정)으로 이관했다. Origin이 허용 목록에 없는 프리플라이트 요청에서 Kong과 동일하게 동작할까?",
      answer: "예, 동일하게 동작합니다(우연히).",
      reason: "`terminate-unmatched-request`의 기본값이 false이므로, Origin이 매칭되지 않아도 요청은 계속 진행되어 결과적으로 200이 반환됩니다. 다만 이는 '의도한 동작'이 아니라 기본값이 우연히 비슷하게 보이는 것이라, 다른 시나리오(GET/HEAD 단순 요청)에서는 CORS 헤더 자체가 붙지 않는 차이가 생길 수 있습니다."
    },
    {
      scenario: "Kong 라우트가 `strip_path: true`로 `/api/v1` 접두사를 벗겨내고 백엔드로 전달했는데, APIM에서는 API의 '기본 경로(base path)' 설정을 그대로 유지한 채 백엔드 URL만 바꿔치기했다. 요청 경로가 깨질까?",
      answer: "예, 깨질 수 있습니다.",
      reason: "APIM은 API의 base path와 백엔드 URL의 경로를 조합해 최종 백엔드 요청 경로를 구성합니다. Kong의 strip_path 동작(접두사 제거)과 APIM의 경로 조합 규칙이 다르므로, `rewrite-uri` 정책으로 명시적으로 맞춰주지 않으면 백엔드가 기대하지 않는 경로로 요청이 전달될 수 있습니다."
    }
  ];

  function renderLabMigration() {
    var body = document.getElementById("labMigrationBody");
    body.innerHTML = LAB_MIGRATION_PITFALLS.map(function (item, idx) {
      return '<div class="quiz-card"><div class="quiz-head"><h3>시나리오 ' + (idx + 1) + '</h3></div><div class="quiz-body">' +
        '<p class="quiz-scenario">' + item.scenario + '</p>' +
        '<div class="quiz-actions">' +
        '<button class="btn" type="button" data-mig-guess="' + idx + '" data-guess="true">발생한다</button>' +
        '<button class="btn" type="button" data-mig-guess="' + idx + '" data-guess="false">발생하지 않는다</button>' +
        "</div>" +
        '<div class="answer-reveal" id="labMigReveal' + idx + '"><span class="tag">정답</span><p class="pwhen" style="margin-top:var(--sp-2);"><strong>' + item.answer + '</strong></p><p class="quiz-scenario">' + item.reason + "</p></div>" +
        "</div></div>";
    }).join("");

    LAB_MIGRATION_PITFALLS.forEach(function (item, idx) {
      body.querySelectorAll('[data-mig-guess="' + idx + '"]').forEach(function (btn) {
        btn.addEventListener("click", function () {
          document.getElementById("labMigReveal" + idx).classList.add("show");
        });
      });
    });
  }
  renderLabMigration();
```

- [ ] **Step 2: 브라우저에서 확인**

"마이그레이션" 탭을 연다. 6개 시나리오 카드가 보이고, "발생한다"/"발생하지 않는다" 버튼 중 아무거나 누르면 정답과 근거가 펼쳐지는지 확인한다.

---

