# Codex Agentic Loop Analysis

## 1. Scope and evidence

이 문서는 공개된 공식 문서, 공개 소스코드, 현재 실행 중인 Codex 세션에서 관찰 가능한 정보, 로컬 Codex 설정/로그를 합쳐 정리한 분석이다.

- 공식 문서
  - Codex CLI: <https://developers.openai.com/codex/cli>
  - Codex CLI features: <https://developers.openai.com/codex/cli/features>
  - Advanced config: <https://developers.openai.com/codex/config-advanced>
  - AGENTS.md guide: <https://developers.openai.com/codex/guides/agents-md>
  - App server: <https://developers.openai.com/codex/app-server>
  - Open-source overview: <https://developers.openai.com/codex/open-source>
  - Workflows: <https://developers.openai.com/codex/workflows>
  - Conversation state: <https://developers.openai.com/api/docs/guides/conversation-state>
  - Compaction: <https://developers.openai.com/api/docs/guides/compaction>
- 공개 소스코드
  - `openai/codex` 저장소: <https://github.com/openai/codex>
- 현재 환경에서 확인한 로컬 자료
  - `~/.codex/config.toml`
  - `~/.codex/log/codex-tui.log`
  - `~/.codex/state_5.sqlite`
  - 현재 세션에 실제로 노출된 system/developer/tool 정의

주의:

- 아래 내용 중 "확인됨"은 공식 문서/공개 코드/현재 세션에서 직접 확인한 것들이다.
- "추론"은 공개 코드 구조와 런타임 관찰을 바탕으로 한 합리적 추정이다.
- OpenAI 내부의 비공개 서버 배치, 모델 서빙 내부 구현, 완전한 프롬프트 조립 정책 전체는 공개되지 않았으므로 그 부분은 단정하지 않았다.

## 2. 한 줄 요약

Codex의 agentic loop는 단순히 "사용자 입력 1회 -> 모델 응답 1회"가 아니라, `세션 상태 관리 -> 컨텍스트/지침 재구성 -> 모델 스트리밍 호출 -> tool call 실행 -> 결과를 히스토리에 주입 -> 필요시 추가 모델 호출 -> 종료 또는 다음 pending input 처리`를 반복하는 다중 단계 오케스트레이션 루프다.

## 3. 전체 구조

큰 그림은 아래와 같다.

1. 클라이언트가 thread/turn을 시작한다.
2. Codex 세션은 현재 turn의 실행 컨텍스트를 만든다.
3. base instructions, developer instructions, AGENTS.md, 환경 정보, skill 주입, 대화 히스토리, 사용자 입력을 조립한다.
4. 현재 turn에 노출할 tool 스펙 목록을 결정한다.
5. 모델 API에 스트리밍 요청을 보낸다.
6. 모델이 일반 텍스트를 계속 내보내면 그대로 사용자에게 스트리밍한다.
7. 모델이 tool call을 내보내면 Codex가 이를 실행한다.
8. tool 결과를 다시 conversation item으로 기록하고, 같은 turn 안에서 후속 모델 호출을 이어간다.
9. 더 이상 tool call이나 추가 입력이 없으면 turn을 종료한다.
10. turn 도중 새 pending input이 생기면 같은 task 루프에서 다음 `run_turn`을 이어서 돌린다.

공개 코드에서 `RegularTask::run()`은 `run_turn(...)`을 한 번 수행한 뒤, `has_pending_input()`이 있으면 다시 루프를 돈다. 즉 turn task 바깥에도 "pending input 처리"용 반복이 있다.

## 4. Agentic loop의 실제 구성

### 4.1 세션 계층

공개 코드의 `Session`은 단순 채팅 객체가 아니라 아래 상태를 가진 실행 관리자다.

- conversation/thread id
- event 송신 채널
- 현재 active turn
- mailbox / idle pending input
- guardian review 세션 관리자
- service 묶음
- feature set
- realtime conversation manager

즉 Codex는 "프롬프트 한 번 보내는 래퍼"가 아니라, 세션 상태와 turn 상태를 별도로 유지하는 에이전트 런타임이다.

### 4.2 turn 계층

`TurnContext`에는 현재 turn에 필요한 거의 모든 실행 파라미터가 있다.

- 모델 정보와 reasoning effort
- provider
- 현재 작업 디렉터리, 날짜, 타임존
- developer instructions
- user instructions
- collaboration mode
- personality
- approval policy / sandbox policy / network policy
- tool config
- dynamic tools
- output schema
- skill context

이 구조만 봐도 "매번 모델에 보내는 프롬프트"는 단순한 대화 로그가 아니라, turn-specific 실행 설정의 함수라는 점이 드러난다.

### 4.3 turn 시작 전 준비

turn이 시작되면 Codex는 보통 다음을 수행한다.

1. startup prewarm된 client session이 있으면 재사용한다.
2. 현재 turn의 context snapshot을 만들고, 이전 turn 대비 diff를 계산한다.
3. full context를 다시 넣을지, 설정 diff만 넣을지 결정한다.
4. skills/plugins/apps/environment/permissions 등 추가 컨텍스트를 붙인다.
5. 이번 turn에서 모델에게 보여줄 tools를 만든다.
6. 모델 샘플링을 시작한다.

공개 코드상 `record_context_updates_and_set_reference_context_item()`은 다음 정책을 가진다.

- baseline reference context가 없으면 full initial context를 주입
- 있으면 settings diff만 추가

이것은 컨텍스트 토큰을 아끼기 위한 핵심 설계다.

### 4.4 모델 샘플링 루프

`run_sampling_request()`는 아래 흐름을 가진다.

1. `built_tools(...)`로 tool router 구성
2. base instructions 확보
3. `build_prompt(...)`로 prompt 생성
4. tool runtime 생성
5. code mode worker 시작
6. `try_run_sampling_request(...)` 실행
7. 스트리밍 오류가 나면 retry 또는 transport fallback 수행

로컬 로그에서도 `session_loop`, `submission_dispatch`, `model_client.stream_responses_websocket`, `ToolCall:` 등이 보였다. 즉 실제 런타임도 한 user turn 내부에서 여러 번의 내부 왕복과 상태 전이를 가진다.

## 5. loop 안에서 실제 수행하는 일

Codex가 loop에서 하는 일은 크게 다섯 종류다.

### 5.1 모델 입력 구성

- base/system 성격의 instructions 결정
- developer message 묶음 구성
- contextual user message 묶음 구성
- 대화 히스토리 정규화
- 사용자 입력 item 추가
- tool 결과 item 추가
- output schema / personality / parallel tool call 여부 설정

### 5.2 tool orchestration

공개 코드 `tools/orchestrator.rs`의 설명 그대로, tool 호출은 대체로 다음 순서를 따른다.

1. approval 판단
2. sandbox 선택
3. 1차 실행
4. 필요시 escalation/retry
5. network approval 후처리

즉 tool call은 그냥 subprocess 실행이 아니라 정책 엔진을 거친다.

### 5.3 대화 상태 관리

- 이전 history 유지
- turn context baseline 유지
- full context 대신 diff만 추가
- 필요시 compaction
- rollout item 기록
- thread/subagent 관계 기록

로컬 `state_5.sqlite`에는 `threads`, `thread_dynamic_tools`, `thread_spawn_edges`, `agent_jobs` 같은 테이블이 있다. 즉 Codex는 대화 자체뿐 아니라 에이전트 실행 상태도 저장한다.

### 5.4 skills/plugins/apps 처리

- 사용자가 명시적으로 skill을 언급했는지 탐지
- implicit invocation 가능 skill 목록 계산
- 필요한 skill 내용을 conversation item으로 주입
- plugin/app section을 developer context에 반영
- MCP/app tool 노출 여부 결정

중요한 점은 skill은 "tool 그 자체"라기보다 "추가 instruction bundle + 보조 자산/스크립트 진입점"에 가깝다는 것이다.

### 5.5 안전성/품질 보강

- sandbox
- approval
- guardian reviewer
- stream retry
- websocket -> HTTPS fallback
- context compaction
- subagent 분기

이것들이 없으면 모델이 tool을 잘 골라도 실제 실행 결과는 불안정해진다.

## 6. 매 turn마다 LLM에 무엇이 전달되는가

공개 코드 `Prompt` 구조체는 핵심 필드를 아래처럼 가진다.

- `input: Vec<ResponseItem>`
- `tools: Vec<ToolSpec>`
- `parallel_tool_calls: bool`
- `base_instructions`
- `personality`
- `output_schema`

즉 개념적으로는 아래와 비슷하다.

```json
{
  "model": "gpt-5.4",
  "instructions": "<base instructions>",
  "input": [
    "<developer message bundle>",
    "<contextual user bundle>",
    "<prior conversation items>",
    "<current user input>",
    "<tool call outputs>"
  ],
  "tools": ["...tool schemas..."],
  "parallel_tool_calls": true,
  "personality": "pragmatic",
  "output_schema": null
}
```

여기서 핵심은 `input`이 단순 메시지 배열이 아니라 `ResponseItem` 배열이라는 점이다. 여기에 들어갈 수 있는 것은 일반 메시지뿐 아니라 function call, function call output, local shell call, tool search call 같은 구조화 item들이다.

### 6.1 base/system instruction

Codex에는 "하나의 고정 system prompt"만 있는 것이 아니다.

구분하면 다음과 같다.

1. base instructions
   - 세션 수준의 기본 모델 지침
   - config의 `base_instructions` 또는 `model_instructions_file`에서 바뀔 수 있음
2. developer instructions
   - 정책, collaboration mode, permissions, apps, skills summary, plugins summary 등이 합쳐짐
3. contextual user instructions
   - AGENTS.md, 환경 컨텍스트, 일부 skill 내용 등
4. 현재 user message
   - 사용자의 실제 질문/요청

즉 "system instruction"이라고 뭉뚱그리기보다, 여러 레이어의 instruction/context를 조합한다고 보는 편이 정확하다.

### 6.2 AGENTS.md와 user instructions

공개 코드상 AGENTS.md는 top-level system prompt가 아니라 특별한 wrapper를 가진 `user` 역할 메시지로 주입된다.

- `instructions/src/user_instructions.rs`의 `UserInstructions`
- 직렬화 형식은 `AGENTS.md instructions for <directory> ... <INSTRUCTIONS> ...`
- `From<UserInstructions> for ResponseItem`은 이것을 `ResponseItem::Message(role="user")`로 만든다

또한 project root에서 현재 cwd까지의 경로에서 여러 `AGENTS.md`를 수집해서 이어 붙이는 로직이 있다. 즉 persona/working style을 markdown 파일로 주는 방식은 실제로 존재한다.

추가로 `~/.codex` 아래의 `AGENTS.md` 또는 `AGENTS.override.md`도 `load_instructions()`를 통해 user instructions 소스로 읽힐 수 있다.

### 6.3 skill injection

skill도 마찬가지로 필요할 때 별도 wrapper를 가진 message item으로 주입된다.

- skill summary 목록은 developer context 쪽에 들어갈 수 있음
- 특정 skill의 본문은 explicit/implicit invocation 시 실제 conversation item으로 주입됨
- 따라서 skill은 모든 turn마다 전체가 다 들어가는 구조가 아니라, 보통 "설명 목록은 상시", "본문은 필요 시"에 가깝다

## 7. system instruction은 매번 바뀌는가

짧게 답하면 "일부는 고정, 일부는 turn마다 달라진다"가 맞다.

### 상대적으로 고정적인 것

- 세션의 base instructions
- 모델 성격에 가까운 기본 personality
- product-level core behavioral prompt
- tool 정의의 큰 틀

### turn마다 바뀌기 쉬운 것

- cwd, date, timezone
- approval/sandbox/network 정책
- collaboration mode 파생 지침
- apps/skills/plugins 노출 상태
- explicit skill 주입
- model switch 메시지
- environment context
- output schema

그리고 Codex는 매번 full context를 다시 통째로 보내기보다, baseline이 있으면 settings diff만 추가하려고 한다. 따라서 "항상 같은 system prompt를 매번 통으로 재전송"하는 구조라고 보는 것은 부정확하다.

## 8. tools 목록은 매번 제한적으로 선별되는가

질문에 대한 정확한 답은 "부분적으로 그렇지만, 단순 semantic shortlist는 아니다"이다.

### 확인된 제한 방식

1. mode/channel 제한
   - 현재 세션만 봐도 `web`은 `analysis` 채널, 개발자 tools는 `commentary` 채널처럼 노출 범위가 나뉜다
2. dynamic tool defer loading
   - `defer_loading=true`인 dynamic tool은 처음부터 모델에 노출하지 않음
3. code mode filtering
   - code mode nested tool은 model-visible list에서 숨길 수 있음
4. app tool direct exposure threshold
   - app tool이 많으면 전부 직접 노출하지 않고 discovery 경로로 우회
5. explicit connector selection
   - 명시적으로 enable된 connector/app만 노출
6. config/feature gating
   - 기능 플래그, 인증 상태, provider capability에 따라 tool surface가 달라짐

### 중요한 반례

공개 코드와 현재 세션만 보면, "사용자 질문 의미를 분석해서 유관 tool 몇 개만 골라 매 turn마다 model에 보내는" 일반화된 보편 정책이 핵심은 아니다. 실제로는 다음이 더 가깝다.

- 고정된 built-in tool 집합이 있음
- 여기에 runtime policy/config/dynamic loading/app exposure 규칙이 덧붙음
- skill은 별도 instruction injection으로 처리됨
- 일부 앱/MCP/dynamic tool만 지연 로딩 또는 직접 노출 제한이 있음

즉 tool surface를 무조건 semantic RAG처럼 매번 재검색해서 축소한다기보다, "정책 기반 필터링 + 필요시 지연 주입" 설계다.

## 9. tool call 외에 Codex와 LLM 사이에 다른 roundtrip이 있는가

있다. 적어도 아래 종류는 확인되거나 강하게 시사된다.

### 9.1 모델 스트리밍 재시도와 transport fallback

공개 코드상 stream 실패 시 retry하고, 예산을 넘기면 websocket에서 HTTPS transport로 fallback할 수 있다.

### 9.2 startup prewarm

`RegularTask::run()`은 startup prewarm된 client session을 소비할 수 있다. 즉 실제 user-visible turn 전에 연결/세션 준비가 선행될 수 있다.

### 9.3 conversation state continuation

공개 코드의 client 레이어는 session lifetime의 `ModelClient`와 turn lifetime의 `ModelClientSession`을 분리한다. 그리고 websocket 연결 재사용, `previous_response_id`, `conversation_id`, turn state header 같은 메커니즘을 사용한다. 이것은 단순 stateless completion 호출과 다르다.

### 9.4 context compaction

컨텍스트가 커지면 compaction API나 summarization prompt를 통해 history를 압축할 수 있다. 공식 API 문서도 conversation state/compaction을 별도로 설명한다.

### 9.5 approvals / guardian / user elicitation

모델이 도구를 부르면 바로 실행하는 것이 아니라, approval reviewer나 guardian이 사이에 끼는 roundtrip이 있을 수 있다.

### 9.6 subagent spawning

현재 세션 도구 정의만 봐도 `spawn_agent`, `wait_agent`, `send_input`, `close_agent`가 있다. 로컬 state DB에도 parent-child thread 관계가 기록된다. 즉 agentic loop는 단일 모델 호출 루프를 넘어 하위 agent orchestration까지 포함한다.

## 10. 이런 흐름이 왜 좋은 결과를 내는가

원리는 비교적 명확하다.

### 10.1 instruction layering

안정적인 지침과 현재 turn의 상황 정보를 분리하면, 모델이 장기적인 행동 규칙과 일시적인 실행 조건을 동시에 더 잘 따를 수 있다.

### 10.2 structured history

tool call, tool output, shell output, user message가 구조화 item으로 남기 때문에 모델이 "무슨 일이 실제로 일어났는지"를 덜 헷갈린다.

### 10.3 controlled tool surface

tool을 무제한 전부 노출하지 않고 정책적으로 제한하면, 잘못된 도구 선택과 프롬프트 오염을 줄일 수 있다.

### 10.4 iterative execution

한 번에 답을 끝내지 않고 `생각 -> 실행 -> 관찰 -> 후속 호출`을 반복하므로, 실제 코딩/조사 작업처럼 환경 피드백을 반영할 수 있다.

### 10.5 safety + retry

sandbox, approval, guardian, transport retry가 있으면 실패 비용을 줄이면서 더 공격적으로 자동화할 수 있다.

### 10.6 context economy

full context 대신 diff 주입과 compaction을 쓰면, 긴 세션에서도 더 많은 유효 작업 맥락을 보존할 수 있다.

## 11. client/server 구조

여기서 "server"는 최소 두 층으로 나눠 봐야 한다.

### 11.1 로컬 Codex app-server

공식 app-server 문서와 공개 코드를 보면 `codex app-server`는 JSON-RPC 기반의 로컬 인터페이스다.

- transport
  - `stdio://` 기본
  - `ws://IP:PORT` 실험적
- core primitive
  - thread
  - turn
  - item
- 주요 API
  - `thread/start`, `thread/resume`, `thread/fork`
  - `turn/start`, `turn/interrupt`
  - `skills/list`, `plugin/list`, `app/list`
  - `fs/readFile`, `fs/writeFile`
  - `command/exec`

즉 rich client가 Codex를 제어하기 위한 로컬 프로토콜 서버다.

### 11.2 클라이언트의 역할

클라이언트는 보통 아래 역할을 맡는다.

- 사용자 입력 수집
- 이벤트 스트림 표시
- turn 시작/중단
- 파일/명령/승인 UI 제공
- skill/plugin/app 목록 조회
- 필요시 remote app-server 접속

CLI/TUI, IDE extension, 데스크톱 앱 등이 이 범주에 들어간다.

### 11.3 server는 어디서 도는가

공개 정보 기준으로는 보통 "Codex가 실행되는 머신"에서 로컬 프로세스로 돈다고 보는 것이 맞다.

- 기본 transport가 `stdio://`
- websocket 리슨도 로컬/원격 모두 가능
- `--remote <ws://...|wss://...>` 옵션도 있음

따라서 일반적인 로컬 CLI 사용에서는 app-server 성격의 런타임이 로컬에서 돌고, remote 구성을 쓰면 별도 머신의 Codex app-server에 붙는 형태도 가능하다.

### 11.4 MCP와의 관계

공개 README에는 Codex CLI가 MCP client로 동작하고, 동시에 MCP server로도 실행할 수 있다고 나온다. 즉 Codex는 자체 app-server 계층과 별개로 MCP 생태계와도 연결된다.

## 12. gpt-5.4 API layer와 실제 LLM layer는 어떻게 보아야 하나

공개 코드와 로그 기준으로 보면 경계는 대략 아래와 같다.

1. Codex client/runtime
   - 세션/turn 관리
   - prompt 조립
   - tool orchestration
   - state 저장
2. Model API layer
   - Responses API 요청 송신
   - websocket/HTTPS transport
   - auth, conversation id, previous response id, turn metadata/state header
   - stream event 수신
3. 실제 모델 서빙 layer
   - `gpt-5.4` 같은 모델이 추론을 수행하는 비공개 백엔드

현재 로컬 로그에는 `responses_websocket`, `api.path="responses"`가 보였다. 즉 Codex는 이 환경에서 OpenAI Responses API를 websocket transport로 사용하고 있었다.

중요한 점:

- "API단"은 도구 스키마, conversation state, 스트리밍 이벤트, 인증, 라우팅 같은 운영 기능을 담당한다.
- "LLM layer"는 실제 토큰 생성과 함수 호출 결정을 담당한다.
- 둘 사이 경계는 논리적으로는 분명하지만, 실제 내부 마이크로서비스 분할은 공개되지 않았다.

OpenAI 서버가 어느 리전에 어떤 형태로 떠 있는지는 공개 자료만으로 특정할 수 없다. 다만 그것은 로컬 Codex app-server와는 다른 층의 OpenAI 관리 인프라다.

## 13. persona를 markdown 등으로 설정하는 부분이 있는가

있다. 다만 한 군데가 아니라 여러 경로가 있다.

### 13.1 AGENTS.md

가장 직접적인 markdown 기반 지침 메커니즘이다.

- 프로젝트 루트에서 현재 디렉터리까지 계층적으로 수집 가능
- user-context message로 주입
- 작업 스타일, 규칙, persona, 코드 컨벤션 등을 적을 수 있음

### 13.2 `~/.codex/AGENTS.md` 또는 `AGENTS.override.md`

공개 코드의 `load_instructions()`는 codex home 아래의 이 파일들도 읽는다. 즉 전역 수준의 사용자 지침/페르소나를 둘 수 있다.

### 13.3 config 기반 personality

`personality` 설정은 별도 enum/설정 항목으로 존재한다. 현재 세션의 visible developer prompt에도 personality에 해당하는 behavioral instruction이 있었다.

### 13.4 model instructions file / developer instructions

config의 `model_instructions_file`, `base_instructions`, `developer_instructions`로도 persona나 작업 태도를 강하게 규정할 수 있다.

즉 Codex에서 persona는 단일 markdown 파일 하나로만 정해지는 것이 아니라:

- markdown 문서형 지침
- config 기반 personality
- developer instruction
- collaboration mode preset

이 네 계층이 함께 작동한다고 보는 편이 맞다.

## 14. 현재 세션에서 직접 확인된 점

이 세션 자체가 Codex의 동작 방식을 일부 보여준다.

- system + developer + environment context + user message가 분리되어 있다
- tool은 namespace와 channel 단위로 정의되어 있다
- `web`은 analysis 전용
- shell/editing/subagent/MCP 관련 도구는 developer tool로 정의되어 있다
- skills는 전체 본문이 처음부터 다 주어진 것이 아니라, "목록 + 필요시 SKILL.md 로드" 구조다

즉 현재 세션만 봐도 "모든 것을 매번 평평하게 system prompt 하나에 넣는 구조"가 아니라는 점이 확인된다.

## 15. 결론

Codex의 agentic loop는 핵심적으로 다음 네 층의 결합이다.

1. instruction layering
2. structured conversation state
3. tool orchestration with policy control
4. iterative sampling with retry/continuation

질문들에 대한 짧은 답을 다시 정리하면:

- system instruction은 일부 고정이고 일부는 turn마다 달라진다.
- tools는 부분적으로 동적으로 제한되지만, 항상 semantic shortlist만 하는 구조는 아니다.
- tool call 외에도 prewarm, retry, continuation, compaction, approval, subagent 같은 roundtrip이 있다.
- client/server 구조는 로컬 app-server 계층과 OpenAI model API/backend 계층을 구분해서 봐야 한다.
- persona는 AGENTS.md, codex home instruction files, personality config, developer instructions로 설정할 수 있다.

## 16. 참고 코드 경로

아래 경로들이 특히 중요했다.

- `openai/codex/codex-rs/core/src/codex.rs`
- `openai/codex/codex-rs/core/src/client.rs`
- `openai/codex/codex-rs/core/src/client_common.rs`
- `openai/codex/codex-rs/core/src/project_doc.rs`
- `openai/codex/codex-rs/core/src/tools/router.rs`
- `openai/codex/codex-rs/core/src/tools/orchestrator.rs`
- `openai/codex/codex-rs/instructions/src/user_instructions.rs`
- `openai/codex/codex-rs/app-server/README.md`
- `openai/codex/codex-rs/codex-api/README.md`

## 17. 남는 불확실성

공개 자료만으로는 아래를 완전히 단정할 수 없다.

- OpenAI 내부 모델 서빙 인프라의 실제 배치
- 모든 제품 변형에서의 정확한 system prompt 전문
- 모든 클라이언트에서 tool exposure가 동일한지 여부
- 비공개 안전성/정책 계층의 전체 동작

따라서 이 문서는 "공개 코드와 현재 환경 관찰에 근거한 매우 구체적인 구조 분석"으로 보는 것이 가장 정확하다.
