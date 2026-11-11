---
name: from-topic
description: 주제로 카드뉴스 원문·카피·그림을 세 체크포인트로 만든다. 콘텐츠(채널 > 프로젝트 > 콘텐츠)를 먼저 찾거나 만들고 그 안에 소재 카드·대본·검증 표·콘티를 쓴 뒤 작품을 붙인다. from-topic, 주제에서 카드, 리서치 후 카드뉴스를 요청하면 이 스킬을 쓴다. 각 단계에서 멈추고 이어갈 수 있다.
---

# 주제에서 카드뉴스

도구는 **원격 MCP** `cardnews-studio`다. 로컬 스토리보드·로컬 엔진을 쓰지 않는다.

## 인증 선확인

첫 동작은 인증 상태 확인이다(`list_organizations`).
`Needs authentication`이면 멈추고 사용자에게 알린다.
비대화형 워커 세션에서는 OAuth 동의 화면이 뜰 수 없어
첫 호출의 동의 화면을 기다리지 않는다.
단 opencode에서는 `opencode mcp auth <name>` 1회 시도 후
`mcp list`로 재확인한다. 세션 도중 인증해도 그 세션에는
도구가 붙지 않을 수 있으니 수임 워커의 첫 동작도 `list_organizations`로 둔다.
기본 체크는 `works:read` + `works:write` + `generate`를 유지한다.

범위: 콘텐츠 찾기·만들기 → 리서치(소재 카드·대본·검증 표) → 카드 초안(콘티) → 사람의 콘티 승인 → 그림 생성 → 게시 기록.
이 스킬은 오케스트레이터다. 카드 구성·프롬프트 명세는 제품 도구가 강제한다.

## 구조

조직 > **채널**(게시 계정) > **프로젝트**(연작) > **콘텐츠**(한 주제) > **작품**(카드뉴스·발표·모션 등 포맷별 산출물).
콘텐츠는 단계를 가진다: `topic`(소재) → `plan`(기획) → `storyboard`(콘티) → `production`(제작) → `export`(내보내기) → `published`(게시).
콘텐츠 하나에 단계 문서 넷이 모인다: `topic_card`(소재 카드), `script`(대본·원문), `verification`(검증 표), `storyboard`(콘티).

- 콘텐츠는 옛 「시나리오」와 같은 것이고 id도 같다. 도구는 `*_content`와 `content_id`를 쓴다. `*_scenario` 도구와 `scenario_id` 인자는 호환용이다.
- **작품부터 만들지 않는다.** 콘텐츠에 붙지 않은 작품은 Studio의 콘텐츠 목록에 보이지 않는다.

## 원칙

1. uid는 도구 인자에 넣지 않는다. 서버가 토큰에서 읽는다.
2. 작품 정본은 `work_id`(UUID)다. `slug`는 활성 조직 별칭이다.
3. 이미지 바이너리를 JSON에 넣거나 base64로 받지 않는다. 픽셀 검수는 `view_media`만 쓴다.
4. 활성 조직이 아니면 `organization_id`를 명시한다.
5. 이미 있는 작품을 이어서 할 때 `create_work`로 새 작품을 만들지 않는다.
6. 콘텐츠가 먼저다. 작품은 `create_work`에 `content_id`를 줘서 만든다.
7. 채널 규칙을 따른다. `list_channels`의 `ops_rules`(검증 필수 `require_verification`, 게시 승인 `require_post_approval`, 하루 게시 한도 `daily_post_limit`, 게시 간격 `min_post_gap_hours`, 스텔스 `stealth`)와 `brand_rules`(어조 `tone`, 색 `palette`, 글꼴 `fonts`, 금지 `forbidden`)를 글·그림에 그대로 적용한다.
8. 콘티 승인은 사람이 Studio 웹 화면에서 한다. 승인 도구는 없다. 에이전트가 관문을 돌아가지 않는다.

## 재개

주제를 받으면 새 콘텐츠·새 작품을 만들기 **전에** 저장본을 읽는다.

```
1. list_organizations   가입 조직과 last_organization_id
2. list_channels        채널과 규칙(ops_rules·brand_rules), 채널별 프로젝트(project_id)
3. list_contents        channel_id 또는 project_id로 걸러 같은 주제의 콘텐츠가 있는지. stage·docs·production_gate를 본다
4. get_content          있으면 documents(소재 카드·대본·검증 표·콘티), approval_history, works
5. list_works           content_id로 걸러 그 콘텐츠의 카드 작품. 콘텐츠가 없으면 같은 주제의 title·slug가 있는지
6. get_work             source, storyboard.sources, cards, approvals, asset_url
```

콘텐츠 판정:

| 저장본 | 할 일 |
| --- | --- |
| 콘텐츠 없음 | `create_content { project_id, title, topic_card? }`. 어느 프로젝트인지 모르면 사용자에게 묻는다. 프로젝트는 Studio 화면에서 만든다 |
| 콘텐츠는 있고 작품이 없음 | 그 `content_id`로 체크포인트 1부터 |
| 작품은 있는데 콘텐츠에 안 붙어 있음(`content_id: null`) | 콘텐츠를 찾거나 만든 뒤 `set_work_scenario { work_id, content_id }`로 붙인다 |
| `production_gate.open`이 false이고 콘티 문서가 있음 | 콘티 승인을 기다리는 중이다. 체크포인트 2의 「승인 기다리기」로 간다 |

작품 판정:

| 저장본 | 시작 체크포인트 |
| --- | --- |
| 작품 없음 | 1 원문. 그때만 `create_work`(`content_id` 포함) |
| `source`가 기본 안내문이거나 `sources`가 비어 있음 | 1 원문. 기존 `work_id`에 `save_work` |
| 원문·출처는 있고 카드 `headline`이 비었거나 `approvals.copy`가 false | 2 카피 |
| `approvals.copy`가 true이고 카드 `asset_url`이 없음 | 3 그림 |
| 카드 이미지가 있음 | 3 그림. 문제 카드만 재생성 |

## 체크포인트 1 — 원문

웹 검색·페이지 조회로 조사한다. Claude Code는 WebSearch/WebFetch, 없으면 동등한 검색·조회 도구.

리서치 계약:

- 근거 링크 없는 문장 금지
- 1차 출처 우선: 법령정보센터, 정부 보도자료, 공식 통계. 언론·블로그는 1차를 확인한 뒤에만
- 법령 인용은 korean-law MCP `verify_citations`로 조문 실존을, `applicable_law`로 시행일을 검증. 못 하면 인용하지 않거나 `[확인 필요]`
- korean-law MCP가 없으면 법제처 Open API 등 동등한 1차 원본으로 직접 확인하고 수단을 기록한다
- korean-law 검증은 5~6건씩 나눠 호출한다. 18건 일괄은 통째로 실패한다
- 개정 조문은 해당 조문을 고친 법령의 부칙·경과조치까지 직접 본다. `applicable_law` 발췌만으로 갈음하지 않는다
- 백분율은 원문 표기 그대로 써도 검사기가 `N%`와 같은 사실로 본다. 가독성용 `100분의 N(N%)` 병기는 선택
- 확인하지 못한 숫자는 `[확인 필요]`
- 회계법인 명의 톤. 단정 대신 요건과 예외를 병기
- 원문 1,500~3,000자(카드 7~12장 기준)
- 출처 3개 이상. 각 `{id, label, url?}`

원문은 마크다운. 문단을 나눠 뒤에 출처 id를 밝힌다. 카드 카피·이미지 프롬프트를 원문에 넣지 않는다.

저장 (MCP 스키마 그대로). 콘텐츠 문서가 정본이고 작품의 `source`는 카드 초안을 뽑는 데 쓰는 같은 글이다:

```
1. write_content_doc  content_id, kind=topic_card, topic_card={...}     소재 카드: 대상·질문·방향·근거 URL
2. write_content_doc  content_id, kind=script, script=원문 markdown     대본(원문)
3. write_content_doc  content_id, kind=verification, verification={...} 검증 표: 주장마다 근거와 결과
4. set_content_stage  content_id, stage=plan
5. create_work        format=card-series, slug, title, sources, content_id
                      source 인자는 없다. 새 작품일 때만. 작품이 그 콘텐츠의 프로젝트에 들어간다
6. save_work          work_id, source=원문 markdown(2와 같은 글)
                      sources를 create에 못 넣었으면 여기서
```

문서는 통째로 쓴다(부분 수정 없음: `get_content`로 읽어 고친 전체를 보낸다). 정해진 항목이 아닌 값은 `extra`에 넣는다. 최상위에 다른 키를 두면 422다.

```json
{
  "content_id": "<content_id>",
  "kind": "topic_card",
  "topic_card": {
    "audience": "올해 본인이나 가족의 안경·콘택트렌즈를 산 직장인",
    "question": "안경점에서 카드로 샀는데 간소화 의료비에 안 보여요. 뭘 따로 받아 와야 하나요?",
    "timeliness": "12월 둘째 주(영수증을 미리 받아 둘 때)",
    "angle": "간소화 목록에 안경 줄만 비어 있는 것을 보여 준다. 안경점에서 받는 확인 영수증 한 장이 답이다",
    "hook": "안경은 병원에서 산 게 아닌데, 의료비예요?",
    "answer": "시력교정용 안경과 렌즈는 의료비에 들어가요. 다만 영수증은 내가 챙겨야 할 수 있어요.",
    "viewer_action": "안경점에 시력교정용 구입 확인 영수증(사용자 이름 포함)을 요청한다",
    "risks": "시력교정용만 해당(선글라스·미용 렌즈 제외), 1인당 한도 있음",
    "sources": [{ "url": "https://call.nts.go.kr/call/qna/selectQnaInfo.do?mi=1318&ctgId=CTG11909", "title": "국세청 의료비 FAQ" }],
    "editorial_status": "우선 제작",
    "review_status": "검증 전",
    "extra": { "묶음": "병원비 묶음" }
  }
}
```

```json
{
  "content_id": "<content_id>",
  "kind": "verification",
  "verification": {
    "rows": [
      { "claim": "시력보정용 안경·콘택트렌즈 구입비는 의료비에 들어간다",
        "basis": "소득세법 시행령 제118조의5 제1항 제4호(2026-01-02 시행)",
        "result": "일치", "location": "원문 2문단", "source": "법령 원문 조회" }
    ],
    "checked_on": "2026-11-11",
    "method": "조문 원문 대조(시행일 기준 현행판)",
    "summary": "주장 1건 일치"
  }
}
```

채널 `ops_rules.require_verification`이 true면 검증 표 없이 다음 체크포인트로 가지 않는다.

`slug`는 소문자·숫자·하이픈만. 사용자에게 원문 요약과 출처 목록을 보여주고 **승인 후에** 체크포인트 2로 간다.

## 체크포인트 2 — 카피

```
1. analyze_source    work_id. format=card-series. 저장된 source를 쓴다
2. 초안 검토         대상 독자·훅·카드 순서. evidence의 quote가 원문과 맞는지
3. save_work         work_id, cards, citations
                     citations는 card_ids로 카드와 출처를 잇는다
4. write_content_doc content_id, kind=storyboard, storyboard={...}   카드 한 장 = 컷 하나
5. set_content_stage content_id, stage=storyboard
6. approve_work      stage=copy. 사용자 확인 후에만
```

콘티 문서는 카드 순서대로 컷을 적는다. `id`는 카드 id, `caption`은 headline·subline, `action`은 그림 방향, `on_screen_text`는 카드에 찍히는 글자다. 원문의 필수 조건이 어느 카드에 있는지 `requirements`로 대조한다.

```json
{
  "content_id": "<content_id>",
  "kind": "storyboard",
  "storyboard": {
    "logline": "간소화에 안 뜨는 안경 영수증, 안경점에서 한 장 받으면 된다",
    "cuts": [
      { "id": "01", "caption": "안경은 병원에서 산 게 아닌데, 의료비예요?", "action": "안경집이 놓인 책상, 빈 목록 카드", "on_screen_text": "안경도 의료비" },
      { "id": "02", "caption": "시력교정용이면 의료비에 들어가요", "action": "안경과 콘택트렌즈 두 물건을 나란히", "on_screen_text": "시력교정용만", "note": "선글라스 제외를 불릿으로" }
    ],
    "requirements": [
      { "requirement": "시력교정용만 해당", "met": true, "note": "카드 02 화면 글자" }
    ],
    "rules": "강조색은 카드당 한 곳"
  }
}
```

### 승인 기다리기

콘티 승인은 사람이 Studio의 콘텐츠 화면에서 한다. 콘티를 쓴 뒤 사용자에게 콘텐츠 주소와 함께 승인을 요청하고 멈춘다.

- `set_content_stage { stage: "production" }`이 409 `storyboard_approval_required`를 주면 아직 승인 전이다. 다시 시도하지 말고 사용자에게 Studio에서 콘티를 승인해 달라고 한다.
- 승인된 콘티를 `write_content_doc`으로 고치면 승인이 취소된다(결과의 `revoked_approvals`). 무엇을 왜 고쳤는지 알리고 다시 승인을 요청한다.
- 채널이 `ops_rules.require_storyboard_approval: false`면 관문이 없다(`production_gate.required: false`).

`analyze_source`가 `unsafe_prompt`·`evidence_mismatch`·`ungrounded_fact`를 내면 원문 또는 초안을 고치고 다시 호출한다. 카드 `id`는 저장된 카드 id만. 없는 id는 422다.

텍스트 패널 넘침 규칙(visual-story 4:5·36% 패널·60px 제목 기준):
제목 ≤16자(한 줄)면 불릿 3개까지, 제목 17~22자(두 줄)면 **불릿 2개·각 ≤20자**.
렌더 검사는 넘친 카드 전부를 한 번에 알려주므로 한 번에 고친다.

사용자에게 카드별 headline·subline·근거 한 줄을 보여주고 **승인 후에** `approve_work(copy)`를 호출한다.

## 체크포인트 3 — 그림

그림 생성은 비용이 든다. 콘티 승인 뒤 `set_content_stage { content_id, stage: "production" }`이 통과한 다음에 시작한다.

```
1. 프롬프트 검토     각 카드 visual.prompt가 도형 명세인지
                    shape/count/proportion/color. 한글·label/caption/title 렌더 지시 없음
                    analyze_source·save_work가 낸 unsafe_prompt를 읽고 고친다
2. create_resource  기준 인물·장소가 필요할 때만. 이어서 필요하면 generate_resource
3. generate_cards   work_id, idempotency_key 필수. job_id를 get_job으로 따라간다
                    timeout·failed면 resume_job
4. view_media       카드 asset MCP URI. 삽화 안 글자·로고를 본다
5. 재생성            문제 카드만 generate_cards에 card_ids.
                     기준 카드(01·continuity_from 조상) 자산이 함께 실리므로
                     anchor를 빼고 요청해도 엔진이 포함한다.
                     그래도 `missing continuity reference`가 나면 anchor를 card_ids에 포함해 다시 요청한다
```

`approve_work(visual)`·`final`은 이 스킬이 자동으로 올리지 않는다. 사용자에게 문제 카드와 재생성 범위를 안내한다.

## 마무리 — 내보내기와 게시 기록

- 렌더·내려받기까지 끝나면 `set_content_stage { stage: "export" }`.
- **게시는 사람이 한다.** 이 MCP는 아무 데도 올리지 않는다. 채널 `ops_rules`의 하루 한도·게시 간격은 `list_publications { channel_id, from, to }`로 확인해 사용자에게 알린다.
- 사용자가 게시했다고 알려 주면 `record_publication { content_id, work_id, url, published_at, caption, credits }`로 기록하고 `set_content_stage { stage: "published" }`로 올린다. 게시처·계정을 생략하면 채널 값이 들어간다.

## 금지

- 도구 인자에 uid, user_id, owner
- base64 이미지 수신. `resources/read`로 큰 PNG를 받지 않는다
- 로컬 storyboard 파일·로컬 엔진으로 MCP를 우회
- 사용자 확인 없이 `approve_work`
- 사용자 승인 없이 공용 라이브러리 복사(`share_resource`)·공개 저장소 동기화
- 재개 중 `create_work`로 새 작품
- 콘텐츠 없이 작품부터 만들기, `content_id` 없이 `create_work`
- 409 `storyboard_approval_required`를 받고 관문을 돌아가기(단계를 건너뛰어 그림 생성, 채널 규칙을 고쳐 관문 끄기)
- 게시하지 않은 것을 `record_publication`으로 기록
