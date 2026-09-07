# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Copayo is a hackathon project: a conversational agent (via Telegram) that estimates a patient's copago (copay) and health insurance coverage, and recommends the cheapest hospital in their network for a given symptom.

Stack: n8n (workflow orchestration, self-hosted VPS) + Notion API (data store) + Groq (LLM, `openai/gpt-oss-120b`) + Telegram Bot API (channel).

The repo holds the delivered MVP: the exported n8n workflow (`n8n/workflow-copago.json`), the full Notion schema and reference data (`notion/schema.md`), the two Groq system prompts and their test guide (`prompts/`), the project docs (`docs/`), and a tagged release. There are no build/lint/test commands because there is nothing to build: the runtime logic lives in the n8n workflow, not in a codebase run locally.

## Non-negotiable architecture rule

Copay calculation (`costo_base × regla_del_plan`) and comparing the 3 hospitals to find the cheapest one **must** happen in an n8n Function/Code node using plain JavaScript — never delegated to the LLM. Groq is only given already-computed data and turns it into natural-language responses; it never calculates or compares prices. This constraint applies to any workflow node, prompt, or code added to this project.

## Flow

```
Telegram Trigger → Notion (lookup) → Function/Code (deterministic calculation) → HTTP Request to Groq (wording) → Telegram (reply)
```

## Repo structure and ownership

Each top-level folder belongs to a different person on the 3-person team; respect that boundary when editing.

- `/docs` — architecture, scope, test cases (shared, each person writes their own section)
- `/n8n` — the exported `workflow.json` (owned by the repo maintainer)
- `/prompts` — Groq system prompt and examples (owned by a teammate)
- `/notion` — schema documentation **and** the full reference data of the 3 Notion bases (owned by a teammate). Exception to the "docs only" rule: the copago is deterministic and small, and the hackathon judges need to see the exact Síntomas/Hospitales/Planes values without Notion access, so `notion/schema.md` carries the complete data tables. Notion remains the source of truth; `schema.md` must be kept in sync with it.

## Deadline

Sunday, September 6, 2026. Deliverable: bot link + repo link sent to hackiathon@viamatica.com.
