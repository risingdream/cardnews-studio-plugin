---
name: from-topic
description: 주제로 카드뉴스 원문·카피·그림을 세 체크포인트로 만든다. from-topic, 주제에서 카드, 리서치 후 카드뉴스를 요청하면 이 스킬을 쓴다. 각 단계에서 멈추고 이어갈 수 있다.
---

# 주제에서 카드뉴스

도구는 **원격 MCP** `cardnews-studio`다. 로컬 스토리보드·로컬 엔진을 쓰지 않는다.
첫 호출에서 OAuth 동의 화면이 뜬다. 기본 체크는 `works:read` + `works:write` + `generate`를 유지한다.

범위: 리서치 → 원문 저장 → 카드 초안 검토 → 그림 생성.
이 스킬은 오케스트레이터다. 카드 구성·프롬프트 명세는 제품 도구가 강제한다.

## 원칙

1. uid는 도구 인자에 넣지 않는다. 서버가 토큰에서 읽는다.
2. 작품 정본은 `work_id`(UUID)다. `slug`는 활성 조직 별칭이다.
3. 이미지 바이너리를 JSON에 넣거나 base64로 받지 않는다. 픽셀 검수는 `view_media`만 쓴다.
4. 활성 조직이 아니면 `organization_id`를 명시한다.
5. 이미 있는 작품을 이어서 할 때 `create_work`로 새 작품을 만들지 않는다.

## 재개

주제를 받으면 새 작품을 만들기 **전에** 저장본을 읽는다.

```
1. list_organizations   가입 조직과 last_organization_id
2. list_works           같은 주제의 title·slug가 있는지
3. get_work             source, storyboard.sources, cards, approvals, asset_url
```

판정:

| 저장본 | 시작 체크포인트 |
| --- | --- |
| 작품 없음 | 1 원문. 그때만 `create_work` |
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
- 확인하지 못한 숫자는 `[확인 필요]`
- 회계법인 명의 톤. 단정 대신 요건과 예외를 병기
- 원문 1,500~3,000자(카드 7~12장 기준)
- 출처 3개 이상. 각 `{id, label, url?}`

원문은 마크다운. 문단을 나눠 뒤에 출처 id를 밝힌다. 카드 카피·이미지 프롬프트를 원문에 넣지 않는다.

저장 (MCP 스키마 그대로):

```
1. create_work    format=card-series, slug, title, sources
                  source 인자는 없다. 새 작품일 때만
2. save_work      work_id, source=원문 markdown
                  sources를 create에 못 넣었으면 여기서
```

`slug`는 소문자·숫자·하이픈만. 사용자에게 원문 요약과 출처 목록을 보여주고 **승인 후에** 체크포인트 2로 간다.

## 체크포인트 2 — 카피

```
1. analyze_source    work_id. format=card-series. 저장된 source를 쓴다
2. 초안 검토         대상 독자·훅·카드 순서. evidence의 quote가 원문과 맞는지
3. save_work         work_id, cards, citations
                     citations는 card_ids로 카드와 출처를 잇는다
4. approve_work      stage=copy. 사용자 확인 후에만
```

`analyze_source`가 `unsafe_prompt`·`evidence_mismatch`·`ungrounded_fact`를 내면 원문 또는 초안을 고치고 다시 호출한다. 카드 `id`는 저장된 카드 id만. 없는 id는 422다.

사용자에게 카드별 headline·subline·근거 한 줄을 보여주고 **승인 후에** `approve_work(copy)`를 호출한다.

## 체크포인트 3 — 그림

```
1. 프롬프트 검토     각 카드 visual.prompt가 도형 명세인지
                    shape/count/proportion/color. 한글·label/caption/title 렌더 지시 없음
                    analyze_source·save_work가 낸 unsafe_prompt를 읽고 고친다
2. create_resource  기준 인물·장소가 필요할 때만. 이어서 필요하면 generate_resource
3. generate_cards   work_id, idempotency_key 필수. job_id를 get_job으로 따라간다
                    timeout·failed면 resume_job
4. view_media       카드 asset MCP URI. 삽화 안 글자·로고를 본다
5. 재생성            문제 카드만 generate_cards에 card_ids
```

`approve_work(visual)`·`final`은 이 스킬이 자동으로 올리지 않는다. 사용자에게 문제 카드와 재생성 범위를 안내한다.

## 금지

- 도구 인자에 uid, user_id, owner
- base64 이미지 수신. `resources/read`로 큰 PNG를 받지 않는다
- 로컬 storyboard 파일·로컬 엔진으로 MCP를 우회
- 사용자 확인 없이 `approve_work`
- 재개 중 `create_work`로 새 작품
