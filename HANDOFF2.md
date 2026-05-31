# 프로젝트 핸드오프 — andrej-karpathy-skills (네이버·워드프레스 연동 작업)

너는 이전 클라우드 세션의 작업을 이어받는다. 아래는 프로젝트 전반과 현재까지의 진행 상황이다. 끝까지 읽고 "현재 상태"를 확인한 뒤 다음 단계를 제안하라.

## 1. 프로젝트 정체
- **저장소:** `kick-the-door/andrej-karpathy-skills` (포크 원본은 `forrestchang/andrej-karpathy-skills`)
- **성격:** Andrej Karpathy의 "LLM 코딩 함정" 관찰을 바탕으로 한 **Claude Code 행동 지침(behavioral guidelines)** 모음. 실행 코드는 없고 마크다운 + 플러그인/스킬 정의만 있다.
- **작업 브랜치:** `claude/env-naver-wordpress-check-QKVve` (main에서 분기)

## 2. 저장소 구조
```
CLAUDE.md                                   # 핵심 행동 지침 4원칙 (프로젝트 규칙)
README.md / README.zh.md                    # 영문/중문 설명
CURSOR.md                                   # Cursor 연동 안내
EXAMPLES.md                                 # 사용 예시
.claude-plugin/plugin.json                  # 플러그인 정의 (skill 경로 등록)
.claude-plugin/marketplace.json             # 마켓플레이스 정의 (karpathy-skills)
skills/karpathy-guidelines/SKILL.md         # 스킬 본문 (CLAUDE.md와 동일한 4원칙)
.cursor/rules/karpathy-guidelines.mdc       # Cursor용 동일 규칙
```

## 3. 반드시 따라야 할 프로젝트 규칙 (CLAUDE.md 4원칙)
1. **Think Before Coding** — 가정을 명시하고, 모호하면 묻는다. 해석이 여러 개면 임의로 고르지 말고 제시한다.
2. **Simplicity First** — 요청한 것만, 최소 코드로. 투기적 추상화·미사용 유연성 금지. 200줄이 50줄이면 다시 써라.
3. **Surgical Changes** — 건드릴 것만 건드린다. 무관한 코드/주석/포맷 "개선" 금지. 내 변경이 만든 고아만 정리한다.
4. **Goal-Driven Execution** — 검증 가능한 성공 기준을 세우고 통과할 때까지 반복한다.

## 4. 현재 진행 중인 실제 작업
환경변수로 주입된 **네이버 OpenAPI**와 **워드프레스 REST API** 연결을 점검하는 일.

### 사용 환경변수
| 변수 | 값 / 상태 | 용도 |
|------|-----------|------|
| `WP_BASE_URL` | `https://4nomads.kr` | 워드프레스 사이트 |
| `WP_USER` | `API` | 워드프레스 사용자 |
| `WP_APP_PASSWORD` | (시크릿) | 워드프레스 앱 비밀번호 |
| `NAVER_CLIENT_ID` | `1xwtwWq6D_W3fM1GFcvK` | 네이버 OpenAPI 클라이언트 ID |
| `NAVER_CLIENT_SECRET` | (시크릿) | 네이버 OpenAPI 시크릿 |

> 로컬에서는 이 변수들을 직접 설정해야 한다. 프로젝트 루트에 `.env`로 만들고 반드시 `.gitignore`에 추가할 것. 시크릿 실제값은 사용자가 보관 중.

## 5. 지금까지 확인된 결과 (클라우드 세션에서)
- 🟢 **워드프레스: 정상.** `GET /wp-json/wp/v2/users/me?context=edit` → HTTP 200. 사용자 `API API`(id=2), 역할 **administrator**, `edit_posts` 권한 보유. 글 작성/발행 가능 상태.
- 🔴 **네이버: 연결 불가(차단).** `openapi.naver.com` 호출이 클라우드 실행 환경의 **송신(egress) 정책**에 막혀 HTTP 403 `Blocked by egress policy`. `naver.com`도 `Host not in allowlist`. → **키 문제가 아니라 클라우드 네트워크 허용목록 문제**라서, 키 유효성은 아직 검증 못 함.

## 6. 로컬(맥북 VSCode)에서 기대되는 변화
- 로컬 네트워크로 직접 나가므로 **네이버 egress 차단이 사라진다.**
- 따라서 이 환경에서 **처음으로 네이버 키 유효성**(Client ID/Secret 유효성, 네이버 개발자센터의 API 사용 신청 여부, 서비스 URL/허용 IP 제한)을 실제로 테스트할 수 있다.

## 7. 첫 작업 지시
1. 위 환경변수가 로컬에 로드돼 있는지 확인(`.env` → `set -a; source .env; set +a`).
2. 워드프레스 연결을 재확인(`/wp-json/wp/v2/users/me`)해 200/권한을 검증.
3. 네이버 검색 API(`https://openapi.naver.com/v1/search/blog.json?query=test&display=1`)를 `X-Naver-Client-Id`/`X-Naver-Client-Secret` 헤더로 호출해 인증·응답 상태를 확인하고, 실패 시 원인(키 무효 / 미신청 API / IP·URL 제한)을 구분해 보고.
4. 두 API 상태를 한 번에 점검하는 작은 스크립트가 필요하면 4원칙(특히 Simplicity First)에 맞춰 제안만 먼저 하고 동의를 받은 뒤 작성.

확인되면 현재 상태를 요약하고 다음 단계를 제안하라.
