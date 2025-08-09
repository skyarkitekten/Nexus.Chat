# Developer Handoff Notes – Azure DevOps LLM Story Builder (MVP with RAG)

## 1. Core Architecture Overview

**Front-End**: Azure DevOps extension UI (TypeScript + React preferred) embedded in Boards work item view.
**Back-End**: Node.js service (Azure Functions or App Service) handling LLM + RAG logic.
**Data Layer**: Azure Cognitive Search (vector search) for backlog embeddings.

---

## 2. Key Components

### A. UI Elements

1. **Generate User Stories Button**

   * Location: Feature work item toolbar.
   * Action: Opens side panel.

2. **Side Panel Layout**

   * **Config Panel**:

     * System Prompt: multi-line text input (editable).
     * Top P: slider (0–1, default 0.7).
     * Acceptance Criteria Template: dropdown (load from saved configs).
   * **Retrieved Backlog Preview**:

     * Show top 3 related work items (title + link + short excerpt).
   * **Generate Button**:

     * Calls backend with feature description + configs.

3. **Output Review**

   * Side-by-side: Generated output vs editable text area.
   * Fields:

     * Story Title
     * Story Description
     * Acceptance Criteria (formatted per template)
     * Test Case Outline (bulleted)
   * Save All → Creates linked backlog items.

---

### B. RAG Pipeline

**Step 1: Backlog Embedding**

* **Data Sources**:

  * Title, Description, Acceptance Criteria from all historical work items (Story, Bug, Task).
* **Frequency**: Nightly job to re-index (Azure DevOps REST API → Azure Cognitive Search).
* **Embedding Model**: `text-embedding-3-large` (OpenAI) or Azure OpenAI equivalent.
* **Storage**: Vector index in Azure Cognitive Search with metadata (Work Item ID, URL, Type).

**Step 2: Retrieval**

* Query vector index with semantic similarity on feature text.
* Return top 5 relevant items (configurable).

**Step 3: LLM Context Injection**

* Construct system prompt:

  ```
  You are an AI assistant helping a Business Analyst write user stories from a feature description. 
  Use the retrieved backlog items below to maintain consistency in tone, structure, and terminology. 
  Do NOT duplicate existing stories.

  Retrieved Backlog Context:
  [LIST OF RELATED BACKLOG ITEMS WITH ACCEPTANCE CRITERIA]
  ```

* Inject user-defined acceptance criteria template into prompt.
* Include top P setting in API request.

---

### C. LLM API Calls

* **Model**: `gpt-4o-mini` for MVP (cost-effective, fast), keep config for upgrading to `gpt-4o`.
* **Request Structure**:

  * `system` message: Constructed from system prompt + backlog context + acceptance criteria template.
  * `user` message: Feature description.
* **Post-processing**:

  * Deduplication: Compare new stories to backlog embeddings (cosine similarity > threshold, e.g., 0.88 → flagged as duplicate).
  * Formatting: Return structured JSON:

    ```json
    {
      "title": "...",
      "description": "...",
      "acceptance_criteria": ["...", "..."],
      "test_case_outline": ["...", "..."]
    }
    ```

---

### D. Azure DevOps Integration

* **Authentication**: Use OAuth via Azure DevOps Extension SDK to access work items on behalf of user.
* **Write Operations**:

  * Create new backlog items (`POST /_apis/wit/workitems/$Product Backlog Item`) linked to originating feature (`relations` property).
  * Add Acceptance Criteria in `Microsoft.VSTS.Common.AcceptanceCriteria` field.
  * Create related Test Case work item if test case outline exists.
* **Error Handling**:

  * If LLM fails: show inline retry option.
  * If duplicate detected: show alert with link to existing item.

---

### E. Config Management

* **Storage**:

  * Team-level configs: in Azure DevOps extension data storage (scoped to project).
  * User-level overrides: browser localStorage or per-user extension storage.
* **Defaults**:

  * System prompt: provided in repo.
  * Top P: 0.7
  * Acceptance criteria template: standard Gherkin (“Given / When / Then”).

---

## 3. Security & Compliance

* **Data Minimization**: Strip PII before sending backlog text to embeddings or LLM.
* **Data Residency**: Use Azure OpenAI + Azure Cognitive Search in same region as customer’s DevOps instance.
* **Logging**: No storing of raw backlog content in logs; store only request IDs and timing metrics.

---

## 4. Testing Strategy

* **Unit Tests**: Prompt construction, deduplication logic, API calls.
* **Integration Tests**: End-to-end generation from feature → saved backlog items.
* **Regression Tests**: Validate style and tone consistency with reference outputs.
* **User Acceptance**: Pilot with 5–10 BAs, track story quality and deduplication accuracy.

---

## 5. Deployment & Release

* **Hosting**: Azure Functions for backend API, static hosting for extension front-end.
* **CI/CD**: GitHub Actions or Azure DevOps Pipelines to package and deploy.
* **Marketplace Release**: Publish as Private extension for pilot; later switch to Public listing.

---

## 6. Open Questions

1. Max token budget for LLM requests — what’s acceptable for cost/performance balance?
2. Deduplication threshold tuning — should BA be able to override if AI flags as duplicate?
3. How much of the retrieved backlog to show in the UI for transparency?

---

Heck yes — here’s a **copy‑pasteable, concrete example** your engineers can use to wire up the API before the UI exists. I’m giving you:

* A **RAG-enabled system prompt template** (with variables).
* A **filled example** (feature + retrieved backlog context).
* The **actual Chat Completions payload** we’ll send.
* A **sample LLM response JSON** your backend should normalize and return to the extension.

---

# RAG-Enabled System Prompt (Template)

```text
You are an assistant that helps Business Analysts create user stories from a provided Feature description.

REQUIREMENTS
- Use the "Retrieved Backlog Context" to align terminology, structure, and scope.
- Prevent duplicates. If a proposed story is substantially similar to any retrieved item, flag it as a potential duplicate and propose a refined alternative or skip.
- Follow the Acceptance Criteria Template exactly.
- Produce test case outlines that a QA engineer can expand into test cases.
- Output strictly in the provided JSON schema. No extra text.

STYLE & CONSTRAINTS
- Story titles: concise, action-oriented.
- Descriptions: explain user value + key edge considerations.
- Keep each story independent and implementation-ready.
- If the feature is too broad, prioritize 3–6 stories.
- If context conflicts, prefer the most recent retrieved item.

ACCEPTANCE CRITERIA TEMPLATE
{ACCEPTANCE_CRITERIA_TEMPLATE}

DUPLICATION RULE
- Consider cosine similarity >= {DUPLICATE_SIMILARITY_THRESHOLD} as potential duplicate.
- If duplicate detected, set "status" to "potential_duplicate" and include "similar_items" with IDs and similarity scores.
- If not duplicate, set "status" to "new".

RETRIEVED BACKLOG CONTEXT
{RETRIEVED_BACKLOG_CONTEXT}

OUTPUT JSON SCHEMA
{
  "stories": [
    {
      "title": "string",
      "description": "string",
      "acceptance_criteria": ["string", "..."],
      "test_case_outline": ["string", "..."],
      "references": {
        "source_feature_id": "{FEATURE_ID}",
        "used_context_item_ids": ["int", "..."]
      },
      "duplicate_check": {
        "status": "new" | "potential_duplicate",
        "similar_items": [
          {"work_item_id": "int", "similarity": "number (0..1)"}
        ]
      }
    }
  ]
}

DO NOT include any text outside the JSON.
```

---

# Filled Example Inputs (for a realistic test)

## Feature (from Azure DevOps)

```json
{
  "feature_id": 12345,
  "title": "Passwordless login for web users",
  "description": "Enable users to sign in without a password using magic links sent to their email. Support rate-limiting, token expiration, and device trust for returning users. Must align with our existing auth domain. Exclude mobile app for MVP."
}
```

## Retrieved Backlog Context (Top 4 similar items)

*(These are the items RAG fetched; include short excerpts only — keep tokens tight.)*

```json
[
  {
    "id": 9811,
    "title": "Email verification for new accounts",
    "summary": "Send verification link with 15-min expiry; lockout after 5 attempts.",
    "acceptance_criteria": [
      "Given unverified email, when user clicks link within 15 min, then account is verified",
      "Given 5+ invalid attempts, then show lockout message for 30 min"
    ],
    "updated_at": "2025-05-02"
  },
  {
    "id": 10477,
    "title": "Rate limiting for authentication endpoints",
    "summary": "Global + per-IP rate limits; reuse shared throttling middleware.",
    "acceptance_criteria": [
      "Given >10 requests/min/IP, then respond 429",
      "Given global burst >1k rpm, then shed load with 429"
    ],
    "updated_at": "2025-06-10"
  },
  {
    "id": 11203,
    "title": "Device trust cookie",
    "summary": "Set signed cookie to remember trusted browser for 30 days.",
    "acceptance_criteria": [
      "Given user opts to trust device, then set signed cookie",
      "Given cookie present and valid, then skip step-up"
    ],
    "updated_at": "2025-04-22"
  },
  {
    "id": 11590,
    "title": "Magic link login (discovery spike)",
    "summary": "Evaluated email-based magic link auth; recommended 10-min token, single-use.",
    "acceptance_criteria": [
      "Token single-use; invalidate after first consumption",
      "Link expired after 10 minutes"
    ],
    "updated_at": "2025-03-14"
  }
]
```

## Acceptance Criteria Template (Team standard; Gherkin)

```text
Given <precondition>
When <action>
Then <expected outcome>
```

## Generation Config (from your extension)

```json
{
  "model": "gpt-4o-mini",
  "top_p": 0.7,
  "temperature": 0.2,
  "max_output_tokens": 1600,
  "duplicate_similarity_threshold": 0.88
}
```

---

# Chat Completions Request (what your backend sends to the LLM)

```json
{
  "model": "gpt-4o-mini",
  "messages": [
    {
      "role": "system",
      "content": "You are an assistant that helps Business Analysts create user stories from a provided Feature description.\n\nREQUIREMENTS\n- Use the \"Retrieved Backlog Context\" to align terminology, structure, and scope.\n- Prevent duplicates. If a proposed story is substantially similar to any retrieved item, flag it as a potential duplicate and propose a refined alternative or skip.\n- Follow the Acceptance Criteria Template exactly.\n- Produce test case outlines that a QA engineer can expand into test cases.\n- Output strictly in the provided JSON schema. No extra text.\n\nSTYLE & CONSTRAINTS\n- Story titles: concise, action-oriented.\n- Descriptions: explain user value + key edge considerations.\n- Keep each story independent and implementation-ready.\n- If the feature is too broad, prioritize 3–6 stories.\n- If context conflicts, prefer the most recent retrieved item.\n\nACCEPTANCE CRITERIA TEMPLATE\nGiven <precondition>\nWhen <action>\nThen <expected outcome>\n\nDUPLICATION RULE\n- Consider cosine similarity >= 0.88 as potential duplicate.\n- If duplicate detected, set \"status\" to \"potential_duplicate\" and include \"similar_items\" with IDs and similarity scores.\n- If not duplicate, set \"status\" to \"new\".\n\nRETRIEVED BACKLOG CONTEXT\n1) [9811] Email verification for new accounts — expiry 15 min; lockout after 5 attempts. AC: verification within 15 min; lockout 30 min after 5 invalid attempts.\n2) [10477] Rate limiting for authentication endpoints — global + per-IP, shared throttling middleware.\n3) [11203] Device trust cookie — remember browser for 30 days; skip step-up when valid.\n4) [11590] Magic link login (discovery spike) — 10-min, single-use token recommended.\n\nOUTPUT JSON SCHEMA\n{\n  \"stories\": [\n    {\n      \"title\": \"string\",\n      \"description\": \"string\",\n      \"acceptance_criteria\": [\"string\", \"...\"],\n      \"test_case_outline\": [\"string\", \"...\"],\n      \"references\": {\n        \"source_feature_id\": \"{FEATURE_ID}\",\n        \"used_context_item_ids\": [\"int\", \"...\"]\n      },\n      \"duplicate_check\": {\n        \"status\": \"new\" | \"potential_duplicate\",\n        \"similar_items\": [\n          {\"work_item_id\": \"int\", \"similarity\": \"number (0..1)\"}\n        ]\n      }\n    }\n  ]\n}\n\nDO NOT include any text outside the JSON."
    },
    {
      "role": "user",
      "content": "FEATURE\nID: 12345\nTitle: Passwordless login for web users\nDescription: Enable users to sign in without a password using magic links sent to their email. Support rate-limiting, token expiration, and device trust for returning users. Must align with our existing auth domain. Exclude mobile app for MVP."
    }
  ],
  "top_p": 0.7,
  "temperature": 0.2,
  "max_tokens": 1600
}
```

---

# Sample LLM Response (what you should expect and return to the extension)

```json
{
  "stories": [
    {
      "title": "Send Magic Link for Web Sign-In",
      "description": "As a web user, I can request a single-use magic link to sign in without a password so I can access my account securely without credential friction. Include rate-limit safeguards and align with existing auth domain and email service.",
      "acceptance_criteria": [
        "Given an existing account email on the web login page\nWhen I request a magic link\nThen a single-use sign-in link is emailed to me"
      ],
      "test_case_outline": [
        "Request magic link with valid email → email sent, contains single-use token",
        "Request magic link >10 times/min from same IP → receive 429 per rate limits",
        "Request magic link for non-existent email → generic success response, no email sent"
      ],
      "references": {
        "source_feature_id": "12345",
        "used_context_item_ids": ["10477", "11590"]
      },
      "duplicate_check": {
        "status": "new",
        "similar_items": [
          {"work_item_id": 11590, "similarity": 0.61}
        ]
      }
    },
    {
      "title": "Consume Magic Link to Authenticate (Single-Use, Expiring Token)",
      "description": "As a web user, I can sign in by clicking a magic link that is single-use and expires after 10 minutes to prevent replay attacks and stale tokens.",
      "acceptance_criteria": [
        "Given a valid, unconsumed magic link within 10 minutes of issuance\nWhen I click the link\nThen I am authenticated into my account in the browser",
        "Given a magic link that is expired or already used\nWhen I click the link\nThen I see an error and I am prompted to request a new link"
      ],
      "test_case_outline": [
        "Click valid link <10 min since issuance → user authenticated",
        "Click link after 10 min → error + prompt to request new link",
        "Use same link twice → second attempt fails; token invalidated"
      ],
      "references": {
        "source_feature_id": "12345",
        "used_context_item_ids": ["11590"]
      },
      "duplicate_check": {
        "status": "new",
        "similar_items": [
          {"work_item_id": 11590, "similarity": 0.79}
        ]
      }
    },
    {
      "title": "Rate Limiting for Magic Link Requests (Reuse Auth Throttling)",
      "description": "As a platform, I enforce per-IP and global rate limits on magic link requests to mitigate abuse, reusing the shared throttling middleware for authentication endpoints.",
      "acceptance_criteria": [
        "Given >10 magic link requests per minute from the same IP\nWhen subsequent requests are made\nThen the service responds with HTTP 429 and does not send emails",
        "Given a global burst exceeding 1000 requests per minute\nWhen sustained for >30 seconds\nThen the service sheds load by returning HTTP 429"
      ],
      "test_case_outline": [
        "Simulate 11 requests/min/IP → last request returns 429, no email",
        "Load test to exceed global threshold → 429 responses observed"
      ],
      "references": {
        "source_feature_id": "12345",
        "used_context_item_ids": ["10477"]
      },
      "duplicate_check": {
        "status": "new",
        "similar_items": [
          {"work_item_id": 10477, "similarity": 0.72}
        ]
      }
    },
    {
      "title": "Remember Trusted Browser After Magic Link Sign-In",
      "description": "As a returning web user, I can mark my current browser as trusted so subsequent sign-ins reduce friction while maintaining security via a signed, 30-day device trust cookie.",
      "acceptance_criteria": [
        "Given I successfully sign in via magic link and choose to trust this browser\nWhen trust is confirmed\nThen a signed device trust cookie is set for 30 days",
        "Given a valid device trust cookie is present\nWhen I request a new magic link and sign in\nThen I bypass additional verification steps"
      ],
      "test_case_outline": [
        "Sign in via magic link → opt-in trust → cookie present with correct signature/expiry",
        "Return within 30 days with cookie → streamlined sign-in path",
        "Cookie expired/invalid → full verification required"
      ],
      "references": {
        "source_feature_id": "12345",
        "used_context_item_ids": ["11203"]
      },
      "duplicate_check": {
        "status": "new",
        "similar_items": [
          {"work_item_id": 11203, "similarity": 0.77}
        ]
      }
    },
    {
      "title": "Email Address Ownership Check During Magic Link Request",
      "description": "As a platform, I prevent information leakage by returning a generic response whether or not the email exists, while logging metrics internally.",
      "acceptance_criteria": [
        "Given an email that does not match any account\nWhen I request a magic link\nThen the UI shows a generic success message and no email is sent"
      ],
      "test_case_outline": [
        "Request magic link with unknown email → generic success, no email delivery",
        "Request with known email → email delivered"
      ],
      "references": {
        "source_feature_id": "12345",
        "used_context_item_ids": ["9811"]
      },
      "duplicate_check": {
        "status": "new",
        "similar_items": [
          {"work_item_id": 9811, "similarity": 0.69}
        ]
      }
    }
  ]
}
```

---

# Notes for Engineers (tiny but important)

* **Truncation**: If retrieved context is long, include only titles + the 1–2 most relevant AC lines per item to stay within token limits.
* **IDs**: Return `used_context_item_ids` so we can display “informed by” references in the UI.
* **Duplicates**: Your service should also run an **independent cosine check** against the vector index for each generated story (belt-and-suspenders), and merge results into `duplicate_check`.
* **Strict JSON**: Reject responses that contain text outside JSON; auto-retry once with a “JSON ONLY” system nudge.
