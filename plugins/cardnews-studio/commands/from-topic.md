---
description: 주제로 카드뉴스 원문·카피·그림을 세 체크포인트로 만든다. 각 단계에서 멈추고 이어갈 수 있다.
argument-hint: "<주제>"
---

# 주제에서 카드뉴스

원격 MCP `cardnews-studio`만 사용한다. 로컬 스토리보드를 쓰지 않는다.
`$ARGUMENTS`가 주제다. 비어 있으면 주제를 묻는다.

시작 전에 `list_channels`(채널 규칙) → `list_contents` → `get_content`로 같은 주제 콘텐츠가 있는지 본다. 없으면 `create_content`로 프로젝트 안에 만든다. 작품은 `list_works { content_id }`·`get_work`로 찾고, 있으면 그 `work_id`를 이어 가고 새 작품을 만들지 않는다.

1. **원문** — 웹 검색으로 조사한다. 근거 없는 문장 금지, 1차 출처 우선, 출처 3개 이상, 1,500~3,000자. 법령은 korean-law `verify_citations`·`applicable_law`. 숫자는 확인하지 못하면 `[확인 필요]`. `write_content_doc`으로 소재 카드(`topic_card`)·대본(`script`)·검증 표(`verification`)를 쓰고, `create_work`(format `card-series`, `sources`, `content_id`) 후 `save_work`(`source`). 요약과 출처를 보여주고 승인을 기다린다.
2. **카피** — `analyze_source` → 독자·훅·순서로 초안을 고친다 → `save_work`(`cards`, `citations`) → `write_content_doc`(`storyboard`: 카드 한 장 = 컷 하나) → 사용자 확인 후 `approve_work`(copy). 콘티 승인은 사용자가 Studio 화면에서 한다.
3. **그림** — `set_content_stage`(production)이 409 `storyboard_approval_required`면 멈추고 콘티 승인을 요청한다. 통과하면 `visual.prompt`가 도형 명세인지 본다(`unsafe_prompt`를 읽음). 필요하면 `create_resource`. `generate_cards` → `get_job` → `view_media`. 문제 카드만 `card_ids`로 재생성한다.
4. **게시 기록** — 게시는 사용자가 한다. 게시한 뒤 `record_publication`으로 기록하고 `set_content_stage`(published).

uid를 인자에 넣지 않는다. 이미지를 base64로 받지 않는다. 콘텐츠 없이 작품부터 만들지 않는다.
