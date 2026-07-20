---
name: tool-matcher
description: Explicitly invoked router that inventories every skill and plugin/connector currently available to Claude (installed Skills, connected MCP tools like Slack/Gmail/Google Drive/Calendar, and Claude Code/Cowork plugins) and matches the best one(s) to whatever task the user describes, before proceeding with the task. Use ONLY when the user calls it by name with "/match" (e.g. "/match 이 엑셀 표 정리해줘"), or explicitly asks Claude to figure out which skill or tool fits a request ("어떤 스킬 써야해?", "이 작업에 맞는 도구 찾아줘", "무슨 플러그인 써야하지"). Do NOT trigger this on ordinary requests — Claude already auto-triggers the right skill/tool most of the time, and running this inventory-and-match process on every message would be redundant and slow. This skill is only for the moments the user wants that matching decision made explicit, visible, and (if ambiguous) put to a choice before Claude starts working.
---

# Tool Matcher

사용자가 `/match <작업 내용>` 형태로 부르거나, "어떤 스킬 써야해?" 같은 질문을 직접 던졌을 때만 작동하는 스킬. 목적은 지금 이 세션에서 실제로 쓸 수 있는 스킬/플러그인을 훑어보고, 사용자의 요청에 제일 잘 맞는 걸 골라서 알려주고 (필요하면 선택지를 주고), 그 다음 실제로 작업을 진행하는 것.

**중요:** 매칭만 하고 끝내지 않는다. 사용자가 순수하게 "뭐 써야해?"만 물어본 게 아니라 실제 작업 요청이 붙어있다면 (`/match 이 파일 정리해서 슬랙에 올려줘` 같은 경우), 매칭 결과를 알려준 뒤 그 스킬/플러그인을 실제로 써서 작업까지 끝낸다.

## 1단계 — 지금 쓸 수 있는 도구 목록 확인

두 가지를 모두 확인한다:

- **설치된 스킬**: 컨텍스트의 `<available_skills>` 블록 (name + description 전체). 이게 이번 세션에서 실제로 쓸 수 있는 스킬 전체 목록이다.
- **연결된 플러그인**: 두 종류가 있을 수 있음
  - MCP 커넥터 (Slack, Gmail, Google Drive, Google Calendar, k-law 등) — 이미 연결되어 바로 호출 가능한 것도 있고, `tool_search`로 찾아야 로드되는 deferred 도구도 있다. 시스템 프롬프트나 도구 목록에 나열된 것들을 확인.
  - Claude Code/Cowork 플러그인 — 해당 환경에서 플러그인 관리 스킬(`cowork-plugin-management` 등)이나 설치된 플러그인 목록이 있으면 같이 확인.
- 요청에 도움이 될 만한데 아직 연결 안 된 커넥터가 있다면 (`search_mcp_registry`로 찾을 수 있는 것), 그것도 "연결하면 쓸 수 있는 옵션"으로 후보에 넣되, 연결 전제라는 걸 분명히 한다.

이 스킬 자신(`tool-matcher`)은 후보 목록에서 제외한다 (자기 자신을 추천하는 건 의미 없음).

## 2단계 — 요청과 매칭

사용자가 `/match` 뒤에 붙인 실제 작업 내용(또는 질문 맥락)을 보고:

- 어떤 스킬의 description이 이 작업의 실질적인 내용(파일 종류, 도메인, 산출물 형태)과 맞아떨어지는지
- 이 작업을 하려면 어떤 플러그인/커넥터로 외부 데이터를 가져오거나 액션을 실행해야 하는지

를 따진다. 스킬 하나 + 플러그인 하나가 같이 필요한 경우도 흔하다 (예: "구글드라이브에 있는 발표자료 다듬어서 슬랙에 공유해줘" → pptx 스킬 + Google Drive + Slack).

판단 시 고려할 것:
- 표면적 키워드 일치가 아니라 실제 필요 여부로 판단한다. "표 만들어줘"가 항상 스프레드시트 파일을 뜻하진 않는다 — 채팅에 보여줄 표일 수도 있다.
- 산출물 형식이 명확하면 (docx/pptx/xlsx/pdf 등) 그에 맞는 스킬을 우선한다.
- "우리 회사", "인생푸드", "삼대미역" 같은 사내 맥락이 언급되면 사내 문서 형식(기안서 등)이나 사내 커넥터(Slack, Drive 등) 필요 가능성을 염두에 둔다.
- 여러 스킬이 겹치는 영역을 다룰 수 있어도, 이번 요청에 실제로 필요한 건 보통 하나다 — 억지로 여러 개 끼워맞추지 않는다.

## 3단계 — 결과 제시

**후보가 하나로 명확할 때:**
어떤 스킬/플러그인을 쓸지와 이유를 한두 줄로 짧게 말하고, 확인 기다리지 않고 바로 작업 진행. 예: "이건 `xlsx` 스킬 쓰면 되겠네요 — 바로 진행할게요." 그리고 실제로 진행한다.

**후보가 여러 개 그럴듯할 때:**
`ask_user_input_v0` 툴로 후보들을 선택지로 제시하고 사용자가 고르게 한다. 각 후보가 뭘 다르게 하는지 짧게 설명 (예: "A안: xlsx 스킬로 표만 정리 / B안: xlsx 스킬 + Slack 커넥터로 정리 후 채널에 바로 공유"). 사용자 선택을 기다린 다음 진행한다.

**딱 맞는 게 없을 때:**
그냥 솔직하게 말한다 ("설치된 스킬/플러그인 중엔 딱 맞는 게 없어서 그냥 일반적인 방식으로 진행할게요") 그리고 평소처럼 작업을 진행한다.

## 4단계 — 실제 작업 수행

매칭이 끝나면 (자동으로 결정됐든 사용자가 골랐든), 실제로 그 스킬의 SKILL.md를 읽고 필요한 도구를 호출해서 작업을 끝낸다. 사용자가 정말로 "뭐 써야해?"만 물어본 거라 작업 내용 자체가 없는 경우에만 매칭 결과 설명에서 멈춘다.
