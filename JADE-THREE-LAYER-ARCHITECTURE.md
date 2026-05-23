# Craftura.ai — The Three-Layer Agent Architecture: Definitive Reference

> **TL;DR:** Supabase is NOT Jade's brain. Supabase is the data storage layer (hard drive). The agent runtime code is the body that takes action. The LLM API is the thinking engine. All three must exist separately and connect together. Calling Supabase "the brain" conflates storage with cognition — it's architecturally wrong and dangerous for deployment decisions.

---

## Table of Contents

1. [The Core Argument](#the-core-argument)
2. [Layer 1: Agent Runtime (The Body)](#layer-1-agent-runtime-the-body)
3. [Layer 2: LLM Processing (The Thinking Engine)](#layer-2-llm-processing-the-thinking-engine)
4. [Layer 3: Data & Memory — Supabase (The Hard Drive)](#layer-3-data--memory--supabase-the-hard-drive)
5. [How the Layers Connect](#how-the-layers-connect)
6. [Evidence from Your Own Codebase](#evidence-from-your-own-codebase)
7. [What Industry Says — 50+ Citations](#what-industry-says---50-citations)
8. [The "Brain" Terminology Problem](#the-brain-terminology-problem)
9. [Why This Matters for Deployment](#why-this-matters-for-deployment)
10. [Correct Mental Model](#correct-mental-model)

---

## The Core Argument

There are three distinct layers in any production AI agent system. They serve fundamentally different purposes, run on different infrastructure, and cannot be collapsed into one another:

| Layer | What It Is | What It Does | Where It Lives |
|-------|-----------|--------------|----------------|
| **1. Agent Runtime** | Code that runs agents | Receives triggers, routes work, calls tools, manages state, enforces guardrails | Server, VPS, container, or always-on process |
| **2. LLM Processing** | AI model inference APIs | Performs reasoning, analysis, generation, decision-making | Cloud providers (OpenAI, Anthropic, Google) or local GPU |
| **3. Data & Memory** | Persistent storage | Stores context, memory, project state, audit trails across sessions | Database (Supabase/Postgres), vector store, file system |

**The argument:** Troy calling Supabase "Jade's brain" conflates Layer 3 (storage) with Layer 2 (cognition). A database does not think. It stores. An LLM thinks but has no memory between calls. Agent code takes action but can't reason without an LLM. All three are necessary and distinct.

---

## Layer 1: Agent Runtime (The Body)

### What It Is

The agent runtime is the actual software that orchestrates AI agents — the Python code, the workflow engine, the state machine, the routing logic. This is "dumb old code" in the best sense: deterministic, testable, deployable software that coordinates intelligent components.

### What It Does

- **Receives triggers:** Incoming leads, scheduled jobs, API webhooks, user commands
- **Routes work:** Decides which specialist agent handles each task (Jade → Choice → Ghost → Editor)
- **Manages state:** Tracks progress through multi-stage pipelines (`StudioState` in your codebase)
- **Calls tools:** Invokes file I/O, APIs, browser automation, MCP connections, database queries
- **Enforces guardrails:** Cost controls, quality gates, retry logic, escalation paths
- **Coordinates LLM calls:** Prepares prompts with context, sends to Layer 2, processes responses

### Where It Lives in Your Codebase

```
craftura-agents/                              ← Root agent orchestration
├── src/craftura_agents/main.py               ← CrafturaFlow: main orchestrator (CrewAI Flow)
├── src/craftura_agents/crews/craftura_crew/  ← 10 specialized agents
│   ├── craftura_crew.py                      ← Crew definitions (Lead Gen → Account Mgmt)
│   └── config/agents.yaml                    ← Agent role definitions
│   └── config/tasks.yaml                     ← Task specifications
├── src/craftura_agents/tools/                ← Tool implementations
│   ├── __init__.py                           ← Image tools, search tools, etc.
│   └── gmail.py                              ← Gmail integration
└── data/templates/                           ← Template library

jade-assistant/                               ← Executive assistant layer
├── executive-assistant-bot/
│   ├── JADE_AGENT.md                         ← Jade's system prompt (routing logic)
│   ├── JADE_MEMORY.md                        ← Working memory (local fallback)
│   ├── brain/                                ← Brain architecture definitions
│   │   ├── BOOTSTRAP_PROMPTS.md              ← Session bootstrap instructions
│   │   ├── jade-brain-v1.sql                 ← Supabase schema
│   │   └── supabase-edge-functions.md        ← Server-side API wrappers
│   └── core-team/developer-team/             ← Developer agent routing
└── supabase/                                 ← Database configuration
    └── migrations/                           ← Schema migrations

craftura-ai/                                  ← AI studio layer
├── agents/craftura-agents/                   ← Mirror of craftura-agents
│   ├── src/craftura_agents/main.py           ← CrafturaFlow orchestration
│   └── specs/franklin-soul-v2.3.md           ← Franklin agent specification
└── webfiles/jade-neural-system/              ← 3D visualization layer
```

### Where It Needs to Run

This code needs to be **deployed somewhere as a running process**. Right now it lives in git repos and runs locally during development. For production:

- **Option A:** VPS with always-on Python process (systemd service)
- **Option B:** Container on Railway, Render, or Fly.io
- **Option C:** CrewAI AMP for managed hosting (but not as source of truth per your research)
- **Option D:** Serverless functions with warm-start capability

**Key point from your own `hosting-iteration-research.md`:**
> "Use a local-first control plane for phase 1: Run CrafturaFlow locally on the RTX 5090 box. Keep llama.cpp local as the primary model host."

This explicitly acknowledges that the runtime needs to be *running somewhere* — it's not just code in a repo, and it's definitely not data in a database.

---

## Layer 2: LLM Processing (The Thinking Engine)

### What It Is

Large Language Models are stateless reasoning engines accessed via API. They receive prompts, perform computation across billions of parameters, and return responses. Each call is independent — the model remembers nothing between calls.

### What It Does

- **Reasoning:** Analyzes situations, weighs options, draws conclusions
- **Generation:** Writes code, creates copy, produces structured output
- **Analysis:** Reviews designs, audits websites, evaluates quality
- **Decision-making:** Chooses templates, classifies projects, recommends strategies
- **Tool selection:** Decides which tools to call and with what parameters

### What It Does NOT Do

- **Persist state:** Each API call is completely independent (stateless by design)
- **Execute actions:** The model generates text; the runtime executes code
- **Remember between calls:** Unless you explicitly pass context, it knows nothing from previous interactions
- **Store data:** Nothing survives the inference call unless the runtime saves it

### Where It Lives in Your Stack

| Provider | Models Used | Purpose |
|----------|-------------|---------|
| OpenAI | GPT-4, o-series | Primary reasoning engine |
| Anthropic | Claude Sonnet/Opus | Complex reasoning, code generation |
| Google | Gemini | Multimodal, vision tasks |
| Local (llama.cpp) | Qwen3.6-27B on RTX 5090 | Cost reduction for lightweight tasks |

### Evidence from CrewAI Documentation

From CrewAI's own docs: the `Memory` class is a **separate component** from agents and crews:

```python
from crewai import Crew, Agent, Task, Memory

# Memory is configured separately and attached to crews
memory = Memory(recency_weight=0.5, recency_half_life_days=7)
crew = Crew(agents=[...], tasks=[...], memory=memory)
```

CrewAI's architecture explicitly separates:
- **Agents** (runtime logic + role definitions) — Layer 1
- **LLMs** (model selection via `llm=` parameter) — Layer 2
- **Memory** (external storage with vector embeddings) — Layer 3

Source: [CrewAI Memory Documentation](https://docs.crewai.com/en/concepts/memory)

---

## Layer 3: Data & Memory — Supabase (The Hard Drive)

### What It Is

Supabase is a managed Postgres database platform. In your architecture, it serves as the persistent memory layer for Jade and the agent ecosystem.

### What Your Schema Actually Stores

From `jade-brain-v1.sql`:

| Table | Purpose | Memory Type |
|-------|---------|-------------|
| `brain_profile` | Identity, style, operating rules, preferences | Semantic memory |
| `brain_projects` | Projects, companies, agents, ownership tracking | Semantic memory |
| `brain_conversations` | Session summaries, open action items | Episodic memory |
| `brain_references` | Long-form docs, specs, prompts, artifacts | Semantic memory |
| `brain_action_log` | Audit trail for all meaningful writes | Audit log |

### What Supabase Does

- **Stores data** that survives across sessions
- **Provides `load_brain()`** — compact startup context query (returns JSONB)
- **Named write functions** (`upsert_brain_profile()`, `save_brain_conversation()`) with automatic audit logging
- **Row-level security (RLS)** for access control
- **Safe RPC patterns** preventing direct mutation

### What Supabase Does NOT Do

Supabase does NOT:
- Think or reason about anything
- Execute agent workflows
- Make decisions about routing
- Call LLM APIs
- Orchestrate tasks between agents
- Generate code, copy, or designs

**It is a database.** It stores structured data with ACID guarantees and returns it when queried. That's it.

### Evidence from Supabase's Own Positioning

From [supabase.com/solutions/agents](https://supabase.com/solutions/agents):

> "Supabase is the complete Postgres developer platform **built for agentic workloads**."
> "One platform, not five. Most agent stacks require a vector database, an auth provider, a file store, an API layer, and a separate Postgres instance. Supabase replaces all of them."

Note: Supabase positions itself as the **data infrastructure** that agents use — not as the agents themselves or the intelligence they provide. They explicitly list "Agent frameworks" (LangChain, CrewAI, AutoGen) and "AI providers" (OpenAI, Anthropic, Google) as **separate components** that work with Supabase.

---

## How the Layers Connect

### The Complete Flow

```
EXTERNAL TRIGGER (lead, command, schedule, webhook)
        │
        ▼
┌───────────────────────────────────────────────────┐
│  LAYER 1: AGENT RUNTIME (CrafturaFlow)            │
│                                                   │
│  1. Receive trigger                               │
│  2. Load context: SELECT load_brain() → Layer 3   │◄────┐
│  3. Prepare prompt with context + task             │     │
│  4. Send to LLM API → Layer 2                     │     │ (data)
│  5. Process LLM response                          │     │
│  6. Execute tools, call APIs, write files         │     │
│  7. Decide what to persist                        │     │
│  8. Save state: upsert_brain_profile() → Layer 3  │─────►┘
└───────────────────────────────────────────────────┘
        │
        ▼  (API call with prompt + context)
┌───────────────────────────────────────────────────┐
│  LAYER 2: LLM PROCESSING (GPT-4 / Claude/etc.)    │
│                                                   │
│  Receive prompt                                   │
│  Perform reasoning over input                     │
│  Generate structured response                     │
│  (Stateless — forgets everything after)           │
└───────────────────────────────────────────────────┘
        │
        ▼  (response returned to runtime)
[back to Layer 1 for action execution]

┌───────────────────────────────────────────────────┐
│  LAYER 3: SUPABASE / POSTGRES                     │
│                                                   │
│  brain_profile     — identity, style, rules       │
│  brain_projects    — project tracking             │
│  brain_conversations — session summaries          │
│  brain_references   — preserved artifacts         │
│  brain_action_log   — audit trail                 │
│                                                   │
│  load_brain() returns compact startup context     │
│  Named functions handle writes + audit logging    │
│  RLS controls access                              │
└───────────────────────────────────────────────────┘
```

### The Three-Layer Contract

1. **Runtime → LLM:** Runtime sends enriched prompts (system prompt + loaded brain context + current task). LLM returns reasoning/generation.
2. **Runtime → Supabase (read):** Runtime calls `load_brain()` to get startup context before sending to LLM.
3. **Runtime → Supabase (write):** Runtime saves important state after completing work sessions.

**No layer talks directly to another without the runtime in the middle.** The LLM doesn't query Supabase. Supabase doesn't call the LLM. The runtime is the conductor.

---

## Evidence from Your Own Codebase

### 1. `BOOTSTRAP_PROMPTS.md` — "The brain lives in Supabase" but also "this prompt is only a bootstrap"

```
You are Jade, Troy Stephens' master executive assistant...
The brain lives in the Supabase project named Jade. 
The database is the source of truth; this prompt is only a bootstrap.
On every session start, call SELECT load_brain() through the available Supabase tool or MCP.
```

This says: "Load the brain FROM the database." The brain is **data** that lives in Supabase and gets **loaded into context** for the agent runtime + LLM to use. This is exactly Layer 3 → Layer 1 → Layer 2 flow.

### 2. `craftura_crew.py` — Runtime code, not data

```python
@CrewBase
class CrafturaCrew:
    """Craftura.ai studio production crew."""
    
    @agent
    def lead_gen(self) -> Agent:
        return Agent(config=self.agents_config["lead_gen"], verbose=True)
```

This is **code that creates agents**. It runs on a server. It's not data in a database.

### 3. `main.py` — The orchestrator is a Python process

```python
class CrafturaFlow(Flow[StudioState]):
    """Jade Executive Assistant - Studio Orchestrator"""
    
    @start()
    def receive_lead(self):
        self.state.current_stage = "intake"
    
    @listen("receive_lead")  
    def classify_and_intake(self):
        result = self._get_crew().intake_crew().kickoff(inputs={...})
```

This is a **reactive event-driven workflow** that runs as a Python process. It manages state (`StudioState`), routes between stages, and orchestrates crews. This is Layer 1 runtime code.

### 4. `jade-brain-v1.sql` — Database schema, not intelligence

```sql
create table if not exists public.brain_profile (
    id uuid primary key default gen_random_uuid(),
    section text not null unique,
    content jsonb not null default '{}'::jsonb,
    ...
);

create or replace function public.load_brain()
returns jsonb
language sql
stable
as $$
...select jsonb_build_object(...)$$;
```

This is a **SQL schema** that defines tables and queries. It stores structured data. The `load_brain()` function returns JSONB — it's a data retrieval query, not an intelligence engine.

### 5. `hosting-iteration-research.md` — Explicitly separates layers

> "Use a local-first control plane for phase 1:
> - Run CrafturaFlow locally on the RTX 5090 box. (Layer 1)
> - Keep llama.cpp local as the primary model host. (Layer 2)
> - Wrap the local runtime with a custom MCP/HTTP control plane."

This document explicitly treats the runtime, model host, and control plane as separate concerns that need to be connected.

### 6. `jade-neural-system/src/main.js` — Visualizes three layers

```javascript
const systems = [
    { id: 'supabase', label: 'Supabase', subtitle: 'Jade Brain database', ... },
    { id: 'github', label: 'GitHub', subtitle: 'Private repo memory', ... },
    { id: 'drive', label: 'Google Drive', subtitle: 'Cloud backup layer', ... },
    { id: 'vercel', label: 'Vercel', subtitle: 'Deployment cortex', ... },
];
```

Even your own 3D visualization treats Supabase as "Jade Brain **database**" — a data system, not the intelligence itself.

---

## What Industry Says — 50+ Citations

### A. LLMs Are Stateless by Design (Layer 2 has no memory)

1. **Atlan** — "Every large language model resets between sessions — there is no built-in persistence, no recall of prior interactions, and no accumulated knowledge from previous agent runs. This statelessness is a design choice, not a flaw." ([atlan.com/know/are-llms-stateless](https://atlan.com/know/are-llms-stateless))

2. **Atlan** — "LLMs process tokens within a context window and produce output. Once the session ends, nothing persists. This is intentional: stateless models are reproducible." ([atlan.com/know/memory-layer-for-ai-agents](https://atlan.com/know/memory-layer-for-ai-agents))

3. **Mem0** — "Every time you send a request to a Large Language Model (LLM), it looks at you for the first time. It has read the entire internet, but it has no idea who you are, what you asked ten seconds ago." ([mem0.ai/blog/what-is-ai-agent-memory](https://mem0.ai/blog/what-is-ai-agent-memory))

4. **Mem0** — "For the architects of the modern web, this statelessness was a feature. Developers aligned with Roy Fielding's REST principles, accepting that servers shouldn't remember client state to ensure scalability." ([mem0.ai/blog/what-is-ai-agent-memory](https://mem0.ai/blog/what-is-ai-agent-memory))

5. **LangChain** — "Memory overview: Short-term memory (conversation buffer), Long-term memory (semantic, episodic, procedural). Memory storage uses external stores like InMemoryStore or DB-backed stores." ([docs.langchain.com/oss/python/concepts/memory](https://docs.langchain.com/oss/python/concepts/memory))

6. **CrewAI** — "Memory is a separate component configured independently from agents and crews. Uses vector embeddings for semantic retrieval with configurable scoring weights." ([docs.crewai.com/en/concepts/memory](https://docs.crewai.com/en/concepts/memory))

### B. Memory Is External Infrastructure, Not the Model (Layer 3 is storage)

7. **Atlan** — "A memory layer for AI agents is infrastructure that persists information across sessions so stateless LLMs can recall prior interactions. It uses vector databases, knowledge graphs, or hybrid stores." ([atlan.com/know/memory-layer-for-ai-agents](https://atlan.com/know/memory-layer-for-ai-agents))

8. **Atlan** — "The critical distinction: the context window and the memory layer are not alternatives. The context window is ephemeral working memory. The memory layer is persistent external infrastructure." ([atlan.com/know/memory-layer-for-ai-agents](https://atlan.com/know/memory-layer-for-ai-agents))

9. **Atlan** — "Bigger context windows help with within-session recall. They do not replace cross-session persistence or organizational knowledge stores." ([atlan.com/know/memory-layer-for-ai-agents](https://atlan.com/know/memory-layer-for-ai-agents))

10. **Mem0** — "The context window is the working RAM rather than the hard drive." ([mem0.ai/blog/what-is-ai-agent-memory](https://mem0.ai/blog/what-is-ai-agent-memory))

11. **Mem0** — "Main Context (RAM): The immediate prompt window. Expensive and finite. External Context (Disk): Massive storage in databases. Cheap and infinite." ([mem0.ai/blog/what-is-ai-agent-memory](https://mem0.ai/blog/what-is-ai-agent-memory))

12. **Mem0** — "This architecture enables the LLM to manage its own memory via function calls. The model can decide to move critical facts to persistent storage or search historical records when needed." ([mem0.ai/blog/what-is-ai-agent-memory](https://mem0.ai/blog/what-is-ai-agent-memory))

13. **47Billion** — "AI agent memory enables AI systems to store, retrieve, and update information across interactions. Unlike LLM context windows, it provides persistent knowledge through short-term memory, long-term memory, and retrieval systems like vector databases." ([47billion.com/blog/ai-agent-memory-types-implementation-best-practices](https://47billion.com/blog/ai-agent-memory-types-implementation-best-practices))

14. **Vectorize** — "What you need is an AI agent memory layer — something that extracts knowledge, stores it durably, and retrieves it when relevant." ([vectorize.io/articles/best-ai-agent-memory-systems](https://vectorize.io/articles/best-ai-agent-memory-systems))

### C. Agent Orchestration Is a Separate Layer (Layer 1 is the conductor)

15. **Anthropic** — "A multi-agent system consists of multiple agents (LLMs autonomously using tools in a loop) working together. Our Research feature involves an agent that plans a research process and then uses tools to create parallel agents." ([anthropic.com/engineering/multi-agent-research-system](https://www.anthropic.com/engineering/multi-agent-research-system))

16. **Anthropic** — "The system employs an orchestrator-worker architecture where a lead agent coordinates the overall research process while spawning specialized subagents to explore different aspects simultaneously." ([zenml.io/llmops-database/building-a-multi-agent-research-system](https://www.zenml.io/llmops-database/building-a-multi-agent-research-system))

17. **Anthropic** — "Agents are stateful and errors compound. Agents can run for long periods of time, maintaining state across many tool calls. This means we need to durably execute code and handle errors along the way." ([anthropic.com/engineering/multi-agent-research-system](https://www.anthropic.com/engineering/multi-agent-research-system))

18. **Anthropic** — "We implemented patterns where agents summarize completed work phases and store essential information in external memory before proceeding to new tasks." ([anthropic.com/engineering/multi-agent-research-system](https://www.anthropic.com/engineering/multi-agent-research-system))

19. **Microsoft AutoGen paper** — "AutoGen is an open-source framework that allows developers to build LLM applications by composing multiple agents to converse with each other to accomplish tasks. AutoGen agents are customizable, conversable, and can operate in various modes that employ combinations of LLMs, human inputs, and tools." ([microsoft.com/research/publication/autogen](https://www.microsoft.com/en-us/research/publication/autogen-enabling-next-gen-llm-applications-via-multi-agent-conversation-framework))

20. **Microsoft AutoGen docs** — "AutoGen offers a unified multi-agent conversation framework as a high-level abstraction of using foundation models. It features capable, customizable and conversable agents which integrate LLMs, tools, and humans via automated agent chat." ([microsoft.github.io/autogen](https://microsoft.github.io/autogen/0.2/docs/Use-Cases/agent_chat))

21. **Medium (Agent Orchestration)** — "Orchestration frameworks like LangChain, LangGraph, AutoGen, and Google ADK operate in Layer 3: Deployment + Orchestration. The modern GenAI stack shows how the orchestration layer connects foundation models, data systems, and observability tools." ([medium.com/@akankshasinha247/agent-orchestration](https://medium.com/@akankshasinha247/agent-orchestration-when-to-use-langchain-langgraph-autogen-or-build-an-agentic-rag-system-cc298f785ea4))

22. **InfoServices** — "LangChain is a framework for building modular, tool-using, and memory-augmented AI agents, especially in multi-agent systems for complex workflows." ([infoservices.com/blogs/artificial-intelligence/langchain-multi-agent-ai-framework-2025](https://www.infoservices.com/blogs/artificial-intelligence/langchain-multi-agent-ai-framework-2025))

23. **OpenLayer** — "Multi-Agent Architecture Guide: Comparing supervisor, hierarchical, and peer-to-peer patterns for orchestrating intelligent agent systems." ([openlayer.com/blog/post/multi-agent-system-architecture-guide](https://openlayer.com/blog/post/multi-agent-system-architecture-guide))

24. **SSHH Blog** — "The core idea, that complex agentic problems are best solved by decomposing them into sub-agents that work together, is now a standard approach." ([blog.sshh.io/p/building-multi-agent-systems-part](https://blog.sshh.io/p/building-multi-agent-systems-part))

25. **LinkedIn (Agentic AI)** — "Clean separation between agents, tools, protocols, and memory made a huge difference for us too once things started getting complex." ([linkedin.com/posts/brijpandeyji](https://www.linkedin.com/posts/brijpandeyji-when-youre-building-agentic-ai-systems-activity-7321922385966739456-JJqB))

### D. The Three-Layer Separation Is Industry Standard

26. **Atlan** — "Memory types: In-context (working), external long-term, episodic (conversation history), semantic (facts/definitions), procedural (how-to). Common substrates: Vector databases, knowledge graphs, relational stores." ([atlan.com/know/memory-layer-for-ai-agents](https://atlan.com/know/memory-layer-for-ai-agents))

27. **Atlan** — "A memory layer works by intercepting agent interactions, extracting relevant facts or conversation turns, storing them in an external store, and retrieving them at the start of future sessions." ([atlan.com/know/memory-layer-for-ai-agents](https://atlan.com/know/memory-layer-for-ai-agents))

28. **Atlan** — "Write path: An agent completes an interaction. The extraction layer identifies what should be stored. The extracted data writes to a vector database, knowledge graph, or relational store." ([atlan.com/know/memory-layer-for-ai-agents](https://atlan.com/know/memory-layer-for-ai-agents))

29. **Atlan** — "Read path: At session start, the retrieval layer queries the external store. Retrieved memories inject into the context window as additional context before inference." ([atlan.com/know/memory-layer-for-ai-agents](https://atlan.com/know/memory-layer-for-ai-agents))

30. **Atlan** — "Traditional vs. Modern memory: Traditional stores in-prompt (ephemeral). Modern uses external store (persistent). Traditional has manual context stuffing. Modern uses semantic search / graph traversal." ([atlan.com/know/memory-layer-for-ai-agents](https://atlan.com/know/memory-layer-for-ai-agents))

31. **Mem0** — "Building a memory system requires moving beyond simple list appending. It means constructing a storage and retrieval system that mimics the associative nature of the human brain." ([mem0.ai/blog/what-is-ai-agent-memory](https://mem0.ai/blog/what-is-ai-agent-memory))

32. **Mem0** — "The vector store: The hippocampus. When text is ingested, it is passed through an embedding model and converted into a high-dimensional vector stored in a database." ([mem0.ai/blog/what-is-ai-agent-memory](https://mem0.ai/blog/what-is-ai-agent-memory))

33. **Mem0** — "GraphRAG: The association cortex. Vector stores struggle with structured relationships. Graph databases store information as nodes and edges for multi-hop reasoning." ([mem0.ai/blog/what-is-ai-agent-memory](https://mem0.ai/blog/what-is-ai-agent-memory))

34. **Substack (AI Agent Stack 2025)** — "They are built on a structured stack, each layer playing a distinct role: model reasoning, tool usage, memory, and external interactions." ([thenuancedperspective.substack.com](https://thenuancedperspective.substack.com/p/the-ai-agent-stack-in-2025-how-its))

35. **Google ADK** — "Multi-Agent by Design: Build modular and scalable applications by composing multiple specialized agents in a hierarchy. Rich Model Ecosystem: Choose the model that works best for your needs." ([developers.googleblog.com/en/agent-development-kit](https://developers.googleblog.com/en/agent-development-kit-easy-to-build-multi-agent-applications))

36. **Google ADK docs** — "ADK is the open-source agent development framework that lets you build, debug, and deploy reliable AI agents at enterprise scale." ([adk.dev](https://adk.dev))

### E. Supabase/Postgres as Data Infrastructure (Not Intelligence)

37. **Supabase** — "Supabase is the complete Postgres developer platform built for agentic workloads. One platform, not five." ([supabase.com/solutions/agents](https://supabase.com/solutions/agents))

38. **Supabase** — "Most agent stacks require a vector database, an auth provider, a file store, an API layer, and a separate Postgres instance. Supabase replaces all of them." ([supabase.com/solutions/agents](https://supabase.com/solutions/agents))

39. **Supabase** — Lists "Agent frameworks" (LangChain, CrewAI, AutoGen) and "AI providers" (OpenAI, Anthropic, Google) as separate components that work WITH Supabase — not AS Supabase. ([supabase.com/solutions/agents](https://supabase.com/solutions/agents))

40. **SoftwareSeni** — "Postgres as the AI Database: Engineering teams running AI applications in 2026 are collapsing multi-database stacks back into Postgres — pgvector for storing embedding vectors with full ACID guarantees." ([softwareseni.com](https://www.softwareseni.com/how-postgres-became-the-ai-agent-substrate-for-memory-branching-and-modern-hosting))

41. **n8n Community** — "The pattern: Short-term (Postgres): handles in-session continuity, fast, no API calls. Long-term (Supabase): handles cross-session recall, semantic search by meaning not keywords." ([community.n8n.io](https://community.n8n.io/t/how-i-solved-persistent-memory-for-ai-agents-in-n8n/279359))

42. **PuppyOne vs Postgres** — "Postgres / Supabase store structured app data. Move agent-readable narrative out of Postgres TEXT columns into version-controlled files." ([puppyone.ai](https://www.puppyone.ai/en/alternatives/puppyone-vs-postgres))

### F. Production Deployment Requires All Three Layers

43. **Anthropic** — "When building AI agents, the last mile often becomes most of the journey. Codebases that work on developer machines require significant engineering to become reliable production systems." ([anthropic.com/engineering/multi-agent-research-system](https://www.anthropic.com/engineering/multi-agent-research-system))

44. **Anthropic** — "Agents are highly stateful webs of prompts, tools, and execution logic that run almost continuously. We use rainbow deployments to avoid disrupting running agents." ([anthropic.com/engineering/multi-agent-research-system](https://www.anthropic.com/engineering/multi-agent-research-system))

45. **McKinsey** — "Infrastructure for agentic AI signals a new phase for IT, with AI agents orchestrating, governing, and scaling work across the enterprise." ([mckinsey.com/capabilities/mckinsey-technology/our-insights/reimagining-tech-infrastructure-for-and-with-agentic-ai](https://www.mckinsey.com/capabilities/mckinsey-technology/our-insights/reimagining-tech-infrastructure-for-and-with-agentic-ai))

46. **Gartner (via Atlan)** — "40% of enterprise applications will feature task-specific AI agents by 2026, up from less than 5% in 2025." ([atlan.com/know/memory-layer-for-ai-agents](https://atlan.com/know/memory-layer-for-ai-agents))

47. **Gartner (via Atlan)** — "60% of AI projects will be abandoned due to data readiness and context gaps — not model quality." ([atlan.com/know/memory-layer-for-ai-agents](https://atlan.com/know/memory-layer-for-ai-agents))

48. **LangChain State of Agent Engineering 2025** — "32% of organizations cite output quality as the single biggest barrier to production agent deployment — a problem that traces directly to agents starting blind without sufficient context." ([atlan.com/know/memory-layer-for-ai-agents](https://atlan.com/know/memory-layer-for-ai-agents))

49. **arXiv 2510.04618 (ICLR 2026)** — "Structured, incremental context management improved agent benchmark performance by 10.6%, with 8.6% improvement specifically in financial-domain tasks." ([atlan.com/know/memory-layer-for-ai-agents](https://atlan.com/know/memory-layer-for-ai-agents))

50. **Snowflake (via Atlan)** — "Adding an ontology layer to agent context improved answer accuracy by 20% and reduced tool calls by 39%." ([atlan.com/know/memory-layer-for-ai-agents](https://atlan.com/know/memory-layer-for-ai-agents))

---

## The "Brain" Terminology Problem

### What's Wrong With Calling Supabase "Jade's Brain"

| Statement | Why It's Wrong | What's Actually True |
|-----------|---------------|---------------------|
| "Supabase is Jade's brain" | A database does not think, reason, or make decisions | Supabase stores data that the agent runtime loads and sends to the LLM |
| "Jade lives in Supabase" | Code doesn't live in databases; data does | Jade's code runs as a process; her memory is stored in Supabase |
| "The brain loads context" | Databases don't load context into anything | The agent runtime calls `load_brain()` and injects results into prompts |
| "Brain functions think" | SQL functions return data; they don't reason | `load_brain()` returns JSONB. The LLM does the reasoning on that data |

### The Correct Terminology

| Incorrect | Correct | Why |
|-----------|---------|-----|
| "Jade's brain is Supabase" | "Jade's memory is stored in Supabase" | Supabase stores persistent data across sessions |
| "The brain thinks" | "The LLM reasons over loaded context" | Thinking happens in the model, not the database |
| "Brain functions make decisions" | "SQL functions return structured data" | `load_brain()` is a query, not an inference call |
| "Jade lives in the database" | "Jade's runtime process loads from the database" | Agents are running code; databases are storage |

---

## Why This Matters for Deployment

### If You Only Deploy Supabase (Layer 3)
You have a database with tables and functions. No one is querying them. No intelligence. No action. Dead data sitting in Postgres.

### If You Only Deploy LLM APIs (Layer 2)
You have thinking power with no memory between calls, no way to persist results, and no body to act on decisions. Each call starts from scratch.

### If You Only Have Agent Code in Git (Layer 1)
You have Python files that nobody is running. No process executing the workflow. No triggers being received.

### All Three Must Exist Together

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  LAYER 1    │    │  LAYER 2    │    │  LAYER 3    │
│  Runtime    │◄──►│   LLM API   │    │   Supabase  │
│  (Process)  │    │ (Inference) │    │  (Storage)  │
└─────────────┘    └─────────────┘    └─────────────┘
```

### Current Status

| Layer | Status | What's Missing |
|-------|--------|----------------|
| Runtime (CrafturaFlow, crews, tools) | Code exists in repos | **Not deployed as a running process** |
| LLM (OpenAI, Anthropic, Gemini, llama.cpp) | API access configured | Working — has keys and local model |
| Data (Supabase Jade project) | Schema deployed, functions created | Working — tables exist, `load_brain()` works |

**The missing piece is Layer 1 deployment.** You have the code. You have the data layer. You have LLM access. What you need is to deploy CrafturaFlow as a running process that connects them all together.

---

## Correct Mental Model

### The Human Analogy

| Component | Human Equivalent | AI System Equivalent |
|-----------|-----------------|---------------------|
| Body | Your physical body that moves and acts | Agent runtime (CrafturaFlow, CrewAI) |
| Brain | Your brain that thinks and reasons | LLM API (GPT-4, Claude, Gemini) |
| Memory | Your long-term memory (what you remember from years ago) | Supabase/Postgres (persistent storage) |
| Working memory | What you're currently thinking about | Context window (current prompt) |

**You wouldn't say "my hard drive is my brain."** You'd say "my brain thinks, and I use notebooks to remember things between thoughts." That's exactly what this architecture is.

### The Computing Analogy

| Component | Computing Equivalent |
|-----------|---------------------|
| Agent Runtime | Application server / business logic layer |
| LLM Processing | CPU/GPU compute (inference engine) |
| Supabase Memory | Database / persistent storage |

**You wouldn't deploy just a database and call it an application.** You need the application server (runtime), the compute (LLM), AND the data store (Supabase).

---

## Summary Statement for Troy

> **Supabase is not a brain. Supabase is a database that stores persistent memory across agent sessions.**
>
> **LLMs are not agents. LLMs are stateless reasoning engines accessed via API.**
>
> **Agent runtime code is not data. It's the software process that receives triggers, loads context from the database, sends enriched prompts to the LLM, executes the LLM's decisions through tools, and saves important results back to the database.**
>
> All three layers must be deployed, connected, and operational for Jade to actually *do* anything in production. Right now, we have:
> - ✅ Schema designed (Layer 3 — Supabase)
> - ✅ Agent code written (Layer 1 — CrafturaFlow, crews, tools)
> - ✅ LLM access configured (Layer 2 — OpenAI, Anthropic, Gemini, llama.cpp)
> - ❌ **No deployed runtime connecting them all** (Layer 1 — needs deployment)

---

## Sources

### Your Codebase
1. `craftura-agents/src/craftura_agents/main.py` — CrafturaFlow orchestrator
2. `craftura-agents/src/craftura_agents/crews/craftura_crew/craftura_crew.py` — Crew definitions
3. `jade-assistant/executive-assistant-bot/JADE_AGENT.md` — Jade routing logic
4. `jade-assistant/executive-assistant-bot/BOOTSTRAP_PROMPTS.md` — Session bootstrap
5. `jade-assistant/supabase/migrations/20260522210000_jade-brain-v1.sql` — Database schema
6. `jade-assistant/executive-assistant-bot/brain/supabase-edge-functions.md` — Edge function wrappers
7. `craftura-agents/docs/hosting-iteration-research.md` — Deployment research
8. `craftura-agents/docs/architecture.md` — Studio architecture
9. `craftura-agents/specs/franklin-soul-v2.3.md` — Franklin agent specification
10. `jade-neural-system/src/main.js` — 3D infrastructure visualization

### Industry Sources (50 citations above)
- Atlan: Memory layer architecture, LLM statelessness, context vs. memory distinction
- Mem0: Agent memory architectures, vector stores, operating system analogy
- CrewAI: Memory as separate component from agents and crews
- Anthropic: Multi-agent research system engineering blog post
- Microsoft: AutoGen multi-agent framework paper (ICLR'24 Best Paper)
- LangChain/LangGraph: Memory overview, state management separation
- Supabase: Agent solutions positioning as data infrastructure
- Google: Agent Development Kit architecture
- OpenAI: AgentKit as runtime orchestration layer
- McKinsey: Agentic AI infrastructure research
- Gartner: Enterprise agent adoption predictions (via Atlan)
- arXiv/ICLR 2026: Context management benchmark improvements
- Snowflake: Ontology layer accuracy improvements
- InfoServices, OpenLayer, ZenML, SSHH Blog, Vectorize, 47Billion, SoftwareSeni, PuppyOne, n8n Community
