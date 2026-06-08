---
title: "AI Agent Design Patterns. Part 3: Reflexion Pattern"
date: 2026-06-07
draft: false
tags: ["ai", "agents", "llm", "reflexion", "self-reflection", "eino", "go"]
categories: ["ai", "engineering"]
summary: "Reflexion is a pattern where an agent learns from its own mistakes through verbal reinforcement and episodic memory. I trace the evolution of self-critique from Constitutional AI to Reflexion, honestly show its limitations (doesn't work without an external verifier), and implement a working example in Go using Eino compose.Graph with an Actor → Evaluator → Reflector → retry loop."
ShowToc: true
series: ["AI Agent Design Patterns"]
---

## 1. Introduction

In [Part 1]({{< ref "ai-agent-design-patterns-1-react" >}}), I covered <abbr title="Reasoning + Acting — a pattern where LLM alternates between reasoning (Thought) and action (Action), receiving observations (Observation) from the environment; the basis for Reflexion's Actor">ReAct</abbr> — a pattern where an agent alternates between reasoning and action. In [Part 2]({{< ref "ai-agent-design-patterns-2-plan-execute" >}}), I covered Plan-and-Execute, where a separate Planner builds an N-step plan. Both patterns share one flaw: **the agent doesn't learn from its mistakes**. Every run starts from scratch. The Reflexion pattern fixes this: the agent makes an attempt, evaluates the result, reflects — and uses the accumulated experience on the next try.

## 2. What is Reflexion

### History

In December 2022, Anthropic published [Constitutional AI](https://arxiv.org/abs/2212.08073) — an approach where a language model critiques its own responses and rewrites them to reduce harmfulness. This was the first large-scale example of the <abbr title="A pattern where an LLM evaluates its own output and suggests improvements">self-critique</abbr> pattern: generate → self-critique → revise. But Constitutional AI used critique for **training** (<abbr title="Further training of a pretrained model on specific data; modifies model weights">finetuning</abbr> via <abbr title="Reinforcement Learning from AI Feedback — training a model using feedback from another LLM instead of human annotators">RLAIF</abbr>), not for <abbr title="Using a trained model to generate output; model weights are not changed">inference</abbr>.

In March 2023, Madaan et al. published [Self-Refine](https://arxiv.org/abs/2303.17651) — iterative improvement through self-feedback. The same <abbr title="Large Language Model — a neural network trained on large text corpora for text generation and understanding (GPT-4, Claude, Llama)">LLM</abbr> plays three roles: generator, critic, and refiner. Result: ~20% average improvement across 7 tasks ([Madaan et al., 2023](https://arxiv.org/abs/2303.17651)). But there's a catch — on reasoning tasks (Math Reasoning), the improvement is **0%**: the model cannot reliably determine whether its reasoning is correct or not.

And here's where it gets interesting. In the same month of March 2023, Shinn et al. published [Reflexion](https://arxiv.org/abs/2303.11366) — solving the core problem of Self-Refine by adding **<abbr title="A store of textual lessons from past attempts; reflections are injected into the Actor's context">episodic memory</abbr>** and **external evaluation**. Instead of a single <abbr title="A critique-then-refine cycle: LLM evaluates its output and regenerates it considering the feedback; the base pattern of Self-Refine">critique-refine</abbr> cycle — a <abbr title="An approach with multiple attempts at solving a task, accumulating experience between them">multi-trial</abbr> process where the agent accumulates verbal "lessons" and uses them on subsequent attempts. Result: 91% <abbr title="Metric: fraction of correct answers on the first attempt">pass@1</abbr> on HumanEval (vs. 80% for GPT-4) and +22% absolute on AlfWorld ([Shinn et al., 2023](https://arxiv.org/abs/2303.11366)).

### Analogy with a developer

Imagine: a junior writes code, runs tests — they fail. What do they do? They don't rewrite from scratch — they read the error, understand the cause, and note "next time I'll check the nil <abbr title="Edge case — input data at the boundary of valid values (empty list, nil, zero length); a frequent source of bugs">edge case</abbr>." That's reflection: not just fixing, but **extracting a lesson for the future**. ReAct is a junior without memory: making the same mistakes every time. Plan-and-Execute is a junior with a plan but without retrospection. Reflexion is a junior who keeps an error diary.

### Formal definition

Reflexion operates in three phases, repeated cyclically:

1. **Act**: The Actor (LLM) generates actions and receives observations from the environment.
2. **Evaluate**: The <abbr title="A deterministic system for evaluating results (unit tests, compiler, game environment); provides an objective PASS/FAIL signal">Evaluator</abbr> assesses the result — with a scalar score or free-form text.
3. **Reflect**: The Self-Reflection model generates <abbr title="Verbal reinforcement — textual feedback describing what went wrong and how to fix it; unlike numeric reward in classical RL">verbal feedback</abbr> — what went wrong and how to fix it. The reflection is stored in **<abbr title="A store of textual lessons from past attempts">episodic memory</abbr>**.

On the next attempt, the Actor receives the contents of episodic memory in context — and can avoid repeating past mistakes.

## 3. What problem it solves

### ReAct's problem: no learning from mistakes

ReAct, as I showed in Part 1, alternates Thought-Action-Observation in a single loop. If the task isn't solved — the agent simply starts over. Past experience? Lost. Every run is the first and last.

### Plan-and-Execute's problem: no retrospection

Plan-and-Execute builds a plan and executes it. The Reviser adjusts the plan when needed — but **within a single run**. Between runs — clean slate. No learning from past mistakes.

### Self-Refine's problem: no memory between attempts

Self-Refine does <abbr title="A critique-then-refine cycle: LLM evaluates its output and regenerates it considering the feedback; the base pattern of Self-Refine">critique-refine</abbr> in a single LLM call. There's improvement on generation tasks (style, format) — but on reasoning tasks **0%**, because the model cannot reliably assess the correctness of its own reasoning without an external arbiter ([Madaan et al., 2023](https://arxiv.org/abs/2303.17651), Table 1: Math Reasoning).

### What Reflexion provides

| Problem | Reflexion's solution |
|---------|---------------------|
| No learning from mistakes | Episodic memory stores verbal lessons between attempts |
| Unreliable self-assessment | Evaluator — external arbiter (tests, compiler, environment) |
| No retrospection | Each attempt is enriched with reflections from past ones |
| <abbr title="Short-sighted — fixing the symptom rather than the cause; in agent context: a local fix without understanding the root problem">Myopic</abbr> fixes | Reflection focuses on the cause of the error, not the symptom |

## 4. Architecture

### Actor → Evaluator → Reflector → Memory → retry

{{< mermaid align="center" >}}
sequenceDiagram
    participant U as User
    participant A as Actor
    participant E as Evaluator
    participant R as Reflector
    participant M as Episodic Memory

    U->>A: Task + reflections from memory
    A->>A: Generate actions (ReAct loop)
    A->>E: Trajectory (actions + observations)
    
    alt Result is correct
        E-->>U: ✅ Success
    else Result has errors
        E->>R: Trajectory + evaluation
        R->>R: Generate verbal reflection<br/>"I failed because..."
        R->>M: Store reflection
        M-->>A: Enriched context for next attempt
        Note over A,M: Retry with accumulated experience
    end
{{< /mermaid >}}

### Components

**Actor** — an LLM generating actions. Can be a ReAct agent or a simple ChatModel. The key difference from regular ReAct: the Actor receives episodic memory contents in the <abbr title="System prompt — a hidden instruction defining the behavior and role of an LLM; not visible to the user, sets context and constraints for the model">system prompt</abbr>, allowing it to account for past mistakes.

**Evaluator** — assesses the Actor's result. Can be:
- *Deterministic*: unit tests, compiler, game environment — gives an objective score
- *LLM-based*: another language model assesses quality — less reliable but applicable for <abbr title="Open-ended tasks — tasks without a single correct answer; the success criterion is subjective or not formally defined">open-ended</abbr> tasks

Why does this matter? It's the external verifier that solves Self-Refine's problem, where the model cannot assess its own correctness.

**Self-Reflection model** — an LLM generating verbal reflection. Receives: the Actor's <abbr title="A sequence of actions and observations by the agent during one attempt; passed to the Reflector for error analysis">trajectory</abbr> (actions + observations), the Evaluator's assessment, past reflections from memory. Generates text like: "I made a mistake handling the empty list <abbr title="Edge case — input data at the boundary of valid values (empty list, nil, zero length); a frequent source of bugs">edge case</abbr>. Next time I need to check len() > 0 before accessing an element."

**Episodic Memory** — a store of reflections. Simple structure: a list of text strings injected into the Actor's context on the next attempt. The more attempts — the richer the memory.

## 5. Evolution of self-critique

Three self-critique patterns emerged within 3 months — each solving the previous one's problem:

{{< mermaid align="center" >}}
flowchart TD
    CAI["🛡️ Constitutional AI<br/><b>Dec 2022</b> • Bai/Anthropic<br/>Self-critique → revise<br/>for <b>training</b> (finetuning)<br/>Goal: harmlessness"]
    SRF["🔄 Self-Refine<br/><b>Mar 2023</b> • Madaan/CMU<br/>Same LLM: generate → critique → refine<br/>for <b>inference</b> (1 call)<br/>~20% improvement on generation"]
    REF["🧠 Reflexion<br/><b>Mar 2023</b> • Shinn/Princeton+NEU<br/>Actor → Evaluator → Reflector<br/>for <b>inference</b> (N attempts)<br/>+ Episodic Memory"]
    HUANG["⚠️ Huang et al.<br/><b>Oct 2023</b> • ICLR 2024<br/><i>LLMs Cannot Self-Correct<br/>Reasoning Yet</i><br/>Without external feedback — doesn't work"]
    RRR["🚀 Reflect, Retry, Reward<br/><b>May 2025</b> • Bensal/Writer<br/>RL-trained reflections<br/>1.5B-7B beats 10x models"]

    CAI -->|"added<br/>inference-time<br/>critique"| SRF
    SRF -->|"added<br/>episodic memory<br/>+ external eval"| REF
    REF -->|"showed<br/>limitations<br/>without verifier"| HUANG
    HUANG -->|"RL-training<br/>better reflections"| RRR

    style CAI fill:#2e7d32,color:#fff
    style SRF fill:#1565c0,color:#fff
    style REF fill:#e65100,color:#fff
    style HUANG fill:#c62828,color:#fff
    style RRR fill:#6a1b9a,color:#fff
{{< /mermaid >}}

### Comparison of three patterns

| Aspect | Constitutional AI | Self-Refine | Reflexion |
|--------|-------------------|-------------|-----------|
| **Date** | Dec 2022 | Mar 2023 | Mar 2023 |
| **Goal** | Harmlessness (safety) | Output quality | Output quality + learning |
| **Critique** | Self-critique | Self-feedback | <abbr title="Self-correction with an external signal (tests, compiler, environment); this is the variant Reflexion uses">External evaluator</abbr> + self-reflection |
| **Memory** | No | No | Episodic memory |
| **Attempts** | 1 | 1 (iterations inside) | N (multi-trial) |
| **Application** | Finetuning (offline) | Inference (online) | Inference (online) |
| **Result** | Improved harmlessness | +20% on generation, 0% on reasoning | +11% HumanEval (91% vs 80% GPT-4), +22% AlfWorld |
| **Evaluation type** | RLAIF (RL from AI judge) | Self-judge | External verifier |

The pattern is clear: each subsequent pattern adds what the previous one lacked. Constitutional AI had no memory and no multi-trial capability. Self-Refine added inference-time critique, but without memory and without an external verifier. Reflexion closed the loop: the external verifier solves the unreliable self-assessment problem, and episodic memory enables learning between attempts.

## 6. When it works / when it doesn't

Why a dedicated section on limitations? Because in October 2023, Huang et al. published ["Large Language Models Cannot Self-Correct Reasoning Yet"](https://arxiv.org/abs/2310.01798) — and proved that self-correction without external feedback **degrades** results.

{{< mermaid align="center" >}}
flowchart TD
    START[Agent task] --> Q{External<br/>verifier available?}
    Q -->|Yes| Q2{Low initial<br/>accuracy?}
    Q -->|No| FAIL["❌ Reflexion won't help<br/>Self-correction degrades results<br/>(Huang et al., 2023)"]
    
    Q2 -->|Yes| WORKS["✅ Reflexion works<br/>+11-22% improvement<br/>(HumanEval, AlfWorld)"]
    Q2 -->|No| WARN["⚠️ May not be worth it<br/>Risk of degradation on simple tasks<br/><abbr title="Diminishing returns — each subsequent attempt brings less improvement than the previous one; after 3-5 attempts cost grows while gain is minimal">Diminishing returns</abbr>"]
    
    WORKS --> REC1["Recommendation:<br/>max 3-5 attempts,<br/>evaluate cost/benefit"]
    WARN --> REC2["Recommendation:<br/>no more than 2 attempts,<br/>monitor quality"]

    style FAIL fill:#c62828,color:#fff
    style WORKS fill:#2e7d32,color:#fff
    style WARN fill:#e65100,color:#fff
{{< /mermaid >}}

### Numbers from Huang et al.

Without an external verifier (<abbr title="Self-correction without external feedback — the model evaluates and fixes itself; degrades results on reasoning tasks (Huang et al., 2023)">intrinsic self-correction</abbr>), quality **drops** across all models:

| Model | GSM8K (before → after) | CommonSenseQA (before → after) |
|-------|----------------------|-------------------------------|
| GPT-3.5 | 75.9 → 74.7 | 75.8 → 41.8 |
| GPT-4 | 95.5 → 89.0 | 82.0 → 80.0 |
| Llama-2-70b | 62.0 → 36.5 | 64.0 → 36.5 |

The reason: LLMs are more likely to change a **correct** answer to an **incorrect** one than vice versa. The fundamental problem is that the model cannot reliably assess the correctness of its own reasoning ([Huang et al., 2023](https://arxiv.org/abs/2310.01798), Figure 1).

### When Reflexion works

| Condition | Why |
|-----------|-----|
| External verifier (tests, compiler, env) | Objective assessment → accurate reflection |
| Low initial accuracy | Room for improvement |
| Multi-trial scenario | Memory accumulates lessons |
| Tasks with objective success criterion | Clear error signal |

### When Reflexion does NOT work

| Condition | Why |
|-----------|-----|
| No external verifier | Model can't assess its own correctness |
| High initial accuracy | Risk of degradation (correct → incorrect) |
| <abbr title="Open-ended tasks — tasks without a single correct answer; the success criterion is subjective or not formally defined">Open-ended</abbr> tasks without criteria | Nothing to evaluate → inaccurate reflection |
| Simple tasks | Cost of reflection isn't justified |

## 7. Implementation variants in Eino

Eino doesn't provide a ready-made `reflection.NewAgent()` — unlike [`react.NewAgent()`]({{< ref "ai-agent-design-patterns-1-react" >}}) from Part 1 or [`planexecute.NewAgent()`]({{< ref "ai-agent-design-patterns-2-plan-execute" >}}) from Part 2. But that's actually a good thing: Reflexion is not a separate agent type, but a **composition pattern** that can be implemented in several ways.

### Variant A: compose.Graph with a cycle

Eino Graph supports cycles via `AddEdge` from a node to itself + `WithMaxRunSteps` to limit iterations. This is the most natural implementation of Reflexion:

{{< mermaid align="center" >}}
flowchart LR
    START(("START")) --> Actor["🎬 Actor<br/>(react.Agent)"]
    Actor --> Evaluator["🔍 Evaluator<br/>(Lambda: run tests)"]
    Evaluator --> Branch{"Pass?"}
    Branch -->|Yes| END(("END"))
    Branch -->|No| Reflector["🪞 Reflector<br/>(ChatModel)"]
    Reflector --> Memory["💾 Memory<br/>(State: []string)"]
    Memory --> Actor
    
    style START fill:#2e7d32,color:#fff
    style END fill:#2e7d32,color:#fff
    style Branch fill:#e65100,color:#fff
    style Actor fill:#1565c0,color:#fff
    style Evaluator fill:#1565c0,color:#fff
    style Reflector fill:#6a1b9a,color:#fff
    style Memory fill:#ad1457,color:#fff
{{< /mermaid >}}

**Pros**: explicit retry cycle, branch support, <abbr title="Saving graph execution state between steps; in Eino: WithCheckPointStore allows interrupting and resuming execution">checkpoint</abbr> via `WithCheckPointStore`, can embed a ReAct agent via `ExportGraph()`.

**Cons**: Graph API uses implicit data passing (entire output → entire input), need to manage state carefully.

### Variant B: compose.Workflow (linear, no cycle)

Workflow is a declarative graph with explicit field mapping. Problem: **Workflow does not support cycles** — always `AllPredecessor`. For Reflexion, this is fatal: no retry-loop.

But if the cycle is implemented **externally** (in Go code), and Workflow is used for a single Actor → Evaluator → Reflector iteration — it works:

```go
// External retry-loop
for attempt := 0; attempt < maxAttempts; attempt++ {
    result, err := workflow.Invoke(ctx, input)
    if result.Passed { break }
    memory = append(memory, result.Reflection)
    // Inject memory into the next call
    input.Reflections = memory
}
```

**Pros**: explicit field mapping, type safety, easier to test a single iteration.

**Cons**: no built-in cycle — have to implement manually, no checkpoint between iterations.

### Variant C: deer-go pattern (State Graph)

[deer-go](https://github.com/cloudwego/eino-examples/tree/main/flow/agent/deer-go) is a Go implementation of ByteDance's DEER-flow on Eino Graph. It uses `Goto`-based routing: each node writes the next target node to `state.Goto`, and an `agentHandOff` function directs execution.

For Reflexion, you could add a Critic node and a `Critic → Actor` edge (retry). This extends the existing architecture, but deer-go doesn't contain ready-made reflection patterns — only a re-planning loop.

**Pros**: ready state graph architecture, checkpoint, <abbr title="Human-in-the-loop — a pattern where the system pauses execution and waits for a human decision before continuing; in Eino: InterruptAndRerun">human-in-the-loop</abbr> via `InterruptAndRerun`.

**Cons**: more complex, requires HTTP server and MCP tools, overkill for simple scenarios.

### Variant comparison

| Aspect | Graph + cycle | Workflow + external loop | deer-go |
|--------|-------------|------------------------|---------|
| **Retry cycle** | Built-in | External (Go code) | Via Goto |
| **Checkpoint** | ✅ `WithCheckPointStore` | ❌ Manual | ✅ Built-in |
| **Complexity** | Medium | Low | High |
| **Field mapping** | Implicit | Explicit | Via State |
| **Production readiness** | High | Medium | High |

My choice for the example: **compose.Graph** — native cycle support, checkpoint, and direct embedding of react.Agent via `ExportGraph()`.

## 8. Code example

Implementing Reflexion on compose.Graph: Actor (ReAct agent) generates Go code, Evaluator runs tests, Reflector analyzes errors, Memory accumulates reflections.

{{< mermaid align="center" >}}
flowchart TD
    START(("START")) --> Actor["🎬 Actor<br/>react.Agent<br/>+ write_code tool"]
    Actor --> Evaluator["🔍 Evaluator<br/>Lambda: go test"]
    Evaluator --> Branch{"Tests pass?"}
    Branch -->|Yes| END(("END ✅"))
    Branch -->|No| Reflector["🪞 Reflector<br/>ChatModel: analyze<br/>test failures"]
    Reflector --> Memory["💾 Append reflection<br/>to episodic memory"]
    Memory --> Actor
    
    style START fill:#2e7d32,color:#fff
    style END fill:#2e7d32,color:#fff
    style Branch fill:#e65100,color:#fff
{{< /mermaid >}}

```go
package main

import (
	"context"
	"fmt"

	"github.com/cloudwego/eino/compose"
	"github.com/cloudwego/eino/components/tool"
	"github.com/cloudwego/eino/components/tool/utils"
	"github.com/cloudwego/eino-ext/components/model/openai"
	"github.com/cloudwego/eino/flow/agent/react"
	"github.com/cloudwego/eino/schema"
)

// reflexionState — shared state for a single Reflexion execution.
// Created fresh with each Invoke.
type reflexionState struct {
	// Task — the original task (e.g., "write a function that sorts a list")
	Task string
	// Reflections — accumulated verbal reflections from past attempts
	Reflections []string
	// Attempt — current attempt number (1-based)
	Attempt int
	// MaxAttempts — maximum number of attempts
	MaxAttempts int
	// Code — generated code (Actor output)
	Code string
	// TestResult — test run result (Evaluator output)
	TestResult string
	// Passed — flag: did tests pass?
	Passed bool
}

// writeCodeTool — Actor's tool: "writes" code to a file.
// In a real application, this would write to disk.
type writeCodeTool struct{}

func (t *writeCodeTool) Info(ctx context.Context) (*schema.ToolInfo, error) {
	return &schema.ToolInfo{
		Name: "write_code",
		Desc: "Write Go code to solve the task. The code will be tested automatically.",
	}, nil
}

func (t *writeCodeTool) InvokableRun(ctx context.Context, args string, opts ...tool.Option) (string, error) {
	// In a real application: write args to a .go file
	return fmt.Sprintf("Code written (%d bytes)", len(args)), nil
}

func main() {
	ctx := context.Background()

	// 1. Create model for Actor and Reflector
	chatModel, err := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		Model: "gpt-4o",
	})
	if err != nil {
		panic(err)
	}

	// 2. Create ReAct agent as Actor
	//    ExportGraph() allows embedding it into compose.Graph
	codeTool := utils.NewTool(&writeCodeTool{}, nil)
	actor, err := react.NewAgent(ctx, &react.AgentConfig{
		ToolCallingModel: chatModel,
		ToolsConfig: compose.ToolsNodeConfig{
			Tools: []tool.BaseTool{codeTool},
		},
		MaxStep: 5,
	})
	if err != nil {
		panic(err)
	}

	// 3. Build the Reflexion Graph
	g := compose.NewGraph[string, string](
		compose.WithGenLocalState(func(ctx context.Context) *reflexionState {
			return &reflexionState{
				MaxAttempts: 3,
			}
		}),
		compose.WithMaxRunSteps(20), // cycle limit
	)

	// Actor: embed ReAct agent via ExportGraph
	actorGraph, actorOpts := actor.ExportGraph()
	g.AddGraphNode("actor", actorGraph, actorOpts...)
	g.AddEdge(compose.START, "actor")

	// Evaluator: Lambda node that "runs tests"
	g.AddLambdaNode("evaluator",
		compose.InvokableLambda(func(ctx context.Context, code string) (string, error) {
			// In a real application: exec.Command("go", "test", "./...")
			// Here: simulation
			if len(code) > 10 {
				return "PASS: all tests passed", nil
			}
			return "FAIL: TestSortEmpty - expected [], got nil", nil
		}),
	)
	g.AddEdge("actor", "evaluator")

	// Reflector: ChatModel analyzes errors
	g.AddChatModelNode("reflector", chatModel)
	g.AddEdge("evaluator", "reflector")

	// Conditional Branch: pass → END, fail → back to Actor
	g.AddBranch("evaluator", compose.NewGraphBranch(
		func(ctx context.Context, testResult string) (string, error) {
			// Read state to check attempt count
			if err := compose.ProcessState[*reflexionState](ctx,
				func(ctx context.Context, s *reflexionState) error {
					s.Attempt++
					s.TestResult = testResult
					// Simple heuristic criterion
					s.Passed = len(testResult) > 4 && testResult[:4] == "PASS"
				},
			); err != nil {
				return "", err
			}

			var target string
			_ = compose.ProcessState[*reflexionState](ctx,
				func(ctx context.Context, s *reflexionState) error {
					if s.Passed || s.Attempt >= s.MaxAttempts {
						target = compose.END
					} else {
						target = "reflector" // reflect first, then retry
					}
					return nil
				},
			)
			return target, nil
		},
		map[string]bool{compose.END: true, "reflector": true},
	))

	// Reflector → Actor: retry with reflection in context
	g.AddEdge("reflector", "actor")

	// END
	g.AddEdge(compose.END, compose.END)

	// 4. Compile and run
	runnable, err := g.Compile(ctx)
	if err != nil {
		panic(fmt.Sprintf("compile error: %v", err))
	}

	result, err := runnable.Invoke(ctx, "Write a function that sorts a slice of integers")
	if err != nil {
		panic(fmt.Sprintf("invoke error: %v", err))
	}

	fmt.Println("Result:", result)
}
```

### What's happening here

1. **State** — `reflexionState` stores the current attempt, reflections, and evaluation result. Created via `WithGenLocalState` on each graph run.

2. **Actor** — embedded via `ExportGraph()`. A ReAct agent with a `write_code` tool. On re-entry (after reflection), it receives updated context.

3. **Evaluator** — a Lambda node that "runs tests". In a real application — `exec.Command("go", "test")`. In the example — simulation: if the code is longer than 10 bytes — PASS.

4. **Reflector** — a ChatModel analyzing errors. Receives test results and generates verbal reflection.

5. **Branch** — conditional branching after Evaluator: PASS → END, FAIL → Reflector → Actor (retry). Checks `Attempt < MaxAttempts`.

6. **Cycle** — `g.AddEdge("reflector", "actor")` closes the loop. `WithMaxRunSteps(20)` limits the total number of steps (protection against infinite loops).

## 9. Engineering scenario

Code generation with tests is the ideal scenario for Reflexion. Why? Because there's an **objective external verifier**: the compiler and unit tests. This isn't a subjective LLM assessment — it's a binary PASS/FAIL.

### <abbr title="Test-Driven Development — write tests first, then code that passes them; in the agent context: tests serve as the Evaluator">TDD</abbr> for agents

{{< mermaid align="center" >}}
sequenceDiagram
    participant Dev as Developer
    participant A as Actor Agent
    participant T as go test
    participant R as Reflector
    participant M as Memory

    Dev->>A: "Write sort([]int)"
    
    Note over A: Attempt 1
    A->>T: sort.go
    T-->>A: ❌ FAIL: TestSortEmpty<br/>expected [], got nil
    
    A->>R: Trajectory + test failure
    R->>R: "Forgot to handle<br/>empty slice"
    R->>M: Store reflection #1
    
    Note over A: Attempt 2 (with reflection)
    M-->>A: "Check empty slice first"
    A->>T: sort_v2.go
    T-->>A: ❌ FAIL: TestSortStable<br/>unstable sort on equal elements
    
    A->>R: Trajectory + test failure
    R->>R: "Used unstable sort,<br/>need stable"
    R->>M: Store reflection #2
    
    Note over A: Attempt 3 (with 2 reflections)
    M-->>A: "Check empty slice first<br/>Use stable sort"
    A->>T: sort_v3.go
    T-->>A: ✅ All tests passed
    
    A-->>Dev: sort_v3.go ✅
{{< /mermaid >}}

### Why this works better than Self-Refine

Self-Refine in the same scenario would give **0%** improvement on Math Reasoning ([Madaan et al., 2023](https://arxiv.org/abs/2303.17651)). Why? Without tests, the model can't distinguish `sort([]int{})` from `sort([]int{1})` — it lacks an objective signal. Reflexion with `go test` as Evaluator solves this problem: tests provide precise error diagnostics → reflection focuses on the real problem → the Actor fixes exactly what's needed.

### Production implementation

In production, the scenario expands:

| Component | Example | In production |
|-----------|---------|---------------|
| Actor | `react.NewAgent` with `write_code` | + `read_file`, `search_docs`, `lint_code` |
| Evaluator | `go test ./...` | + `go vet`, `golangci-lint`, coverage ≥ 80% |
| Reflector | ChatModel "analyze failures" | Prompt with specific error patterns |
| Memory | `[]string` in state | Redis / file with reflection history |
| Max attempts | 3 | 5 (HumanEval: 91% achieved in 12 attempts, but 3-5 usually sufficient) |

## 10. Practical recommendations

### When to apply Reflexion

| Scenario | Applicability | Rationale |
|----------|--------------|-----------|
| Code generation + tests | ✅ Excellent | Objective verifier (compiler/tests) |
| Game agents | ✅ Excellent | Environment provides clear <abbr title="Reinforcement signal — a numeric evaluation of an agent's action by the environment; in RL: reward for successful/unsuccessful action">reward signal</abbr> |
| Data pipeline with validation | ✅ Good | Schema validation as verifier |
| Code review automation | ⚠️ Cautious | LLM assessment less reliable than tests |
| Creative writing | ⚠️ Cautious | No objective success criterion |
| Math / reasoning | ❌ Not recommended | Without external verifier — degrades results |

### Hyperparameter tuning

**Max attempts**: 3-5 for tasks with a fast verifier (tests). HumanEval reached 91% in 12 attempts, but <abbr title="Diminishing returns — each subsequent attempt brings less improvement than the previous one; after 3-5 attempts cost grows while gain is minimal">diminishing returns</abbr> start after 3-5. More is more expensive, but not better.

**Episodic memory size**: store the last 5-10 reflections. Too many — the context grows and the model loses focus. Too few — doesn't account for older mistakes.

**Evaluator choice**: a deterministic verifier (tests, compiler) is always better than LLM-based. If the verifier is unreliable — Reflexion degrades into Self-Refine with its problems.

**Reflector prompt**: specific > abstract. Not "analyze the error", but "identify: (1) which test failed, (2) what input caused the failure, (3) what assumption was wrong, (4) what to change in the code".

### Cost management

Each Reflexion attempt = a full Actor + Evaluator + Reflector cycle. With 3 attempts — 3x cost. <abbr title="Mitigation — measures to reduce the negative effect; strategies for lowering cost or risk">Mitigations</abbr>:
- Use a cheap model for the Evaluator (deterministic checking doesn't require GPT-4)
- Stop early: if 2 attempts didn't help — the third probably won't either
- Cache reflections for similar tasks

## 11. Summary

The Reflexion pattern is ReAct + self-assessment + episodic memory. Key takeaways:

1. **External verifier is mandatory**. Without it, self-correction degrades results ([Huang et al., 2023](https://arxiv.org/abs/2310.01798)). With it — Reflexion delivers +11% on HumanEval and +22% on AlfWorld.

2. **Episodic memory is the key difference from Self-Refine**. Not just <abbr title="A critique-then-refine cycle: LLM evaluates its output and regenerates it considering the feedback">critique-refine</abbr>, but accumulating verbal lessons between attempts. This transforms a one-shot agent into a learning one.

3. **Not a silver bullet**. Reflexion doesn't work on tasks without an objective success criterion and can degrade results when initial accuracy is already high.

4. **In Eino — compose.Graph with a cycle**. Workflow doesn't work (no cycles). Graph + `AddEdge("reflector", "actor")` + `WithMaxRunSteps` — the natural implementation.

What's next? In [Part 4]({{< ref "ai-agent-design-patterns-4-multi-agent" >}}) — Multi-Agent patterns: when one agent isn't enough, and you need a team. And the topic of agent memory — long-term, episodic, semantic — I'll cover in detail in Part 6.

{{< collapse title="📚 References" >}}

- **Reflexion**: Shinn, N., Cassano, F., Labash, A., Gopinath, A., Narasimhan, K., & Yao, S. (2023). [Reflexion: Language Agents with Verbal Reinforcement Learning](https://arxiv.org/abs/2303.11366). NeurIPS 2023.
- **Self-Refine**: Madaan, A., Tandon, N., Gupta, P., Hallinan, S., Gao, L., Wiegreffe, S., ... & Yang, D. (2023). [Self-Refine: Iterative Refinement with Self-Feedback](https://arxiv.org/abs/2303.17651). NeurIPS 2023.
- **Constitutional AI**: Bai, Y., Kadavath, S., Kundu, S., Askell, A., Kernion, J., Jones, A., ... & Kaplan, J. (2022). [Constitutional AI: Harmlessness from AI Feedback](https://arxiv.org/abs/2212.08073). Anthropic.
- **Huang et al.**: Huang, J., Chen, X., Mishra, S., Zheng, H. S., Yu, A. W., Song, Q., & Zhou, D. (2023). [Large Language Models Cannot Self-Correct Reasoning Yet](https://arxiv.org/abs/2310.01798). ICLR 2024.
- **Reflect, Retry, Reward**: Bensal, Y., Kim, S., Bosselut, A., & Guha, N. (2025). [Reflect, Retry, Reward: Training LLM Agents to Reflect and Retry with Reward-Guided Self-Reflection](https://arxiv.org/abs/2505.24726).
- **Eino**: CloudWeGo. [Eino: The ultimate LLM/AI application development framework in Go](https://github.com/cloudwego/eino).
- **deer-go**: CloudWeGo. [DEER-flow Go implementation](https://github.com/cloudwego/eino-examples/tree/main/flow/agent/deer-go).

{{< /collapse >}}
