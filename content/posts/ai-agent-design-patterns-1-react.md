---
title: "AI Agent Design Patterns. Part 1: ReAct Pattern"
date: 2026-06-06
draft: false
tags: ["ai", "agents", "llm", "react-pattern", "eino", "go"]
categories: ["ai", "engineering"]
summary: "ReAct (Reasoning + Acting) — the foundational AI agent design pattern that interleaves reasoning and action in a single loop. I break down the architecture, numbers from Yao et al. 2022, framework landscape, and a minimal working Go example via Eino."
ShowToc: true
series: ["AI Agent Design Patterns"]
---

## 1. Introduction

I'm starting a series on AI agent design patterns. The goal: a reference built on original research papers, working Go code, and honest limitation analysis. Every claim traces back to the source — no secondary summaries. Audience — experienced engineers who don't need basics explained.

ReAct Pattern (Yao et al., 2022) is the foundation upon which Reflexion, Plan-and-Execute, ReWOO, and all agent architectures are built. Without it, the other patterns don't form a coherent system.

## 2. What is ReAct Pattern

### History

In October 2022, a team from Princeton University and Google Brain — Shunyu Yao, Jeffrey Zhao, Dian Yu, Nan Du, Izhak Shafran, Karthik Narasimhan, Yuan Cao — published [«ReAct: Synergizing Reasoning and Acting in Language Models»](https://arxiv.org/abs/2210.03629). Project page: [react-lm.github.io](https://react-lm.github.io).

The idea is simple: an LLM should not only reason (as in Chain-of-Thought) or only act (as in tool-use agents), but **interleave** reasoning and action, forming a closed feedback loop.

### Analogy with Human Thinking

When you solve a complex problem — say, navigating an unfamiliar city — you don't plan the entire route in your head and then execute. You: check the map → think → walk → see a sign → adjust → keep going. This very pattern — **interleaved Reasoning + Acting** — is what ReAct formalizes.

### Contrast with Predecessors

| Approach | Reasoning | Action | Grounding | Self-correction |
|----------|-----------|--------|-----------|-----------------|
| **CoT** (Wei et al., 2022, [arXiv:2201.11903](https://arxiv.org/abs/2201.11903)) | Yes | No | No | No |
| **Act-only** | No | Yes | Yes | No |
| **ReAct** | Yes | Yes | Yes | Yes |

CoT generates a chain of reasoning but cannot verify facts. Act-only calls tools but doesn't reflect on results. ReAct combines both worlds.

### Formal Definition

A ReAct agent operates in a loop:

{{< mermaid align="center" >}}
graph LR
    Q((Question)) --> T1[Thought]
    T1 --> A1[Action]
    A1 --> O1[Observation]
    O1 --> T2[Thought]
    T2 --> A2[Action]
    A2 --> O2[Observation]
    O2 --> TN[...]
    TN --> Ans((Answer))
{{< /mermaid >}}

Example trace (task: "What's the weather in Paris?"):

```
Thought: I need to check the current weather in Paris
Action:  get_weather(city="Paris")
Observation: 18°C, clear, humidity 45%
Thought: Data received, I can answer now
Answer: It's currently 18°C in Paris, clear skies, humidity 45%
```

Each **Thought** is the model's reasoning about what to do next. Each **Action** is a tool invocation. Each **Observation** is a result the model considers in the next step.

## 3. What Problem Does It Solve

### The CoT Problem: Hallucination

Chain-of-Thought impresses on tasks where the answer can be derived from context. But as soon as external facts are needed — the model hallucinates. Classic example: CoT confidently generates "the president of Nepal is Hari Bahadur Basnet" — the name sounds plausible but is factually wrong. No grounding — no guarantee.

### The Act-only Problem: No Reflection

An agent that only acts can call a tool and get a result, but cannot synthesize an answer. It doesn't "think" about the observation — just passes it along. This works for simple queries but breaks on multi-step tasks.

### The Error Propagation Problem

In long CoT chains, an error at an early step goes unnoticed and cascades, making the final answer nonsensical. Without the ability to verify through an external environment, the model cannot course-correct.

### What ReAct Provides

| Problem | ReAct Mechanism |
|---------|-----------------|
| Hallucination | Grounding through tool calls (Observation = fact) |
| No reflection | Thought after each Observation — model analyzes the result |
| Error propagation | Self-correction: model sees erroneous result and adjusts plan |
| Opacity | Interpretability: each Thought is a readable reasoning trace |
| Rigid plan | Dynamic planning: plan is revised at each step |

### Numbers from Yao et al., 2022

Key results from the [original paper](https://arxiv.org/abs/2210.03629):

- **HotpotQA + FEVER**: ReAct overcomes CoT hallucination through Wikipedia API — the model retrieves facts instead of making them up ([Yao et al., 2022, §4.1](https://arxiv.org/abs/2210.03629)).
- **ALFWorld**: +34% success rate over imitation learning — in a text-based home interaction environment, ReAct significantly outperforms the baseline ([Yao et al., 2022, §4.2](https://arxiv.org/abs/2210.03629)).
- **WebShop**: +10% success rate over reinforcement learning — even in the complex online shopping task, ReAct demonstrates an advantage ([Yao et al., 2022, §4.2](https://arxiv.org/abs/2210.03629)).

### Pattern Applicability

Importantly: the pattern has been validated across a broad class of models and tasks. The specific numbers from Yao et al. were obtained on PaLM-540B, but the interleaved reasoning + acting principle is model-independent — it works with GPT-4, Claude, and open-source models that support tool calling.

## 4. Architecture and Diagrams

### The Thought → Action → Observation Loop

{{< mermaid align="center" >}}
sequenceDiagram
    participant U as User
    participant A as ReAct Agent
    participant T as Tools
    
    U->>A: Question
    loop ReAct Loop
        A->>A: Thought (reasoning)
        A->>T: Action (tool call)
        T-->>A: Observation (result)
    end
    A->>A: Final Thought
    A-->>U: Answer
{{< /mermaid >}}

At each iteration, the agent forms a **Thought** — internal reasoning about the current state and next step. Then it executes an **Action** — calling one of the available tools. The resulting **Observation** is added to the context, and the cycle repeats. State (message history) accumulates: all previous Thoughts, Actions, and Observations are available at the next step.

### State-Graph Representation

In frameworks like Eino, ReAct is implemented as a directed graph (State Graph):

{{< mermaid align="center" >}}
graph TB
    START((START)) --> CM[ChatModel]
    CM -->|tool_calls| B{Branch}
    CM -->|no tool_calls| END((END))
    B -->|has tool calls| TN[ToolsNode]
    B -->|no tool calls| END
    TN --> CM
{{< /mermaid >}}

The graph representation matters for three reasons:

- **State**: all messages are stored in a single graph state — no manual context management needed.
- **Streaming**: each graph node can stream its result — the user sees the agent's reasoning in real time.
- **Callbacks**: handlers can be attached to each node — logging, metrics, tracing.

### Evolution of Approaches 2022–2025

{{< mermaid align="center" >}}
graph LR
    A["Prompt-based<br/>ReAct (2022)"] --> B["Tool Calling<br/>API (2023)"]
    B --> C["Agent<br/>Frameworks (2024)"]
    C --> D["Multi-Agent<br/>Orchestration (2025)"]
{{< /mermaid >}}

- **Prompt-based ReAct (2022)**: original implementation — few-shot prompts, tools via text interface. Worked but fragile and non-scalable.
- **Tool Calling API (2023)**: models gained native function calling support — tools became structured and reliable. Schick et al., 2023 ([arXiv:2302.04761](https://arxiv.org/abs/2302.04761)) showed that LLMs can learn to call tools autonomously.
- **Agent Frameworks (2024)**: LangGraph, LlamaIndex, Eino — frameworks that encapsulate ReAct into reusable components with graph architecture.
- **Multi-Agent Orchestration (2025)**: multiple ReAct agents coordinate to solve complex tasks — each specialized in its domain.

## 5. Framework Landscape

ReAct is a universal pattern, implemented across all major frameworks. Summary table:

| Framework | Language | Implementation | Key Feature |
|-----------|----------|----------------|-------------|
| **LangGraph** | Python | `create_react_agent` | De facto standard in Python, graph model |
| **LlamaIndex** | Python | `ReActAgent` | Deep RAG and index integration |
| **OpenAI Agents SDK** | Python | `Agent` + tools | Native GPT-4/GPT-4o integration |
| **Anthropic Claude API** | Python/TS | Tool use + system prompt | Maximum reasoning via extended thinking |
| **Google ADK** | Python | `Agent` + tools | Gemini and Google Cloud integration |
| **Eino** | Go | `react.NewAgent` | Go-native, production-tested at ByteDance |

### LangGraph — De Facto Standard

[LangGraph](https://github.com/langchain-ai/langgraph) has become the standard for agent systems in the Python world. The `create_react_agent` function creates a ready-made ReAct agent from a model and tool list in a few lines. The graph architecture allows conditional transitions, loops, and human-in-the-loop patterns. If you're in Python — this is the first candidate.

### Eino — Why Go

[Eino](https://www.cloudwego.io/docs/eino/overview/eino_open_source/) is a framework from CloudWeGo (ByteDance), production-tested in Doubao, TikTok, and Coze. Chosen for the practical section for three reasons:

1. **Go-native**: typed tools, interfaces, no `interface{}` chaos.
2. **Production-tested**: serves hundreds of millions of requests per day inside ByteDance.
3. **Graph architecture**: `compose.Graph` under the hood of the ReAct agent — the same model as LangGraph, but in Go.

No Python code examples are provided in this article — this is intentional. This blog focuses on Go.

## 6. Practice: ReAct with Eino, Go

### Installation

```bash
go get github.com/cloudwego/eino@latest
go get github.com/cloudwego/eino-ext/components/model/openai@latest
```

API documentation: [pkg.go.dev/github.com/cloudwego/eino](https://pkg.go.dev/github.com/cloudwego/eino)

### Minimal ReAct Agent

```go
package main

import (
    "context"
    "fmt"

    "github.com/cloudwego/eino-ext/components/model/openai"
    "github.com/cloudwego/eino/components/tool"
    "github.com/cloudwego/eino/compose"
    "github.com/cloudwego/eino/flow/agent/react"
    "github.com/cloudwego/eino/schema"
)

func main() {
    ctx := context.Background()

    // Model with tool calling support
    chatModel, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
        Model: "gpt-4o",
    })

    // Create agent with a single tool
    agent, _ := react.NewAgent(ctx, &react.AgentConfig{
        ToolCallingModel: chatModel,
        ToolsConfig: compose.ToolsNodeConfig{
            InvokableTools: []tool.InvokableTool{weatherTool()},
        },
    })

    // Invoke agent
    msg, _ := agent.Generate(ctx, []*schema.Message{
        schema.UserMessage("What's the weather in Paris?"),
    })
    fmt.Println(msg.Content)
}
```

### Typed Tool via utils.NewTool

```go
type WeatherRequest struct {
    City string `json:"city" jsonschema:"description=City to get weather for"`
}

type WeatherResponse struct {
    City        string `json:"city"`
    Temperature int    `json:"temperature"`
    Condition   string `json:"condition"`
}

func weatherTool() tool.InvokableTool {
    return utils.NewTool(
        &schema.ToolInfo{
            Name: "get_weather",
            Desc: "Get current weather for a specified city",
            ParamsOneOf: schema.NewParamsOneOfByParams(map[string]*schema.ParameterInfo{
                "city": {Type: "string", Desc: "City", Required: true},
            }),
        },
        func(ctx context.Context, input *WeatherRequest) (*WeatherResponse, error) {
            // Real weather API call goes here
            return &WeatherResponse{
                City: input.City, Temperature: 18, Condition: "clear",
            }, nil
        },
    )
}
```

### Key Parameters (without code)

- **`ToolCallingModel`** — model must support tool calling (`ToolCallingChatModel` interface).
- **`ToolsConfig`** — tool node configuration: `InvokableTools` and `StreamableTools`.
- **`MaxStep`** — graph step limit (default 12 = up to 6 full ChatModel + Tools cycles).
- **`MessageModifier`** — function to modify messages before model call (e.g., adding system prompt).
- **`ToolReturnDirectly`** — tools whose result is returned directly to the user, bypassing the next model call.

### Links for Deep Dive

- [ReAct Agent Manual](https://www.cloudwego.io/docs/eino/core_modules/flow_integration_components/react_agent_manual/) — complete parameter reference
- [eino-examples/flow/agent/react](https://github.com/cloudwego/eino-examples/tree/main/flow/agent/react) — full working example (Food Recommender demo)
- [GitHub: cloudwego/eino](https://github.com/cloudwego/eino) — source code

## 7. When ReAct is NOT the Right Choice

ReAct is not a silver bullet. Each Thought→Action→Observation cycle is a separate LLM call, meaning: cost grows linearly with the number of steps, latency accumulates, and the planning horizon is limited by the model's context window.

### Hierarchy of Complexity

The Microsoft Azure Architecture Center's [guide to AI agent orchestration](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns) articulates a principle: **use the lowest level of complexity that reliably solves the problem**.

| Level | Description | When it's enough |
|-------|-------------|------------------|
| **Direct model call** | Single LLM call, no tools | Classification, summarization, translation |
| **Single agent + tools (ReAct)** | Agent with reasoning and tools | Dynamic tool selection within a single domain |
| **Multi-agent orchestration** | Multiple specialized agents | Cross-domain tasks, distinct security boundaries |

ReAct occupies the middle level. If a direct model call solves the task — you don't need an agent. If a single agent can't cope due to prompt complexity, tool overload, or security requirements — move to multi-agent. But not before.

| If your task... | ...consider this alternative |
|-----------------|------------------------------|
| Has a fixed workflow with known steps | **Plan-and-Solve** — planning without iterative search |
| Is cost-sensitive (many LLM calls) | **ReWOO** — planning without interleaved model calls ([Xu et al., 2023](https://arxiv.org/abs/2305.18323)) |
| Requires learning from past mistakes | **Reflexion** (a.k.a. maker-checker, evaluator-optimizer) — ReAct + self-evaluation ([Shinn et al., 2023](https://arxiv.org/abs/2303.11366)) |
| Needs a tree of hypotheses | **Tree of Thoughts** — branching reasoning ([Yao et al., 2023](https://arxiv.org/abs/2305.10601)) |
| Has a long planning horizon | **Plan-and-Execute** — decomposition into subtasks |

A detailed comparison of patterns — in Part 5 of this series.

## 8. What's Next in the Series

- **Part 2: Plan-and-Execute Pattern** — decomposing tasks into subtasks with a separate planner and executor.
- **Part 3: Reflexion Pattern** — ReAct + self-evaluation: an agent that learns from its own mistakes.
- **Part 4: ReWOO Pattern** — planning without interleaved model calls: cheaper, faster, but without dynamic adjustment.
- **Part 5: Pattern Comparison + Multi-Agent Orchestration** — when to choose which pattern, and how multiple agents coordinate for complex tasks.

Discussion welcome — comments on the site or [GitHub Issues](https://github.com/triumphpc/blog/issues).

## 9. References

### Research Papers

1. **Yao et al., 2022 — ReAct**: Synergizing reasoning and acting in language models. Formalization of the Thought→Action→Observation loop. [arXiv:2210.03629](https://arxiv.org/abs/2210.03629)
2. **Wei et al., 2022 — Chain-of-Thought**: Step-by-step reasoning without external environment interaction. Predecessor to ReAct. [arXiv:2201.11903](https://arxiv.org/abs/2201.11903)
3. **Schick et al., 2023 — Toolformer**: LLMs learn to call tools autonomously. Bridge between prompt-based and API-based approaches. [arXiv:2302.04761](https://arxiv.org/abs/2302.04761)
4. **Shinn et al., 2023 — Reflexion**: Extending ReAct with self-evaluation and verbal reinforcement. [arXiv:2303.11366](https://arxiv.org/abs/2303.11366)
5. **Yao et al., 2023 — Tree of Thoughts**: Generalizing CoT to a tree of hypotheses with search. [arXiv:2305.10601](https://arxiv.org/abs/2305.10601)
6. **Xu et al., 2023 — ReWOO**: Planning without interleaved model calls — demonstrates ReAct's cost/latency limitations. [arXiv:2305.18323](https://arxiv.org/abs/2305.18323)

### Documentation and Examples

- [Eino ReAct Agent Manual](https://www.cloudwego.io/docs/eino/core_modules/flow_integration_components/react_agent_manual/) — all parameters and configuration
- [Eino Open Source Announcement](https://www.cloudwego.io/docs/eino/overview/eino_open_source/) — framework overview
- [GitHub: cloudwego/eino](https://github.com/cloudwego/eino) — source code
- [GitHub: eino-examples/flow/agent/react](https://github.com/cloudwego/eino-examples/tree/main/flow/agent/react) — full working example (Food Recommender)
- [LangGraph ReAct Agent template](https://github.com/langchain-ai/react-agent) — Python implementation
- [React Project Page](https://react-lm.github.io) — original paper's project page

### Architecture Guides

- [AI agent design patterns — Microsoft Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns) — complexity hierarchy (direct call → single agent → multi-agent), orchestration patterns, production recommendations for reliability, security, cost optimization

