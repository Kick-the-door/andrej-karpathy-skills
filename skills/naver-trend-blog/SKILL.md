---
name: naver-trend-blog
description: 네이버 트렌드(데이터랩+뉴스)를 읽어 워드프레스에 이슈성 글을 초안으로 작성하고, 네이버 블로그용 주제(경제·자기계발) 10개를 매일 추천한다. "트렌드 글 써줘", "워드프레스 초안", "블로그 주제 추천", "오늘 블로그 뭐 쓰지" 같은 요청과 매일 자동 실행에 사용.
license: MIT
---

# Naver Trend Blog

네이버 트렌드를 기반으로 ① 워드프레스 **이슈성 글 초안**을 만들고 ② 네이버 블로그용 **주제 10개**를 추천하는 에이전트.

## 솔직한 전제 (먼저 읽기)

- **자동 발행 안 함**: 워드프레스 글은 항상 `status: draft`(초안)로만 만든다. 검토 후 직접 발행한다.
- **네이버 블로그는 추천만**: 네이버 블로그는 공식 글쓰기 API가 없어 **발행 불가**. 주제 10개 목록까지만 만든다.
- **네이버 실시간 검색어 없음**: 2021년 폐지. 데이터랩은 *지정 키워드*의 상대 추이만 주므로, 후보 키워드를 넣어 상승분을 찾고 뉴스 API로 화제를 보강한다.
- **키 없으면 건너뛴다**: 필요한 환경변수가 없으면 해당 단계를 생략하고 명시한다. 없는 데이터는 지어내지 않는다.

## 필요한 환경변수

| 변수 | 용도 |
|------|------|
| `NAVER_CLIENT_ID` / `NAVER_CLIENT_SECRET` | 데이터랩·검색 API ([Naver Developers](https://developers.naver.com)에서 발급, "검색"+"데이터랩" 체크) |
| `WP_BASE_URL` | 워드프레스 사이트 URL (예: `https://example.com`) |
| `WP_USER` / `WP_APP_PASSWORD` | 워드프레스 REST용 (관리자 → 사용자 → 애플리케이션 비밀번호) |

시작 시 존재 여부부터 확인하고, 빠진 키에 해당하는 단계는 생략한다:
```bash
[ -n "$NAVER_CLIENT_ID" ] && [ -n "$NAVER_CLIENT_SECRET" ] || echo "네이버 키 없음 → 트렌드 단계 생략"
[ -n "$WP_BASE_URL" ] && [ -n "$WP_APP_PASSWORD" ] || echo "워드프레스 키 없음 → 발행 단계 생략(초안 미리보기만)"
```

## 워크플로

### 1. 트렌드 수집 (네이버 키 있을 때)
**후보 키워드** — 경제·자기계발 축으로 시드 키워드를 잡는다(예: 금리, 환율, 부동산, 절세, 재테크, 부업, 생산성, 습관, 자기계발). 사용자가 관심 키워드를 주면 우선 사용.

**데이터랩 상승 키워드** — 최근 30일 상대 추이:
```bash
curl -s -X POST "https://openapi.naver.com/v1/datalab/search" \
  -H "X-Naver-Client-Id: $NAVER_CLIENT_ID" \
  -H "X-Naver-Client-Secret: $NAVER_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{"startDate":"<30일전>","endDate":"<오늘>","timeUnit":"date",
       "keywordGroups":[{"groupName":"금리","keywords":["금리"]}, ...]}'
```
최근 값이 기간 평균 대비 오른 키워드를 **상승 키워드**로 추린다.

**뉴스 화제** — 상승 키워드별 최신 뉴스에서 구체 이슈를 잡는다:
```bash
curl -s "https://openapi.naver.com/v1/search/news.json?query=<kw>&display=5&sort=date" \
  -H "X-Naver-Client-Id: $NAVER_CLIENT_ID" -H "X-Naver-Client-Secret: $NAVER_CLIENT_SECRET"
```
제목/요약에서 오늘의 이슈 각도를 정한다.

### 2. 워드프레스 이슈성 글 초안 작성 (워드프레스 키 있을 때)
상승 키워드 1개를 골라 이슈성 글을 쓴다 — 제목, 도입(왜 지금 이슈인지), 본문(뉴스 근거 요약 + 해설), 마무리. 출처 뉴스 링크를 본문에 남긴다. **사실 확인이 안 된 수치·인용은 쓰지 않는다.**

초안으로 등록 (`status: draft`):
```bash
curl -s -X POST "$WP_BASE_URL/wp-json/wp/v2/posts" \
  -u "$WP_USER:$WP_APP_PASSWORD" \
  -H "Content-Type: application/json" \
  -d '{"title":"<제목>","content":"<HTML 본문>","status":"draft"}'
```
응답의 글 ID와 편집 링크(`$WP_BASE_URL/wp-admin/post.php?post=<id>&action=edit`)를 보고한다. 워드프레스 키가 없으면 초안 본문을 채팅으로만 미리보여 준다.

### 3. 네이버 블로그 주제 10개 추천
경제·자기계발 각도로 **오늘의 주제 10개**를 만든다. 가능하면 1단계의 상승 키워드/뉴스 화제와 엮는다. 각 주제에 한 줄 후킹 포인트를 붙인다(예: "왜 지금?" 또는 타깃 독자). 5:5 또는 사용자가 정한 비율로 경제/자기계발 분배.

### 4. 출력
- 채팅에 ① 워드프레스 초안 링크(또는 미리보기) ② 주제 10개 목록을 정리.
- 매일 실행이면 날짜를 머리에 붙인다("YYYY-MM-DD 블로그 자동화").

## 출력 형식 예시
```markdown
# 2026-05-30 블로그 자동화

## ✍️ 워드프레스 초안
- 제목: "기준금리 동결, 내 대출이자에 생기는 변화 3가지"
- 상태: 초안(draft) · [편집 링크](https://example.com/wp-admin/post.php?post=123&action=edit)
- 근거 뉴스: [한국경제 …], [연합뉴스 …]

## 📝 네이버 블로그 주제 10개
**경제 (5)**
1. 환율 1,400원 시대, 해외직구족이 지금 점검할 것 — 왜 지금?: 환율 급등
2. …
**자기계발 (5)**
6. 아침 1시간 루틴으로 생산성 2배 만드는 법 — 타깃: 직장인
7. …
```

## 매일 자동 실행 설정
이 컨테이너는 임시 환경이라 자체적으로 24시간 돌지 않는다. 매일 돌리려면 둘 중 하나:
- **Claude Code on the web 예약 트리거**: 이 스킬을 매일 정해진 시각에 실행하도록 스케줄 등록.
- **외부 cron**: 서버/PC의 cron이 이 스킬 절차를 실행(네이버·워드프레스 키를 환경변수로 주입).

`/loop` 스킬로 세션이 열려 있는 동안 주기 실행도 가능하지만, 세션이 닫히면 멈춘다.

## 원칙
- 워드프레스는 항상 초안. 자동 공개 금지.
- 출처 링크를 남겨 검증 가능하게.
- 키가 없으면 그 단계는 "(키 없음 — 생략)"으로 표시하고 나머지는 진행.
