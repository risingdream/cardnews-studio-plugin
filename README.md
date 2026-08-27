# cardnews-studio-plugin

Cardnews Studio 원격 MCP와 작품 조회 스킬을 묶은 에이전트 플러그인 마켓플레이스다.
Claude Desktop·Claude Code·Codex가 이 주소 하나로 같은 플러그인을 설치하고 갱신한다.

조회 정본은 원격 MCP `https://zent-studio.risingdream.workers.dev/mcp`다.
이 저장소는 매니페스트·스킬·라이선스·보안 정책만 담는다. 서비스 엔진 소스는 포함하지 않는다.

### 공개 범위

| 층 | 라이선스 | 공개 |
| --- | --- | --- |
| 이 저장소(매니페스트·스킬·MCP URL·문서) | MIT | 공개 |
| 계산·생성 실행 | — | 원격 MCP 엔드포인트만. 구현은 비공개 |
| 서비스 소스·Worker·웹 | — | 비공개 |

패키지에 Secret·토큰·쿠키·API 키는 없다. 인증은 설치 후 OAuth 동의로만 이루어진다.

## 플러그인 설치

| 클라이언트 | 방법 |
| --- | --- |
| Claude Desktop | 설정 → 플러그인 → 저장소에서 추가 → `risingdream/cardnews-studio-plugin` |
| Claude Code | `/plugin marketplace add risingdream/cardnews-studio-plugin` 후 `/plugin install cardnews-studio@cardnews-studio-plugin` |
| Codex | `codex plugin marketplace add risingdream/cardnews-studio-plugin` 후 `codex plugin add cardnews-studio@cardnews-studio-plugin` |

첫 도구 호출에서 OAuth 동의 화면이 한 번 뜬다. 기본 체크는 작품 조회(`works:read`)만이다.

## 들어 있는 것

```text
plugins/cardnews-studio/
  .claude-plugin/plugin.json   name cardnews-studio, 0.1.0
  .codex-plugin/plugin.json    Codex용 동일 메타
  .mcp.json                    cardnews-studio → 원격 MCP URL
  skills/inspect-works         조직·작품·리소스·카드 조회
  commands/inspect-works.md    같은 조회 커맨드
```

## 갱신

| 층 | 어떻게 |
| --- | --- |
| MCP 도구 | 서버 재배포. 플러그인 재설치 없음 |
| 스킬·매니페스트 | `plugin.json`의 `version`이 올라가면 마켓플레이스 갱신 |

- Claude Code: `/plugin marketplace update cardnews-studio-plugin` 후 필요하면 `/plugin update cardnews-studio@cardnews-studio-plugin`
- Codex: `codex plugin marketplace upgrade cardnews-studio-plugin` 후 `codex plugin add cardnews-studio@cardnews-studio-plugin`

`plugins/cardnews-studio/`는 비공개 서비스 저장소에서 동기화한다.
여기에 직접 낸 PR은 다음 동기화 때 덮어쓰이므로, 스킬 수정 제안·버그 신고는
[Issues](https://github.com/risingdream/cardnews-studio-plugin/issues)로 보낸다.
서비스 본체 버그는 비공개 저장소에서 다룬다.

## 제거와 OAuth 철회

| 동작 | 방법 |
| --- | --- |
| 플러그인 제거 | Claude `/plugin uninstall cardnews-studio@cardnews-studio-plugin`. Codex `codex plugin remove cardnews-studio` |
| 마켓 제거 | 선택. Claude `/plugin marketplace remove cardnews-studio-plugin`. Codex `codex plugin marketplace remove cardnews-studio-plugin` |
| OAuth 철회 | 서비스 설정에서 grant 삭제. 플러그인 제거만으로는 서버 grant가 안 지워질 수 있다 |

## 지원

플러그인 설치·문서 문제는 이 저장소 Issues.
취약점은 공개 이슈가 아니라 [SECURITY.md](SECURITY.md) 경로로 제보한다.
개인정보 고지는 [PRIVACY.md](PRIVACY.md)다.

## 라이선스

이 저장소의 내용(스킬 문서·MCP 설정·매니페스트)은 MIT다. 서비스 엔진 소스는 포함되지 않으며 비공개다.
