# 프로젝트 배경

이 디렉터리는 **Kong + Cassandra/PostgreSQL + Kibana/Elasticsearch 기반 기존 시스템을 Azure API Management(APIM) + Azure Monitor로 전환**하는 프로젝트를 위한 개인 학습·정리 공간이다.

- 아직 Azure/Copilot/Jira 실 접속 권한이 없는 상태에서 준비 학습을 진행 중이다.
- 회사 방법론상 AIDD(AI-Driven Development, Copilot 활용) 개발 + Jira에 AI 작업 히스토리 기록이 요구된다.

## 참고 공식 문서

- Azure API Management: https://learn.microsoft.com/ko-kr/azure/api-management/
- Azure Monitor: https://learn.microsoft.com/ko-kr/azure/azure-monitor/

## 산출물 규칙

- 학습 도구는 단일 HTML Artifact로 제작한다(빌드 도구 없이 즉시 열람/공유 가능해야 함).
- APIM 정책 XML, KQL(Kusto) 예시 등 기술 콘텐츠는 반드시 실제 문법을 따라야 한다 — 허구의 문법 금지.
- "KQL"이라는 약어가 Kibana Query Language와 Azure의 Kusto Query Language 두 곳에서 쓰여 혼동되기 쉬우므로, 콘텐츠에서 명시적으로 구분한다.
- 새 학습 콘텐츠를 만들 때는 동료가 이미 만든 정적 대응표 페이지와 겹치지 않도록, 대응표 자체보다 "직접 작성해보고 정답과 비교"하는 능동회상 방식을 우선한다.
- 대시보드/체크리스트류 진행률 UI는 "동료 대비 나의 갭"으로 프레이밍하지 않는다. 동료의 주간보고는 프로젝트 진행 로드맵의 출처일 뿐이며, 진행률은 프로젝트 자체의 현재 상태를 보여주는 용도로만 표시한다.
- 콘텐츠 범위는 배경에 명시된 전환 대상 전체("Kong + Cassandra/PostgreSQL + Kibana/Elasticsearch")를 다뤄야 한다 — Kong/Kibana 쪽에만 치우치지 않도록, Cassandra/PostgreSQL 데이터 계층과 Application Gateway·SSL 인증서/방화벽·Azure 기초 개념(구독/리소스 그룹/RBAC/관리 ID)도 레퍼런스·퀴즈로 다룬다.
- 레퍼런스 페이지에 실린 항목은 전부 퀴즈로도 자동 변환한다("리스트업된 내용은 모두 퀴즈에도 추가"). 새 레퍼런스 데이터셋을 추가할 때는 `renderRefPage`/`buildFullQuiz` 공용 함수를 그대로 재사용하고, 손수 다시 작성하지 않는다.

## 커뮤니케이션

- 응답/주석/문서/커밋 메시지는 한국어로 작성한다(사용자 전역 CLAUDE.md 규칙과 동일).
