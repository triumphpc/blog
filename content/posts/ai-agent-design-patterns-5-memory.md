---
title: "AI Agent Design Patterns. Part 5: Agent Memory Management"
date: 2026-06-08
draft: false
tags: ["ai", "agents", "llm", "memory", "eino", "go"]
categories: ["ai", "engineering"]
summary: "Context window ≠ memory. I break down agent amnesia, the academic taxonomy (Du et al., 2026), five approaches to memory — from OpenClaw's file-based brain to mem0's managed layer — and how our team solves this in practice."
ShowToc: true
series: ["AI Agent Design Patterns"]
---

## 1. The Problem: Agent Amnesia

In [Part 1]({{< ref "ai-agent-design-patterns-1-react" >}}), I covered the <abbr title="Reasoning + Acting — pattern where LLM alternates between reasoning and acting">ReAct</abbr> agent. In [Part 3]({{< ref "ai-agent-design-patterns-3-reflexion" >}}) — Reflexion, where the agent learns from mistakes through episodic memory. Four patterns, four sessions — and none solves a fundamental problem: **the context window is not memory**.

Picture this: your debug assistant on Friday found that a middleware bug was a race condition on a shared cache. On Monday, you ask the same agent about a similar bug — and it starts from scratch. It doesn't remember the conclusion, the context, or even that the cache exists. Every Monday is Groundhog Day.

And this isn't exotic. It's the norm. <abbr title="Large Language Model">LLMs</abbr> are stateless by nature: process tokens, return result, forget. The context window — 128K, 200K, even 1M tokens — is always finite. Session ends, window clears, agent knows nothing about your project.

But humans don't need to remember everything either — they store what matters and forget the rest. An agent doesn't need infinite memory, it needs **the right** memory. What to save? How to compress? When to surface into context? These are the questions I'll tackle.

## 2. A Taxonomy of Agent Memory

The systematic survey by Du et al. ([Memory for Autonomous LLM Agents, 2026](https://arxiv.org/abs/2603.07670)) formalizes agent memory as a **write-manage-read loop**:

{{< mermaid >}}
graph LR
    W["✏️ WRITE<br>What & when to save"] --> M["🔧 MANAGE<br>Compression, deduplication,<br>conflict resolution"]
    M --> R["📖 READ<br>What & when to<br>surface into context"]
    R -->|"new experience"| W
{{< /mermaid >}}

Three phases, three fundamentally different engineering decisions:

- **Write**: not everything is worth saving. "User asked about weather" — noise. "User prefers YAML over JSON" — fact. Filtering is essential.
- **Manage**: facts go stale, duplicate, contradict each other. "Bob leads the ML team" and "Alice manages the ML team" — you need a resolution mechanism.
- **Read**: surface the right thing at the right time. Not the entire archive, just what's relevant. And fit it within the token budget.

Du identifies three dimensions for classification: **temporal scope** (short-term / long-term), **representation** (text / vectors / graphs), and **control policy** (fixed rules / learnable). Combinations yield five mechanism families — from context-resident compression to policy-learned management. But in practice, engineers choose from ready-made solutions. And there are surprisingly many.

## 3. Approach 1: File-First Brain (OpenClaw)

[OpenClaw](https://github.com/mem0ai/openclaw) — a project from the team behind [mem0](https://github.com/mem0ai/mem0) (58K ⭐ on GitHub, YC S24), a popular managed memory layer for AI agents. If mem0 externalizes memory into a service, OpenClaw goes to the opposite extreme — a radical idea: **the filesystem = the agent's brain**.

The agent stores everything in markdown files:

| File | Purpose | Size |
|------|---------|------|
| `SOUL.md` | Identity: who I am, my values | ~2K tokens |
| `AGENTS.md` | Procedures: how I work | ~3K tokens |
| `MEMORY.md` | Curated long-term memory | ~5K tokens |
| `memory/YYYY-MM-DD.md` | Raw daily logs | No limit |

Every session starts with a boot sequence — the agent reads SOUL.md, AGENTS.md, MEMORY.md, and recent daily logs. This is its "morning coffee": 4K–10K tokens just to start, before it says a single word.

{{< mermaid >}}
graph TB
    START["🚀 New Session"] --> SOUL["📖 SOUL.md<br>identity"]
    SOUL --> AGENTS["📖 AGENTS.md<br>procedures"]
    AGENTS --> MEMORY["📖 MEMORY.md<br>curated facts"]
    MEMORY --> LOGS["📖 daily logs<br>recent entries"]
    LOGS --> READY["✅ Agent Ready"]

    style START fill:#1565c0,color:#fff
    style READY fill:#2e7d32,color:#fff
{{< /mermaid >}}

Why files, not a vector database? OpenClaw's philosophical argument: **RAG is for looking up information; an agent needs a brain**. A vector DB is fragmented — semantic search returns chunks without context. Files are whole, natively readable by the agent, human-editable.

But there's a cost. 10K tokens per boot is 10K tokens not used for the actual task. At 5M tokens/day (typical OpenClaw burn), this is a serious budget. And the bigger MEMORY.md grows, the more expensive each session becomes.

## 4. Approach 2: Structured Compression (MemPalace)

[MemPalace](https://github.com/milla-jovovich/mempalace) solves OpenClaw's main problem — token crushing. The idea: **don't load everything upfront; surface on demand**.

Architecture is a Memory Palace hierarchy: Wing (project/person) → Room (sub-topic) → Hall (memory type: facts, events, discoveries, preferences, advice). At each level — a compressed description that LLM reads natively. Compression isn't magic AAAK — it's plain truncation: snippets are cut to 200–300 characters and grouped by `room`.

Four memory layers ([layers.py](https://github.com/milla-jovovich/mempalace/blob/develop/mempalace/layers.py)):

| Layer | What it stores | How it works | Tokens |
|-------|---------------|--------------|--------|
| **L0 — Identity** | Who I am, my principles, key people | Reads `~/.mempalace/identity.txt` (user-written) | ~100 |
| **L1 — Essential Story** | Top-15 important moments from the entire palace | Scans up to 2000 drawers, ranks by `importance`/`emotional_weight`/`weight`, groups by `room`, truncates to 3200 chars | ~500–800 |
| **L2 — On-Demand** | Filtered retrieval by wing/room | Metadata filter in ChromaDB (not semantic!), up to N drawers | ~200–500 per call |
| **L3 — Deep Search** | Full semantic search | `col.query(query_texts=...)` across the entire palace with similarity ranking | Unlimited |

Startup — only L0 + L1, ~600–900 tokens. L2 and L3 surface via <abbr title="Model Context Protocol — open protocol for integrating LLMs with external tools and data">MCP</abbr> tools when the agent encounters a task requiring context.

{{< mermaid >}}
graph TB
    BOOT["🚀 wake-up<br>~600-900 tokens"] --> L0["L0: Identity<br>~100 tokens<br>identity.txt file"]
    BOOT --> L1["L1: Essential Story<br>~500-800 tokens<br>top-15 drawers by weight"]
    L0 --> L2["L2: On-Demand<br>~200-500 tokens<br>filter by wing/room"]
    L1 --> L2
    L2 --> L3["L3: Deep Search<br>unlimited<br>semantic search"]
    L3 --> WING["🏰 Wing<br>project / person"]
    WING --> ROOM["🏠 Room<br>sub-topic"]
    ROOM --> HALL["🚪 Hall<br>memory type"]
    
    style BOOT fill:#2e7d32,color:#fff
    style L2 fill:#e65100,color:#fff
    style L3 fill:#c62828,color:#fff
{{< /mermaid >}}

Bonus: Knowledge Graph with **temporal validity** — facts have expiration dates. "Bob leads the ML team" is valid until a certain date. When Alice becomes the lead — the fact automatically expires. This solves the conflict resolution problem from the write-manage-read loop.

Result: 96.6% R@5 on LongMemEval (<abbr title="Long-Term Memory Evaluation — benchmark for evaluating long-term agent memory; the agent is given a series of dialogues, then tested on which facts it remembers">long-term memory benchmark</abbr>) at ~600–900 tokens per wake-up (L0 + L1). Versus 10K for OpenClaw. A 10–15x difference — MemPalace recalls better while spending an order of magnitude less context on boot. And this is without an LLM at search time: pure ChromaDB, pure semantic search, zero API calls.

## 5. Approach 3: Managed Memory Layer

The third approach — externalize memory into a separate service. The agent stores nothing itself; it requests context from an external layer.

**mem0** ([GitHub](https://github.com/mem0ai/mem0), 58K ⭐) — the "universal memory layer." A managed service (YC S24) that automatically extracts facts from conversations, deduplicates, and returns relevant context on request. New algorithm (April 2026): 94.8% on LongMemEval at 6.8K tokens. Pros: no need to think about write/manage — the service does it all. Cons: vendor lock-in, dependency on an external API.

Now **Zep** is a different beast. It's an open-source memory server, and worth a closer look because it solves a problem mem0 doesn't: the structure of relationships between facts.

Picture this: the agent knows "Bob works on the ML team" and "The ML team uses <abbr title="PyTorch — popular deep learning framework for Python, developed by Meta">PyTorch</abbr>." A vector DB (like mem0's) will find each fact separately — but won't infer that Bob likely works with PyTorch. Zep adds a <abbr title="Knowledge Graph — stores entities (Bob, ML team, PyTorch) and relationships (works_on, uses), enabling logical inference">Knowledge Graph</abbr> on top of vectors — powered by its Graphiti engine. Entities and relationships are extracted automatically from conversations, and namespace-based isolation separates different users' contexts.

Why are we considering it? Because in real projects, facts don't live in a vacuum — they're connected. "Client X switched to plan Y" and "Plan Y doesn't support feature Z" — the agent should draw a conclusion, not just return both facts. A graph makes that possible.

But there's a cost. Zep requires infrastructure: vector DB + graph + embedding service. Not one service, but a stack. If you don't have DevOps capacity — deployment will hurt.

**LangMem** ([docs](https://langchain-ai.github.io/langmem/), by LangChain) — SDK with three memory types: **semantic** (facts), **episodic** (past experiences), **procedural** (evolving behavior). Procedural memory is unique: the agent updates its own prompt based on feedback, "learning" to behave better. Pros: native LangGraph integration, namespace isolation for privacy. Cons: tied to the LangChain ecosystem.

What's common? All three are **middleware**: they sit between the agent and the LLM, intercept context, and enrich it with relevant facts. Differences lie in storage (vectors vs. graph vs. prompt), management (auto vs. curated), and deployment (SaaS vs. self-hosted vs. embedded).

## 6. How We Solve It

Our team is tackling this exact problem — designing memory for AI agents on our platform. And the architecture we've arrived at looks suspiciously like a hybrid of OpenClaw and MemPalace.

Four memory layers:

{{< mermaid >}}
graph TB
    SESSION["🔄 Session Layer<br>current dialogue,<br>auto-cleanup"]
    AGENT["🤖 Agent Layer<br>SOUL.md, ABOUT.md,<br>character & profile"]
    USER["👤 User Layer<br>preferences,<br>user context"]
    PROJECT["📁 Project Layer<br>AGENTS.md, TOOLS.md,<br>workspace + Virtual FS"]
    
    SESSION --> AGENT --> USER --> PROJECT
    
    style SESSION fill:#1565c0,color:#fff
    style AGENT fill:#6a1b9a,color:#fff
    style USER fill:#e65100,color:#fff
    style PROJECT fill:#2e7d32,color:#fff
{{< /mermaid >}}

Runtime context assembly: on every request, the system builds context layer by layer — from session (cheapest, always in window) to project (most expensive, loaded on demand). Like MemPalace: cheap startup, detail on demand.

The agent's file structure is directly inspired by OpenClaw:

| File | OpenClaw Analog | Purpose |
|------|-----------------|---------|
| `SOUL.md` | SOUL.md | Agent's character |
| `ABOUT.md` | — | Profile/description |
| `AGENTS.md` | AGENTS.md | Agent creation instructions |
| `TOOLS.md` | — | Auto-assembly from skills + MCP |

From MemPalace, we adopted **layered loading**: don't load everything into context at once, surface layers as needed. And **temporal validity** — facts have expiration dates; stale ones don't make it into context. But what does this mean in practice?

### Layered loading: how it works for us

MemPalace has four layers ([layers.py](https://github.com/milla-jovovich/mempalace/blob/develop/mempalace/layers.py)), each solving a specific problem:

- **L0 — Identity** (~100 tokens). Plain-text file `~/.mempalace/identity.txt`, written by the user. "I am Atlas, a personal AI assistant for Alice. Traits: warm, direct. People: Alice (creator), Bob (Alice's partner). Project: A journaling app." This is a constant — doesn't change between sessions, not computed, just read from disk.

- **L1 — Essential Story** (~500–800 tokens). Auto-generated from the palace: scans up to 2000 drawers (memory chunks), ranks by weight (`importance`, `emotional_weight`, `weight`), takes top-15, groups by `room` for readability, truncates to 3200 characters. This is "the most important things that happened" — not the full archive, but a digest. The algorithm isn't perfect: e.g., a drawer with high `emotional_weight` might crowd out a more useful but less "emotional" fact. But for the boot task — giving the agent minimal context — it works.

- **L2 — On-Demand** (~200–500 tokens per call). Filtered retrieval: "give me everything from wing=driftwood, room=auth-migration." This is a metadata filter in ChromaDB, not semantic search — simply `WHERE wing = ? AND room = ?`. Surfaces when a specific topic comes up in conversation.

- **L3 — Deep Search** (unlimited). Full semantic search: `col.query(query_texts=["why did we switch to GraphQL"])` across the entire palace. Returns results with similarity ranking. This is the heavy artillery — when L1 and L2 didn't provide enough context.

Startup — `wake_up()` → L0 + L1, ~600–900 tokens. L2 and L3 surface via <abbr title="Model Context Protocol — open protocol for integrating LLMs with external tools and data">MCP</abbr> tools when the agent encounters a task requiring context.

We adapted this model, but with a key difference: MemPalace is an external service, while our layers are assembled **inside the agent's runtime**. No need to call MCP for memory — context is already assembled by the time the LLM starts generating.

What the assembly looks like on each request:

| Layer | What's loaded | When | Tokens |
|-------|--------------|------|--------|
| Session | Current dialogue | Always | 0 (already in window) |
| Agent | SOUL.md + ABOUT.md | Always | ~1K |
| User | Preferences, context | Always (for current user) | ~500 |
| Project | AGENTS.md + TOOLS.md + workspace | On demand (first request to project) | ~2–5K |

Total at startup: ~1.5K tokens. More than MemPalace (~600–900), but an order of magnitude less than OpenClaw (~10K). A compromise.

Why load Project on demand? Imagine: a user enters a chat, says hello. The agent doesn't know which project the work will involve. Why spend 5K tokens loading AGENTS.md and TOOLS.md for a project that might never be needed? So: wait until the task requires project context — and only then surface the layer.

And how does our L1 analog work — ranking "the most important"? We don't have one. SOUL.md and ABOUT.md are themselves L0/L1, curated manually. No automatic ranking from 2000 drawers, but no risk of an algorithm with high `emotional_weight` crowding out a useful but "boring" fact either. For a platform with hundreds of agents, this is a deliberate choice: predictability over automation.

### Temporal validity: facts with expiration dates

This is the second idea from MemPalace we ported. In MemPalace, the Knowledge Graph is built on SQLite: each fact is a triple `(subject, predicate, object)` with `valid_from` and `valid_to` fields ([knowledge_graph.py](https://github.com/milla-jovovich/mempalace/blob/main/mempalace/knowledge_graph.py)). When a fact becomes stale, it's not deleted — instead, `valid_to` is set. This enables queries like "what was true on date X?" via filtering: `valid_from <= X AND (valid_to >= X OR valid_to IS NULL)`.

```python
# MemPalace: adding a temporal fact
kg.add_triple("Kai", "works_on", "Orion", valid_from="2025-06-01")

# Kai leaves Orion
kg.invalidate("Kai", "works_on", "Orion", ended="2026-03-01")

# Query: what's true now?
kg.query_entity("Kai")
# → [Kai → works_on → Orion (ended), Kai → recommended → Clerk]

# Query: what was true on Jan 20, 2026?
kg.query_entity("Kai", as_of="2026-01-20")
# → [Kai → works_on → Orion (active)]
```

Why is this needed? The most common memory problem isn't missing facts — it's **stale** facts. The agent remembers "the project uses REST API," but the team migrated to gRPC three months ago. Instead of the right answer, the agent drags a false fact into context. Temporal validity solves this: every fact has an expiration date, expired ones are automatically excluded from context.

Our implementation is simpler than MemPalace's: instead of a separate SQLite graph, we use metadata in the Virtual FS. Each agent memory file (MEMORY.md, ABOUT.md) has `last_modified` — and if a fact hasn't been updated longer than TTL (configurable), it's marked stale and excluded from context. No full graph query with `as_of`, but for our use case (a platform with hundreds of agents, not a single coding assistant) this is sufficient. Graph queries are Zep's territory, and if we need logical inference over a fact graph, we know where to look.

### Runtime architecture: from request to context

Above I described abstract memory layers. Here's what it looks like at runtime — at the code level.

The main orchestrator is `Chat UseCase`. Every user request goes through an 8-stage pipeline:

{{< mermaid >}}
graph TB
    REQ["📥 User Request"] --> PREP["1️⃣ prepareSession<br>Load Session + AgentVersion"]
    PREP --> REG["2️⃣ RegisterInAgent<br>Bind tools to agent"]
    REG --> PRE["3️⃣ ExecutePreAgent<br>RAG pre-search"]
    PRE --> ASM["4️⃣ Context Assembly<br>Session[] + SystemPrompt + Tool Injections"]
    ASM --> LLM["5️⃣ LLM react-loop<br>eino/adk Runner"]
    LLM -->|"needs context"| TOOLS["6️⃣ Tool Execution<br>Workspace files / RAG / Skills"]
    TOOLS -->|"loaded into context"| LLM
    LLM -->|"done"| POST["7️⃣ PostAgent + Hooks<br>cleanup + side effects"]
    POST --> SAVE["8️⃣ Update Session<br>Context[]"]

    style REQ fill:#1565c0,color:#fff
    style TOOLS fill:#e65100,color:#fff
    style SAVE fill:#2e7d32,color:#fff
{{< /mermaid >}}

Key insight: context is assembled **before** the LLM starts generating. By the time `runner.Run()` is called, everything is already assembled — session, system prompt, tool injections. The LLM receives a ready-made context and works with it.

But this doesn't mean all memory is loaded at once. SOUL.md and ABOUT.md are already in the system prompt — this is "hot" memory. Files from Workspace the agent requests itself during the react-loop (step 5→6 on the diagram), when it realizes it lacks context. This is on-demand loading from MemPalace — except instead of <abbr title="Model Context Protocol — open protocol for integrating LLMs with external tools and data">MCP</abbr> tools we use eino tools.

Five Workspace scopes and their S3 mapping:

| Scope | Virtual Path | S3 Prefix | Used by |
|-------|-------------|-----------|---------|
| `session` | `/storage/session/...` | `projects/{pid}/sessions/{sid}/workspace/` | Current dialogue artifacts |
| `user` | `/storage/user/...` | `projects/{pid}/users/{uid}/workspace/` | User files and preferences |
| `agent_user` | `/storage/agent_user/...` | `projects/{pid}/agents/{aid}/users/{uid}/workspace/` | Agent-user pair memory |
| `skills` | `/storage/skills/...` | `projects/{pid}/skills/` | SKILL.md, project documents |
| `hooks` | `/storage/hooks/...` | `projects/{pid}/hooks/` | Hook scripts (post-agent) |

And unlike both OpenClaw and MemPalace, we don't need an external memory service — everything runs inside the platform via Workspace (Virtual FS → S3) and Session Context (PostgreSQL).

Difference from OpenClaw: we don't store raw daily logs. Instead — curated memory with automatic compression. Difference from mem0: no external service dependency — everything runs inside the platform via Virtual FS and S3 mount.

## 7. Takeaways

Five approaches to agent memory — from file-based brain to managed service:

| Approach | Project | Boot cost | Storage | Management | Best for |
|----------|---------|-----------|---------|------------|----------|
| File-First | OpenClaw | ~10K tokens | Markdown files | Manually curated | Coding agents, full control |
| Structured Compression | MemPalace | ~600–900 tokens | Hierarchy + ChromaDB | Auto (ranking + temporal) | Multi-project agents, strict budget |
| Managed Layer | mem0 | ~6.8K tokens | Vector DB | Auto (SaaS) | Quick start, no infra overhead |
| Vector + Graph | Zep | Depends on query | Vectors + Knowledge Graph | Auto (self-hosted) | Structured relationships, privacy |
| Embedded SDK | LangMem | Depends on type | LangGraph store | Auto + procedural memory | LangChain ecosystem |

No silver bullet. OpenClaw gives maximum control but burns tokens. MemPalace is elegant but requires structural discipline. mem0 is easy to integrate but vendor-locked. Zep is powerful for graph relationships but heavy to deploy. LangMem is perfect for LangChain but useless outside it.

We chose a hybrid: memory layers like MemPalace, file structure like OpenClaw, runtime assembly instead of static boot. Because our task isn't a coding agent or a chatbot — it's a platform where agents of different types live and work together. And memory must serve that diversity.

---

**Previous article in the series:** [Part 4: Multi-Agent Patterns]({{< ref "ai-agent-design-patterns-4-multi-agent" >}})
