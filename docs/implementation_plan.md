# Implementation Plan — BizMatch

Living roadmap for the Business Entity Resolution pipeline (Amazon ML
Challenge 2026). This document breaks the problem into **phases**
(major pipeline components) and **stages** (concrete, independently
testable steps within each phase).

For the full problem specification, see [`problem_description.md`](./problem_description.md).

## How to use this document

- Work through stages **in order within a phase**, and phases roughly
  in order — later phases depend on outputs from earlier ones.
- Where a stage has multiple viable approaches, the table lists a
  **default** (the classical/obvious choice, used to get a working
  end-to-end pipeline first) and **alternatives** worth trying once the
  default is working, not before.
- Update the **Progress Tracker** at the bottom as stages are completed.
  It's the single source of truth for "where are we right now."

---

## Phase 1 — Foundations

Goal: a reproducible project skeleton and an honest, data-driven picture
of what we're working with, before any design decisions get locked in.

| Stage | Goal | Default approach | Alternatives / notes |
|---|---|---|---|
| 1. Repository & environment setup | Reproducible structure, pinned dependencies | `uv`-managed venv, `src/` + `dataset/` + `output/` + `docs/` layout | — |
| 2. Full-scale EDA | Quantify noise patterns, distributions, and referential integrity on the real data | Programmatic checks (`src/eda/`): structural/referential integrity, distributional profiling, regex-based noise quantification + empirical discovery | — |
| 3. Validation split design | A held-out split that gives an honest read on generalization | Split by **Source 1 entity** (never by row), stratified by country + match-count bucket | Time-based or source-based splits if temporal/source drift shows up later |

---

## Phase 2 — Preprocessing / Normalization

Goal: one clean, consistent representation of every record that every
later phase can trust. Every rule here should trace back to a number
from Phase 1's EDA, not a hunch.

| Stage | Goal | Default approach | Alternatives / notes |
|---|---|---|---|
| 1. Generic text cleanup | Strip junk symbols, casing, punctuation, diacritics | Regex-based cleanup | Unicode normalization libraries (`ftfy`) if patterns get messier at scale |
| 2. Name normalization | Canonical legal-suffix form + suffix-stripped "core name" | Suffix dictionary + regex substitution, built empirically from EDA's trailing-token discovery | Learned/clustering-based suffix detection if the dictionary approach plateaus |
| 3. Address normalization | Abbreviation mapping, PIN/street-number extraction, landmark flag | Regex-based parsing | A proper address parser (`usaddress`, `libpostal`) for stronger component extraction — heavier to integrate, worth it if regex plateaus |
| 4. Non-Latin script flagging | Explicit flag when name/address is non-Latin — string similarity is meaningless across scripts | Boolean flag only, no transliteration | Algorithmic transliteration (`indic-transliteration`, `aksharamukha`) as an *additional* feature later — legitimate under fair-play rules (deterministic script conversion, not a data lookup), but re-check the rules before shipping it |
| 5. Schema finalization & persistence | Freeze the normalized schema every later stage depends on | Write normalized versions of all source files once, reuse everywhere | — |

---

## Phase 3 — Candidate Generation (Blocking)

Goal: shrink millions-by-millions comparisons down to a manageable,
high-recall candidate set. **This phase sets the recall ceiling for
everything downstream** — no amount of modeling later fixes a true match
that never made it into the candidate set.

| Stage | Goal | Default approach | Alternatives / notes |
|---|---|---|---|
| 1. Exact-key blocking | Cheap, high-precision blocks | PIN/ZIP exact match (hash/dict grouping) | — |
| 2. Phonetic blocking | Catch spelling/transliteration variants in Latin-script names | Soundex on first name token | Metaphone / Double Metaphone / NYSIIS — worth an A/B recall comparison later |
| 3. Prefix / canopy blocking | Catch near-identical name prefixes cheaply | Sorted-neighborhood style: group by `(country, first-k-chars-of-core-name)` | Multiple prefix lengths unioned; sliding-window canopy clustering |
| 4. Address-based blocking | Fallback for non-Latin-name records (address stays Latin even when name doesn't) | Address-prefix key, same dict-grouping technique | — |
| 5. Approximate-similarity blocking | Catch fuzzy/typo'd name matches without pairwise search | MinHash LSH (banding) on character n-grams | FAISS approximate nearest-neighbor on TF-IDF or embedding vectors — larger research surface, see note below |
| 6. Union + recall measurement | Combine strategies, measure the actual recall ceiling | Union of stages 1–5; recall measured against ground truth | — |

**All blocking strategies must be O(n) or O(n log n)** — never a global
pairwise search — given the real dataset's scale (~2M records/source).
This is the phase with the largest research literature if you want to go
beyond the classical techniques above; useful search terms: *"blocking
techniques entity resolution survey"*, *"sorted neighborhood method"*,
*"MinHash LSH record linkage"*. The toolkits **Dedupe.io** and
**py_entitymatching (Magellan)** document many more strategies with
trade-offs.

---

## Phase 4 — Feature Engineering

Goal: turn each candidate pair into a fixed-width vector the model can
learn a decision boundary over.

| Stage | Goal | Default approach | Alternatives / notes |
|---|---|---|---|
| 1. Name similarity features | Quantify name closeness, robust to different noise types | Exact match, token Jaccard, Levenshtein ratio, token-sort ratio, phonetic match, char n-gram Jaccard | TF-IDF cosine similarity; embedding cosine similarity (Phase 5 alt.) |
| 2. Address similarity features | Quantify address closeness | Token Jaccard, Levenshtein ratio, PIN match, street-number match, landmark flag | Parsed-component-wise comparison if a proper address parser is adopted in Phase 2 |
| 3. Structural / cross-field features | Signals not captured by string similarity alone | Same-country flag, source-pair indicator, non-Latin flag, trailing-ID match, domain-root match, blank-address flags | — |
| 4. Feature validation | Catch bugs before they reach the model | Distribution checks per feature, missingness rates, correlation with label on labeled train candidates | SHAP analysis once a model exists (Phase 5) |
| 5. Scale / performance | Keep feature computation from bottlenecking the pipeline | Vectorized/batched computation (precompute shingle sets per unique string, batch string-distance calls, bounded-memory batching) | Multiprocessing across chunks if single-threaded throughput isn't enough |

---

## Phase 5 — Matching Model

Goal: classify each candidate pair as match / no-match, tuned
specifically for the challenge's **macro F-beta(0.5)** metric.

| Stage | Goal | Default approach | Alternatives / notes |
|---|---|---|---|
| 1. Train/val split | Avoid leakage across candidate pairs of the same entity | Split by S1 entity (reuse Phase 1's split) | — |
| 2. Baseline classifier | A strong, fast, interpretable tabular model | Gradient-boosted trees (LightGBM) on Phase 4 features | XGBoost/CatBoost — similar performance class; embedding-based Siamese/bi-encoder models — bigger effort, worth trying after the classical baseline works |
| 3. Threshold selection | Optimize for macro F-beta(0.5), not accuracy or a default 0.5 cutoff | Threshold sweep on validation set, maximizing macro F-beta(0.5) directly | Per-entity top-k logic instead of / alongside a flat threshold, if match counts are usually small |
| 4. Error analysis | Understand what's driving mistakes | Manual inspection of false positives/negatives by country, feature importances | SHAP values for per-prediction explanation |

The natural next experiment after a working classical baseline is a
**hybrid**: add one embedding-similarity feature to the same tree model,
rather than replacing it outright.

---

## Phase 6 — Inference Pipeline (Test-Time)

Goal: run the exact same logic on test data and produce the two required
output files.

| Stage | Goal | Approach |
|---|---|---|
| 1. Apply Phases 2–4 to test data | Same normalization/blocking/features, no ground truth available | Reuse the same functions — no forked train/test code paths |
| 2. Score candidates | Apply the trained model | `predict_proba` |
| 3. Apply decision logic | Threshold (and/or top-k) from Phase 5 | Reuse the tuned threshold |
| 4. Assemble outputs | Exact format compliance | Produce `matching_results.tsv` and `candidate_pairs.tsv` — one row per S1 test entity, empty string for singletons |

---

## Phase 7 — Validation & Submission Packaging

Goal: never lose a leaderboard submission to a format rejection.

| Stage | Goal | Approach |
|---|---|---|
| 1. Local format validation | Catch every rule from the spec before uploading | Run Amazon's own [`validate_submission.py`](../utils/validate_submission.py) locally |
| 2. Coverage/consistency checks | Independent sanity checks beyond the validator | Every test S1 entity present, no duplicate rows, no duplicate/invalid IDs, matches ⊆ candidates |
| 3. Package the submission zip | Match the required structure exactly | `output/`, `code/business_entity_resolution/{src,README,requirements.txt}`, filled methodology doc |
| 4. Methodology write-up | Required deliverable | Fill [`solution_template.md`](./solution_template.md) as each phase finishes — easier than reconstructing it at the end |

---

## Phase 8 — Iteration

Ongoing, after Phase 6 produces a working end-to-end run with a real
validation F-beta(0.5) score. Iteration is driven by **error analysis
(Phase 5, Stage 4), not guesswork** — go back to whichever earlier phase
is the actual bottleneck. Priority order, since each phase caps what's
achievable in the next:

1. Blocking recall (Phase 3) first
2. Feature quality (Phase 4) next
3. Model / threshold (Phase 5) last

---

## Progress Tracker

- [ ] **Phase 1 — Foundations**
  - [ ] Stage 1: Repository & environment setup
  - [ ] Stage 2: Full-scale EDA
  - [ ] Stage 3: Validation split design
- [ ] **Phase 2 — Preprocessing / Normalization**
  - [ ] Stage 1: Generic text cleanup
  - [ ] Stage 2: Name normalization
  - [ ] Stage 3: Address normalization
  - [ ] Stage 4: Non-Latin script flagging
  - [ ] Stage 5: Schema finalization & persistence
- [ ] **Phase 3 — Candidate Generation (Blocking)**
  - [ ] Stage 1: Exact-key blocking
  - [ ] Stage 2: Phonetic blocking
  - [ ] Stage 3: Prefix / canopy blocking
  - [ ] Stage 4: Address-based blocking
  - [ ] Stage 5: Approximate-similarity blocking (MinHash LSH)
  - [ ] Stage 6: Union + recall measurement
- [ ] **Phase 4 — Feature Engineering**
  - [ ] Stage 1: Name similarity features
  - [ ] Stage 2: Address similarity features
  - [ ] Stage 3: Structural / cross-field features
  - [ ] Stage 4: Feature validation
  - [ ] Stage 5: Scale / performance
- [ ] **Phase 5 — Matching Model**
  - [ ] Stage 1: Train/val split
  - [ ] Stage 2: Baseline classifier
  - [ ] Stage 3: Threshold selection
  - [ ] Stage 4: Error analysis
- [ ] **Phase 6 — Inference Pipeline (Test-Time)**
  - [ ] Stage 1: Apply Phases 2–4 to test data
  - [ ] Stage 2: Score candidates
  - [ ] Stage 3: Apply decision logic
  - [ ] Stage 4: Assemble outputs
- [ ] **Phase 7 — Validation & Submission Packaging**
  - [ ] Stage 1: Local format validation
  - [ ] Stage 2: Coverage/consistency checks
  - [ ] Stage 3: Package the submission zip
  - [ ] Stage 4: Methodology write-up
- [ ] **Phase 8 — Iteration** *(ongoing, not a one-time checklist item)*
