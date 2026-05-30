---
name: morning-briefing
description: 연결된 업무 소스(Notion·Google Drive·GitHub 등)와 네이버 트렌드를 읽어 오늘 할 일을 우선순위(P1/P2/P3)로 정리하고 Notion 페이지에 기록한다. "오늘 할 일", "아침 브리핑", "우선순위 정리", "오늘 뭐부터", "daily briefing", "what should I work on today" 같은 요청에 사용.
license: MIT
---

# Morning Briefing (아침 업무 브리핑)

연결된 업무 소스를 한 번에 훑어서 **오늘 할 일과 우선순위**를 정리하고, 결과를 Notion 페이지로 남기는 에이전트.

## 솔직한 전제 (먼저 읽기)

- Claude는 **카카오톡 등 임의 모바일 앱에 직접 접근할 수 없다.** 이 환경에서 **MCP로 연결된 소스 + 네이버 오픈 API**만 읽는다.
- 매번 **실제로 연결된 것만** 사용하고, 빠진 소스는 결과에 `(연결 안 됨)`으로 표시한다. 없는 데이터를 지어내지 않는다.
- 출처/출력 위치가 불명확하면 **멈추고 사용자에게 한 번 묻는다** (추측 금지).
- 모든 수집 단계는 **읽기 전용**이다. 쓰기는 마지막 Notion 페이지 생성뿐.

## 워크플로

### 1. 소스 탐색 (read-only)
연결 가능한 소스를 확인한다. 보통 다음이 후보다 — Notion(연동 소스로 Slack·Jira·Linear·MS Teams·Google Drive·SharePoint·GitHub 포함), Google Drive, GitHub. 연결 안 된 것은 메모만 하고 넘어간다.

### 2. 신호 수집
연결된 것에 한해 오늘 관련 항목을 모은다.

- **Notion**
  - `notion-search` (query_type `internal`): 나에게 온 멘션, 액션아이템, 할당된 작업. AI 검색이면 Slack/Jira/Linear/Teams 내용까지 포괄된다.
  - `notion-query-meeting-notes`: 최근 회의에서 나온 액션아이템.
  - 작업용 데이터베이스가 있으면 `notion-query-database-view`로 내게 할당된/마감 임박 항목.
- **Google Drive**
  - `list_recent_files` (orderBy `lastModifiedByMe` 또는 `recency`): 최근 손댄 문서.
  - 핵심 확인이 필요하면 `read_file_content`로 해당 파일만 본다.
- **GitHub** (`mcp__github__*`)
  - `search_issues` 로 `assignee:@me is:open`, `search_pull_requests` 로 `review-requested:@me is:open` 및 내가 연 `is:open author:@me`.
  - 최근 업데이트 순으로 정렬해 오늘 손볼 것을 추린다.

### 3. 정규화
수집한 항목을 다음 형태로 통일한다:

```
{ 제목, 출처(앱), 링크, 마감일?, 긴급신호, 예상소요 }
```
긴급신호 예: `마감 오늘/초과`, `남을 막는 중`, `답변 대기`, `직접 요청받음`.

### 4. 우선순위 산정 (긴급 × 중요)
- **P1 — 지금**: 오늘/초과 마감, 다른 사람을 막고 있음, 나를 기다리는 직접 요청.
- **P2 — 오늘 중**: 중요하지만 덜 긴급, 다가오는 마감.
- **P3 — 여유 되면**: 보조·후속 작업.

동점이면 마감일 → 대기 인원수 순으로 정렬. **각 항목에 "왜 이 순위인지" 한 줄 근거**를 단다.

### 5. 네이버 트렌드 키워드 (환경변수 있을 때만)
`NAVER_CLIENT_ID` / `NAVER_CLIENT_SECRET`가 설정돼 있는지 먼저 확인한다. **없으면 이 단계를 건너뛰고** 결과에 `(네이버 키 미설정 — 생략)`으로 표시한다.

```bash
[ -n "$NAVER_CLIENT_ID" ] && [ -n "$NAVER_CLIENT_SECRET" ] || echo "네이버 키 미설정 — 트렌드 생략"
```

**후보 키워드 도출**: 2~4단계에서 모은 업무 항목/주제에서 핵심 키워드(프로젝트명·제품명·관심 토픽)를 2~5개 뽑는다. 뽑기 애매하면 사용자에게 한 번 묻는다.

**데이터랩 검색어 트렌드** — 후보 키워드의 최근 30일 상대 검색 추이:
```bash
curl -s -X POST "https://openapi.naver.com/v1/datalab/search" \
  -H "X-Naver-Client-Id: $NAVER_CLIENT_ID" \
  -H "X-Naver-Client-Secret: $NAVER_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "startDate": "2026-05-01", "endDate": "2026-05-30", "timeUnit": "date",
    "keywordGroups": [
      {"groupName": "키워드A", "keywords": ["키워드A"]},
      {"groupName": "키워드B", "keywords": ["키워드B"]}
    ]
  }'
```
응답의 시계열로 **상승/하락 키워드**를 정리한다 (최근 값 vs 기간 평균).

**검색 API 뉴스 화제** — 키워드별 최신 뉴스에서 반복되는 화제어:
```bash
curl -s "https://openapi.naver.com/v1/search/news.json?query=키워드A&display=5&sort=date" \
  -H "X-Naver-Client-Id: $NAVER_CLIENT_ID" \
  -H "X-Naver-Client-Secret: $NAVER_CLIENT_SECRET"
```
제목/요약에서 자주 등장하는 단어를 모아 화제 키워드로 정리한다.

두 결과를 **"오늘의 트렌드 키워드"** 섹션으로 묶고, 내 업무 항목과 겹치는 키워드는 표시한다.

### 6. 출력
- **채팅 요약**: P1/P2/P3 그룹 + 트렌드 키워드 섹션을 간결히.
- **Notion 기록**: `notion-create-pages`로 **"오늘의 할 일 — YYYY-MM-DD"** 페이지 생성.
  - 우선순위별 체크박스(to-do) 목록, 각 항목에 출처 링크와 한 줄 근거.
  - 하단에 **트렌드 키워드** 섹션.
  - 부모 페이지를 모르면 사용자에게 한 번 묻고, 답이 없으면 workspace 루트의 private 페이지로 만든다.

## 출력 형식 예시

```markdown
# 오늘의 할 일 — 2026-05-30

## 🔴 P1 — 지금
- [ ] PR #214 리뷰 (GitHub) — 동료가 머지 대기 중, 나를 막고 있음 · [링크]
- [ ] 결제 버그 회신 (Slack via Notion) — 고객 답변 대기, 오늘 마감 · [링크]

## 🟡 P2 — 오늘 중
- [ ] 기획안 v2 검토 (Drive) — 내일 회의 자료 · [링크]

## ⚪ P3 — 여유 되면
- [ ] 백로그 정리 (Jira via Notion) · [링크]

## 📈 오늘의 트렌드 키워드
- 상승: **키워드A**(▲, 최근 7일 +40%) — 관련 뉴스 "…"
- 화제: 키워드B 관련 뉴스 3건 (내 업무 항목과 겹침)
```

## 원칙
- 간결하게. 항목이 없으면 "없음"이라고 적는다 — 채우려고 지어내지 않는다.
- 우선순위 근거는 한 줄. 장황한 설명 금지.
- 모든 항목에 출처 링크를 남겨 검증 가능하게 한다.
