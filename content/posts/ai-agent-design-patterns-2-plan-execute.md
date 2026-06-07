---
title: "AI Agent Design Patterns. Part 2: Plan-and-Execute Pattern"
date: 2026-06-07
draft: false
tags: ["ai", "agents", "llm", "plan-execute", "eino", "go"]
categories: ["ai", "engineering"]
summary: "Plan-and-Execute separates strategic planning from tactical execution. I break down the Planner+Executor+Reviser architecture, ReWOO and LLMCompiler variants with numbers from original papers, prompt injection defense via control-flow integrity, and a minimal working Go example with Eino."
ShowToc: true
series: ["AI Agent Design Patterns"]
---

## 1. Introduction

In [Part 1]({{< ref "ai-agent-design-patterns-1-react" >}}) I covered ReAct — a pattern where the agent interleaves reasoning and action. The problem: ReAct plans one step ahead. For tasks that need a global plan — incident analysis, multi-step debugging, service orchestration — the myopic approach breaks. Plan-and-Execute solves this by separating planning from execution.

## 2. What is Plan-and-Execute

### History

In May 2023, Wang et al. published [Plan-and-Solve Prompting](https://arxiv.org/abs/2305.04091) — a zero-shot strategy that first devises a plan to divide a task into subtasks, then carries out the subtasks. This was a direct response to three pitfalls of Zero-shot-CoT: missing-step errors, calculation errors, and semantic misunderstanding errors.

In March 2023, Shen et al. introduced [HuggingGPT](https://arxiv.org/abs/2303.17580) — an LLM-powered controller that plans a task, selects AI models from Hugging Face based on their descriptions, executes each subtask with the chosen model, and summarizes the result. This was the first production-scale implementation of the pattern.

So. The pattern crystallized: a **Planner** (strategist), an **Executor** (tactician), and an optional **Reviser** (reviewer). [LangChain](https://www.langchain.com/) — a Python framework for building LLM applications, including agent primitives, RAG pipelines, and orchestration tools — popularized this as the `Plan-and-Execute Agent` in 2024, but the idea originated in 2023.

### Team analogy

Imagine: a tech lead (Planner) decomposes a task into tickets, a developer (Executor) picks up a ticket and implements it, a QA engineer (Reviser) verifies — and if they find a problem, the ticket goes back for rework. ReAct is a solo developer who sets their own one-step task, does it, checks it, sets the next one. For simple tasks — fine. For complex ones — you need a manager.

### Formal definition

Plan-and-Execute operates in three phases:

1. **Plan**: The Planner generates a list of steps `[S₁, S₂, ..., Sₙ]` to solve the task.
2. **Execute**: The Executor executes each step Sᵢ, calling tools and receiving observations.
3. **Revise**: The Reviser evaluates progress and modifies the remaining plan if necessary.

The key difference from ReAct: planning is **global**, not step-by-step. The Planner sees the entire task and builds an N-step plan. The Executor receives one step at a time — not the full plan, not the user input. This creates a natural security boundary.

## 3. What problem it solves

### ReAct's problem: myopic planning

ReAct, as I showed in Part 1, plans one step ahead. Each Thought is a reaction to the previous Observation, not to an overall strategy. For tasks with known structure (log analysis → filtering → correlation → diagnosis), this is inefficient: the agent "wanders" instead of following a plan.

### ReAct's problem: cost

Each ReAct step is a full LLM call with the entire context (all previous Thoughts + Actions + Observations). With 10 steps — 10 calls to an expensive model. ReWOO (Xu et al., 2023) showed this is unnecessary: up to **5x** reduction in token consumption on HotpotQA with a **4.4%** accuracy improvement ([Xu et al., 2023](https://arxiv.org/abs/2305.18323)).

### ReAct's problem: no explicit re-planning

ReAct doesn't revise a plan — there simply isn't one. If a step leads to a dead end, the model "implicitly" corrects through the next Thought. But that's not re-planning — it's reactive wandering. Plan-and-Execute makes re-planning **explicit** through the Reviser.

### What Plan-and-Execute provides

| Problem | Plan-and-Execute solution |
|---------|---------------------------|
| Myopic planning | Planner sees the whole task, builds an N-step plan |
| High cost | Executor can use a cheap model — it only needs one step |
| No re-planning | Reviser explicitly evaluates progress and modifies the plan |
| Prompt injection | Executor doesn't see user input or the full plan |
| Opacity | The plan is a readable, auditable artifact |

## 4. Architecture

### Plan → Execute → Revise

{{< mermaid align="center" >}}
sequenceDiagram
    participant U as User
    participant P as Planner
    participant E as Executor
    participant T as Tools
    participant R as Reviser

    U->>P: Task
    P->>P: Generate plan [S₁, S₂, ..., Sₙ]
    loop For each step
        P->>E: Step Sᵢ
        E->>T: Call tool
        T-->>E: Observation
        E-->>R: Step result
        R->>R: Evaluate progress
        alt Plan needs revision
            R->>P: Revised plan
        else Step OK
            R->>E: Next step Sᵢ₊₁
        end
    end
    R-->>U: Final answer
{{< /mermaid >}}

The Planner receives the task and generates a plan. The Executor executes step by step, calling tools. The Reviser checks each step's result and decides: continue the plan, revise it, or finish.

### Eino State Graph

In Eino, the pattern is implemented via `compose.Graph` — a directed graph with three agent nodes:

{{< mermaid align="center" >}}
graph TB
    START((START)) --> PL[Planner<br/>BaseChatModel]
    PL --> EX[Executor<br/>ToolCallingChatModel<br/>+ ToolsNode]
    EX --> RV{Reviser<br/>BaseChatModel}
    RV -->|needs revision| PL
    RV -->|step OK, more steps| EX
    RV -->|all done| END((END))
{{< /mermaid >}}

The state struct stores: messages (history), current plan (steps), step number. The `maxStep` parameter limits iterations — protection against infinite loops.

### Pattern evolution

{{< mermaid align="center" >}}
graph LR
    C["CoT<br/>Wei 2022"] --> R["ReAct<br/>Yao 2022"]
    R --> PS["Plan-and-Solve<br/>Wang 2023"]
    PS --> PA["Plan-and-Execute<br/>LangChain 2024"]
    PA --> RW["ReWOO<br/>Xu 2023"]
    PA --> LC["LLMCompiler<br/>Kim 2023"]
{{< /mermaid >}}

CoT (Chain-of-Thought, [Wei et al., 2022](https://arxiv.org/abs/2201.11903)) gave step-by-step reasoning without actions. ReAct ([Yao et al., 2022](https://arxiv.org/abs/2210.03629)) added actions. Plan-and-Solve ([Wang et al., 2023](https://arxiv.org/abs/2305.04091)) introduced explicit planning. Plan-and-Execute formalized this as an architectural pattern with separate agents. ReWOO and LLMCompiler are optimizations of the base pattern.

## 5. Variants

### ReWOO: Reasoning Without Observation

ReWOO ([Xu et al., 2023](https://arxiv.org/abs/2305.18323)) eliminates interleaved LLM calls. The Planner generates a plan with variables once, and the Executor substitutes results — **without calling the LLM at each step**. Result: **5x token reduction** and **+4.4% accuracy** on HotpotQA. An additional bonus: you can distill the Planner from 175B (GPT-3.5) to 7B (LLaMA) — and it works.

Here's how it looks on our log analysis scenario:

{{< mermaid align="center" >}}
graph LR
    subgraph "1. Planner (LLM, 1 call)"
        P["Plan:<br/>#E1 = query_logs('error spike')<br/>#E2 = filter_by_time(#E1, 'last hour')<br/>#E3 = correlate_deployments(#E1)<br/>#E4 = suggest_fix(#E2, #E3)"]
    end
    subgraph "2. Executor (no LLM)"
        E1["#E1 → query_logs<br/>→ result into #E1"]
        E2["#E2 → filter_by_time(#E1)<br/>→ result into #E2"]
        E3["#E3 → correlate(#E1)<br/>→ result into #E3"]
        E4["#E4 → suggest_fix(#E2,#E3)<br/>→ final answer"]
    end
    P --> E1
    E1 --> E2
    E1 --> E3
    E2 --> E4
    E3 --> E4

    style P fill:#4a9eff,color:#fff
    style E1 fill:#2d8659,color:#fff
    style E2 fill:#2d8659,color:#fff
    style E3 fill:#2d8659,color:#fff
    style E4 fill:#2d8659,color:#fff
{{< /mermaid >}}

The key insight: the Planner uses the LLM **once** to generate the plan. Then the Executor simply calls tools and substitutes results into variables — like a template engine. The LLM is only needed for the next Planner call if the Reviser decides to re-plan.

### LLMCompiler: parallel execution

LLMCompiler ([Kim et al., 2023](https://arxiv.org/abs/2312.04511)) borrows from classical compilers: the Planner generates a DAG (Directed Acyclic Graph) of step dependencies, the Task Fetching Unit dispatches independent steps in parallel, and the Executor runs them concurrently. Result: **3.7x latency speedup**, **6.7x cost savings**, **~9% accuracy improvement** over ReAct.

On our scenario it looks like this:

{{< mermaid align="center" >}}
graph LR
    subgraph "1. Planner → DAG"
        P["Plan with dependencies:<br/>#E1 = query_logs('error')<br/>#E2 = filter_by_time(#E1)<br/>#E3 = correlate_deployments(#E1)<br/>#E4 = query_metrics('cpu')<br/>#E5 = suggest_fix(#E2, #E3, #E4)"]
    end
    subgraph "2. Parallel execution"
        E1["#E1 query_logs<br/>⏱ 200ms"]
        E2["#E2 filter_by_time<br/>⏱ 100ms"]
        E3["#E3 correlate<br/>⏱ 150ms"]
        E4["#E4 query_metrics<br/>⏱ 120ms"]
        E5["#E5 suggest_fix<br/>⏱ 300ms"]
    end
    P --> E1
    E1 --> E2
    E1 --> E3
    E1 --> E4
    E2 --> E5
    E3 --> E5
    E4 --> E5

    style P fill:#4a9eff,color:#fff
    style E1 fill:#2d8659,color:#fff
    style E2 fill:#e6a817,color:#333
    style E3 fill:#e6a817,color:#333
    style E4 fill:#e6a817,color:#333
    style E5 fill:#2d8659,color:#fff
{{< /mermaid >}}

Yellow highlights the steps that run **in parallel**: after #E1 (query_logs) completes, steps #E2, #E3, and #E4 launch simultaneously — each depends only on #E1, not on each other. Total time: 200ms + max(100, 150, 120) + 300ms = **650ms** instead of sequential 870ms. With dozens of steps in real tasks, the gain grows up to 3.7x.

### HuggingGPT: LLM as controller

HuggingGPT ([Shen et al., 2023](https://arxiv.org/abs/2303.17580)) is a specific implementation: ChatGPT plans the task, selects AI models from Hugging Face based on their descriptions, executes each subtask with the chosen model, and summarizes the result. Not a separate variant, but an illustration of the pattern's applicability to real systems.

{{< mermaid align="center" >}}
graph LR
    U["User:<br/>Generate an image<br/>and describe it with voice"]
    P["ChatGPT<br/>(Planner)"]
    M1["Stable Diffusion<br/>(text-to-image)"]
    M2["Whisper<br/>(speech-to-text)"]
    M3["Bark<br/>(text-to-speech)"]
    S["ChatGPT<br/>(Summarizer)"]

    U --> P
    P -->|"1. image generation"| M1
    P -->|"2. describe image"| M2
    P -->|"3. voice output"| M3
    M1 --> S
    M2 --> S
    M3 --> S
    S --> U

    style P fill:#4a9eff,color:#fff
    style M1 fill:#9b59b6,color:#fff
    style M2 fill:#9b59b6,color:#fff
    style M3 fill:#9b59b6,color:#fff
    style S fill:#4a9eff,color:#fff
    style U fill:#555,color:#fff
{{< /mermaid >}}

Purple — specialized AI models from Hugging Face. ChatGPT acts as the Planner: it parses the request, selects models based on their capability descriptions, passes results between them, and forms the final answer. Unlike ReWOO and LLMCompiler, the Executor here isn't an LLM — it's external models. But the pattern is the same: planning → execution → summarization.

Full deep-dive on ReWOO and LLMCompiler — in Part 4 of this series.

## 6. Security: Control-Flow Integrity

And here's where it gets interesting. Separating the Planner and Executor isn't just an architectural flourish. It's a defense against prompt injection (an attack where a malicious instruction is embedded in data that an LLM processes — for example, in tool output or user-supplied content).

Del Rosario et al. (2025) in ["Architecting Resilient LLM Agents"](https://arxiv.org/abs/2509.08646) formulate the principle of **control-flow integrity**: if the Executor only sees one step of the plan (not the full plan, not the user input), then an attacker controlling tool output cannot redirect the entire workflow. The blast radius is limited to one step.

Contrast with ReAct: in a ReAct agent, the full context (including user input) is available at every step. If a tool returns malicious content, the model may interpret it as a new instruction — and change behavior. In Plan-and-Execute, the Executor receives an isolated task: "call tool X with parameters Y."

Additional defenses from Del Rosario 2025:

- **Principle of Least Privilege**: each step gets the minimum set of tools
- **Task-scoped tool access**: the Executor for "filter logs" shouldn't have access to "delete records"
- **Sandboxed execution**: code generated by the Executor runs in a sandbox
- **HITL (Human-in-the-Loop)**: critical steps require human confirmation

These aren't theoretical recommendations. Del Rosario et al. provide implementation blueprints for LangGraph, CrewAI, and AutoGen — with working code.

## 7. Practice: Plan-and-Execute with Eino, Go

### Scenario: production incident analysis

An on-call engineer gets an alert: latency spike on the orders service. Need to: query logs → filter by time → correlate with deployments → query metrics → suggest fix. Five steps, known structure — a perfect Plan-and-Execute case.

### Installation

```bash
go get github.com/cloudwego/eino@latest
go get github.com/cloudwego/eino-ext/components/model/openai@latest
```

### Minimal Plan-and-Execute agent

```go
package main

import (
    "context"
    "fmt"

    "github.com/cloudwego/eino-ext/components/model/openai"
    "github.com/cloudwego/eino/components/tool"
    "github.com/cloudwego/eino/compose"
    "github.com/cloudwego/eino/flow/agent/multiagent/planexecute"
    "github.com/cloudwego/eino/schema"
)

func main() {
    ctx := context.Background()

    plannerModel, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
        Model: "gpt-4o",
    })
    executorModel, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
        Model: "gpt-4o-mini",
    })
    reviserModel, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
        Model: "gpt-4o",
    })

    agent, _ := planexecute.NewAgent(ctx, &planexecute.AgentConfig{
        Planner:   plannerModel,
        Reviser:   reviserModel,
        Executor:  executorModel,
        ToolsConfig: compose.ToolsNodeConfig{
            InvokableTools: []tool.InvokableTool{
                queryLogsTool(),
                filterByTimeTool(),
                correlateDeploymentsTool(),
                queryMetricsTool(),
                suggestFixTool(),
            },
        },
        MaxStep: 10,
    })

    msg, _ := agent.Generate(ctx, []*schema.Message{
        schema.UserMessage(
            "Latency spike on the orders service. " +
            "Find the root cause and suggest a fix.",
        ),
    })
    fmt.Println(msg.Content)
}
```

Note: **Planner and Reviser use `gpt-4o`** (strategic decisions), while **Executor uses `gpt-4o-mini`** (calling a single tool). This saves cost: a cheap model for tactical work, an expensive one for strategy. In ReAct, this optimization isn't available — every step requires a full-capability model.

Tools are defined via `utils.NewTool` — the same way as in Part 1 for ReAct. Full example with tools and prompts: [eino-examples/flow/agent/multiagent/plan_execute](https://github.com/cloudwego/eino-examples/tree/main/flow/agent/multiagent/plan_execute).

### Execution trace

Expand the blocks below to see intermediate agent outputs:

<details>
<summary><strong>Step 1: Planner — plan generation</strong></summary>

```
Task: Latency spike on the orders service. Find the root cause and suggest a fix.

Plan:
1. query_logs — query orders service logs for the last hour
2. filter_by_time — filter logs to the spike window (14:00-14:30)
3. correlate_deployments — check deployments in the same window
4. query_metrics — request CPU/memory metrics for the orders service
5. suggest_fix — based on the data, suggest a fix
```

</details>

<details>
<summary><strong>Step 2: Executor — query_logs</strong></summary>

```
Call: query_logs(service="orders", level="ERROR", limit=100)
Result: 47 ERROR records in the last hour. Main errors:
- "timeout connecting to payments service" (23 occurrences)
- "connection pool exhausted" (18 occurrences)
- "slow query: SELECT * FROM orders WHERE..." (6 occurrences)
```

</details>

<details>
<summary><strong>Step 3: Executor — filter_by_time + correlate_deployments</strong></summary>

```
Call: filter_by_time(logs=<step2_result>, start="14:00", end="14:30")
Result: Spike started at 14:12. Before 14:12 — normal error levels.

Call: correlate_deployments(service="orders", start="14:00", end="14:30")
Result: Deploy of payments-service v2.3.1 at 14:10 (2 minutes before spike).
```

</details>

<details>
<summary><strong>Step 4: Executor — query_metrics</strong></summary>

```
Call: query_metrics(service="orders", metric="cpu,memory", start="14:00", end="14:30")
Result: orders CPU — normal (45%). Memory — normal (62%).
But: payments-service latency jumped from 50ms to 2000ms starting at 14:10.
```

</details>

<details>
<summary><strong>Step 5: Reviser — validation + suggest_fix</strong></summary>

```
Reviser: Root cause found — deploy of payments-service v2.3.1 at 14:10 caused
latency increase in payments, which led to timeouts in orders. Plan complete,
no re-planning needed.

Call: suggest_fix(root_cause="payments-service v2.3.1 latency regression",
                   affected_service="orders")
Result:
1. Immediate: rollback payments-service to v2.3.0
2. Short-term: increase timeout and connection pool for orders→payments
3. Long-term: add circuit breaker between orders and payments
```

</details>

### Key parameters

- **`Planner`** — model for plan generation (`BaseChatModel` interface).
- **`Reviser`** — model for progress evaluation and re-planning (`BaseChatModel`).
- **`Executor`** — model for step execution (`ToolCallingChatModel` + `ToolsNode`).
- **`MaxStep`** — graph iteration limit (default 12). Protection against infinite loops.
- **`ToolsConfig`** — tools available to the Executor.

### Links for deeper study

- [Eino Plan-and-Execute Agent Manual](https://www.cloudwego.io/docs/eino/core_modules/eino_adk/agent_implementation/plan_execute/) — complete parameter reference
- [eino-examples/flow/agent/multiagent/plan_execute](https://github.com/cloudwego/eino-examples/tree/main/flow/agent/multiagent/plan_execute) — full working example
- [GitHub: cloudwego/eino](https://github.com/cloudwego/eino) — source code

## 8. Comparative tables

### Architecture: ReAct vs Plan-and-Execute

| Dimension | ReAct | Plan-and-Execute |
|-----------|-------|-------------------|
| Planning horizon | Myopic: 1 step | Global: N steps |
| LLM calls per step | Full model every step | Cheap Executor per step, expensive Planner once |
| Context management | Scratchpad: all Thoughts/Actions/Observations accumulate | State: plan + step results |
| Re-planning mechanism | Implicit: through next Thought | Explicit: Reviser modifies the plan |
| Error recovery | Retry in-place: model corrects on next step | Re-plan from checkpoint: Reviser rebuilds remaining plan |
| Architectural complexity | Single loop (ChatModel → ToolsNode) | Three agents (Planner → Executor → Reviser) |
| Prompt injection resilience | Low: full context available at every step | Higher: Executor sees only one step ([Del Rosario et al., 2025](https://arxiv.org/abs/2509.08646)) |

### Performance and cost

| Metric | ReAct | Plan-and-Execute (naive) | ReWOO | LLMCompiler |
|--------|-------|--------------------------|-------|-------------|
| Token consumption (vs ReAct) | 1x | ~1.2x (Planner/Reviser overhead) | **0.2x** ([Xu 2023](https://arxiv.org/abs/2305.18323)) | ~0.5x (parallel execution) |
| LLM calls | N steps × full model | 1 Planner + N × Executor + k × Reviser | 1 Planner + N × tool calls (no LLM) | 1 Planner + parallel Executors |
| Latency | Sequential: N × T_step | Sequential: T_plan + N × T_exec | **Sequential but no LLM per step** | **Parallel: ~T_plan + T_slowest_step** ([Kim 2023](https://arxiv.org/abs/2312.04511)) |
| Accuracy (HotpotQA) | Baseline | Comparable | **+4.4%** ([Xu 2023](https://arxiv.org/abs/2305.18323)) | **+~9%** vs ReAct ([Kim 2023](https://arxiv.org/abs/2312.04511)) |
| Latency speedup | 1x | ~1x | ~1x | **3.7x** ([Kim 2023](https://arxiv.org/abs/2312.04511)) |
| Cost savings | 1x | ~0.8x (cheap Executor) | **~5x** ([Xu 2023](https://arxiv.org/abs/2305.18323)) | **6.7x** ([Kim 2023](https://arxiv.org/abs/2312.04511)) |

### Plan-and-Execute variants

| Variant | Tokens vs ReAct | Speedup | Accuracy (delta) | Complexity | Key innovation |
|---------|-----------------|---------|-------------------|------------|----------------|
| **Naive** (LangChain-style) | ~1.2x | ~1x | ~0% | Low | Planning + sequential execution |
| **ReWOO** ([Xu 2023](https://arxiv.org/abs/2305.18323)) | **0.2x** (-5x) | ~1x | **+4.4%** | Medium | Variable substitution, LLM only for Planner |
| **LLMCompiler** ([Kim 2023](https://arxiv.org/abs/2312.04511)) | ~0.5x | **3.7x** | **+~9%** | High | DAG dependencies, parallel execution |

### Pattern selection guide

| Task characteristic | Recommended pattern | Example |
|--------------------|--------------------|---------|
| Simple, single API call with parsing | **ReAct** | "What's the weather in Paris?" → one weather API call |
| Complex, multi-step, known structure | **Plan-and-Execute** | Production incident analysis: logs → filter → correlate → diagnose |
| Parallelizable subtasks | **LLMCompiler** | "Compare iPhone prices across 5 stores" → 5 parallel calls |
| Cost-sensitive with fixed tool set | **ReWOO** | Regular monitoring: same step sequence, same tools |
| Requires learning from past errors | **Reflexion** | Code review: agent checks, finds bug, re-checks the fix |

## 9. When NOT to use

The Microsoft Azure Architecture Center in their [AI agent orchestration guide](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns) formulates the principle: **use the minimum level of complexity that solves the task**.

| If your task... | ...consider this alternative |
|-----------------|------------------------------|
| Is solved by a direct model call | **Direct model call** — classification, summarization |
| Needs one or two tools, steps unknown in advance | **ReAct** — as covered in Part 1 |
| Has a fixed workflow without re-planning | **ReWOO** — cheaper, no Reviser overhead |
| Requires trees of hypotheses | **Tree of Thoughts** ([Yao et al., 2023](https://arxiv.org/abs/2305.10601)) |
| Spans multiple domains, different security boundaries | **Multi-agent** — Part 5 of this series |

Plan-and-Execute is a middle complexity tier between ReAct and multi-agent. Don't jump to it if ReAct solves the task. Don't stay on it if you need multi-agent coordination.

## 10. What's next

- **Part 3: Reflexion Pattern** — ReAct + self-evaluation: the agent learns from its own mistakes through verbal reinforcement.
- **Part 4: ReWOO and LLMCompiler** — deep dive into two Plan-and-Execute optimizations: variable substitution and parallel DAG execution.
- **Part 5: Pattern Comparison + Multi-Agent Orchestration** — when to choose which pattern, and how multiple agents coordinate to solve complex tasks.

Discussion welcome — comments on the site or [GitHub Issues](https://github.com/triumphpc/blog/issues).

## 11. References

### Research papers

1. **Wang et al., 2023 — Plan-and-Solve Prompting**: Zero-shot task decomposition into subtasks. Addresses missing-step errors of Zero-shot-CoT. [arXiv:2305.04091](https://arxiv.org/abs/2305.04091)
2. **Shen et al., 2023 — HuggingGPT**: LLM as a controller orchestrating AI models from Hugging Face. First production-scale implementation of the pattern. [arXiv:2303.17580](https://arxiv.org/abs/2303.17580)
3. **Xu et al., 2023 — ReWOO**: Decoupling reasoning from observations. Variable substitution, 5x token efficiency, +4.4% accuracy on HotpotQA. 175B→7B distillation. [arXiv:2305.18323](https://arxiv.org/abs/2305.18323)
4. **Kim et al., 2023 — LLMCompiler**: Parallel function calling via DAG. 3.7x latency speedup, 6.7x cost savings, +~9% accuracy. [arXiv:2312.04511](https://arxiv.org/abs/2312.04511)
5. **Del Rosario et al., 2025 — Plan-then-Execute Security**: Control-flow integrity as prompt injection defense. Least privilege, task-scoped tool access, sandboxed execution. [arXiv:2509.08646](https://arxiv.org/abs/2509.08646)
6. **Yao et al., 2022 — ReAct**: The base Reasoning + Acting pattern from which Plan-and-Execute evolves. [arXiv:2210.03629](https://arxiv.org/abs/2210.03629)
7. **Wei et al., 2022 — Chain-of-Thought**: Step-by-step reasoning — the predecessor of all agent patterns. [arXiv:2201.11903](https://arxiv.org/abs/2201.11903)

### Documentation and examples

- [Eino Plan-and-Execute Agent Manual](https://www.cloudwego.io/docs/eino/core_modules/eino_adk/agent_implementation/plan_execute/) — complete parameter reference
- [Eino Open Source Announcement](https://www.cloudwego.io/docs/eino/overview/eino_open_source/) — framework overview
- [GitHub: cloudwego/eino](https://github.com/cloudwego/eino) — source code
- [GitHub: eino-examples/flow/agent/multiagent/plan_execute](https://github.com/cloudwego/eino-examples/tree/main/flow/agent/multiagent/plan_execute) — full working example

### Architecture guides

- [AI agent design patterns — Microsoft Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns) — complexity hierarchy, orchestration patterns, production recommendations
