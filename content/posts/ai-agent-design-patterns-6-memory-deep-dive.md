---
title: "AI Agent Design Patterns. Part 6: Agent Memory in Practice"
date: 2026-06-11
draft: false
tags: ["ai", "agents", "llm", "memory", "eino", "go"]
categories: ["ai", "engineering"]
summary: "Deep dive into agent memory mechanics: five memory files, four context assembly strategies, TOOLS.md as a dual-source responsibility, and Dreaming — background consolidation from Anthropic (Auto Dream, Dreams API, KAIROS) and OpenAI (Dreaming V3, 82.8% factual recall). Engineering decisions behind a production platform."
ShowToc: true
series: ["AI Agent Design Patterns"]
---

In [Part 5]({{< ref "ai-agent-design-patterns-5-memory" >}}) I surveyed approaches to agent memory: from OpenClaw's file-based brain to mem0's managed layer. We chose a hybrid — MemPalace-style memory layers, OpenClaw-style file structure. But a survey is one thing, and engineering mechanics is quite another. How exactly do files end up in context? Why are some always injected and others only on demand? How does the agent even know it needs to read something? And what happens when memory turns into a dumpster — who cleans it up?

This article is a deep dive into the mechanics. Five memory files, four context assembly strategies, TOOLS.md with two sources of responsibility, and — the most interesting part — Dreaming: background memory consolidation that Anthropic and OpenAI both shipped in 2026. I'll show the engineering decisions made behind the scenes, why we chose this particular path, and when we'll need to change it.

## 1. Five Memory Files: Complete Catalog

In the first part I described four memory layers and mentioned SOUL.md, ABOUT.md, TOOLS.md. But the real file model is richer. Five files, each with its own responsibility zone, its own way of entering context, and its own isolation scope.

### 1.1. Exhaustive Table

| # | File | Purpose | Scope | When it enters context | How | Who writes |
|---|------|---------|-------|----------------------|-----|-----------|
| 1 | **SOUL.md** | Agent identity and personality: values, communication style, persona | `agent` | Always (every request) | Bootstrap injection into system prompt | User via UI / Agent via memory tools |
| 2 | **ABOUT.md** | Agent profile and description: role, specialization, tasks | `agent` | Always (every request) | Bootstrap injection into system prompt | User via UI / Agent via memory tools |
| 3 | **TOOLS.md** | User notes on tools: quirks, examples, limitations | `agent` | Always (every request) | Bootstrap injection (merged with auto-generated) | User via UI / Agent via memory tools. **Lazy lifecycle**: file is not created by default |
| 4 | **USER.md** | User profile: name, preferences, work context — for a specific agent | `agent_user` | On demand (agent decides) | Through memory tools | Agent via memory tools / User via chat |
| 5 | **MEMORY.md** | Persistent memory: facts, decisions, dialog context for a specific agent+user pair | `agent_user` | On demand (agent decides) | Through memory tools | Agent via memory tools (auto-capture) |

The key distinction is **injection** vs **on-demand**. Three files (SOUL, ABOUT, TOOLS) are injected into the system prompt on every request. This is "hot" memory — the agent always sees its identity and tool descriptions. Two files (USER, MEMORY) are on-demand, through memory tools. The agent decides for itself when it needs user context.

{{< mermaid >}}
graph TB
    subgraph "Bootstrap Injection (~1-2K tokens, EVERY request)"
        SOUL["SOUL.md<br>identity"]
        ABOUT["ABOUT.md<br>profile"]
        TOOLS["TOOLS.md<br>tool notes"]
    end

    subgraph "On-demand (via memory tools, agent decides)"
        USER["USER.md<br>user profile"]
        MEMORY["MEMORY.md<br>long-term memory"]
    end

    SOUL --> SP["System Prompt"]
    ABOUT --> SP
    TOOLS --> SP
    SP --> LLM["LLM"]

    LLM -->|"memory tools"| USER
    LLM -->|"memory tools"| MEMORY

    style SOUL fill:#6a1b9a,color:#fff
    style ABOUT fill:#6a1b9a,color:#fff
    style TOOLS fill:#6a1b9a,color:#fff
    style USER fill:#e65100,color:#fff
    style MEMORY fill:#e65100,color:#fff
{{< /mermaid >}}

### 1.2. SOUL.md — Identity

SOUL.md is "who I am." Values, personality, communication style. Limit — 2000 characters. Injected into every request as the `## Agent Identity` section in the system prompt.

Why not more? Because every character in SOUL.md is paid for on every request. 2000 characters ≈ 500 tokens. If you increase it to 10K — the agent will spend 2.5K tokens just on its own identity before saying a single word. For a platform with hundreds of agents, that's unacceptable.

Who writes it? The user via UI — setting the personality at agent creation. Or the agent itself via memory tools — when the user asks "be more formal" or "use Russian by default."

### 1.3. ABOUT.md — Profile

ABOUT.md is "what I do." Role, specialization, typical tasks. Same limit — 2000 characters, same injection on every request.

Why a separate file instead of merging with SOUL? The identity/profile split isn't accidental. Identity (SOUL) is a constant — it rarely changes. Profile (ABOUT) is more dynamic: "right now I'm helping with gRPC migration" → a month later → "helping set up monitoring." Separate files let you update the profile without touching the core personality.

### 1.4. USER.md — User Profile for a Specific Agent

Now things get interesting. USER.md is stored in the `agent_user` scope — meaning each agent has its own profile for each user. Agent A knows that "Alice prefers YAML" — but Agent B in the same project doesn't, until it reads it itself.

What's stored:

| Category | Examples |
|----------|----------|
| Identification | Name, timezone, language, role |
| Preferences | "Answer concisely", "use YAML", "table format" |
| Work context | "Works with Python, uses FastAPI", "PM of project X" |
| Tools | "Primary calendar — Google", "Todoist for tasks" |
| Standing notes | "Standup at 9:00 Mon-Fri", "Avoid Friday meetings" |

Limit — 2000 characters. And here's why USER.md is on-demand rather than injection: not all agents need a user profile. A coding assistant — sure, it matters that you prefer Go. But a text summarization agent — it doesn't care about your timezone. Why spend 500 tokens on a profile the agent won't use?

### 1.5. MEMORY.md — Long-Term Memory

MEMORY.md is "what I know." Key decisions, client context, current tasks, conclusions from conversations. Scope `agent_user` — tied to the agent+user pair. Limit — 5000 characters (soft limit — when exceeded, the agent receives an instruction to compress).

MEMORY.md implements the write-manage-read pattern I described in the first part:

**Write** (who saves and when):
- Agent — via memory tools, when it identifies an important fact
- **Auto-capture** (future): before compaction/session end — automatic flush of important facts

**Manage** (compression, deduplication):
- Agent receives instruction: "Regularly compress MEMORY.md, remove outdated facts"
- If MEMORY.md > 5000 characters — instruction to the agent to compress
- Future: automatic deduplication via LLM

**Read** (when it's loaded):
- Agent reads via memory tools **on demand**
- Instruction in system prompt: "search memory before acting" (OpenClaw pattern)
- NOT injected into system prompt on boot (token economy)

Why is MEMORY.md on-demand rather than injection? Because MEMORY.md grows. Today — 500 characters, a month later — 5000. If injected — every request will spend more and more tokens on memory. OpenClaw solves this with truncation (cuts off > 20K characters), but truncation means data loss. We prefer the agent to decide for itself what it needs from memory right now.

### 1.6. Scope Summary

```
┌──────────────────────────────────────────────────────────┐
│                AGENT MEMORY LAYERS                        │
├──────────────┬──────────────┬───────────────┬───────────┤
│ Scope        │ Files        │ Loading       │           │
├──────────────┼──────────────┼───────────────┤           │
│ Agent        │ SOUL.md      │ Always        │           │
│              │ ABOUT.md     │ Always        │ Injection │
│              │ TOOLS.md     │ Always        │           │
├──────────────┼──────────────┼───────────────┤           │
│ Agent+User   │ USER.md      │ On demand     │ On-demand │
│              │ MEMORY.md    │ On demand     │ (tools)   │
└──────────────┴──────────────┴───────────────┴───────────┘

Agent receives Prompt Hint in system prompt:
"You have memory tools. Load USER.md and MEMORY.md when needed."
```

Two scopes, two isolation levels. `agent` — shared agent files, visible to all users. `agent_user` — personal memory, isolated per agent+user pair. No context leakage between users.

## 2. Context Assembly Strategy: Why Hybrid v2

In the first part I said we chose a hybrid strategy — injection for hot context, tools for the rest. But I didn't cover which alternatives we considered and rejected. And that's perhaps the most important architectural decision in the entire memory system.

### 2.1. Option 1: Full Injection (like OpenClaw)

All memory files are read from storage and injected into the system prompt on every request.

{{< mermaid >}}
graph LR
    S3[(Storage)] -->|"read all files"| ASM[Context Assembler]
    ASM --> SP[SystemPrompt]
    SP --> LLM[LLM]

    subgraph "Always injected (~5-10K tokens)"
        SOUL["SOUL.md"]
        ABOUT["ABOUT.md"]
        USER["USER.md"]
        MEMORY["MEMORY.md"]
        TOOLS["TOOLS.md"]
    end

    SOUL --> ASM
    ABOUT --> ASM
    USER --> ASM
    MEMORY --> ASM
    TOOLS --> ASM
{{< /mermaid >}}

**Pros**: simple implementation (~30-50 lines), predictable context, easy to debug.

**Cons**: 5-10K tokens per request, multiple storage reads, MEMORY.md grows → truncation → data loss, agent can't update files through tools.

OpenClaw does exactly this. And it works — for a coding assistant with one user. But on a platform with hundreds of agents where every request costs money, burning 10K tokens on boot is a luxury.

### 2.2. Option 2: On-Demand (Tool-Based)

Memory files are NOT injected. The agent receives memory tools and decides for itself what to read.

**Pros**: 0 tokens from memory on boot, agent can update files, scales without limit.

**Cons**: agent may "forget" to read context → poor responses. Every read = one react-loop iteration. Harder to debug — context depends on agent behavior.

This is the opposite extreme from Full Injection. Cheap at start, but unreliable. An agent without identity is like a person without a name — they can work, but they don't know who they are.

### 2.3. Option 3: Hybrid v1 — L0/L1 Boot + L2 Tools

L0 (SOUL, ABOUT) — always injected. L1 (USER) — also injected. L2+ (MEMORY) — through tools.

**Pros**: ~2-3K on boot, predictable base, close to MemPalace pattern.

**Cons**: USER.md burns tokens even when the agent doesn't need a user profile. On a platform with 50+ agent types, far from all need USER.md — a summarization agent, a translator, a code reviewer don't interact with users personally.

### 2.4. Option 3b: Hybrid v2 — Minimal Injection + Prompt Hint ← chosen

The key insight from studying OpenClaw: even OpenClaw **doesn't inject MEMORY.md into the system prompt**. If memory tools are available, the agent gets a hint "use memory_search" and loads MEMORY.md itself. USER.md in the Codex integration isn't always injected either.

Hence — Hybrid v2: minimal injection (SOUL + ABOUT + TOOLS) + prompt hint directing the agent to memory tools for USER.md and MEMORY.md.

{{< mermaid >}}
graph TD
    subgraph "System Prompt Injection (~1-2K tokens)"
        SOUL2["SOUL.md / ABOUT.md<br>agent identity"]
        TOOLS2["TOOLS.md<br>tool notes"]
        AUTO["Auto-generated tools<br>tool descriptions"]
    end

    subgraph "On-demand via memory tools"
        USER2["USER.md<br>user profile"]
        MEMORY2["MEMORY.md<br>long-term memory"]
    end

    SOUL2 --> SP2["System Prompt"]
    TOOLS2 --> SP2
    AUTO --> SP2
    SP2 --> LLM2["LLM"]

    LLM2 -->|"memory tools"| USER2
    LLM2 -->|"memory tools"| MEMORY2
{{< /mermaid >}}

**Prompt Hint** (added to system prompt):
> You have access to memory tools. Load USER.md (user profile) and MEMORY.md (long-term memory) when needed.

### 2.5. Comparing the Four Options

| Criterion | Full Injection | On-Demand | Hybrid v1 | **Hybrid v2** |
|------------|---------------|-----------|-----------|---------------|
| Boot tokens | ~5-10K | 0 | ~2-3K | **~1-2K** |
| Context predictability | ✅ All files always | ❌ Agent may forget | ✅ L0/L1 always | ⚠️ L1 via prompt hint |
| Agent updates files | ❌ API only | ✅ Via tools | ⚠️ L2+ via tools | ✅ All via tools |
| Storage reads on boot | 5+ | 0 | 3-4 | **2-3** |
| Scalability | ❌ MEMORY.md grows | ✅ | ⚠️ USER.md burns tokens | ✅ |
| Implementation complexity | Simple | Medium | Complex | **Medium** |

Why not Hybrid v1? Because USER.md in injection is tokens wasted when the agent doesn't need a profile. On a platform with dozens of agent types (summarizer, translator, code analyst), far from every agent interacts with the user personally. Why load USER.md into every request?

Bottom line: Hybrid v2 — minimal token budget on boot, maximum flexibility, consistent with how OpenClaw actually works (doesn't inject MEMORY.md, gives a hint instead).

### 2.6. Prompt Hint — How the Agent Knows What to Read

The agent won't read USER.md or MEMORY.md on its own — it needs to be told. The Prompt Hint is an instruction added to the system prompt:

> You have memory tools. Load USER.md (user profile) and MEMORY.md (long-term memory) when needed.

This is analogous to the OpenClaw approach: the agent gets a hint "use memory_search" and loads MEMORY.md itself. We added "when needed" — the agent decides whether it needs user context in the current request.

Risk: the agent may "forget" to load USER.md → responses without personal context. Mitigation: clear Prompt Hint + examples in system prompt ("At the start of a session, load USER.md and MEMORY.md"). In practice — USER.md is small (~200-500 tokens), loading takes one react-loop iteration.

## 3. TOOLS.md: Two Sources — Two Responsibilities

TOOLS.md is the most non-trivial memory file. The system already has a programmatic mechanism for describing tools: `ToolSet.RegisterInAgent()` generates `SystemPromptInjection` with tool descriptions for the LLM. Why add another source?

### 3.1. The Problem: Mechanics ≠ Practice

The system **automatically** generates a description of each tool: name, parameters, types, brief description. This is enough for the LLM to understand **how to call** a tool. But the auto-generated description **doesn't contain** practical experience — and that's what determines **when and why** the agent will use the tool.

**What auto-generated provides** (mechanics):
- Tool names and parameter names
- Data types and parameter required/optional
- Brief description from ToolSettings

**What auto-generated DOESN'T provide** (practice):
- When to use a specific tool versus another
- Quirks of working with specific data
- Typical usage patterns in the project context
- Limitations not obvious from the description (rate limits, response size)
- User preferences for working with tools

The gap between "how to call" and "when to use" is the gap between API documentation and real-world experience with it. TOOLS.md fills this gap.

### 3.2. Solution: Auto-generated + User Extensions

The System Prompt (tools section) is formed from two sources:

```
{auto-generated description from RegisterInAgent()}     ← mechanics (in memory)
---
## Tool Notes
{contents of TOOLS.md, if exists}                       ← practice (from storage)
```

**Two sources — two responsibilities:**

| Source | Where stored | Who updates | What it contains |
|--------|-------------|-------------|-----------------|
| Auto-generated | In memory (system prompt) | Runtime on every request | Mechanics: names, parameters, types, lists |
| TOOLS.md | In storage | User (UI) / Agent (memory tools) | Practice: tips, quirks, examples, warnings |

Key invariant: the auto-generated part exists **only in memory** and is never written to storage. TOOLS.md is stored persistently and **is never automatically overwritten**. The two sources don't conflict.

### 3.3. Examples

#### Knowledge DB (RAG)

**Auto-generated** (formed on every request):
```
search_knowledge_db: Search documents in knowledge databases.
Parameters: query (string), db_name (string, optional)

Available knowledge databases:
- "Product Docs" (product documentation)
- "Internal Wiki" (internal processes)
```

**TOOLS.md** (user notes):
```markdown
## Knowledge DB Notes

- For Product Docs search, use exact product names, not descriptions
- Internal Wiki is updated weekly — if you don't find current info,
  ask the user to confirm
- Recommended pattern: first search → then read_knowledge_db_document
- Search result limit is 5 chunks, if you need more — refine the query
```

### 3.4. Why Not "Fully Auto" and Not "Fully Manual"

**Fully auto** (an option we considered): only auto-generated description, no TOOLS.md. Pro — always current, no sync issues. Con — the user can't add quirks, tips, usage context. The agent sees "search_knowledge_db(query, db_name)" but doesn't know that "Product Docs needs exact names."

**Fully manual** (like OpenClaw): user/agent writes TOOLS.md manually, the system generates nothing. Pro — full customization. Con — when a new skill or MCP tool is added, TOOLS.md becomes stale. The user must remember to update. And if they forget — the agent sees descriptions for three tools when five are connected.

Splitting into auto-generated + user extensions solves both problems: mechanics are always current (generated from ToolSettings), practice is added manually (TOOLS.md). And when MCP arrives — the auto-generated part automatically includes MCP descriptions.

### 3.5. Lazy Lifecycle

Another nuance: TOOLS.md is not created by default. The file appears in storage only when the user or agent first writes content to it. An "empty" TOOLS.md is an **absent** file, not an empty string.

This same rule applies to all memory files. Missing file is **not an error**:

| Scenario | Behavior |
|----------|----------|
| TOOLS.md doesn't exist at injection | `## Tool Notes` section is not added |
| SOUL.md / ABOUT.md don't exist | Corresponding sections are omitted |
| Agent requests a non-existent file via memory tools | Empty result, not an error |
| Agent writes a new file via memory tools | File is created |

Graceful degradation — a new agent without memory files doesn't crash, it works with minimal context. The prompt hint is always added — even if files don't exist yet, the agent knows the tools are there.

## 4. Dreaming: Background Memory Consolidation

And now — the most interesting part. Everything I've described above is an **inline mechanism**: the agent writes to MEMORY.md during a session, manages size through a "compress regularly" instruction. There's no consolidation (deduplication, contradiction resolution, stale data removal).

The question: should we add a **background consolidation process**, triggered after sessions end? And if so — how? Anthropic and OpenAI have already answered "yes" and implemented it in 2026. Let's look at how they do it.

### 4.1. Anthropic: Auto Dream (Claude Code)

A 4-phase background process, triggered **between sessions** by a separate sub-agent ([solaius, 2026](#ref-solaius)):

| Phase | Action | Result |
|-------|--------|--------|
| **1. Orient** | Reads current state of MEMORY.md and topic files | Baseline memory map |
| **2. Gather Signal** | Searches completed session logs for new facts, drift, contradictions. Grep, not full read (token economy) | Signals for update |
| **3. Consolidate** | Merges duplicates, resolves contradictions, normalizes time ("yesterday" → "2026-03-15"), removes stale entries | Updated memory files |
| **4. Prune & Index** | Rebuilds MEMORY.md as an index, updates references, enforces 200-line cap | Clean index |

{{< mermaid >}}
graph LR
    LOGS["Completed session<br>logs"] -->|"Phase 1: Orient"| MAP["Baseline memory<br>map"]
    MAP -->|"Phase 2: Gather Signal"| SIGNAL["Signals:<br>new facts, drift,<br>contradictions"]
    SIGNAL -->|"Phase 3: Consolidate"| UPDATED["Updated<br>memory files"]
    UPDATED -->|"Phase 4: Prune & Index"| CLEAN["Clean index<br>MEMORY.md"]

    style LOGS fill:#1565c0,color:#fff
    style CLEAN fill:#2e7d32,color:#fff
{{< /mermaid >}}

**Trigger**: double gate — **24+ hours** since last consolidation **AND** **5+ new sessions**. Not "after every conversation" but "when enough material has accumulated." Smart: frequent short sessions don't trigger dreaming every minute.

**Safety**: the sub-agent only writes to `memory/`, has no git/npm/MCP tools. It can't break code, can't reach the internet — only memory cleanup.

**Results (Harvey, legal AI)**: 6x increase in task completion rate ([solaius, 2026](#ref-solaius)). But Harvey is a legal AI where memory is critical. A realistic estimate for typical scenarios — **1.5–3x**.

**Cost**: standard Claude token rates. 913 sessions consolidated in 8–9 minutes. At "once per day with 5+ sessions" frequency — pennies compared to main LLM calls.

### 4.2. Anthropic: Dreams API (Managed Agents, Enterprise)

Enterprise version of Auto Dream with additional capabilities ([solaius, 2026](#ref-solaius)):

| Property | Value |
|----------|-------|
| Scale | Up to 100 past sessions per dream |
| Isolation | Input store is not modified — a new output store is created |
| Review gate | Auto-apply or review before applying |
| Pattern types | Recurring mistakes, workflow convergence, shared preferences |
| Versioning | Each mutation creates an immutable `memver_...` with audit trail |

The review gate is the key difference from basic Auto Dream. Enterprise can't afford "LLM silently rewrote memory." Every change goes through review. Versioning is immutable — you can roll back to any previous version. This is critical for regulated workflows (healthcare, legal, finance).

### 4.3. KAIROS (unreleased, internal Anthropic)

And here's a preview of the architecture for always-on agents ([solaius, 2026](#ref-solaius)). Append-only logs + nightly dreaming.

- Logs: `logs/YYYY/MM/YYYY-MM-DD.md` — append-only, **never modified**
- Nightly `/dream` distills logs into topic files and MEMORY.md
- Analogy with WAL (<abbr title="Write-Ahead Log — a database journaling technique; all changes are first written to the WAL, then applied to the main data structure">Write-Ahead Log</abbr>) in databases: full audit trail + lean working memory

Why does this matter? Because it solves the fundamental problem of current approaches: **what if dreaming deletes something important?** In KAIROS — it won't, ever. Logs are append-only, they don't change. `/dream` creates a new representation (MEMORY.md), but the original stays. Like WAL in PostgreSQL: even if the checkpoint crashes — the logs are there, you can recover.

It's the same principle as in event sourcing: state is a function of event history. MEMORY.md is a projection, and logs are the event store. A projection can be recreated, an event store cannot.

### 4.4. OpenAI: Dreaming V3 (June 2026)

A background synthesis process that completely replaced manual "saved memories" ([OpenAI, 2026](#ref-openai-dreaming)):

| Generation | Mechanism | Factual Recall |
|-----------|----------|---------------|
| 2024: Saved Memories | Manual list, explicit "remember this" | 41.5% |
| 2025: Dreaming V0 | Supplemental synthesis overlay on top of saved memories | 67.9% |
| 2026: Dreaming V3 | Fully automatic synthesis, replaces saved memories | **82.8%** |

The evolution is impressive: from 41.5% to 82.8% in two years ([OpenAI, 2026](#ref-openai-dreaming)). But even more interesting — how Dreaming V3 works:

- Reads the **entire dialog history** of the user
- Automatically synthesizes a profile: facts, preferences, instructions, implicit patterns
- **Self-updating**: "user is traveling to Singapore in July" → after the trip, automatically rewritten to "user traveled to Singapore in July 2026"
- Runs **between** conversations, not during

Self-updating is the killer feature. In our approach, MEMORY.md contains "user is traveling to Singapore in July." After July — that's a stale fact. In Dreaming V3 — the fact automatically updates. No manual management.

But there are problems:

- **The platform silently rewrites records** — no revision log ([OpenAI, 2026](#ref-openai-dreaming); [solaius, 2026](#ref-solaius)). You can't find out what was there before the rewrite.
- **Audit trail is opaque** — critical for regulated workflows. In healthcare, you're obligated to know why the agent made a decision, and if memory silently changed — you won't know.
- **5x compute reduction** enabled expansion to the Free tier — but at the cost of quality for complex scenarios.

### 4.5. Comparison of Approaches

| Aspect | Auto Dream (Anthropic) | Dreams API (Enterprise) | KAIROS (unreleased) | Dreaming V3 (OpenAI) | Our approach (inline) |
|--------|----------------------|------------------------|--------------------|--------------------|--------------------|
| When capture | After session | After session | Nightly | After session | During session |
| What it analyzes | Full logs | Up to 100 sessions | All logs | Entire history | Only current context |
| Consolidation | Auto: merge, contradictions, normalization | Auto + review gate | Nightly distillation | Auto: synthesis + self-update | Instruction "compress MEMORY.md" |
| Trigger | 24h + 5 sessions | On demand | Nightly | Between conversations | No trigger |
| Audit | — | Immutable versions | Append-only logs (WAL) | No revisions | Last-write-wins |
| Cost | Additional LLM call | Additional LLM call | Additional LLM call nightly | Additional compute (5x reduced) | No additional cost |
| Update latency | 24h+ | On demand | 1 night | Between conversations | Immediate |

### 4.6. Why We're NOT Doing Dreaming Yet

Four reasons:

1. **The basic mechanism is sufficient** — inline write via memory tools + "compress" instruction covers 80% of the value. Memory works, facts are saved, agents recall context. The "memory turns into a dumpster" problem is theoretical — we haven't seen it in production yet.

2. **Infrastructure** — dreaming requires a worker, task queue (NATS stream or BullMQ), session completion triggers, consolidation prompt. None of this exists, and it's not "a couple lines of code."

3. **No data** — without accumulated experience with MEMORY.md in production, it's unclear how critical the problem is. Maybe the "compress" instruction is sufficient. Maybe not. We need to look at metrics: MEMORY.md size, staleness frequency, duplicate count.

4. **Cost and risk** — the LLM might incorrectly merge different facts, delete important things as "stale." At this stage, a review gate is needed (not auto-apply), and that's a different UX.

**Recommended path**:

1. Current epic — implement the inline mechanism
2. Observation — collect metrics on MEMORY.md in production
3. Next epic — if metrics show degradation → implement Option B (batch consolidation) with review gate
4. Future — if audit trail is needed → Option C (append-only + nightly distillation)

Approximate cost of batch consolidation: ~5500 tokens per run (3000 for session log + 1000 for MEMORY.md + 500 for prompt + 1000 for output). At "once per 24h with 5+ sessions" frequency and a cheap model (OSS 20B) — ~$0.01–0.05 per agent/day. That's 1–3% of the cost of serving the agent. Pennies — but only if the basic mechanism already works.

## 5. Write-Manage-Read in Practice

In the first part I described the write-manage-read loop (Du et al., 2026) as an abstract model. Let's see what it looks like in our implementation.
### 5.1. Write: Who, When, Through What

| File | Who writes | When | Through what |
|------|-----------|------|-------------|
| SOUL.md | User (UI) / Agent | At agent creation, on user request | UI / memory tools |
| ABOUT.md | User (UI) / Agent | At creation, on role change | UI / memory tools |
| TOOLS.md | User (UI) / Agent | When practical experience accumulates | UI / memory tools |
| USER.md | Agent / User | At first interaction, on preference update | Memory tools / UI |
| MEMORY.md | Agent | When an important fact is identified | Memory tools |

### 5.2. Manage: Compression, Deduplication, Staleness

The current mechanism is an **instruction to the agent**:

> Regularly compress MEMORY.md. Remove outdated facts. Merge duplicates. If MEMORY.md > 5000 characters — compress it.

This isn't automatic consolidation, but delegation to the agent. Does it work? In most cases — yes. An LLM is perfectly capable of compressing 5000 characters of facts down to 3000, removing duplicates and stale data. But not always correctly — it might delete something important or merge things that should stay separate. Hence the soft limit, not hard truncation.

**Future**: automatic deduplication via LLM (like Auto Dream Phase 3). But — after accumulating production metrics.

**Temporal validity**: every memory file has a `last_modified`. If a fact hasn't been updated longer than a configurable TTL — it's marked stale. Simpler than MemPalace's Knowledge Graph with `valid_from`/`valid_to`, but sufficient for a platform with hundreds of agents. Graph queries are Zep's territory.

### 5.3. Read: Injection vs On-Demand

| Mechanism | Files | When | Tokens |
|-----------|-------|------|--------|
| Bootstrap injection | SOUL.md, ABOUT.md, TOOLS.md | Every request | ~1-2K |
| Prompt Hint | — | Every request | ~50 |
| On-demand (memory tools) | USER.md, MEMORY.md | Agent's decision | ~0.5-5K |

Total on boot: ~1-2K tokens. On first request with USER.md + MEMORY.md load: ~2-4K. On each subsequent request: depends on the task — the agent might not access memory at all.

## 6. Takeaways

### 6.1. Memory Files Summary

| File | Scope | Loading | Char limit | Who writes | Key purpose |
|------|-------|---------|-----------|-----------|-------------|
| SOUL.md | agent | Injection | 2000 | UI / agent | Identity |
| ABOUT.md | agent | Injection | 2000 | UI / agent | Profile |
| TOOLS.md | agent | Injection | 2000 | UI / agent | Tool practice |
| USER.md | agent_user | On-demand | 2000 | agent / UI | User profile |
| MEMORY.md | agent_user | On-demand | 5000 (soft) | agent only | Long-term memory |

### 6.2. Context Assembly Strategies

| Strategy | Boot tokens | Predictability | Flexibility | Scalability |
|----------|------------|---------------|-------------|-------------|
| Full Injection | ~5-10K | ✅ | ❌ | ❌ |
| On-Demand | 0 | ❌ | ✅ | ✅ |
| Hybrid v1 | ~2-3K | ✅ | ⚠️ | ⚠️ |
| **Hybrid v2** | **~1-2K** | **⚠️** | **✅** | **✅** |

### 6.3. Dreaming: Not "If" but "When"

Anthropic and OpenAI have already implemented background memory consolidation. The results are impressive: 82.8% factual recall at OpenAI, 6x task completion at Harvey (Anthropic). But both approaches have tradeoffs: OpenAI silently rewrites without an audit trail, Anthropic requires a separate sub-agent and double gate.

Our path — inline mechanism first, then metrics, then dreaming. Not because we don't believe in consolidation, but because we need to understand **what exactly** to consolidate. Without production data — it's shooting in the dark.

When we do implement it — most likely Option B (batch consolidation with review gate), like Anthropic. And if audit trail is needed — Option C (append-only logs + nightly distillation), like KAIROS. But that's a whole different story.

---

**Previous article in the series:** [Part 5: Agent Memory Management]({{< ref "ai-agent-design-patterns-5-memory" >}})

## References

1. <a id="ref-openai-dreaming"></a>OpenAI. "Dreaming: Better memory for a more helpful ChatGPT." June 4, 2026. [openai.com/index/chatgpt-memory-dreaming](https://openai.com/index/chatgpt-memory-dreaming/)
2. <a id="ref-solaius"></a>solaius (Red Hat Research). "Claude Memory & Dreaming Deep Dive." June 2026. [github.com/solaius/ai-asset-registry](https://github.com/solaius/ai-asset-registry/blob/main/agent-memory/research/10-claude-memory-dreaming.md)
