---
description: Orbit Studio에서 조직·채널·콘텐츠·작품·리소스·카드를 조회한다. 저장·생성은 하지 않는다.
---

# 작품 조회

원격 MCP `cardnews-studio`의 조회 도구만 사용한다.

1. `list_organizations`로 가입 조직과 마지막 활성 조직을 확인한다.
2. `list_channels`로 채널·규칙·프로젝트를, `list_contents`로 콘텐츠(단계·편성·문서·관문)를 본다. 본문과 승인·게시 기록은 `get_content`다.
3. `list_works`(카드 계열)·`list_motion_works`(모션)로 작품 목록을 받는다. `channel_id`·`project_id`·`content_id`로 거른다. 다른 조직이면 `organization_id`를 넣는다.
4. 대상이 정해지면 `get_work`(모션은 `get_motion_work`)로 메타를 본다. 정본은 `work_id`다.
5. 리소스가 필요하면 `list_resources`를 쓴다.
6. 카드나 기준 이미지를 눈으로 볼 때만 `view_media`에 MCP media URI를 넘긴다.
7. 모션 리뷰 지적은 `list_motion_comments`, 게시 기록은 `list_publications`로 본다.

uid를 인자에 넣지 않는다. 저장·생성 도구는 호출하지 않는다.
