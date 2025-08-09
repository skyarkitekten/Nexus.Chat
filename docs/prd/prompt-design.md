# PRD: Prompt Design, Slash Commands & Hashtag Context Awareness for AI Retrieval Agent

## Summary

A structured prompt interaction layer for our chat-based feature definition tool, enabling precise, context-rich outputs from the AI retrieval agent. Users interact via **slash commands** to declare intent and **hashtags** to scope and filter context. This design ensures consistent, grounded outputs while reducing ambiguity and improving retrieval accuracy.

---

## Problem Statement

Without structured input, conversational AI can:

* Misinterpret intent.
* Retrieve irrelevant or incomplete historical data.
* Produce inconsistent output formats.
* Waste cycles asking for clarifications.

This leads to slower workflows, reduced trust in outputs, and more manual correction.
By introducing **structured commands** and **tag-based context signaling**, we give both humans and the AI a shared protocol for interaction — resulting in faster, more accurate, and reusable outputs.

---

## Goals

### Business Goals

* Improve retrieval accuracy by ≥25% over freeform queries.
* Reduce clarification turns per request by 50%.
* Increase adoption of AI-generated feature definitions and specs.

### User Goals

* Quickly issue precise commands to the agent without extra typing.
* Apply filters (domain, goal, phase, risk, region) to focus results.
* Receive outputs in a consistent, predictable structure.
* Trust that outputs are grounded in historical data with citations.

### Non-Goals

* Full natural language understanding without commands (will still be supported, but not optimized).
* Automatic creation of tags from context (future enhancement).
* Prioritization or scoring logic for proposed features.

---

## User Stories

1. As a PM, I can type `/define "One-Click Checkout" #domain/payments #goal/conversion` and get a grounded feature definition citing past work.
2. As a designer, I can use `/compare "Feature A" vs "Feature B" #domain/onboarding` to understand trade-offs.
3. As an engineer, I can filter retrieval to my scope by adding `#risk/security #tech/web`.
4. As a new hire, I can discover what tags are available via inline autocomplete.

---

## User Experience (Step-by-Step Flow)

1. **User invokes a slash command**:

   * `/define`, `/stories`, `/find`, `/compare`, `/push`.
2. **Optional hashtags** refine context:

   * `#domain/payments`, `#goal/conversion`, `#risk/security`.
3. **Agent parses intent + tags**:

   * Commands → output type & format.
   * Tags → retrieval filters & boosts.
4. **Semantic + lexical retrieval** performed over historical feature corpus.
5. **RAG generation** produces:

   * Human-readable section.
   * Machine-readable JSON block (retrieval metadata, citations, confidence).
6. **Output returned to chat** in consistent, structured format.
7. **User iterates** with follow-up commands, reusing tags or adjusting scope.

---

## Narrative

Imagine asking your AI assistant for “that checkout thing we talked about last year.”
Without structure, you might get partial results, irrelevant discussions, or an overlong summary.

Now, you type:
`/define "One-Click Checkout" #domain/payments #goal/conversion #region/us`

The agent instantly:

1. Retrieves the most relevant features and decisions from history.
2. Drafts a new definition grounded in actual past artifacts.
3. Cites those artifacts inline, so you can verify them.
4. Presents everything in a predictable format your team trusts.

Commands give precision. Tags give context. Together, they turn the chat into a powerful product memory interface.

---

## Success Metrics

* **Retrieval precision**: % of retrieved items rated relevant by users (target: ≥85%).
* **Clarification reduction**: Avg. clarifying Qs per request (target: -50%).
* **Citations rate**: % of outputs with ≥2 relevant citations (target: 90%).
* **Command adoption**: % of AI requests using slash commands (target: ≥70%).

---

## Technical Considerations

* **Command Parser**:

  * Regex + NLP fallback for extracting command + quoted strings.
  * Validation of allowed commands for V1 scope.
* **Tag Parser**:

  * Matches against canonical taxonomy (`#domain`, `#goal`, `#phase`, `#risk`, `#region`, `#tech`, `#persona`).
  * Ignores unknown tags but logs for analytics.
* **Retrieval Pipeline**:

  * Semantic + lexical search hybrid.
  * Hard filters from tags; soft boosts for ranking relevance.
* **RAG Prompt Template**:

  * System prompt for grounding.
  * Context table of retrieved items.
  * Output instructions for structure + citations.
* **Output Contract**:

  * Human-readable section for users.
  * JSON metadata block for downstream systems.
* **Latency Targets**:

  * Retrieval <800ms, generation <3.5s.

---

## Slash Commands (V1 Scope)

* `/define "<feature>" [#tags]` – Draft feature definition.
* `/stories "<feature>" [#tags]` – Generate user stories.
* `/find "<query>" [#tags]` – Search historical artifacts.
* `/compare "<A>" vs "<B>" [#tags]` – Produce trade-off analysis.
* `/push "<feature>" to [Jira|Linear]` – Send spec to backlog.

---

## Hashtag Taxonomy (V1)

* **Domain**: `#domain/payments`, `#domain/onboarding`
* **Goal**: `#goal/conversion`, `#goal/activation`
* **Phase**: `#phase/discovery`, `#phase/build`
* **Risk**: `#risk/security`, `#risk/latency`
* **Region**: `#region/eu`, `#region/us`
* **Persona**: `#persona/pm`, `#persona/engineer`

---

## Milestones & Sequencing

1. **XX weeks** — Implement command parser + core command set.
2. **XX weeks** — Tag parser + canonical taxonomy.
3. **XX weeks** — Retrieval filters/boosts from tags.
4. **XX weeks** — RAG prompt integration with structured output.
5. **XX weeks** — Autocomplete for commands + tags in chat UI.

---
