# User Query Complexity Assessment Framework

When a support bot gives a poor response, there are two very different explanations:

1. The model failed on a tractable query — a real quality issue
2. The query was genuinely difficult — multi-intent, ambiguous, legally sensitive, or partially out of scope

Without input-level scoring, these look identical in your evaluation data. You end up penalising good models for hard inputs, or missing real failures hidden behind easy ones. Routing decisions also become arbitrary — based on topic category rather than actual difficulty.

This framework gives you a principled way to score queries before evaluation, so complexity becomes a first-class variable in your analysis.

---

## What's in this repo

| File | Description |
| :--- | :--- |
| `user_query_quality_framework.md` | The full framework — parameters, scoring, routing signals, annotation guidance |
| `context_note_dataset_decisions.md` | Why the examples are manually constructed rather than drawn from public datasets |

**Coming eventually:**
- Python implementation — LLM-based scorer using these rules
- Annotated examples — calibration set for inter-annotator agreement
- Classifier — trained on operational data if it becomes available

---

## How the framework works

Queries are scored on four parameters:

- **Multi-Intent** — does the query contain more than one distinct request?
- **Relevance** — how well does it map to the product or service in scope?
- **Policy Constraints** — does it contain content that requires careful handling (PII, abuse, legal risk, security)?
- **Intent Clarity** — how clearly does the user express what they want?

These combine into a **Complexity Score** (0 to High), plus a set of **routing flags** for cases that require a different handler regardless of complexity — legal escalation, security events, prohibited content.

The framework operates in two layers: **Layer 1** produces the score and flags (deployment-agnostic); **Layer 2** translates those into routing decisions (deployment-specific, advisory only).

---

## What it's not

- Not a model evaluation framework — it scores inputs, not outputs
- Not a topic classifier — complexity cuts across topics
- Not validated on real operational data — public datasets don't contain the edge cases this framework is designed to handle (see `context_note_dataset_decisions.md`)
- Not a black box — every scoring decision is explicit and annotator-interpretable

---

## Who it's for

Teams building or evaluating customer support AI at the point where input quality starts mattering as much as output quality.

---

## Status

v2.4 — framework complete and documented. Python implementation and classifier are planned.

Feedback welcome — open an issue or reach out directly.

---

*Kseniia Briling | 2026*
