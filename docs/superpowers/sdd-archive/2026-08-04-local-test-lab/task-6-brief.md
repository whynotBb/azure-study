## Task 6: 로그 데이터셋 + "로그 비교" 탭 UI

**Files:**
- Modify: `azure-migration-tracker.html` (JS, Task 5의 `renderLabPolicy();` 호출 다음 줄에 추가)

**Interfaces:**
- Produces: `LAB_LOG_QUESTIONS` — Task 6 내부에서만 쓰임. 기존 `quiz-card`/`answer-in`/`answer-reveal` CSS 클래스를 재사용(새 CSS 불필요).

- [ ] **Step 1: 로그 샘플 + 질문 데이터**

```js
  /* ---------------- 로컬 테스트 랩: 로그 비교 탭 ---------------- */
  var LAB_KONG_LOG_SAMPLE =
    '{"client_ip":"203.0.113.10","method":"GET","request_uri":"/orders","status":200,"latencies":{"request":42},"started_at":1735600000}\n' +
    '{"client_ip":"203.0.113.11","method":"GET","request_uri":"/orders","status":429,"latencies":{"request":3},"started_at":1735600005}\n' +
    '{"client_ip":"203.0.113.10","method":"GET","request_uri":"/users/1","status":403,"latencies":{"request":5},"started_at":1735600010}\n' +
    '{"client_ip":"198.51.100.20","method":"GET","request_uri":"/legacy/report","status":503,"latencies":{"request":1},"started_at":1735600020}\n' +
    '{"client_ip":"203.0.113.10","method":"GET","request_uri":"/orders","status":200,"latencies":{"request":812},"started_at":1735600030}';

  var LAB_LOG_QUESTIONS = [
    {
      q: "Azure Monitor의 ApiManagementGatewayLogs 테이블에서, 최근 1일 동안 5xx 응답만 필터링하려면?",
      answerKql:
        "ApiManagementGatewayLogs\n" +
        "| where TimeGenerated > ago(1d)\n" +
        "| where ResponseCode >= 500\n" +
        "| project TimeGenerated, ApiId, Method, Url, ResponseCode, BackendResponseCode, CallerIpAddress",
      answerResult: "위 샘플 로그 기준: /legacy/report (503) 1건이 해당됩니다."
    },
    {
      q: "가장 느린 호출 top 3를 총 소요시간(TotalTime) 기준으로 뽑으려면?",
      answerKql:
        "ApiManagementGatewayLogs\n" +
        "| top 3 by TotalTime desc\n" +
        "| project TimeGenerated, ApiId, Url, TotalTime, BackendTime",
      answerResult: "위 샘플 로그 기준: /orders (812ms) 요청이 가장 느린 호출로 1위에 나와야 합니다."
    },
    {
      q: "호출자 IP(CallerIpAddress)별 요청 수를 집계해 많은 순으로 정렬하려면?",
      answerKql:
        "ApiManagementGatewayLogs\n" +
        "| summarize RequestCount = count() by CallerIpAddress\n" +
        "| sort by RequestCount desc",
      answerResult: "위 샘플 로그 기준: 203.0.113.10이 3건으로 1위입니다."
    }
  ];

  function renderLabLogs() {
    var body = document.getElementById("labLogsBody");
    var html = '<h2 class="section-title">Kong/Nginx 원본 로그 샘플</h2><div class="code-block">' + LAB_KONG_LOG_SAMPLE + "</div>";
    html += '<h2 class="section-title" style="margin-top:var(--sp-6);">질문</h2>';
    LAB_LOG_QUESTIONS.forEach(function (item, idx) {
      html += '<div class="quiz-card"><div class="quiz-head"><h3>질문 ' + (idx + 1) + '</h3></div><div class="quiz-body">' +
        '<p class="quiz-scenario">' + item.q + '</p>' +
        '<textarea class="answer-in" data-log-q="' + idx + '" placeholder="KQL을 직접 작성해 보세요"></textarea>' +
        '<div class="quiz-actions"><button class="btn primary" type="button" data-log-reveal="' + idx + '">정답 보기</button></div>' +
        '<div class="answer-reveal" id="labLogReveal' + idx + '"><span class="tag">정답 예시</span><div class="code-block">' + item.answerKql + '</div><p class="pwhen" style="margin-top:var(--sp-2);">' + item.answerResult + '</p></div>' +
        "</div></div>";
    });
    body.innerHTML = html;

    LAB_LOG_QUESTIONS.forEach(function (item, idx) {
      var ta = body.querySelector('[data-log-q="' + idx + '"]');
      var uid = "lab-log-" + idx;
      ta.value = state.quiz[uid] || "";
      ta.addEventListener("input", function () { state.quiz[uid] = ta.value; persist(); });
      body.querySelector('[data-log-reveal="' + idx + '"]').addEventListener("click", function () {
        document.getElementById("labLogReveal" + idx).classList.add("show");
      });
    });
  }
  renderLabLogs();
```

- [ ] **Step 2: 브라우저에서 확인**

"로그 비교" 탭을 연다. 원본 로그 샘플(5줄 JSON)이 code-block으로 보이고, 그 아래 질문 3개가 기존 퀴즈 카드와 동일한 스타일로 보인다. 텍스트 영역에 아무 텍스트나 입력 후 새로고침해 값이 유지되는지 확인(저장 가능 환경 기준), "정답 보기"를 누르면 정답 KQL과 해설이 펼쳐지는지 확인한다.

---

