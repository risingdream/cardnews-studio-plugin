---
name: inspect-works
description: Cardnews Studio MCP로 조직·작품·리소스·카드를 조회한다. 작품 목록, 메타, 리소스, 카드 이미지 확인을 요청하면 이 스킬을 쓴다. 조회만 하며 저장·생성은 다루지 않는다.
---

# 작품 조회

도구는 **원격 MCP** `cardnews-studio`다. 로컬 바이너리를 설치하지 않는다.
첫 호출에서 OAuth 동의 화면이 뜬다. 기본 체크는 `works:read`만 유지한다.

범위: 조직 목록 → 작품 목록 → 작품 메타 → 리소스 메타 → (필요 시) 카드 PNG.
범위 밖: 작품 저장, 리소스 생성, 카드 생성, 렌더.

## 원칙

1. uid는 도구 인자에 넣지 않는다. 서버가 토큰에서 읽는다.
2. 작품 정본은 `work_id`(UUID)다. `slug`는 활성 조직 별칭이다.
3. 이미지 바이너리를 JSON에 넣거나 base64로 받지 않는다. 픽셀 검수는 `view_media`만 쓴다.
4. 활성 조직이 아니면 `organization_id`를 명시한다.

## 흐름

```
1. list_organizations   가입 조직과 last_organization_id
2. list_works           활성 조직 작품 목록. 필요하면 organization_id
3. get_work             원문·스토리보드·승인·리소스·카드 URL. PNG 없음
4. list_resources       해당 작품 리소스 메타
5. view_media           MCP media URI의 PNG. 큰 파일은 resources/read 대신 이 도구
```

`get_work`와 `list_resources`의 이미지 필드는 `https://zent-studio.risingdream.workers.dev/mcp/resources/media/<key>` 형태다. query `token`을 붙이지 않는다.

없는 작품은 404로 끝낸다. 추측해서 다른 조직의 작품을 열지 않는다.
