---
description: Cardnews Studio에서 조직·작품·리소스·카드를 조회한다. 저장·생성은 하지 않는다.
---

# 작품 조회

원격 MCP `cardnews-studio`의 조회 도구만 사용한다.

1. `list_organizations`로 가입 조직과 마지막 활성 조직을 확인한다.
2. `list_works`로 작품 목록을 받는다. 다른 조직이면 `organization_id`를 넣는다.
3. 대상이 정해지면 `get_work`로 메타를 본다. 정본은 `work_id`다.
4. 리소스가 필요하면 `list_resources`를 쓴다.
5. 카드나 기준 이미지를 눈으로 볼 때만 `view_media`에 MCP media URI를 넘긴다.

uid를 인자에 넣지 않는다. 저장·생성 도구는 호출하지 않는다.
