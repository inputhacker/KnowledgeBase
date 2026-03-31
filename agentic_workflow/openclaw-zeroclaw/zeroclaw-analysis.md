# ZeroClaw 분석

## 1. 디렉토리/프로젝트의 목적

`zeroclaw`는 Rust 단일 바이너리 기반의 로컬 우선 personal assistant 런타임이다. OpenClaw와 비슷한 기능 표면을 가지지만, 내부 구현은 훨씬 직접적이고 단일 프로세스 중심적이다.

핵심 축은 다음과 같다.

- `agent`: LLM 호출, tool loop, system prompt builder
- `gateway`: 웹 대시보드/채널/세션 운영면
- `channel`: Telegram/Discord/Slack 등 메시지 입력 표면
- `tools`: built-in tool + MCP + delegate/swarm/cron
- `skills`: 워크스페이스 skill + open-skills
- `memory`: 장기 기억, daily memory, response cache

OpenClaw보다 "구성 요소가 내부 모듈로 직접 연결"되어 있고, prompt builder와 tool loop도 repo 안에서 명시적으로 보인다.

## 2. 핵심 구성요소와 역할

### agent

[`zeroclaw/src/agent/agent.rs`](zeroclaw/src/agent/agent.rs)가 중심이다.

- `AgentBuilder`가 provider, tools, memory, prompt builder, allowed_tools, skills, security summary 등을 조립한다.
- `build_system_prompt()`에서 `PromptContext`를 만들고 `SystemPromptBuilder`로 system prompt를 조립한다.
- `turn()` / `turn_streamed()`에서 history, memory context, tool loop를 수행한다.

### gateway

- gateway는 채널, 대시보드, MCP, skills, system prompt 생성을 묶는 운영 계층이다.
- `channels/mod.rs`에서 채널 런타임용 system prompt를 직접 생성한다.
- 즉 ZeroClaw는 "agent용 prompt builder"와 "channel 런타임용 prompt builder"가 병존한다.

### channel

- channel path에서는 `build_system_prompt_with_mode_and_autonomy(...)`를 통해 system prompt를 조립한다.
- 비-CLI 채널에서는 autonomy 설정, non-cli excluded tools, deferred MCP tool section 등이 반영된다.

### tools

- built-in tools는 Rust trait 객체로 등록된다.
- MCP는 eager 등록도 가능하고, deferred loading도 가능하다.
- `tool_search`를 통해 deferred MCP tool schema를 나중에 활성화할 수 있다.

### skills

- workspace `skills/`에서 `SKILL.md` 또는 `SKILL.toml`을 읽는다.
- open-skills 저장소도 opt-in으로 동기화할 수 있다.
- skill 디렉토리는 security audit를 통과해야 로드된다.

## 3. md 파일들의 역할

ZeroClaw는 md 파일을 더 노골적으로 "정체성/행동/문맥 파일"로 본다.

### personality loader 경로

[`zeroclaw/src/agent/personality.rs`](zeroclaw/src/agent/personality.rs)는 다음 파일들을 personality file로 로드한다.

- `SOUL.md`
- `IDENTITY.md`
- `USER.md`
- `AGENTS.md`
- `TOOLS.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md`
- `MEMORY.md`

이 파일들은 `IdentitySection`에서 prompt에 렌더링된다.

### channel용 bootstrap 경로

[`zeroclaw/src/channels/mod.rs`](zeroclaw/src/channels/mod.rs)의 `load_openclaw_bootstrap_files(...)`는 채널 runtime prompt에 다음 파일을 주입한다.

- 기본: `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`
- 있으면: `BOOTSTRAP.md`
- 장기 기억: `MEMORY.md`

그리고 "이미 prompt에 주입되어 있으니 file_read로 다시 읽으라고 말하지 말라"는 지시까지 같이 붙인다.

### MEMORY.md와 daily memory의 구분

ZeroClaw는 두 종류를 분리한다.

- `MEMORY.md`: curated long-term memory, prompt에 주입 가능
- `memory/*.md`: 일일 메모, 기본 prompt에는 넣지 않고 memory tool로 접근

이 분리가 코드와 온보딩 문구 둘 다에서 분명하다.

## 4. skills의 역할과 특징

### skill 로딩

[`zeroclaw/src/skills/mod.rs`](zeroclaw/src/skills/mod.rs) 기준:

- 기본 workspace `skills/`
- opt-in open-skills repo
- 보안 audit 통과 후만 로드
- `allow_scripts=false`가 기본값

즉 skill은 OpenClaw보다도 더 "감사된 로컬 capability"에 가깝다.

### prompt 주입 방식

ZeroClaw는 config로 두 가지 모드를 가진다.

- `SkillsPromptInjectionMode::Full`
- `SkillsPromptInjectionMode::Compact`

[`zeroclaw/src/skills/mod.rs`](zeroclaw/src/skills/mod.rs)의 `skills_to_prompt_with_mode(...)`가 핵심이다.

#### Full

- skill 설명
- location
- instructions
- skill tools metadata

를 system prompt에 전부 넣는다.

#### Compact

- name / description / location 중심 요약만 넣는다.
- 자세한 내용은 `read_skill(name)`로 필요 시 로드하게 한다.

### 일반 질의 vs skill 사용 질의

ZeroClaw는 여기서 OpenClaw와 차이가 크다.

- `full` 모드:
  - skill instruction이 이미 prompt 안에 있다.
  - 일반 질의든 skill 관련 질의든 prompt 구조는 거의 동일하다.
  - 모델은 이미 skill 내용을 본 상태에서 판단한다.
- `compact` 모드:
  - skill 본문은 prompt에 없다.
  - skill 관련 질의가 들어오면 `read_skill(name)`을 호출해 상세 내용을 읽어야 한다.

즉 ZeroClaw는 config에 따라 "사전주입형"과 "온디맨드형"을 모두 지원한다.

## 5. tools의 역할과 특징

### tool 등록

`AgentBuilder::build()` 시점에 `tools`가 들어오고, `allowed_tools`가 있으면 여기서 바로 잘린다.

- `allowed_tools=None`: 전체 tool 유지
- `allowed_tools=Some([...])`: 이름이 일치하는 tool만 남김

이건 단순 표시용이 아니라 실제 registry 자체를 줄이는 강한 제한이다.

### iteration마다 tool spec 재구성

[`zeroclaw/src/agent/loop_.rs`](zeroclaw/src/agent/loop_.rs)의 `run_tool_call_loop(...)`는 각 iteration마다:

- 현재 registry
- excluded_tools
- activated deferred tools

를 바탕으로 tool spec을 다시 계산한다.

즉 ZeroClaw는 실행 중 tool surface가 달라질 수 있다.

### deferred MCP

`mcp.deferred_loading=true`이면:

- 처음에는 MCP tool 전체 schema를 안 보낸다.
- 대신 `tool_search`를 등록한다.
- 필요할 때 model이 `tool_search`로 MCP tool을 검색/활성화한다.

이건 prompt 크기와 tool explosion을 줄이는 명시적 전략이다.

## 6. system instruction / tools / user input 전달 흐름

### agent path

`Agent::build_system_prompt()` 기준:

1. `tool_dispatcher.prompt_instructions(&self.tools)`
2. `PromptContext` 생성
3. `SystemPromptBuilder::with_defaults()`
4. 섹션별 조립

기본 섹션:

- DateTime
- Identity
- ToolHonesty
- Tools
- Safety
- Skills
- Workspace
- Runtime
- ChannelMedia

### channel path

`channels/mod.rs` 쪽은 별도의 procedural builder를 쓴다.

- No Tool Narration
- Tool Honesty
- Tools
- Hardware
- Your Task
- Safety
- Skills
- Bootstrap files / identity
- Date & Time
- Runtime

즉 ZeroClaw는 agent 경로와 channel 경로 모두에서 system prompt를 내부에서 직접 만들며, 그 구조가 코드로 노출되어 있다.

### user input 전달

`turn()`에서는 user message 앞에 현재 날짜/시간과 memory loader context를 붙인 뒤 history에 넣는다.

즉 ZeroClaw의 user input은:

- current date/time
- retrieved memory context
- user message

로 enrich되어 provider에 전달된다.

## 7. query 연계 tool filtering 여부

### 결론

ZeroClaw에는 query 연계 tool filtering이 실제로 존재한다. 다만 범위는 주로 MCP tool 쪽이다.

### 1) allowed_tools

- query 기반은 아니고 명시적 allowlist 기반
- tool registry 자체를 줄인다

### 2) tool_filter_groups

[`zeroclaw/src/agent/loop_.rs`](zeroclaw/src/agent/loop_.rs)의 `filter_tool_specs_for_turn(...)`가 핵심이다.

- built-in tools는 기본적으로 통과
- `mcp_` prefix 도구는 group rule에 따라 포함/제외
- `dynamic` group은 user message에 keyword가 있어야 활성화

즉 "user message와 연계된 tool filtering"이 실제 코드로 구현되어 있다.

### 3) deferred MCP + tool_search

- tool schema를 처음부터 다 주지 않고
- 필요한 경우에만 검색/활성화하게 함

이 역시 넓은 의미의 tool surface filtering이다.

### 4) context analyzer

[`zeroclaw/src/agent/context_analyzer.rs`](zeroclaw/src/agent/context_analyzer.rs)에 heuristic analyzer가 있다.

- 이전 tool call
- 직전 assistant 메시지 키워드

를 보고 suggested tools를 계산한다.

다만 이번에 확인한 주 실행 경로에서는 이 analyzer가 `run_tool_call_loop(...)`에 직접 연결되어 있는 증거는 찾지 못했다. 따라서 "존재는 하지만 현재 핵심 실행 경로에서 강하게 쓰인다고 단정하기는 어렵다"고 보는 것이 안전하다.

## 8. 일반 질의 vs skill/tool 사용 질의에서 system prompt 차이

### 일반 질의

- skill이 `full` 모드면 이미 모든 skill instruction이 prompt에 들어간 상태다.
- tool spec도 현재 허용된 범위 내에서 함께 provider에 전달된다.

### skill 사용 질의

- `full` 모드에서는 별도 prompt 변경 없이 model이 바로 skill guidance를 활용한다.
- `compact` 모드에서는 `read_skill(name)` 호출이 필요해진다.

### tool 사용 질의

- system prompt는 매 턴 재조립되거나 history 첫 system message로 유지되지만
- 실제 provider로 보내는 tool spec은 iteration별로 재계산된다.
- MCP의 경우 query keyword에 따라 일부만 노출될 수 있다.

즉 ZeroClaw는 "질의 종류에 따라 prompt 내용이 약간 달라진다"기보다,
"질의와 config에 따라 실제 tool surface가 iteration마다 달라진다"는 쪽이 더 정확하다.

## 9. 특징 요약

- ZeroClaw는 Rust 내부 모듈에서 prompt, tool loop, filtering이 매우 명시적으로 보인다.
- md 파일은 personality/bootstrap 계층으로 강하게 통합되어 있다.
- skill은 `full`/`compact` 두 모드로 주입 전략이 명확하다.
- tool filtering은 OpenClaw보다 더 직접적이며, query keyword 기반 MCP filtering도 실제 구현돼 있다.
- deferred MCP와 `tool_search`는 prompt/도구 폭발을 제어하는 중요한 설계 포인트다.
