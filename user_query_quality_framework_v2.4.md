# User Query Complexity Assessment Framework

### For Customer Support AI Systems

---

## Overview

Most AI evaluation frameworks assess **output quality** — how well the model responds. This framework addresses the upstream question: **how complex is the input query**, and in what way does that complexity affect the model's task.

Query complexity is not a single property. It emerges from a combination of factors that each make the bot's job harder in different ways. This framework identifies those factors systematically, enabling:

- Calibrated output evaluation and failure attribution — a poor response to a complex query ≠ a poor response to a simple one  
- Better routing logic (when to escalate, when to request clarification)  
- Training data stratification by complexity level

**Business impact:** Complexity scores make routing decisions defensible. Simple queries can be handled by lightweight models or self-service flows; complex queries can be flagged for human review or escalated to more capable models. Without input-level scoring, routing is either arbitrary or based solely on topic category — which does not capture difficulty.

**Scope note:** This framework evaluates properties of the **query itself**, not properties of the deployment (system access, integrations, SLA tiers). The same query receives the same scores regardless of which system processes it — with one deliberate exception: **Relevance** is deployment-relative by design, since scope is defined by the product, not the query. See parameter 2 for details. The framework also treats each query as a standalone unit; conversational dependency — where difficulty arises from prior turns, unresolved references, or ticket history — is not modelled.

---

## Framework Architecture

The framework operates in two layers.

**Layer 1 — Query Assessment** is the core of this document: it defines the parameters and produces a complexity score and a set of flags. It is deployment-agnostic.

**Layer 2 — Routing Signals** describes what those scores and flags typically mean for routing decisions. It is intentionally non-prescriptive: thresholds, escalation paths, and actions are deployment-specific and must be defined by each implementing team. Layer 2 guidance is advisory only.

---

## Processing Logic

Before scoring, a query is assessed on two pre-conditions. If either pre-condition applies, complexity scoring does not proceed and a flag is issued instead.

```
Query received
     │
     ▼
[Policy Constraints check]
     │
     ├─ PROHIBITED → Flag issued; complexity score = N/A
     │
     ▼
[Relevance check]
     │
     ├─ Not Relevant (1) → Flag issued; complexity score = N/A
     │
     ▼
Score remaining parameters → Calculate Complexity Score
```

**Note on processing order:** Policy Constraints is checked first for safety reasons — a prohibited query should not proceed through further assessment regardless of relevance. This processing order is independent of parameter numbering, which reflects conceptual grouping rather than sequence.

---

## Parameters

### 1\. Multi-Intent

*Does the query contain more than one distinct request?*

**Type:** Boolean flag

| Value | Description |
| :---- | :---- |
| `SINGLE` | One clear request or question |
| `MULTI` | Two or more distinct requests within one query |

**Effect on complexity:** MULTI increases coordination complexity, even when individual intents are simple. Each additional intent requires the model to track, address, and sequence multiple goals — multiplying the risk of partial or misaligned responses.

**Note on degree:** In practice, queries rarely contain more than two or three distinct intents. When a query contains three or more, annotators should note the count explicitly alongside the MULTI flag — this informs downstream analysis even if the flag itself remains binary.

**Examples:**

- `"How do I reset my password?"` → `SINGLE`  
- `"My payment failed and I want a refund — also how do I change my email address?"` → `MULTI`

---

### 2\. Relevance

*How well does the query map to the product or service in scope?*

| Score | Label | Description | Effect on complexity |
| :---- | :---- | :---- | :---- |
| 3 | Fully relevant | Query clearly concerns the product/service in scope | None |
| 2 | Partially relevant | Query mixes in-scope and out-of-scope content | **Increases** |
| 1 | Not relevant | Query is entirely outside the scope of this system | N/A (flag issued) |

**Note on deployment-relativity:** Unlike other parameters, Relevance cannot be assessed without knowing the deployment context. The same query may score 3 in one system and 1 in another. This is intentional: scope is a property of the product, not the query, and relevance cannot be meaningfully evaluated in a vacuum.

**Note on the symmetry:** A fully irrelevant query is straightforward — the correct response is deflection, and that decision requires no judgment. A fully relevant query is also straightforward — the model can proceed. Complexity arises specifically at score 2, where the model must decide how to handle the in-scope portion without overreaching on the out-of-scope portion. Mixed-scope queries create additional pressure on the model to over-answer: LLMs are prone to responding to out-of-scope portions instead of selectively refusing them, which increases the risk of confabulated or inappropriate output.

**Examples:**

- `"Why isn't my invoice downloading?"` (billing SaaS support) → `3`  
- `"Why isn't my invoice downloading, and also can you recommend an accountant?"` → `2`  
- `"What's the weather like in Budapest?"` (billing SaaS support) → `1`

---

### 3\. Policy Constraints

*Does the query contain content that constrains or prevents normal processing?*

Policy Constraints covers any content that requires the model to handle the query differently from a standard request — including sensitive data, privacy-relevant information, and harmful language.

| Score | Label | Description | Effect on complexity |
| :---- | :---- | :---- | :---- |
| 3 | Clean | No flags; query can be processed normally | None |
| 2 | Caution required | Query contains elements that require careful handling but do not prevent a response | **Increases** |
| 1 | Prohibited | Query cannot be processed in the standard pipeline | N/A (flag issued) |

**Score 2 — Caution required** covers several distinct situations. Annotators should apply one or more secondary tags to enable downstream analysis:

| Tag | Description | Risk type |
| :---- | :---- | :---- |
| `PII` | Identifying data present (name \+ account number \+ contact details in combination) or sensitive data types (payment card numbers, passwords, government IDs) | Handling risk |
| `NEGATIVE` | Expression of dissatisfaction, complaints about service quality, or statements of intended consequences (negative reviews, cancellation, escalation) — regardless of register | Handling risk |
| `ABUSE` | Hostile or threatening content that requires a de-escalation response — regardless of whether directed at a human agent, the system, or a third party | Handling risk |
| `LEGAL` | Content touching legal disputes, regulatory matters, or liability-sensitive topics | Routing risk |
| `SECURITY` | Account security concerns, suspected fraud, or unauthorised access | Routing risk |

**On the distinction between risk types:** `LEGAL` and `SECURITY` carry **routing risk**: the query requires a different handler regardless of other factors. `PII`, `NEGATIVE`, and `ABUSE` carry **handling risk**: the standard pipeline can proceed, but must adapt its approach.

**On the distinctions between `NEGATIVE`, `ABUSE`, and `PROHIBITED`:** `NEGATIVE` covers dissatisfaction directed at the service — complaints, cancellation threats, and statements of intended consequences. `ABUSE` covers hostile or threatening content that requires a de-escalation response. The distinction is based on content, not register: an angry complaint is `NEGATIVE`; a threatening message is `ABUSE`. If the stated consequence involves a regulatory body, ombudsman, or legal authority, apply `LEGAL` in addition to `NEGATIVE`; consequences directed at private platforms (reviews, social media) are `NEGATIVE` only. The boundary between `ABUSE` (score 2\) and `PROHIBITED` (score 1\) is whether a legitimate underlying service request is present. When uncertain, apply `ABUSE`.

**Score 1 — Prohibited** covers:

- Threats or harassment with no legitimate underlying service request  
- Requests for information the system is prohibited from providing  
- Content that triggers mandatory escalation regardless of query content

When score 1 applies, the flag `PROHIBITED` is issued and no complexity score is assigned. The appropriate response is deployment-specific.

---

### 4\. Intent Clarity

*How clearly does the query express what the user wants?*

| Score | Label | Description | Effect on complexity |
| :---- | :---- | :---- | :---- |
| 3 | Explicit | The user's goal is directly stated | None |
| 2 | Implicit | The goal can be confidently inferred from context | Moderate increase |
| 1 | Unclear | What the user wants cannot be determined reliably | High increase |

**Examples:**

- `"Cancel my subscription"` → `3 / Explicit`  
- `"I've been a customer for three years and this is really disappointing"` → `2 / Implicit` (complaint; likely wants acknowledgment and resolution, but no explicit ask)  
- `"It's not working"` → `1 / Unclear` (no product, no action, no error — no basis for inference)

---

## Complexity Score

The complexity score aggregates the parameters above into a single operational signal.

**Calculation:**

Start at baseline **0**. Add points for each complexity-increasing condition:

| Condition | Points |
| :---- | :---- |
| MULTI\_INTENT | \+2 |
| Relevance \= 2 (partial) | \+1 |
| Policy Constraints \= 2 (caution required) | \+1 |
| — `ABUSE` | \+1 (additional) |
| — `LEGAL` / `SECURITY` | \+0 (routing signal only, see Layer 2\) |
| Intent Clarity \= 2 (implicit) | \+1 |
| Intent Clarity \= 1 (unclear) | \+2 |
| Policy Constraints \= 1 (prohibited) | Score not assigned |
| Relevance \= 1 (not relevant) | Score not assigned |

**Complexity levels:**

| Total | Level | Interpretation |
| :---- | :---- | :---- |
| 0 | Simple | Straightforward query; standard processing |
| 1–2 | Moderate | One or two complicating factors; careful handling needed |
| 3–4 | Complex | Multiple complicating factors; elevated risk of poor response |
| 5+ | High complexity | Routing signal for human review; see Layer 2 guidance |

**Override rules:** Certain parameter combinations are disproportionately harder than their additive score suggests. The following combinations automatically trigger **High Complexity** regardless of total points:

| Combination | Rationale |
| :---- | :---- |
| MULTI\_INTENT \+ Intent Clarity \= 1 | Multiple goals with no clear basis for inference |
| MULTI\_INTENT \+ Policy Constraints \= 2 `ABUSE` | De-escalation required across multiple intents |

**Note on asymmetry between `ABUSE`, `NEGATIVE`, and `LEGAL`/`SECURITY`:** `NEGATIVE` receives no additional points beyond the base \+1 for PC \= 2\. `LEGAL` and `SECURITY` receive no additional points and have no override rules — they are routing signals handled entirely in Layer 2\. `ABUSE` is the only tag that acts through both mechanisms: an additional \+1 in the scoring table, and an override rule when combined with MULTI\_INTENT.

**Note on scoring approach:** The complexity score is intentionally heuristic and prioritises interpretability over predictive accuracy. It does not fully capture multiplicative interactions between parameters — override rules above address the most critical cases; edge cases should be noted qualitatively by annotators.

---

## Annotation Examples

| Query | Multi-Intent | Relevance | Policy Constraints | Intent Clarity | Complexity |
| :---- | :---- | :---- | :---- | :---- | :---- |
| "How do I reset my password?" | SINGLE | 3 | 3 | 3 / Explicit | **0** |
| "My payment failed and I want a refund — also how do I change my email?" | MULTI | 3 | 3 | 3 / Explicit | **2** |
| "Why isn't my invoice downloading, and can you recommend an accountant?" | MULTI | 2 | 3 | 3 / Explicit | **3** |
| "I've been a customer for three years and this is really disappointing" | SINGLE | 3 | 2 `NEGATIVE` | 2 / Implicit | **2** |
| "It's not working" | SINGLE | 3 | 3 | 1 / Unclear | **2** |
| "My card number is 4111 1111 1111 1111 and the charge is wrong" | SINGLE | 3 | 2 `PII` | 3 / Explicit | **1** |
| "I'll report you to my lawyer if this isn't fixed today" | SINGLE | 3 | 2 `LEGAL` `NEGATIVE` | 2 / Implicit | **2** (+ `LEGAL` routing signal, see Layer 2) |
| "What's the weather like in Budapest?" (billing SaaS) | SINGLE | 1 | 3 | 3 / Explicit | **N/A** |

---

## Edge Cases

Some queries do not fit cleanly into the scoring logic. Documented here as annotator guidance.

**"Hello" / "Hi there" / no content** Single intent, clean, no goal stated. Intent Clarity \= 1 (Unclear) by default, but this is a distinct failure mode — the query is not ambiguous, it is simply empty of content. Tag `EMPTY` and exclude from complexity scoring. These queries should be filtered separately in downstream analysis.

**Queries in an unexpected language** Score normally on all dimensions. Language mismatch is a deployment constraint, not a property of the query itself. Flag `LANG_MISMATCH` for routing purposes.

**Quoted or forwarded content ("My colleague sent me this error: \[paste\]")** The outer query may be clear while the embedded content is not. Score the outer intent; note the embedded complexity separately if it materially affects the response task.

**Sarcasm or irony ("Great, another outage, love this service")** Intent Clarity \= 2 (Implicit) — the likely goal (complaint, request for update) is inferable from context. Tag `TONE_IRONIC` if your deployment uses tone signals for response calibration.

---

## Layer 2 — Routing Signals

The following guidance describes how complexity scores and flags typically inform routing decisions. It is advisory: thresholds and escalation paths are deployment-specific and must be defined by each implementing team.

| Signal | Typical routing consideration |
| :---- | :---- |
| Complexity \= 0 | Standard processing; lightweight model or self-service flow may be sufficient |
| Complexity \= 1–2 | Standard pipeline with careful handling; review outputs where Policy Constraints flags are present |
| Complexity \= 3–4 | Consider routing to more capable model or flagging for post-response review |
| Complexity \= 5+ or High (override) | Consider human review before or after response; deployment thresholds apply |
| Flag: `PROHIBITED` | Block or escalate immediately; response pipeline should not proceed |
| Flag: Relevance \= 1 | Deflect; no complexity scoring needed |
| Tag: `NEGATIVE` | Consider prioritisation or retention-focused routing; does not require escalation out of standard pipeline |
| Tag: `LEGAL` | Routing risk — consider legal team notification or scripted response regardless of complexity score |
| Tag: `SECURITY` | Routing risk — consider security escalation path regardless of complexity score |

---

## Relationship to Output Evaluation

Query complexity scores are designed to be used **alongside** output evaluation, not instead of it. When a response fails quality review, complexity scores provide a diagnostic layer:

| Input complexity | Output quality | Likely cause |
| :---- | :---- | :---- |
| Low | Poor | Model failure |
| High | Poor | May be complexity-induced; review before penalising |
| High | Good | Model handled a hard case well — positive signal |
| Low | Good | Expected baseline performance |

This prevents systematic undervaluation of good responses to hard queries, and avoids attributing input problems to model deficiencies.

---

## Annotation Notes

**On the illustrative examples:** The annotation examples in this document are constructed manually to demonstrate parameter combinations. Validation on real-world data was attempted but not completed due to structural limitations of available public datasets. For the full rationale and dataset validation attempts, see `context_note_dataset_decisions.md`.

**Known limitations:**

- Policy Constraints boundary cases (score 2 vs. score 1\) carry the highest subjectivity load; annotator calibration examples are the primary alignment mechanism  
- Intent Clarity boundary between score 2 and score 1 is the most annotator-dependent dimension; calibration examples are essential  
- Complexity Score is additive by design; multiplicative interactions are partially addressed by override rules but not fully modelled  
- Relevance scoring requires deployment context and cannot be applied dataset-wide without first defining the target scope  
- The framework has been developed and validated in English only; cross-lingual applicability requires separate validation

---

*Kseniia Briling | 2026* *Feedback and contributions welcome.*  
