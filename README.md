# Hermes Agent Internal Architecture

A technical reference documenting the internal workings of the Hermes Agent framework by Nous Research.

## Scope

This book covers only the **agent loop system** — how Hermes thinks, acts, delegates, remembers, and stops. It deliberately excludes:
- Terminal, web, and desktop UI implementations
- Gateway/messaging platform adapters
- Installation and setup procedures
- Provider-specific API adapters

## Chapters

1. [Architecture & The Conversation Loop](./src/architecture/index.md) — The core agent loop, message flow, and system prompt construction
2. [Guardrails & Safety](./src/guardrails/index.md) — Prompt-injected rules, runtime checks, and enforcement mechanisms
3. [Agent Spawning & Delegation](./src/spawning/index.md) — How and why Hermes spawns subagents, and the isolation boundaries between them
4. [Self-Correction](./src/self-correction/index.md) — Error classification, retry strategies, fallback chains, and recovery mechanisms
5. [Termination Conditions](./src/termination/index.md) — When and how Hermes knows to stop a turn or session
6. [LLM Influence on Behavior](./src/llm-influence/index.md) — How different models, providers, and modalities shape agent behavior
7. [Skill Creation & Management](./src/skills/index.md) — How skills are authored, loaded, dispatched, and curated
8. [Tool Creation & Dispatch](./src/tools/index.md) — Tool registration, schema generation, execution, and the registry system
9. [Core Memory Design](./src/memory/index.md) — The memory stack: core memory, session search, auxiliary clients, and memory providers

## Source Reference

All descriptions are derived from the Hermes Agent source code
(`NousResearch/hermes-agent`, commit `65bc6727b4`, 2026-10-06).
