# BizMatch

**Business Entity Resolution across noisy, multi-source data — built for the Amazon ML Challenge 2026.**

> ⚠️ **Work in progress.** This project is not yet completed and is being developed actively. 

---

## Overview

Large commercial platforms often receive information about the same business from multiple independent sources — a name here, a partial address there, none of them sharing a common ID. **Entity Resolution** is the task of figuring out, from these noisy, partial fragments, which records actually refer to the same real-world business.

This project tackles that problem for the Amazon ML Challenge 2026's **Business Entity Resolution** track: given business records from three independent, unlabeled data sources, determine — for every entity in a deduplicated reference source — which records (if any) from the other two sources refer to the same business.

## Problem Statement

- **Source 1** is a deduplicated reference source. For every Source 1 entity, the task is to find all matching records in **Source 2** and **Source 3**.
- A Source 1 entity may match **zero, one, or many** records across the other two sources.
- Each record has only four fields: `entity_id`, `business_name`, `business_address`, and `country` — no shared identifiers, and no structured metadata to lean on.
- The data is intentionally noisy: abbreviations, legal-suffix inconsistencies, transliteration variants, typos, missing address components, landmark-based addresses, and more.
- The **country** field is an open set: training data covers `US` and `India`; the test set additionally includes `France`, unseen during training.
- **No external data lookups are permitted** — no geocoding APIs, no business registries, no commercial entity-resolution services. The entire signal has to come from the noisy text itself.

Full specification: [`docs/problem_description.md`](docs/problem_description.md).

## Evaluation

Submissions are scored with a **macro-averaged F-beta(0.5) score**, computed per Source 1 entity and averaged across all entities:

```
F_0.5 = (1.25 × Precision × Recall) / (0.25 × Precision + Recall)
```

This metric is deliberately **precision-heavy** — a false merge (linking two different businesses) is penalized twice as heavily as a missed match. Entities with no true matches ("singletons") score a full 1.0 for a correctly predicted empty match list, and 0.0 for any false match — so correctly recognizing "no match" carries real weight in the final score, not just correctly finding matches that exist.

## Deliverables

Per the challenge rules, the pipeline produces two output files:

- **`matching_results.tsv`** — final predicted matches (the file scored on the public/private leaderboard)
- **`candidate_pairs.tsv`** — the candidate set produced by the blocking stage, before the matching model narrows it down (used to evaluate blocking recall and reduction ratio)

along with a full submission package containing runnable code, a methodology write-up (based on the provided [`docs/solution_template.md`](docs/solution_template.md)), and pinned dependencies.

## Constraints

- Final model must be licensed **MIT or Apache 2.0**, and no larger than **8B parameters**.
- All approaches, methodologies, and code are subject to review for academic-integrity and fair-play compliance.

## Project Status & Roadmap

This project is being built in clearly defined phases — preprocessing, candidate generation (blocking), feature engineering, matching model, inference, and validation/packaging — with each phase broken into independently testable stages.

The living roadmap, including which stages are complete and what's next, is tracked in [`docs/implementation_plan.md`](docs/implementation_plan.md).

## References

Files below were provided as part of the challenge:

- [`docs/problem_description.md`](docs/problem_description.md) — full problem specification
- [`docs/solution_template.md`](docs/solution_template.md) — required methodology write-up template
- [`utils/validate_submission.py`](utils/validate_submission.py) — official local submission-format validator

## License

This project is open-source and available under the **MIT License**.

Note: this governs the *code license* only. The trained model itself must independently satisfy the challenge's MIT/Apache-2.0, ≤8B-parameter constraint described above.
