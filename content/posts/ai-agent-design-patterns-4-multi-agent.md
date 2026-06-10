---
title: "AI Agent Design Patterns. Part 4: Multi-Agent Patterns"
date: 2026-06-08
draft: false
tags: ["ai", "agents", "llm", "multi-agent", "eino", "go"]
categories: ["ai", "engineering"]
summary: "Five multi-agent orchestration architectures — Supervisor, Swarm, Debate, Assembly Line, Hierarchical — with Mermaid diagrams, code examples in Eino (Go), and an honest look at limitations. When one agent isn't enough, and when you're better off staying solo."
ShowToc: true
series: ["AI Agent Design Patterns"]
---

## 1. Introduction

In [Part 1]({{< ref "ai-agent-design-patterns-1-react" >}}), I covered <abbr title="Reasoning + Acting — a pattern where LLM alternates between reasoning and action, receiving observations from the environment">ReAct</abbr> — an agent that thinks and acts. In [Part 2]({{< ref "ai-agent-design-patterns-2-plan-execute" >}}) — Plan-and-Execute, where a planner builds an N-step plan. In [Part 3]({{< ref "ai-agent-design-patterns-3-reflexion" >}}) — Reflexion, where an agent learns from its mistakes. Three patterns, three solo agents.

And here we hit a ceiling. One <abbr title="Large Language Model — a neural network trained on large text corpora; the foundation for AI agents">LLM</abbr> agent can't be an expert at everything. Well, it can — but poorly. One system prompt — one "role." Writer ≠ Reviewer ≠ Researcher in the same head. Conflicting interests, hallucinations, loss of focus. And even Reflexion won't help if the task requires fundamentally different expertise — an agent can't "reflect" a skill it doesn't have into existence.

So we need a team. Multiple agents, each with its own role, coordinating to solve a shared task. Sounds simple, but here's where it gets interesting: **how exactly do they coordinate?** Who makes decisions? How do you pass context? How do you avoid infinite handoff loops?

I'll break down five multi-agent orchestration architectures — from a centralized supervisor to a parallel debate — with Mermaid diagrams, code in Eino (Go), and honest limitations. All examples use the same scenario: a code review pipeline with Researcher, Coder, and Reviewer — easier to compare that way.

## 2. Supervisor Pattern

### Motivation

The most common scenario: you have a task requiring different expertise, but you need someone to decide **which agent to delegate what to**. Without centralized coordination, agents will talk simultaneously, interrupt each other, or — even worse — pass the task around in circles.

Supervisor is <abbr title="Centralized coordination — one router agent makes all delegation decisions; eliminates routing conflicts">centralized coordination</abbr>: one router agent receives the task, decides who to delegate to, collects results, and decides the next step.

### Architecture

{{< mermaid >}}
graph TB
    USER["👤 User"] -->|"task"| SUP["🧑‍💼 Supervisor<br/>delegates, aggregates"]
    SUP -->|"delegate: research"| RES["🔍 Researcher<br/>gathers context"]
    SUP -->|"delegate: code"| COD["💻 Coder<br/>writes code"]
    SUP -->|"delegate: review"| REV["🔎 Reviewer<br/>reviews code"]
    RES -->|"result"| SUP
    COD -->|"result"| SUP
    REV -->|"result"| SUP
    SUP -->|"final answer"| USER

    style SUP fill:#2e7d32,color:#fff
    style RES fill:#1565c0,color:#fff
    style COD fill:#e65100,color:#fff
    style REV fill:#6a1b9a,color:#fff
{{< /mermaid >}}

The Supervisor sees the full picture: each sub-agent returns its result to the supervisor, not to the next agent. This eliminates chaos, but creates a <abbr title="Bottleneck — a single point through which all data flows; as agent count grows, the supervisor becomes the limiting factor">bottleneck</abbr>.

### Implementation in Eino

Eino ADK provides a ready-made `supervisor.New`:

```go
package main

import (
    "context"
    "log"

    "github.com/cloudwego/eino/adk"
    "github.com/cloudwego/eino/adk/prebuilt/supervisor"
    "github.com/cloudwego/eino-ext/components/model/openai"
)

func main() {
    ctx := context.Background()

    // Initialize ChatModel for all agents
    chatModel, err := openai.NewChatModel(ctx, &openai.ChatModelConfig{
        APIKey:  os.Getenv("OPENAI_API_KEY"),
        Model:   os.Getenv("OPENAI_MODEL"),
        BaseURL: os.Getenv("OPENAI_BASE_URL"),
    })
    if err != nil {
        log.Fatal(err)
    }

    // Create sub-agents
    researcher, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
        Name:        "researcher",
        Description: "Researches best practices and gathers context for coding tasks",
        Instruction: "You are a research specialist. Find relevant information, API docs, and best practices.",
        Model:       chatModel,
    })

    coder, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
        Name:        "coder",
        Description: "Writes Go code based on research findings",
        Instruction: "You are a Go developer. Write clean, idiomatic Go code.",
        Model:       chatModel,
    })

    reviewer, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
        Name:        "reviewer",
        Description: "Reviews code for bugs, style issues, and correctness",
        Instruction: "You are a code reviewer. Check for bugs, edge cases, and style issues.",
        Model:       chatModel,
    })

    // Supervisor agent (coordinator)
    coordinator, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
        Name:        "coordinator",
        Description: "Coordinates research, coding, and review sub-agents",
        Instruction: "You are a project coordinator. Delegate tasks to the right specialist.",
        Model:       chatModel,
    })

    // Assemble the Supervisor pattern
    supervisorAgent, err := supervisor.New(ctx, &supervisor.Config{
        Supervisor: coordinator,
        SubAgents:  []adk.Agent{researcher, coder, reviewer},
    })
    if err != nil {
        log.Fatal(err)
    }

    // Run
    runner := adk.NewRunner(ctx, adk.RunnerConfig{
        Agent: supervisorAgent,
    })

    result, _ := runner.Query(ctx, "Implement a thread-safe LRU cache in Go")
    _ = result
}
```

Key point: `supervisor.Config` takes a `Supervisor` (coordinator) and `SubAgents` (array of specialists). The Supervisor itself decides who to call and when.

### Limitations

1. **Bottleneck**: all results pass through the supervisor → latency grows linearly as agent count increases
2. **Context explosion**: the supervisor must hold results from all sub-agents in its context → tokens burn fast
3. **Single point of failure**: if the supervisor delegates incorrectly, the entire process goes off track

## 3. Swarm / Handoff Pattern

### Motivation

But what if you don't need a supervisor? What if agents already know who to hand off to? This is <abbr title="Decentralized routing — agents decide themselves who to transfer control to; no single coordinator">decentralized coordination</abbr>: an agent completes its part and transfers (handoff) to the next one.

Sounds tempting — no bottleneck, no single point of failure. But there's a price: who decides when to hand off? The agent must understand when its work is done and who to pass the baton to.

### Architecture

{{< mermaid >}}
graph LR
    RES["🔍 Researcher"] -->|"handoff:<br/>context ready"| COD["💻 Coder"]
    COD -->|"handoff:<br/>code ready"| REV["🔎 Reviewer"]
    REV -->|"handoff:<br/>needs fix"| COD
    REV -->|"done"| USER["👤 User"]

    style RES fill:#1565c0,color:#fff
    style COD fill:#e65100,color:#fff
    style REV fill:#6a1b9a,color:#fff
{{< /mermaid >}}

Notice: the Reviewer can hand the task back to the Coder (found a bug → fix it). This is a cycle — unlike the Assembly Line, where the flow is strictly unidirectional.

### Implementation in Eino

Eino provides `host.NewMultiAgent` — a Host pattern where a host agent routes to specialists:

```go
package main

import (
    "context"
    "log"

    "github.com/cloudwego/eino/flow/agent/multiagent/host"
    "github.com/cloudwego/eino-ext/components/model/openai"
)

func main() {
    ctx := context.Background()

    chatModel, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
        APIKey:  os.Getenv("OPENAI_API_KEY"),
        Model:   os.Getenv("OPENAI_MODEL"),
        BaseURL: os.Getenv("OPENAI_BASE_URL"),
    })

    // Host agent: router
    hostAgent := &host.Host{
        ChatModel:    chatModel,
        SystemPrompt: "You manage a code review pipeline. Route requests to the right specialist.",
    }

    // Specialists (implement the host.Specialist interface)
    researcher := NewResearchSpecialist(chatModel)
    coder := NewCodeSpecialist(chatModel)
    reviewer := NewReviewSpecialist(chatModel)

    // Assemble multi-agent
    multiAgent, err := host.NewMultiAgent(ctx, &host.Config{
        Host:        hostAgent,
        Specialists: []host.Specialist{researcher, coder, reviewer},
    })
    if err != nil {
        log.Fatal(err)
    }

    // Run
    result, _ := multiAgent.Run(ctx, "Implement a thread-safe LRU cache in Go")
    _ = result
}
```

Eino's Host pattern is a hybrid: the host agent routes tasks, but specialists can be invoked by context, not just by direct order. This is closer to swarm than to a pure supervisor.

### Limitations

1. **Deadlock**: Agent A → B → A → B → ... infinite handoff loop. No supervisor to break the cycle
2. **Routing quality**: if an agent incorrectly assesses who to hand off to, the entire chain breaks
3. **Context isolation**: each agent sees only its own context + handoff message. No global picture

## 4. Debate Pattern

### Motivation

What if the task has no single correct answer? Or if it's critical to test multiple hypotheses in parallel? Du et al. showed that multi-agent debate improves <abbr title="Factual accuracy — how well generated text matches real facts; reduces hallucinations">factual accuracy</abbr> by +8% on GSM8K through debate between N agents ([Du et al., 2023](https://arxiv.org/abs/2305.14325)).

The idea: several agents independently solve the task, then critique each other's solutions — and arrive at a consensus. Like Minsky's <abbr title="Society of Minds — Marvin Minsky's concept: intelligence emerges from the interaction of many simple agents, not from a single center">society of minds</abbr>, but in practice.

### Architecture

{{< mermaid >}}
graph TB
    TASK["📋 Task"] --> A1["🤖 Agent A"]
    TASK --> A2["🤖 Agent B"]
    TASK --> A3["🤖 Agent C"]
    A1 <-->|"critique"| A2
    A2 <-->|"critique"| A3
    A1 <-->|"critique"| A3
    A1 --> J["⚖️ Judge / Consensus"]
    A2 --> J
    A3 --> J
    J --> RESULT["✅ Final Answer"]

    style A1 fill:#2e7d32,color:#fff
    style A2 fill:#1565c0,color:#fff
    style A3 fill:#e65100,color:#fff
    style J fill:#c62828,color:#fff
{{< /mermaid >}}

Three agents solve the task in parallel, critique each other, and a <abbr title="Judge — a separate arbiter agent that receives all solutions and picks the best one; alternative is majority vote, where the option with the most votes wins">Judge (or majority vote)</abbr> selects the final answer. This is not a pipeline — it's **parallel consensus**.

### Implementation in Eino

There's no ready-made Debate pattern in Eino. We build one using `compose.Graph`:

```go
package main

import (
    "context"
    "fmt"

    "github.com/cloudwego/eino/compose"
    "github.com/cloudwego/eino/schema"
    "github.com/cloudwego/eino-ext/components/model/openai"
)

func main() {
    ctx := context.Background()

    chatModel, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
        APIKey:  os.Getenv("OPENAI_API_KEY"),
        Model:   os.Getenv("OPENAI_MODEL"),
        BaseURL: os.Getenv("OPENAI_BASE_URL"),
    })

    g := compose.NewGraph[string, string]()

    // Three independent agents — generate solutions in parallel
    g.AddGraphNode("agent_a", func(ctx context.Context, task string) (string, error) {
        msgs := []*schema.Message{
            schema.SystemMessage("You are a senior Go developer. Solve the task."),
            schema.UserMessage(task),
        }
        resp, _ := chatModel.Generate(ctx, msgs)
        return resp.Content, nil
    })

    g.AddGraphNode("agent_b", func(ctx context.Context, task string) (string, error) {
        msgs := []*schema.Message{
            schema.SystemMessage("You are a careful code reviewer. Solve the task with emphasis on correctness."),
            schema.UserMessage(task),
        }
        resp, _ := chatModel.Generate(ctx, msgs)
        return resp.Content, nil
    })

    g.AddGraphNode("agent_c", func(ctx context.Context, task string) (string, error) {
        msgs := []*schema.Message{
            schema.SystemMessage("You are a performance expert. Solve the task with emphasis on efficiency."),
            schema.UserMessage(task),
        }
        resp, _ := chatModel.Generate(ctx, msgs)
        return resp.Content, nil
    })

    // Judge: aggregates and picks the best answer
    g.AddGraphNode("judge", func(ctx context.Context, input string) (string, error) {
        // input contains results from all three agents
        msgs := []*schema.Message{
            schema.SystemMessage("You are a judge. Compare three solutions and pick the best one. Explain your choice."),
            schema.UserMessage(input),
        }
        resp, _ := chatModel.Generate(ctx, msgs)
        return resp.Content, nil
    })

    // Parallel generation → Judge
    g.AddEdge(compose.START, "agent_a")
    g.AddEdge(compose.START, "agent_b")
    g.AddEdge(compose.START, "agent_c")
    g.AddEdge("agent_a", "judge")
    g.AddEdge("agent_b", "judge")
    g.AddEdge("agent_c", "judge")
    g.AddEdge("judge", compose.END)

    r, _ := g.Compile(ctx)
    result, _ := r.Invoke(ctx, "Implement a thread-safe LRU cache in Go")
    fmt.Println(result)
}
```

Key point: `compose.START` → three nodes in parallel → Judge aggregates. Graph automatically parallelizes nodes whose inputs are all ready.

### Limitations

1. **3x cost**: three LLM calls instead of one (minimum). Judge is the fourth
2. **Doesn't work without a Judge**: simple majority vote can converge on an error (collective hallucination)
3. **Latency**: parallel is faster than sequential, but still at least max(agent_a, agent_b, agent_c) + judge

## 5. Assembly Line Pattern

### Motivation

What if the task is strictly <abbr title="Sequential processing — each stage receives the previous stage's output and passes it to the next; a pipeline">sequential</abbr>? Research → Code → Review — each stage depends on the previous one. No parallelism, no debate. Just a pipeline.

This is exactly what ChatDev ([Qian et al., 2023](https://arxiv.org/abs/2307.07924)) and MetaGPT ([Hong et al., 2023](https://arxiv.org/abs/2308.00352)) implemented. MetaGPT added <abbr title="Standard Operating Procedures — standardized rules defining the sequence and format of agent interactions">SOP</abbr>: each agent receives a clear input/output format, which reduces hallucinations by 1.8x compared to ChatDev.

### Architecture

{{< mermaid >}}
graph LR
    TASK["📋 Task"] --> RES["🔍 Researcher<br/>gathers context"]
    RES -->|"research<br/>findings"| COD["💻 Coder<br/>writes code"]
    COD -->|"code<br/>draft"| REV["🔎 Reviewer<br/>reviews"]
    REV -->|"approved<br/>code"| DONE["✅ Result"]

    style RES fill:#1565c0,color:#fff
    style COD fill:#e65100,color:#fff
    style REV fill:#6a1b9a,color:#fff
{{< /mermaid >}}

Strict sequence. No cycles, no parallel branches. Each agent receives the previous one's output — and only that.

### Implementation in Eino

Assembly Line is a classic `compose.Workflow` (or Chain):

```go
package main

import (
    "context"
    "fmt"

    "github.com/cloudwego/eino/compose"
    "github.com/cloudwego/eino/schema"
    "github.com/cloudwego/eino-ext/components/model/openai"
)

func main() {
    ctx := context.Background()

    chatModel, _ := openai.NewChatModel(ctx, &openpi.ChatModelConfig{
        APIKey:  os.Getenv("OPENAI_API_KEY"),
        Model:   os.Getenv("OPENAI_MODEL"),
        BaseURL: os.Getenv("OPENAI_BASE_URL"),
    })

    // Assembly Line via Chain
    chain := compose.NewChain[string, string]()

    // Stage 1: Research
    chain.AppendChatTemplate(
        prompt.FromMessages(schema.FString,
            schema.SystemMessage("You are a research specialist. Gather context and best practices for the task. Output: structured research findings."),
            schema.UserMessage("{input}"),
        ),
    ).AppendChatModel(chatModel)

    // Stage 2: Code
    chain.AppendChatTemplate(
        prompt.FromMessages(schema.FString,
            schema.SystemMessage("You are a Go developer. Write code based on the research findings. Output: complete Go source code."),
            schema.UserMessage("{input}"),
        ),
    ).AppendChatModel(chatModel)

    // Stage 3: Review
    chain.AppendChatTemplate(
        prompt.FromMessages(schema.FString,
            schema.SystemMessage("You are a code reviewer. Review the code for bugs, edge cases, and style issues. Output: approved code or list of fixes."),
            schema.UserMessage("{input}"),
        ),
    ).AppendChatModel(chatModel)

    r, _ := chain.Compile(ctx)
    result, _ := r.Invoke(ctx, "Implement a thread-safe LRU cache in Go")
    fmt.Println(result)
}
```

`compose.Chain` is a `Workflow` under the hood. Each `ChatTemplate + ChatModel` pair is one pipeline stage. The previous stage's output is automatically fed as input to the next.

### Limitations

1. **Sequential = slow**: no parallelism, each stage waits for the previous one
2. **Cascading errors**: if Researcher provides bad context, Coder writes bad code, and Reviewer might not catch the root problem
3. **No cycles**: if Reviewer finds a bug — the entire pipeline needs to restart (unlike Swarm, where handoff back is possible)

## 6. Hierarchical Pattern

### Motivation

What if the task is so complex that one level of delegation isn't enough? A project with 10 subtasks, each breaking down into 3-4 micro-tasks. A single supervisor would drown in context.

Hierarchical is <abbr title="Hierarchical decomposition — multi-level task breakdown: the top level delegates subtasks, each of which can have its own sub-tasks and coordinators">multi-level decomposition</abbr>. The CEO delegates to Team Leads, who delegate to specialists. Each level manages only its own scope.

### Architecture

{{< mermaid >}}
graph TB
    USER["👤 User"] --> CEO["🎯 CEO Agent<br/>strategy"]
    CEO -->|"design<br/>research"| TL1["📋 Team Lead: Research"]
    CEO -->|"implement"| TL2["📋 Team Lead: Dev"]
    TL1 --> R1["🔍 Researcher 1"]
    TL1 --> R2["🔍 Researcher 2"]
    TL2 --> D1["💻 Coder 1"]
    TL2 --> D2["💻 Coder 2"]
    TL1 --> CEO
    TL2 --> CEO
    CEO --> USER

    style CEO fill:#c62828,color:#fff
    style TL1 fill:#2e7d32,color:#fff
    style TL2 fill:#2e7d32,color:#fff
    style R1 fill:#1565c0,color:#fff
    style R2 fill:#1565c0,color:#fff
    style D1 fill:#e65100,color:#fff
    style D2 fill:#e65100,color:#fff
{{< /mermaid >}}

The key difference from Supervisor: here we have **two levels of delegation**. The CEO doesn't know about Researcher 1 and Coder 2 — it only works with Team Leads. This reduces context load at each level.

### Implementation in Eino

Eino ADK provides `deep.New` — a DeepAgent with built-in <abbr title="Task management — automatic goal decomposition into subtasks, progress tracking, and delegation to sub-agents">task management</abbr>:

```go
package main

import (
    "context"
    "log"

    "github.com/cloudwego/eino/adk"
    "github.com/cloudwego/eino/adk/prebuilt/deep"
    "github.com/cloudwego/eino-ext/components/model/openai"
)

func main() {
    ctx := context.Background()

    chatModel, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
        APIKey:  os.Getenv("OPENAI_API_KEY"),
        Model:   os.Getenv("OPENAI_MODEL"),
        BaseURL: os.Getenv("OPENAI_BASE_URL"),
    })

    // Create sub-agents
    researcher, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
        Name:        "researcher",
        Description: "Researches best practices and gathers context",
        Instruction: "You are a research specialist. Find relevant information and API docs.",
        Model:       chatModel,
    })

    coder, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
        Name:        "coder",
        Description: "Writes Go code based on requirements",
        Instruction: "You are a Go developer. Write clean, idiomatic Go code.",
        Model:       chatModel,
    })

    reviewer, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
        Name:        "reviewer",
        Description: "Reviews code for correctness and quality",
        Instruction: "You are a code reviewer. Check for bugs and style issues.",
        Model:       chatModel,
    })

    // DeepAgent: automatically breaks down tasks into subtasks
    // and delegates them to sub-agents
    deepAgent, err := deep.New(ctx, &deep.Config{
        Name:        "project_manager",
        Description: "Manages complex coding projects by breaking them into subtasks",
        ChatModel:   chatModel,
        Instruction: "You are a project manager. Break down tasks and delegate to specialists.",
        SubAgents:   []adk.Agent{researcher, coder, reviewer},
    })
    if err != nil {
        log.Fatal(err)
    }

    runner := adk.NewRunner(ctx, adk.RunnerConfig{
        Agent: deepAgent,
    })

    result, _ := runner.Query(ctx, "Implement a thread-safe LRU cache in Go with tests")
    _ = result
}
```

DeepAgent uses the built-in `write_todos` tool for planning and the `task` tool for delegating to sub-agents. Context is isolated between the main agent and sub-agents — this prevents <abbr title="Context pollution — when irrelevant information from one agent leaks into another's context, degrading response quality">context pollution</abbr>.

### Limitations

1. **Overhead**: two levels of coordination = two levels of LLM calls. CEO + Team Lead before the task reaches the executor
2. **Debugging complexity**: an error at a lower level can silently bubble up
3. **Over-engineering for simple tasks**: if a task fits in a single supervisor — hierarchy is overkill

## 7. Pattern Comparison

| Pattern | Centralization | Parallelism | Coordination complexity | Token overhead | When to use |
|---------|:---:|:---:|:---:|:---:|---|
| **Supervisor** | High | Low | Low | Medium | Clear roles, need control over execution order |
| **Swarm / Handoff** | Low | Medium | High | Medium | Agents know their boundaries; flexible routing |
| **Debate** | Low (or Judge) | High | Medium | High | Ambiguous tasks, need hypothesis testing |
| **Assembly Line** | Low | None | Low | Low | Strict sequential stages, SOP |
| **Hierarchical** | High (multi-level) | Medium | High | High | Complex projects with subtask decomposition |

How to read the table: **Centralization** — how many decision points exist. **Parallelism** — can agents work simultaneously. **Coordination complexity** — how much effort goes into routing. **Token overhead** — how many extra tokens are spent on coordination.

## 8. Honest Limitations

### Communication overhead

Every exchange between agents = full context in the prompt. With 3 agents and 2 debate rounds — 3×2×context_length tokens. With long context, this can cost $0.50+ per request. <abbr title="Mitigation — measures to reduce the negative effect; strategies for lowering cost or risk">Mitigation</abbr>: summarize intermediate results before passing them on.

### Coordination errors

The Supervisor can delegate incorrectly. An agent can hand off to the wrong specialist. The Judge can pick the worst answer. Li et al. showed that <abbr title="Sampling-and-voting — a method where N agents independently generate answers, then the most popular one is selected by vote; similar to self-consistency, but with different system prompts">"more agents" improves results through sampling-and-voting</abbr>, but this only works **when the base model is already good enough** ([Li et al., 2024](https://arxiv.org/abs/2402.05120)). If the model gives <50% accuracy, more agents won't help.

### Diminishing returns

More agents ≠ better results. Li et al. showed that <abbr title="Sampling-and-voting — a method where N agents independently generate answers, then the most popular one is selected by vote; similar to self-consistency, but with different system prompts">sampling-and-voting</abbr> scales with the number of agents, but with **diminishing returns**: each additional agent contributes less than the previous one. In practice, 3-5 agents is the optimum. Beyond 5-7, costs grow while gains are minimal.

### Deadlock / Infinite loop

In Swarm/Handoff, Agent A passes to B, B to A, ad infinitum. In Debate, agents can't agree. In Hierarchical, Team Leads keep bouncing the task to each other. <abbr title="Mitigation — measures to reduce the negative effect; strategies for lowering cost or risk">Mitigation</abbr>: `WithMaxRunSteps` in Eino Graph (as in Reflexion), timeout on handoff count, or circuit breaker when re-transfer to the same agent occurs.

### When one agent is better

If the task:
- Doesn't require different expertise
- Has a clear success criterion
- Fits in a single system prompt

...then multi-agent is unnecessary complexity. ReAct with the right prompt often works better than three agents coordinating through a supervisor. Don't add agents for the sake of adding agents.

## 9. Summary

Five multi-agent orchestration patterns — not competing, but complementary:

- **Supervisor** — when you need control and predictability
- **Swarm** — when agents know their boundaries and can self-organize
- **Debate** — when hypothesis testing and factual accuracy matter
- **Assembly Line** — when stages are strictly sequential and SOP matters more than flexibility
- **Hierarchical** — when task complexity requires multi-level decomposition

In Eino: three patterns out of the box (`supervisor.New`, `host.NewMultiAgent`, `deep.New`), two via `compose.Graph`. Choosing a pattern is choosing a trade-off between control and flexibility, parallelism and cost, simplicity and scalability.

And the main point: **adding agents doesn't fix a bad prompt**. Multi-agent is a tool for coordinating expertise, not a magic pill for hallucinations.

---

**Next in the series:** [Part 5: Agent Memory Management]({{< ref "ai-agent-design-patterns-5-memory" >}})
