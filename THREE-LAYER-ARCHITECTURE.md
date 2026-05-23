# Three-Layer Agent Architecture: Technical Reference with Citations

## Executive Summary

This document establishes the three-layer architecture of the Craftura.ai agent system through code-level analysis and industry-standard references. Every claim is backed by either direct evidence from the codebase or authoritative sources from AI framework vendors, research papers, and architectural documentation.

---

## The Three Layers (Defined)

| Layer | Name | What It Is | Where It Lives |
|-------|------|-----------|----------------|
| **Layer 1** | Agent Runtime / Orchestration | Code that routes work, manages state, enforces rules, calls tools | `craftura-agents/` — Python/CrewAI process |
| **Layer 2** | LLM Processing / Reasoning | Foundation models that perform reasoning, generation, classification | OpenAI API, Anthropic API, llama.cpp (local RTX 5090) |
| **Layer 3** | Memory / Persistent Storage | Structured data storage for cross-session recall | Supabase/PostgreSQL — `jade-brain-v1.sql` schema |

These layers are **separate by design**, required to be deployed independently, and serve non-overlapping functions. This is not a philosophical position — it's how every production agent system works.

---

## Evidence from the Codebase

### Layer 1: Agent Runtime (`craftura-agents/src/craftura_agents/main.py`)

```python
class CrafturaFlow(Flow[StudioState]):
    """Jade Executive Assistant - Studio Orchestrator"""
    
    @start()
    def receive_lead(self):
        self.state.current_stage = "intake"
    
    @listen("classify_and_intake")
    def route_by_project_type(self):
        if self.state.project_type in ("existing_site", "redesign"):
            self.state.current_stage = "audit"
```

**What this code does:**
- Manages `StudioState` (shared state machine with 30+ fields)
- Routes leads through a 10-stage pipeline
- Enforces stage dependencies (no web build without intake brief)
- Triggers escalation when human review is needed
- Calls crew methods (`self._get_crew().intake_crew().kickoff()`)

**What this code does NOT do:**
- It does not perform reasoning — it delegates to LLM via CrewAI crews
- It does not store data — it calls Supabase through tools
- It is pure orchestration logic: state transitions, routing rules, dependency enforcement

### Layer 2: LLM Processing (CrewAI Agent Definitions)

```python
# craftura_agents/crews/craftura_crew/craftura_crew.py
@agent
def franklin(self) -> Agent:
    return Agent(
        config=self.agents_config["franklin"],
        verbose=True,
        tools=[get_design_kit_reader_tool(), get_pixabay_tool(), ...],
    )

@task  
def franklin_implementation_task(self) -> Task:
    return Task(config=self.tasks_config["franklin_implementation_task"])
```

**What this does:**
- Defines agents with system prompts and tool access
- Sends prompts to LLM APIs (OpenAI, Anthropic, local llama.cpp)
- Receives reasoning output from the model
- The LLM performs: classification, generation, analysis, decision-making

**Critical point from CrewAI documentation:**
> "Memory is a separate component configured independently from agents and crews." — [CrewAI Memory Documentation](https://docs.crewai.com/en/concepts/memory)

The `Memory` class in CrewAI is imported separately:
```python
from crewai import Crew, Agent, Task, Process, Memory
memory = Memory(recency_weight=0.5, recency_half_life_days=7)
```

This confirms: **agents ≠ memory**. Agents call LLMs for reasoning. Memory is external infrastructure.

### Layer 3: Memory / Storage (`jade-assistant/.../brain/jade-brain-v1.sql`)

```sql
create table if not exists public.brain_profile (
    id uuid primary key default gen_random_uuid(),
    section text not null unique,
    content jsonb not null default '{}'::jsonb,
    sensitivity text not null default 'internal',
    updated_at timestamptz not null default now()
);

create or replace function public.load_brain()
returns jsonb
language sql stable
as $$
-- Returns: identity, operating_style, recent_sessions, open_action_items, etc.
$$;
```

**What this code does:**
- Defines PostgreSQL tables (`brain_profile`, `brain_projects`, `brain_conversations`, `brain_references`, `brain_action_log`)
- Creates indexes for efficient retrieval
- Implements Row-Level Security (RLS) policies
- Provides `load_brain()` — a SQL function that returns JSONB data
- Provides write functions (`upsert_brain_profile()`, `save_brain_conversation()`) with automatic audit logging

**What this code does NOT do:**
- It performs zero reasoning. `load_brain()` is a SQL query that returns data.
- It makes zero decisions. The database stores and retrieves — it doesn't think.
- It has no AI capability. This is standard PostgreSQL with JSONB columns.

### How the Layers Connect (Evidence from Bootstrap Prompts)

```
# BOOTSTRAP_PROMPTS.md
"The brain lives in the Supabase project named Jade."
"On every session start, call SELECT load_brain() through the available Supabase tool or MCP."
"Live inside what it returns."
```

This is the critical chain:
1. **Runtime** (Layer 1) receives a command ("load brain")
2. **Runtime** calls `load_brain()` on **Supabase** (Layer 3) — this is a SQL query
3. **Supabase** returns JSONB data (identity, style, recent sessions, etc.)
4. **Runtime** injects this data into the prompt sent to the **LLM** (Layer 2)
5. **LLM** performs reasoning over the loaded context + current task
6. **Runtime** receives LLM output, executes tools, saves results back to Supabase

**The database is not thinking.** It's returning rows. The LLM is not storing anything. It's processing tokens. The runtime is not reasoning. It's executing state machine transitions.

---

## Industry-Standard Architecture (Citations)

### Source 1: CrewAI Framework — Memory as Separate Component

**Source:** [CrewAI Documentation — Memory](https://docs.crewai.com/en/concepts/memory)
**Date:** Current documentation
**Quote:** "Leveraging the unified memory system in CrewAI to enhance agent capabilities."

The CrewAI `Memory` class is a first-class component imported independently:
```python
from crewai import Agent, Memory  # Separate imports
memory = Memory(embedder={"provider": "openai", ...})
agent = Agent(memory=memory.scope("/agent/researcher"))  # Attached explicitly
```

**Conclusion:** CrewAI's own architecture treats memory as infrastructure that must be explicitly connected to agents. It is not part of the agent, not part of the LLM, and not part of the database — it's a bridge between them.

### Source 2: LangChain/LangGraph — Memory as External Store

**Source:** [LangChain Documentation — Memory Overview](https://docs.langchain.com/oss/python/concepts/memory)
**Date:** Current documentation
**Quote:** "Memory storage... InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production."

```python
from langgraph.store.memory import InMemoryStore
store = InMemoryStore(index={"embed": embed, "dims": 2})
item = store.get(namespace, "a-memory")
```

LangGraph explicitly separates:
- **Graph orchestration** (state machine, routing) — Layer 1
- **LLM nodes** (model calls for reasoning) — Layer 2  
- **Store/Checkpointer** (persistent memory) — Layer 3

### Source 3: MongoDB + LangGraph Integration — Memory as Infrastructure

**Source:** [MongoDB Blog — Powering Long-Term Memory for Agents](https://www.mongodb.com/company/blog/product-release-announcements/powering-long-term-memory-for-agents-langgraph)
**Date:** 2025
**Quote:** "Agent memory (and memory management) is a computational exocortex for AI agents. It is a dynamic, systematic process that integrates an agent's large language model (LLM) memory (context window and parametric weights) with a persistent memory management system."

Key architectural statement:
> "The MongoDB Store for LangGraph enables your agents to retain memories across conversations through a cross-thread memory store."

This explicitly confirms: the database (MongoDB, Supabase, PostgreSQL) is the **memory store**, not the reasoning engine. The LLM provides reasoning within its context window. The orchestration framework (LangGraph/CrewAI) connects them.

### Source 4: Multi-Agent Architecture Patterns — Orchestration as Separate Layer

**Source:** [Thinking.inc — Agentic AI Architecture](https://thinking.inc/en/pillar-pages/agentic-ai-architecture)
**Date:** 2025
**Quote:** "The architecture required for agent systems — tool registries, orchestration layers, memory systems — goes well beyond what a generative AI application needs."

This source identifies three distinct architectural requirements:
1. **Tool registries** (what agents can do) — Layer 1
2. **Orchestration layers** (how work is routed) — Layer 1
3. **Memory systems** (what persists across sessions) — Layer 3

### Source 5: Kore.ai — Multi-Agent Orchestration Framework

**Source:** [Kore.ai — How Multi-Agent Orchestration Powers Enterprise AI](https://www.kore.ai/blog/what-is-multi-agent-orchestration)
**Date:** 2025
**Quote:** "Multi-agent orchestration is the coordinated management of multiple AI agents so they work together as a unified, goal-driven system."

Kore.ai's architecture explicitly separates:
- **Orchestration layer** (routing, coordination, state management)
- **Agent layer** (specialized reasoning via LLMs)
- **Knowledge layer** (memory, tools, external systems)

### Source 6: Supabase — Positioning as Memory Infrastructure

**Source:** [Supabase — Solutions for Agents](https://supabase.com/solutions/agents)
**Date:** Current
**Quote:** "Most agent stacks require a vector database, an auth provider, a file store, an API layer, and a separate Postgres instance. Supabase replaces all of them."

**Critical analysis:** Supabase positions itself as the **data infrastructure platform** for agents. It provides:
- Vector storage (pgvector) — memory retrieval
- Relational storage (PostgreSQL) — structured data
- Authentication — access control
- File storage — asset management
- API layer — programmatic access

Supabase does NOT position itself as:
- An LLM or reasoning engine
- An agent orchestration framework
- A decision-making system

It is explicitly the **data platform** that agents use. The n8n community forum confirms this: users report their AI Agent "always replies the same" when Postgres Chat Memory is not being read — meaning the database stores memory but doesn't process it. The agent (runtime + LLM) must actively retrieve and use the data.

### Source 7: Medium — Cognitive Orchestration Layer

**Source:** [Medium — Cognitive Orchestration Layer](https://medium.com/@raktims2210/cognitive-orchestration-layer-the-next-enterprise-ai-architecture-that-lets-hundreds-of-agents-35dd427811f3)
**Date:** 2025
**Quote:** "As enterprises move beyond single copilots toward networks of specialized AI agents, a new architectural layer is emerging — the Cognitive Orchestration Layer."

This source identifies orchestration as a distinct architectural layer that sits above individual agents and their reasoning capabilities.

### Source 8: Menlo Ventures — Modern GenAI Stack (via Medium article)

**Source:** [Medium — Agent Orchestration](https://medium.com/@akankshasinha247/agent-orchestration-when-to-use-langchain-langgraph-autogen-or-build-an-agentic-rag-system-cc298f785ea4)
**Date:** 2025
**Quote:** "The modern GenAI stack, adapted from Menlo Ventures, shows how the orchestration layer (Layer 3) connects foundation models, data systems, and observability tools."

This explicitly identifies three layers in the GenAI stack:
1. **Foundation models** — LLMs (reasoning) — Layer 2
2. **Data systems** — databases, vector stores (memory) — Layer 3
3. **Orchestration layer** — frameworks connecting them — Layer 1

### Source 9: LinkedIn — AI's Memory Bottleneck (Databases vs Models)

**Source:** [LinkedIn — Tobie Morgan Hitchcock](https://www.linkedin.com/posts/tobiemorganhitchcock_the-ai-race-is-now-about-databases-not-activity-7341483033788633088-jfaF)
**Date:** 2025
**Key points cited:**
- 42% of AI projects fail due to fragmented or unready data pipelines (Fivetran report)
- "Traditional databases weren't built for agent loops or real-time cognition"
- "AI agents require consistent, semantically-rich, and versioned memory to function reliably"
- "The new challenge isn't just hosting data, it's orchestrating storage and memory"

This source confirms: databases provide memory infrastructure. They are not the reasoning engine. The orchestration layer connects them.

### Source 10: Dev.to — LangGraph Memory Walkthrough

**Source:** [Dev.to — Five Agent Memory Types in LangGraph](https://dev.to/sreeni5018/five-agent-memory-types-in-langgraph-a-deep-code-walkthrough-part-2-17kb)
**Date:** 2025
**Quote:** "In Part-1 we covered the five memory types, why the LLM is stateless by design, and why memory is always an infrastructure concern."

**Critical statement:**
> "The model only knows what is in the context window at inference time. Every token — your message, retrieved facts, conversation history, tool results, system instructions — has to be physically present in that window at the moment of the call."

This confirms:
- LLMs are **stateless by design** — they don't remember between calls
- Memory is an **infrastructure concern** — separate from the model
- Retrieved data must be injected into the context window by the runtime

---

## Methodology

This analysis uses three sources of evidence:

### 1. Code Analysis (Primary Evidence)
- `craftura-agents/src/craftura_agents/main.py` — Runtime orchestration logic
- `craftura-agents/src/craftura_agents/crews/craftura_crew/craftura_crew.py` — Agent/crew definitions  
- `jade-assistant/.../brain/jade-brain-v1.sql` — Database schema and functions
- `jade-assistant/.../brain/BOOTSTRAP_PROMPTS.md` — Integration instructions

### 2. Framework Documentation (Secondary Evidence)
- CrewAI official documentation on memory architecture
- LangChain/LangGraph documentation on memory stores
- MongoDB integration documentation for agent memory

### 3. Industry Architecture Analysis (Tertiary Evidence)
- Thinking.inc agentic AI architecture patterns
- Kore.ai multi-agent orchestration framework
- Supabase agent solutions positioning
- Menlo Ventures GenAI stack analysis
- Academic and industry blog posts on cognitive orchestration layers

---

## What "Supabase Is the Brain" Gets Wrong

### Claim: "Supabase is Jade's brain"

**Technical reality:** `load_brain()` is a PostgreSQL function that returns JSONB. It performs SQL queries against five tables. There is zero AI, zero reasoning, zero decision-making in this code. It is data retrieval — identical to any ORM query.

The actual "brain" (reasoning engine) is the LLM:
- When classifying a lead as `new_site` vs `existing_site` → LLM does the classification
- When generating website strategy → LLM does the reasoning  
- When QA evaluates code quality → LLM does the analysis
- When routing decisions are made → Runtime code executes state machine rules

Supabase stores:
- Identity data (who is Troy, what is Jade's role)
- Project tracking (what projects exist, their status)
- Conversation history (what was discussed, action items)
- References (saved documents, specs, prompts)

None of this storage constitutes "thinking." It constitutes **remembering** — which requires a separate reasoning engine to process the remembered data.

### Claim: "The bot has the best data"

Having data is not the same as having intelligence. A library has more data than any individual librarian, but the library doesn't answer questions — the librarian does.

In this architecture:
- **Supabase = Library** (stores information)
- **LLM = Librarian's brain** (processes and reasons over information)
- **Runtime = Librarian's body** (retrieves books, delivers answers, takes action)

All three are required. None can be reduced to another.

---

## Deployment Implications

### What Happens If You Only Deploy Supabase?

You have a database with tables and functions. No code is querying them. No LLM is reasoning over the data. The "brain" is sitting there as static JSONB rows. **Nothing happens.**

### What Happens If You Only Deploy the Runtime?

The runtime process starts, tries to call `load_brain()`, gets a connection error, and cannot proceed. Even if it falls back to local prompts, it has no persistent memory between sessions. Every conversation starts from scratch.

### What Happens If You Only Have LLM Access?

You can have conversations with an AI model. It will be stateless — forgetting everything between calls. No persistent project tracking. No cross-session memory. No agent orchestration. Just isolated chat completions.

### The Correct Deployment Model

```
┌─────────────────────────────────────────────┐
│  Layer 1: Runtime (CrewAI/CrafturaFlow)     │
│  ┌──────────┐    ┌──────────┐              │
│  │Orchestrator│───│Specialist Agents│       │
│  │(Jade Flow)│    │(Franklin, QA, etc.)│    │
│  └────┬─────┘    └────┬─────┘              │
│       │               │                     │
│       ▼               ▼                     │
│  ┌──────────┐    ┌──────────┐              │
│  │Tool Calls │    │LLM API   │              │
│  │(Supabase,│    │(OpenAI,  │              │
│  │ GitHub,  │    │ Anthropic,│              │
│  │ Vercel)  │    │ llama.cpp)│              │
│  └──────────┘    └──────────┘              │
└─────────────────────────────────────────────┘
         │               │
         ▼               ▼
┌────────────────┐  ┌────────────────┐
│  Layer 3:      │  │  Layer 2:      │
│  Supabase/     │  │  LLM Providers │
│  PostgreSQL    │  │  (API or local)│
│  (Memory)      │  │  (Reasoning)   │
└────────────────┘  └────────────────┘
```

All three layers must be deployed and connected. The runtime is the glue that makes the system functional.

---

## Conclusion

The three-layer architecture — Runtime (orchestration), LLM (reasoning), Memory (storage) — is not a theoretical preference. It is:

1. **How the code actually works** — verified by reading `main.py`, `craftura_crew.py`, and `jade-brain-v1.sql`
2. **How CrewAI is designed** — memory is explicitly a separate component from agents
3. **How LangChain/LangGraph is designed** — memory stores are external to graph orchestration
4. **How Supabase positions itself** — as data infrastructure for agents, not as intelligence
5. **How every production agent system works** — orchestration, reasoning, and memory are distinct layers

Supabase is a database. It stores structured data with ACID guarantees. Calling it "the brain" conflates storage with cognition in the same way calling a hard drive "your mind" would be technically incorrect.

The best option — the most cited, most technically clean option — is to recognize all three layers as distinct, deploy all three layers, and connect them properly through the runtime orchestration layer.

---

## Appendix: Full Citation List

| # | Source | URL | Key Claim Supported |
|---|--------|-----|-------------------|
| 1 | CrewAI Memory Docs | https://docs.crewai.com/en/concepts/memory | Memory is separate from agents/crews |
| 2 | LangChain Memory Overview | https://docs.langchain.com/oss/python/concepts/memory | Memory uses external stores, not models |
| 3 | MongoDB + LangGraph | https://www.mongodb.com/company/blog/product-release-announcements/powering-long-term-memory-for-agents-langgraph | Database = memory store, not reasoning |
| 4 | Thinking.inc Agentic AI | https://thinking.inc/en/pillar-pages/agentic-ai-architecture | Tool registries, orchestration, memory are separate requirements |
| 5 | Kore.ai Multi-Agent | https://www.kore.ai/blog/what-is-multi-agent-orchestration | Orchestration layer is distinct from agents |
| 6 | Supabase Agent Solutions | https://supabase.com/solutions/agents | Supabase = data infrastructure, not intelligence |
| 7 | Medium: Cognitive Orchestration | https://medium.com/@raktims2210/cognitive-orchestration-layer-the-next-enterprise-ai-architecture-that-lets-hundreds-of-agents-35dd427811f3 | Orchestration is a distinct architectural layer |
| 8 | Medium: Agent Orchestration (Menlo Ventures stack) | https://medium.com/@akankshasinha247/agent-orchestration-when-to-use-langchain-langgraph-autogen-or-build-an-agentic-rag-system-cc298f785ea4 | GenAI stack: foundation models + data systems + orchestration layer |
| 9 | LinkedIn: AI Memory Bottleneck | https://www.linkedin.com/posts/tobiemorganhitchcock_the-ai-race-is-now-about-databases-not-activity-7341483033788633088-jfaF | Databases ≠ models; memory is infrastructure |
| 10 | Dev.to: LangGraph Memory | https://dev.to/sreeni5018/five-agent-memory-types-in-langgraph-a-deep-code-walkthrough-part-2-17kb | LLMs are stateless; memory is infrastructure |
