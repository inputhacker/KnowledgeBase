#openclaw-zeroclaw
이 디렉토리는 agentic workflow 관점에서 openclaw, zeroclaw 각각의 요소들을 살펴보고,
어떠한 요소들이 openclaw와 zeroclaw로 하여금 각각 어떠한 특징을 가지게 하는지 분석하기
위한 목적을 위해 생성되었다.

openclaw를 이루는 요소들(agent, gateway, channel 등)과 md 파일들(AGENT.md, SOUL.md, MEMORY.md등), tools, skills 등에 대하여 각각의 역할과 특징을 분석하고 정리한다.
특히, agent 측면에서는 LLM에게 system instruction, tools information, user input등이 어떻게 전달되는지에 대한 깊이있는 분석이 필요하다. 특히, 사용자의 일반 query를 LLM에 보내는 경우와 skills나 tools을 사용해야 하는 경우 각각에 대하여 system instruction이 어떻게 달라지는지에 대한 내용을 포함해. 그리고 skills를 포함한 tools가 많은 경우 사용자의 쿼리와 연계된 tools를 openclaw내에서 의도적으로 tools filtering을 하여 tools의 일부를 선정하도록 하는 작업들이 수행하고 있는지 여부 등도 함께 포함해줘.

zeroclaw를 이루는 요소들(agent, gateway, channel 등)과 md 파일들(AGENT.md, SOUL.md, MEMORY.md등), tools, skills 등에 대하여 각각의 역할과 특징을 분석하고 정리한다.
특히, agent 측면에서는 LLM에게 system instruction, tools information, user input등이 어떻게 전달되는지에 대한 깊이있는 분석이 필요하다. 특히, 사용자의 일반 query를 LLM에 보내는 경우와 skills나 tools을 사용해야 하는 경우 각각에 대하여 system instruction이 어떻게 달라지는지에 대한 내용을 포함해. 그리고 skills를 포함한 tools가 많은 경우 사용자의 쿼리와 연계된 tools를 zeroclaw내에서 의도적으로 tools filtering을 하여 tools의 일부를 선정하도록 하는 작업들이 수행하고 있는지 여부 등도 함께 포함해줘.

## 분석 결과 문서

- `openclaw 분석`: [`openclaw-analysis.md`](./openclaw-analysis.md)
- `zeroclaw 분석`: [`zeroclaw-analysis.md`](./zeroclaw-analysis.md)
