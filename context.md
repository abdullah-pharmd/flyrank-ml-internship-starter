# Project Context Log — FlyRank ML Internship

Running summary of what was built, what was learned, and what comes next.
Updated after each completed task. Read this before starting a new task.

---

## The Big Picture

**Lane**: Content Refresh Prioritisation — predicting which pages are likely losing
search traffic so editors can refresh them first.

**Core question**: Given Google Search Console signals (impressions, clicks, CTR,
position) and content metadata (age, word count), which pages are most likely to be
in traffic decline in the next period?

**Honest answer the data gives us**: A Random Forest trained on client-grouped data
can rank pages by decline risk at **Precision@50 = 0.540** — meaning 27 out of every
50 top-flagged pages are genuinely declining. That is better than a hand-written
threshold rule (baseline = 0.280) but far from perfect. The model is a **triage tool**
that cuts reviewer time, not an oracle.

---

## Completed Tasks

### ML-02 · w01 — Research Question
**File**: `work/notebooks/w01_research_question.ipynb`

Defined the lane: content refresh prioritisation for SEO teams.
Framed the question as a ranking/classification problem: predict `is_declining` (trend
direction = "down") from observable signals available before a refresh decision.
Justified the lane by tying it directly to FlyRank's live "refresh" flags.

---

### ML-03 · w02 — ML Task Framing
**File**: `work/notebooks/w02_ml_task_framing.ipynb`

Formalised the ML task:
- **Input**: one row per content item per time window (impressions, CTR, position, age, word count)
- **Output**: probability score; top-K list ranked by that score
- **Metric**: Precision@50 (how many of the top 50 flagged pages are real declines)
- **Baseline to beat**: a deterministic hand-written rule
- **Validation design**: client-grouped split (same client never in train and test)

---

### ML-04 · w03 — Data Contract
**File**: `work/notebooks/w03_data_contract.ipynb`

Proved the dataset is usable:
- Grain: one row = one content item snapshot (verified no duplicate grain)
- All required feature columns present and typed correctly
- Ran a leakage experiment: features that are computed *from* the label give
  near-perfect ROC-AUC (~0.97); honest features give ~0.61. Documented the
  boundary clearly.
- Confirmed label (`trend_direction`) is available for ~84% of rows.

---

### ML-07 · w04 — Baseline Score
**File**: `work/notebooks/w04_baseline_score.ipynb`

Built the deterministic rule-based baseline:
```
flag = (days_since_update ≥ 180) AND (impressions ≥ 100) AND (CTR < median_CTR)
```
**Signal audit** (two bucket tables, one-word verdicts):
- Staleness (days_since_update): CONFIRMED — older pages decay at higher rates
- CTR vs position: CONFIRMED — below-median CTR pages show elevated decline rates

**Baseline result**: Precision@50 = **0.280**
This is the number every future model must beat.

---

### ML-08 · w05 — Model
**File**: `work/notebooks/w05_model.ipynb`

Trained a Random Forest classifier (100 trees, max_depth=8, random_seed=42).

**Key design choice**: client-grouped split (GroupShuffleSplit, 75/25).
Same client is never in both train and test, preventing data leakage across clients.

| Split type | Precision@50 | ROC-AUC |
|------------|:------------:|:-------:|
| Random (optimistic) | 0.960 | 0.984 |
| Client holdout (honest) | **0.540** | **0.611** |

Top features by importance: `impressions_90d`, `avg_position`, `days_since_last_update`, `ctr`.

**Lesson**: random splits produce wildly optimistic numbers. The 42-point gap (0.96 vs 0.54)
is "optimism bias" — the model is memorising client-specific patterns, not generalising.

---

### ML-09 · w06 — Validation Audit
**File**: `work/notebooks/w06_validation_audit.ipynb`

Audited the FlyRank research paper and then turned the same lens on our own model.

**Paper critique (two findings)**:
1. "Refreshed pages saw 3× traffic recovery" — label comes from pages that *were chosen*
   for refresh. Selection bias: whoever picks which pages to refresh is likely already
   targeting the worst performers, inflating the measured recovery gap.
2. "Position improvements averaged 4.2 places" — no control group; natural position
   fluctuations are not separated from the refresh effect.

**Self-audit**:
- Quantified the optimism bias: random split P@50 = 0.960 vs grouped P@50 = 0.540
- No feature leakage found — all features predate the label window
- Real failure examples inspected: model over-flags high-impression pages in low-CTR
  niches where low CTR is normal (e.g. informational queries)

---

### ML-10 · w07 — Content Action Playbook
**File**: `work/notebooks/w07_action_playbook.ipynb`

Turned the validated model output into a human-usable decision-support playbook.

**Five action archetypes with reason codes**:
| Action | Reason Code | Trigger |
|--------|-------------|---------|
| `refresh_and_review_ctr` | `low_ctr_visible_page` | impressions ≥ 500 AND CTR < 0.5% AND position 1-20 |
| `refresh` | `stale_visible_page` | days_since_update ≥ 180 AND impressions ≥ 500 |
| `expand_and_refresh` | `thin_visible_page` | word_count < 1200 AND impressions ≥ 250 |
| `refresh` | `page_one_decay_risk` | position ≤ 10 AND content_age ≥ 180 days |
| `monitor` | `general_refresh_review` | none of the above |

**Hard limits**:
- Not valid for: new clients (<90 days data), non-English content, pages <50 impressions
- Should NEVER be automated: publishing refreshes, bulk batch actions, page redirects

**Monitoring trigger**: retrain if Precision@50 drops below 0.45 on a fresh holdout.

**Exports committed**:
- `work/outputs/ml_action_score.csv` — full scored queue (~28k rows)
- `work/outputs/metrics_receipt.json` — P@50=0.540, AUC=0.611, base_rate=0.517
- `work/figures/feature_importances.png` — for the paper
- `work/figures/precision_at_k_curve.png` — for the paper

---

## Key Numbers to Know

| What | Number |
|------|--------|
| Dataset rows | ~28,000 content snapshots |
| Clients in dataset | 32 |
| Base rate (% declining) | 51.7% |
| Baseline rule Precision@50 | 0.280 |
| Model Precision@50 (honest) | **0.540** |
| Model ROC-AUC (honest) | 0.611 |
| Optimism bias (random vs grouped split) | +0.420 in P@50 |

---

## What Comes Next

**ML-11 · capstone.ipynb** — Write the full research paper in the notebook.
The paper builds directly on the outputs in `work/outputs/` and figures in `work/figures/`.
Template is at `work/capstone_report_template.md`.

**ML-12** — Demo outline, social post, employer-facing summary (lives in the closing
cells of `capstone.ipynb`).
