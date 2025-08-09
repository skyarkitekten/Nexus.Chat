# PRD – Azure DevOps LLM Story Builder (MVP, with RAG)

## Summary

A lightweight Azure DevOps extension for business analysts that transforms approved features into grounded, high-quality user stories with acceptance criteria and test case outlines — using LLM + RAG from the historical backlog. Users can tweak AI behavior with configurable system prompts, top P, and predefined acceptance criteria templates. The RAG setup ensures generated stories reflect existing truth, avoid duplication, and maintain team-specific language.

## Narrative

In the old world, a BA writing stories had two problems: the blank page, and the “isn’t this already in the backlog?” worry.

In the new world, she selects a feature, clicks **Generate User Stories**, and the AI first scans her historical backlog. It finds relevant work items — past stories, acceptance criteria, even phrasing patterns — and uses them to ground new stories.

The result? Output that feels like it’s been written by her team before, with zero duplicates, matching tone, and aligned with real history. Developers trust it. Testers trust it. She spends her time refining, not retyping.

---

## Problem Statement

Business analysts spend significant time converting product owner–approved features into well-structured user stories. This is slow, inconsistent, and prone to rework. Even worse, without visibility into historical backlog context, new stories often duplicate existing ones or conflict with prior work.

Existing Azure DevOps tooling provides no intelligent assistance or deep backlog awareness. We need a solution that accelerates story creation while grounding it in historical truth and established team standards.

---

## Goals

### Business Goals

* Reduce BA story-writing time by 50%.
* Improve story quality and consistency across teams.
* Avoid duplicated or conflicting backlog items.
* Build trust in AI outputs by ensuring they’re based on real, historical backlog context.

### User Goals

* Draft user stories from features in seconds.
* Automatically include acceptance criteria and test case outlines.
* Match style and terminology of prior stories.
* Avoid creating duplicates or contradictions.
* Stay entirely within Azure DevOps.

### Non-Goals

* No epic splitting in MVP.
* No AI-driven prioritization or estimation.
* No analytics dashboards in MVP.

---

## User Stories

1. **As a BA**, I want the AI to check the historical backlog before drafting, so I don’t create duplicate or conflicting stories.
2. **As a BA**, I want the AI to use the terminology and style from our prior backlog items, so the stories feel consistent.
3. **As a BA**, I want to define acceptance criteria templates, so all generated stories match our format.
4. **As a BA**, I want to adjust system prompt and top P for tone, detail, and creativity.
5. **As a Tester**, I want AI-generated stories to include grounded test case outlines so I can quickly expand them into test plans.
6. **As a Developer**, I want stories that already match our technical language, so there’s no guesswork.

---

## User Experience – Step-by-Step Flow

1. **Entry Point**

   * BA opens a feature in Azure DevOps and clicks **Generate User Stories**.

2. **RAG Context Retrieval**

   * The extension queries the historical backlog for related items (based on feature text and metadata).
   * Returns relevant work items and acceptance criteria to the LLM.

3. **Configuration Panel**

   * System prompt field (editable), top P slider, and acceptance criteria template dropdown.
   * Previews key retrieved backlog items so BA sees what the AI is using for context.

4. **Generation**

   * LLM uses retrieved backlog context + BA configurations to produce:

     * Story title
     * Story description (aligned to prior stories)
     * Acceptance criteria (from template)
     * Test case outline

5. **Review & Edit**

   * Side-by-side preview with edit-in-place.
   * Highlighted references showing which backlog items informed the AI output.

6. **Save to Azure DevOps**

   * Creates backlog items linked to originating feature.
   * Acceptance criteria stored in standard field; test case outline added as related work item.

---

## Success Metrics

* Time to create a user story reduced by **≥50%**.
* **0 duplicate stories** generated across pilot teams.
* ≥85% of generated stories require **minor or no edits**.
* ≥90% of pilot users report increased trust in AI output.

---

## Technical Considerations

* **Azure DevOps Extension SDK** for UI embedding.
* **Authentication**: Azure DevOps user credentials + API token.
* **LLM API**: GPT-4o or similar via OpenAI; pluggable API layer for model swaps.
* **RAG Pipeline**:

  * Embed historical backlog (titles, descriptions, acceptance criteria) in vector DB (e.g., Azure Cognitive Search + embeddings).
  * Query relevant items by semantic similarity before generation.
  * Pass retrieved items into LLM context.
* **Deduplication**: Compare new story candidates to backlog embeddings before finalizing output.
* **Config Storage**: Store prompt/top P/templates in Azure DevOps extension storage or secure external DB.
* **Security**: Strip PII before embedding or sending to LLM.
* **Rate Limits**: Batch retrieval + generation where possible.

---

## Milestones & Sequencing

1. **Week 0–2** – Extension scaffold; UI framework in Azure DevOps.
2. **Week 3–4** – Vector DB + backlog embedding pipeline.
3. **Week 5–6** – LLM integration with RAG context injection.
4. **Week 7–8** – Configurable system prompt/top P/templates.
5. **Week 9–10** – Test case outline generation + deduplication checks.
6. **Week 11–12** – QA, pilot rollout, Marketplace publish.

---

With this RAG element, your MVP shifts from “AI story writer” to **“AI story writer that already knows your backlog”** — which is a massive differentiator in Azure DevOps land.

I can also give you **a sample RAG-enabled system prompt design** so your dev team knows exactly how to inject retrieved backlog context into the LLM.

Do you want me to drop that in next? It’ll make the PRD even more execution-ready.
