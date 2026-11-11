---
name: inspect-works
description: Orbit Studio MCP로 조직·채널·콘텐츠·작품·리소스·카드를 조회한다. 채널별 콘텐츠와 단계, 콘텐츠의 소재 카드·대본·검증 표·콘티, 작품 목록·메타, 리소스, 카드 이미지, 모션 리뷰 코멘트, 게시 기록 확인을 요청하면 이 스킬을 쓴다. 조회만 하며 저장·생성은 다루지 않는다.
---

# 작품 조회

도구는 **원격 MCP** `cardnews-studio`다. 로컬 바이너리를 설치하지 않는다.
첫 호출에서 OAuth 동의 화면이 뜬다. 플러그인 기본 체크는 `works:read` + `works:write` + `generate`다. 이 스킬은 조회 도구만 쓴다.

범위: 조직 → 채널 → 콘텐츠(단계·문서·승인·게시 기록) → 작품 목록 → 작품 메타 → 리소스 메타 → (필요 시) 카드 PNG, 모션 리뷰 코멘트.
범위 밖: 콘텐츠·문서 쓰기, 작품 저장, 리소스 생성, 카드 생성, 렌더, 코멘트 해결 표시.

## 구조

조직 > **채널**(게시 계정) > **프로젝트**(연작) > **콘텐츠**(한 주제) > **작품**(카드뉴스·발표·원페이지·모션 등 포맷별 산출물).
콘텐츠의 단계는 `topic`(소재) → `plan`(기획) → `storyboard`(콘티) → `production`(제작) → `export`(내보내기) → `published`(게시)다.
콘텐츠는 옛 「시나리오」와 같은 것이고 id도 같다(`content_id = scenario_id`). 조회에는 `*_content` 도구와 `content_id`를 쓴다. `list_scenarios`·`get_scenario`는 호환용이다.

위에서 아래로 좁혀 간다: 채널 → 콘텐츠 → 그 콘텐츠의 작품. 작품 전체 목록부터 훑지 않는다.

## 원칙

1. uid는 도구 인자에 넣지 않는다. 서버가 토큰에서 읽는다.
2. 작품 정본은 `work_id`(UUID)다. `slug`는 활성 조직 별칭이다.
3. 이미지 바이너리를 JSON에 넣거나 base64로 받지 않는다. 픽셀 검수는 `view_media`만 쓴다.
4. 활성 조직이 아니면 `organization_id`를 명시한다.

## 흐름

```
1. list_organizations   가입 조직과 last_organization_id
2. list_channels        채널과 규칙(brand_rules·ops_rules), 채널별 프로젝트(project_id), 채널 없는 프로젝트(unassigned_projects)
3. list_contents        콘텐츠 목록. channel_id('none' = 채널 없음)·project_id·stage·편성 날짜(from·to)·unscheduled로 거른다
4. get_content          documents(소재 카드·대본·검증 표·콘티 본문), approval_history, production_gate, publications, works
5. list_works           카드뉴스·발표·원페이지 목록. channel_id·project_id·content_id로 거른다
   list_motion_works    모션 작품 목록. 같은 필터
6. get_work             원문·스토리보드·승인·리소스·카드 URL. PNG 없음 (모션은 get_motion_work)
7. list_resources       해당 작품 리소스 메타
8. view_media           MCP media URI의 PNG. 큰 파일은 resources/read 대신 이 도구
```

`get_work`와 `list_resources`의 이미지 필드는 `https://studio.risingdream.kr/mcp/resources/media/<key>` 형태다. query `token`을 붙이지 않는다.

## 읽는 법

| 볼 것 | 어디 | 뜻 |
|---|---|---|
| 어느 단계인가 | `list_contents` 항목의 `stage`, `scheduled_on` | 편성 날짜는 제안이지 예약이 아니다 |
| 문서가 있는가 | `docs.topic_card`·`script`·`verification`·`storyboard` | null이면 아직 없다. `version`은 쓸 때마다 오른다 |
| 콘티가 승인됐는가 | `approvals.storyboard`, `production_gate` | `production_gate.open`이 false면 콘티 승인 전이라 제작 단계로 넘어가지 못한다. `required`가 false면 채널이 관문을 껐다 |
| 승인이 왜 풀렸는가 | `get_content`의 `approval_history` | `revoke_reason: doc_changed`는 승인 뒤 문서를 고친 것이다 |
| 어떤 작품이 붙었는가 | `get_content`의 `works`, 또는 `list_works`·`list_motion_works { content_id }` | 항목의 `content_id`가 null이면 콘텐츠에 안 붙은 작품이다. `scenario_id`는 `content_id`와 같은 값이다 |
| 채널별 작품 | `list_works`·`list_motion_works { channel_id }` | `'none'`은 채널 없는 프로젝트의 작품과 미분류 작품 |
| 게시됐는가 | `get_content`의 `publications`, `list_publications { channel_id, from, to }` | 기록일 뿐이다. 이 MCP는 아무 데도 올리지 않는다 |
| 리뷰 지적 | `list_motion_comments { work_id, status }` | 사람이 모션 뷰어에 꽂은 코멘트. `t_ms`(시각)·`x`·`y`(화면 위치)·`scene`·`body`·`status`(open·resolved)·`replies` |

없는 작품은 404로 끝낸다. 추측해서 다른 조직의 작품을 열지 않는다.
