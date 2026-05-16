# Context Note: Dataset Decisions

## Why illustrative examples instead of an annotated dataset

Validation on real-world data was attempted but not completed due to structural limitations of available public datasets.

**Bitext Customer Support Dataset** (Hugging Face, 27k rows) is synthetically generated and cleaned for intent classification. As a result:
- Every query maps cleanly to a single intent and a single in-scope topic
- Multi-intent queries are absent by design
- PII and sensitive content are absent or anonymised
- Relevance = 1 or 2 does not occur — all queries are fully in-scope for the target domain

**Twitter Customer Support Dataset** (Kaggle, ~2.8M rows) is real but sourced from a public channel. As a result:
- PII is absent — users do not share sensitive data publicly
- LEGAL and SECURITY cases are rare
- After filtering for primary inbound Apple Support queries, the sample was dominated by a single event (iOS 11 update complaints), limiting diversity

**General finding:** Public customer support datasets are either synthetic and cleaned for classification tasks (which removes the edge cases this framework is designed to handle), or sourced from public channels (which removes sensitive content by nature). Meaningful coverage of low-frequency but high-stakes parameters — Multi-Intent, PII, LEGAL, SECURITY — requires access to real operational data, which was not available.

The illustrative examples in the framework are therefore constructed manually to demonstrate parameter combinations. This is noted explicitly in the document.

---

## Why Multi-Intent is in the framework despite being rare in public datasets

The absence of multi-intent queries in public datasets is a property of those datasets, not of real customer support traffic.

Multi-intent handling is a documented operational problem in production systems. Zendesk released a specific update to their AI Agents product (late 2025) to address it — prior versions either failed to match multi-intent text to any intent and returned a default response, or answered only the first intent and ignored the rest. This is consistent with industry writing on the topic (ML6, InternalNote) which treats multi-intent as a known failure mode requiring explicit architectural handling.

The rarity of multi-intent in classification-oriented datasets reflects a deliberate cleaning decision by dataset creators, not the distribution of real user queries. Users writing to support under frustration or time pressure naturally combine requests.

The framework treats Multi-Intent as a first-class parameter on these grounds: it is a real phenomenon, it is specifically harder for models to handle than single-intent queries of equivalent complexity, and its absence from training data makes it more likely to cause failures in production — not less important to account for.
