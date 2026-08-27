# 보안 정책

이 저장소는 Cardnews Studio 플러그인 매니페스트·스킬·원격 MCP URL만 담는다.
인증 비밀·토큰·쿠키·API 키는 패키지에 없다.

## 제보

취약점은 GitHub Private vulnerability reporting 또는 저장소 소유자에게 비공개로 전달한다.
공개 이슈에 exploit, 토큰, 개인정보, 실제 작품 이미지를 넣지 않는다.

## 이 패키지가 담지 않는 것

- OAuth access/refresh token, client secret
- Firebase 세션, 서비스 계정 키
- `OPENROUTER_API_KEY` 및 모든 API 키
- localhost·내부 URL, Wrangler 환경 이름, D1/R2 식별자
- 서비스 소스와 실제 작품 PNG

설치 후 클라이언트는 사용자가 동의한 MCP OAuth로 라이브 오리진에만 붙는다.
토큰은 클라이언트와 서버가 관리하며, 이 저장소로 돌아오지 않는다.

MCP 위협 모델의 정본은 비공개 서비스 문서다. 공개 보안 정책은 보고 경로와 “비밀을 패키지에 넣지 않는다”만 적는다.
