# Agent Architecture — One Pager

## The Three Layers (Non-Negotiable)

### Layer 1: AGENT RUNTIME — The Body
**What:** Python code that runs agents, routes work, calls tools, manages state
**Your code:** `CrafturaFlow`, `CrafturaCrew`, Jade routing logic
**Where it lives:** Server, VPS, container — needs to be a running process
**Status:** ✅ Code written, ❌ Not deployed

### Layer 2: LLM PROCESSING — The Thinking Engine  
**What:** Stateless AI models that reason, generate, analyze (GPT-4, Claude, Gemini)
**Your stack:** OpenAI + Anthropic + Google + local llama.cpp on RTX 5090
**Where it lives:** Cloud APIs or local GPU inference server
**Status:** ✅ Working

### Layer 3: DATA & MEMORY — The Hard Drive (Supabase)
**What:** Postgres database that stores context across sessions
**Your schema:** `brain_profile`, `brain_projects`, `brain_conversations`, `brain_references`, `brain_action_log`
**Where it lives:** Supabase cloud (managed Postgres)
**Status:** ✅ Working

## The Flow

```
Trigger → Runtime loads brain from Supabase → Runtime sends prompt to LLM → 
LLM returns reasoning → Runtime executes action → Runtime saves results to Supabase
```

## What's Wrong With "Supabase Is Jade's Brain"

| Claim | Reality |
|-------|---------|
| "Supabase is the brain" | Supabase is a **database**. It stores data. It does not think. |
| "Jade lives in Supabase" | Jade's code runs as a **process**. Her memory is stored in Supabase. |
| "Brain functions make decisions" | `load_brain()` returns JSONB. The **LLM** makes decisions on that data. |

## Correct Mental Model

```
Body (Runtime) ↔ Brain (LLM) ↔ Memory (Supabase)
   Code           Thinking       Storage
  Runs somewhere  Stateless API  Database
```

- A body without a brain can't think → Runtime without LLM = dead code
- A brain without a body can't act → LLM without runtime = text with no action  
- A brain without memory forgets everything → LLM is stateless by design
- Memory without a brain is just data → Supabase without runtime = empty database

## What's Missing

**Deploy Layer 1.** You have all three layers built. What you need is CrafturaFlow running as an always-on process that connects them together.

---

*Sources: CrewAI docs, Anthropic multi-agent engineering blog, Atlan memory layer analysis, Mem0 architecture, Microsoft AutoGen (ICLR'24), Supabase agent solutions page, your own codebase across 13 repos.*
