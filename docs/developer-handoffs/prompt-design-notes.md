# Developer Handoff: Prompt Design, Slash Commands & Hashtag Context Awareness

## 1. Overview

The system must:

1. **Parse commands** and **hashtags** from user input.
2. Use parsed data to:

   * Shape the retrieval query.
   * Filter or boost relevant artifacts.
   * Select the correct generation template.
3. Return:

   * **Human-readable output** (for the chat).
   * **Machine-readable JSON** (for downstream integration).

---

## 2. Command Parsing Rules

**Format:**

```
/<command> "<quoted text>" [#tag1 #tag2 ...] [free text]
```

**Supported Commands (V1):**

| Command    | Purpose                                     | Output Type      |
| ---------- | ------------------------------------------- | ---------------- |
| `/define`  | Draft a new feature definition              | Feature Spec     |
| `/stories` | Generate user stories & acceptance criteria | Stories List     |
| `/find`    | Search historical artifacts                 | Search Results   |
| `/compare` | Compare two features or ideas               | Comparison Table |
| `/push`    | Send finalized feature to backlog tool      | Confirmation     |

**Parsing Logic:**

* **Command**: String starting with `/` and matching allowed commands.
* **Quoted String**: First quoted phrase after command → main feature/query target.
* **Hashtags**: Strings starting with `#` → parsed into `{category}/{value}`.
* **Free Text**: Any remaining text after hashtags.

---

## 3. Hashtag Parsing Rules

**Canonical Categories:**

* `#domain/*` – Product domain (payments, onboarding, etc.)
* `#goal/*` – Business or user goal (conversion, activation, retention)
* `#phase/*` – Stage (discovery, build, rollout)
* `#risk/*` – Risk category (security, latency, regulatory)
* `#region/*` – Geographic scope
* `#tech/*` – Technology reference
* `#persona/*` – Target user type

**Behavior:**

* **Hard filter**: Limit retrieval to artifacts matching tags exactly (e.g., `#domain/payments`).
* **Soft boost**: Increase ranking score for semantic matches with tag values.
* **Fallback**: If no tags provided, infer from chat thread context.

---

## 4. Retrieval API Contract

**Request Payload:**

```json
{
  "command": "/define",
  "query": "One-Click Checkout",
  "tags": {
    "domain": ["payments"],
    "goal": ["conversion"],
    "region": ["us"]
  },
  "free_text": "",
  "retrieval": {
    "top_k": 8,
    "filters": {
      "domain": ["payments"],
      "region": ["us"]
    },
    "boosts": {
      "goal": ["conversion"]
    }
  }
}
```

**Response Payload (Retrieval Layer):**

```json
{
  "results": [
    {
      "id": "F-221",
      "title": "Instant Checkout",
      "summary": "Rolled out in EU, improved conversions by 7%.",
      "date": "2022-04-10",
      "status": "shipped",
      "decision": "implemented in EU only",
      "similarity": 0.83,
      "link": "https://..."
    },
    {
      "id": "RFC-91",
      "title": "Express Pay",
      "summary": "Proposed in 2023, blocked by PSP limitations.",
      "date": "2023-02-14",
      "status": "rejected",
      "decision": "not feasible with current provider",
      "similarity": 0.77,
      "link": "https://..."
    }
  ]
}
```

---

## 5. RAG Generation API Contract

**Request Payload:**

```json
{
  "command": "/define",
  "feature_name": "One-Click Checkout",
  "tags": ["#domain/payments", "#goal/conversion", "#region/us"],
  "retrieved_items": [
    {
      "id": "F-221",
      "title": "Instant Checkout",
      "decision": "implemented in EU only",
      "key_learnings": "High conversion uplift, minor fraud increase",
      "similarity": 0.83
    },
    {
      "id": "RFC-91",
      "title": "Express Pay",
      "decision": "blocked by PSP limitations",
      "key_learnings": "Technical constraint: tokenization support missing",
      "similarity": 0.77
    }
  ],
  "instructions": "Draft a grounded feature definition. Cite all IDs."
}
```

**Response Payload:**

```json
{
  "human_readable": "**Feature:** One-Click Checkout\n\n**Problem:** Checkout flow has multiple steps causing drop-off...\n\n**Goals:** ...\n\n**User Stories:** ...\n\n**Citations:** [F-221], [RFC-91]",
  "machine_readable": {
    "command": "/define",
    "feature_name": "One-Click Checkout",
    "citations": ["F-221", "RFC-91"],
    "confidence": 0.74,
    "open_questions": ["What PSPs will support tokenization in US?"]
  }
}
```

---

## 6. Output Structure Requirements

**Human-Readable Section (always first):**

* Must be in markdown.
* Sections vary by command (see PRD output templates).

**Machine-Readable JSON (always second):**

* Includes:

  * Command
  * Main entity (feature name/query)
  * Parsed tags
  * Retrieval metadata (IDs, similarity scores)
  * Citations array
  * Confidence score
  * Open questions (if any)

---

## 7. Example End-to-End Flow

**User Input:**

```
/define "One-Click Checkout" #domain/payments #goal/conversion #region/us
```

**Steps:**

1. Parser extracts:

   * Command: `/define`
   * Feature: `"One-Click Checkout"`
   * Tags: domain=payments, goal=conversion, region=us
2. Retrieval query built with filters + boosts.
3. Top 8 historical artifacts retrieved.
4. RAG prompt generated with system + retrieval context.
5. LLM outputs:

   * Human-readable spec.
   * JSON block with citations & metadata.

---

## 8. Open Implementation Questions

* Should hashtags be **validated against a fixed taxonomy** or allow freeform entry with soft matching?
* Should we **require citations** in every output, even if no strong matches are found (flag as “assumptions”)?
* Do we want **autocomplete** for commands and tags in the chat UI for V1, or V2?

---

## 9. Few-Shot Prompt Examples (for Prompt Engineering & QA)

These examples show **full input → retrieval context → generation output** for each core command.
They are designed to:

* Demonstrate correct parsing of **slash commands** and **hashtags**.
* Show **retrieval filters/boosts** in action.
* Enforce **citations** and **grounded outputs**.
* Provide QA teams with **acceptance baselines**.

---

### Example 1 — `/define`

**User Input:**

```
/define "One-Click Checkout" #domain/payments #goal/conversion #region/us
```

**Parsed Command:**

```json
{
  "command": "/define",
  "feature_name": "One-Click Checkout",
  "tags": {
    "domain": ["payments"],
    "goal": ["conversion"],
    "region": ["us"]
  }
}
```

**Retrieval Output:**

```json
{
  "results": [
    {
      "id": "F-221",
      "title": "Instant Checkout",
      "decision": "Implemented in EU only",
      "key_learnings": "High conversion uplift (+7%), minor fraud increase",
      "similarity": 0.83
    },
    {
      "id": "RFC-91",
      "title": "Express Pay",
      "decision": "Blocked by PSP limitations",
      "key_learnings": "No tokenization support in US PSPs",
      "similarity": 0.77
    }
  ]
}
```

**Generation Prompt Sent to LLM:**

```
Context:
- Domain: payments
- Tags: #goal/conversion #region/us
- User Request: Draft a grounded feature definition for "One-Click Checkout".
- Retrieved Items:
  1. F-221 | Instant Checkout | Implemented in EU only | High conversion uplift (+7%), minor fraud increase
  2. RFC-91 | Express Pay | Blocked by PSP limitations | No tokenization support in US PSPs

Task:
1) Check for duplicates or overlap.
2) Draft a full feature definition (problem, goals, scope, user stories, risks).
3) Cite IDs inline.
4) List open questions if unknowns remain.
```

**Expected Output:**

```markdown
**Feature:** One-Click Checkout

**Problem:** Current US checkout requires multiple steps, contributing to drop-off rates at payment entry [F-221].

**Goals:**
- **Business:** Increase US checkout conversion by ≥5%.
- **User:** Reduce checkout completion time to <10 seconds.

**Scope:**
- Single-step purchase flow for returning customers with stored payment details.
- PSP integration must support tokenization [RFC-91].

**Non-Goals:**
- Multi-item cart optimization.
- Subscription flows.

**User Stories:**
1. As a returning user with stored payment details, I can confirm my purchase in one click so that I check out faster.
2. As a security analyst, I can ensure tokenization compliance so that PCI risk is minimized.

**Risks:**
- Fraud increase seen in EU pilot [F-221].
- PSP integration blockers [RFC-91].

**Open Questions:**
- Which US PSPs can support tokenization without API changes?

**Citations:** [F-221], [RFC-91]
```

**Machine-Readable JSON:**

```json
{
  "command": "/define",
  "feature_name": "One-Click Checkout",
  "citations": ["F-221", "RFC-91"],
  "confidence": 0.74,
  "open_questions": [
    "Which US PSPs can support tokenization without API changes?"
  ]
}
```

---

### Example 2 — `/compare`

**User Input:**

```
/compare "Social Login" vs "Passwordless Magic Link" #domain/auth #risk/security #region/eu
```

**Expected Highlights in Output:**

* Table format with **target user**, **flow changes**, **tech dependencies**, **security implications**, **effort estimate**, **prior results**.
* Explicit security trade-off callouts (due to `#risk/security`).
* EU compliance considerations (due to `#region/eu`).

---

### Example 3 — `/find`

**User Input:**

```
/find "usage-based pricing" #domain/billing #phase/discovery
```

**Expected Highlights in Output:**

* Ranked list of historical pricing model experiments.
* Show decision outcomes and reasons for rejection or adoption.
* Include links for each artifact.

---

### Example 4 — `/stories`

**User Input:**

```
/stories "In-App Upsell Banner" #domain/engagement #goal/activation
```

**Expected Highlights in Output:**

* 6–8 user stories across personas (PM, designer, marketer).
* Acceptance criteria for each story.
* Tracking event suggestions (marked as “proposed”).

---

### QA Acceptance Criteria for All Commands

* **Must** include at least 2 citations when retrieval finds matches.
* **Must** list open questions if context is incomplete.
* **Must** respect tag filters in retrieval results.
* **Must** output human-readable markdown before machine JSON.
* **Must** flag low-confidence results with a warning banner.

---

### Notes for Prompt Engineers

* Treat **slash commands** as the primary signal for output type and formatting.
* Treat **hashtags** as hard filters first, then soft semantic boosts.
* Always ask **1 clarifying question max** if tags conflict or no retrieval match.
* Ensure retrieval chunk size is optimized for:

  * PRDs: 1–2k tokens
  * Tickets: 400–800 tokens
  * Decision logs: atomic

