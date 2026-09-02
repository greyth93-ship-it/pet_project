# Pet Care MCP — `.mcpb` 패키징

Claude Desktop 등 MCP 호스트에 [`mcp-server.sh`](../mcp-server.sh)(stdio, `mcp` 프로파일)를
더블클릭 한 번으로 등록하기 위한 [MCP Bundle](https://github.com/anthropics/mcpb) 패키지다.
루트 `CLAUDE.md` "Phase: MCP 대화형 입구 → 원격 커넥터 전환 — 단계 기획" ④ 항목.

## 산출물 구성 판단 — jar 번들 포함 대신 "로컬 경로 참조"를 택한 이유

`.mcpb`는 서버 파일을 통째로 zip에 넣어 완전히 포터블하게 만들 수도 있다(`server/` 안에
jar를 넣고 `${__dirname}` 변수로 참조). 이 프로젝트는 그렇게 하지 않고,
**`manifest.json`의 `mcp_config.command`가 저장소 안의 `mcp-server.sh`를 절대경로로 직접
가리키는 방식**을 택했다. 이유:

1. **시크릿을 다시 다룰 필요가 없다.** `mcp-server.sh`는 이미 `pet_backend`로 `cd`한 뒤
   `application.properties`의 `spring.config.import=optional:file:.env[.properties]`
   (상대경로) 메커니즘으로 `.env`를 읽는다. jar를 번들에 넣으면 이 상대경로 트릭이 깨져
   Desktop이 시크릿 값을 `mcp_config.env`나 mcpb의 `user_config`(설치 시 사용자 입력)로
   다시 주입해야 하는데, 이건 시크릿 취급 경로가 하나 더 생기는 것이라 배포 전 백엔드
   규칙("시크릿 키는 `.env`에만") 위반 위험이 커진다.
2. **`mcp-server.sh`가 이미 "어디서 실행되든 스스로 위치를 찾는" 스크립트다** —
   내부에서 `cd "$(dirname "${BASH_SOURCE[0]}")"`로 자기 위치 기준으로 이동한다. 그래서
   Desktop이 어떤 작업 디렉터리에서 이 스크립트를 실행하든 항상 올바르게 동작한다 —
   `.mcpb`의 `${__dirname}` 트릭과 동일한 문제를 이미 해결해 둔 상태라, 굳이 jar를
   다시 패키징할 이유가 없다.
3. **용도가 개인·팀용이다** (루트 CLAUDE.md: "MCP 서버는 사용자가 자기 LLM 앱에 등록해야
   동작한다 → 대중 채널이 아니다"). 이 저장소를 이미 클론해 둔 팀원 대상이므로, 완전
   포터블한 원클릭 배포(불특정 다수용)보다 "저장소 경로만 맞으면 바로 되는" 가벼운 방식이
   실익 대비 구현 위험이 낮다.

**트레이드오프**: `manifest.json`의 `mcp_config.command`가 절대경로
(`/Users/guyeongju/team/pet_project/pet_backend/mcp-server.sh`)로 고정돼 있어, 클론 위치가
다른 팀원은 설치 전에 그 경로를 자신의 로컬 클론 경로로 고쳐야 한다(아래 설치법 참고).
완전 자동 설치가 필요해지면 후속으로 "jar 번들 + mcpb `user_config`로 시크릿 프롬프트"
전환을 검토할 것(지금은 하지 않음 — 위 1번 이유).

## 설치법

1. (처음 한 번) `cd pet_backend && ./gradlew bootJar` — `mcp-server.sh`가 실행할 jar를 만든다.
2. 이 저장소를 클론한 경로가 `manifest.json`의 `server.mcp_config.command`와 다르면, 그
   값을 자신의 로컬 절대경로(`.../pet_backend/mcp-server.sh`)로 수정한다.
3. `pet_backend/mcpb/`에서 패키징:
   ```bash
   npx @anthropic-ai/mcpb validate manifest.json
   npx @anthropic-ai/mcpb pack . pet-care.mcpb
   ```
4. 생성된 `pet-care.mcpb`를 더블클릭(또는 Claude Desktop의 확장 설치 화면에 드래그)해
   설치한다 — config 파일을 손으로 편집할 필요 없이 이 한 번의 클릭으로 등록된다.
5. Claude Desktop에서 "pet-care" MCP 서버가 연결됐는지 확인한다.

`.mcp.json`(Claude Code용, 저장소 루트)과 이 `.mcpb`(Claude Desktop용)는 같은
`mcp-server.sh`를 실행 방식만 다르게 등록하는 것 — 서버 코드나 도구는 완전히 동일하다.
