# Calibration Protocol — Two Sessions to α ≥ 0.7

> How to calibrate human raters to inter-rater Krippendorff α ≥ 0.7 in two sessions. Developed for [jak.ma](https://jak.ma) and refined during the Baidu ERNIE Mentor Program.

---

## Why this matters

Uncalibrated rater scores are noise. Two competent raters scoring the same response on the same rubric will diverge by ~1 point on a 0–4 scale if they have not been calibrated. With 5 dimensions, that's ~5 points of unexplained variance per response. The aggregate signal is gone.

Krippendorff's α ≥ 0.7 is the industry threshold for "raters agree well enough to trust the aggregate." Below that, the score reflects rater identity more than response quality. Above it, you can talk meaningfully about whether a model regressed.

This protocol gets two raters from "competent but uncalibrated" to α ≥ 0.7 in two sessions, ~3 hours total.

---

## The protocol at a glance

```
Session 1 (90 min):                Session 2 (90 min, 1-2 days later):
- Walk through rubric (15 min)     - Re-score the same 20 anchors (30 min)
- Rate 20 anchor responses (45)    - Measure α
- Discuss every disagreement (30)  - Discuss residual disagreement (45 min)
                                   - Sign off on calibration (15 min)
```

After Session 2, raters are calibrated for that rubric. Re-calibration is required after any rubric change or every 6 weeks (the protocol drifts).

---

## Session 1 — Walk-through and first pass

### Materials

1. The rubric document (see [`evaluation-rubric.md`](./evaluation-rubric.md)).
2. A **20-anchor anchor set** — 20 responses pre-scored by the rubric author. Anchors must span the 0–4 range for each dimension and include at least 3 "edge" cases per dimension (where the score is between two anchors).
3. A rating sheet with one row per anchor × dimension (20 × 5 = 100 cells).

### Procedure

1. **Rubric walkthrough (15 min).** The session lead reads each dimension out loud, including all 0–4 anchors, including the calibration example. Raters ask questions; the lead answers.
2. **Independent rating (45 min).** Each rater scores all 20 anchors across all 5 dimensions. **No discussion.** Raters do not see each other's scores.
3. **Collect scores, compute pairwise α** for each dimension. Highlight every cell where the raters' scores differ by ≥2.
4. **Disagreement walkthrough (30 min).** For every flagged cell, both raters explain their reasoning. The lead arbitrates against the anchor reference. The rubric is updated if (and only if) the disagreement reveals an ambiguity in the anchors.

### Common Session 1 outcomes

- **Per-dimension α typically 0.4–0.6** after Session 1. This is normal.
- **The dimension with the lowest α is almost always `naturalness`** (most subjective).
- **Disagreements concentrate at 2 vs 3 boundary.** This is also normal. The discussion sharpens the anchors.

---

## Session 2 — Re-score and finalize

### Materials

Same 20 anchors. Updated rubric if Session 1 surfaced ambiguities.

### Procedure

1. **Independent re-rating (30 min).** Raters score the same 20 anchors with the (possibly updated) rubric. No reference to Session 1 scores.
2. **Compute α per dimension.** Target ≥ 0.7.
3. **Residual disagreement (45 min).** For each remaining ≥2-point gap, raters discuss. If the rubric is well-defined, residual disagreements are about *specific anchors*, not the rubric.
4. **Sign-off (15 min).** Each rater signs off on the rubric. If α < 0.7 on any dimension, that dimension is re-anchored and a Session 3 is scheduled (rare; under 10% of calibration cycles).

### Common Session 2 outcomes

- **Per-dimension α ≥ 0.7 on 4 of 5 dimensions** is typical.
- **`Naturalness` often lands at 0.65–0.70.** Acceptable if other dimensions are ≥0.75.
- **If α stays < 0.5 on any dimension after Session 2**, the rubric anchors are wrong, not the raters. Re-write anchors.

---

## What goes in the 20-anchor set

The anchor set is the calibration; the rubric is the theory. Quality of the anchor set predicts calibration success more than the rubric does.

### Composition

For a 5-dimension, 0–4 rubric:

- **4 anchors at score 4** (one per dimension where 4 is the *defining* dimension).
- **4 anchors at score 0** (same).
- **8 anchors at scores 1, 2, 3** (mix; cover the middle).
- **4 anchors that are "tricky"** — high on some dimensions, low on others, deliberately misleading.

### Diversity

- **Trade categories.** Cover at least 6 of 12.
- **Cities.** Cover at least 6 of 12.
- **Query types.** Mix `find_worker`, `ask_price`, `complaint`, `smalltalk`.
- **Languages.** Mix `darija_arabic`, `darija_arabizi`, `french`, `mixed`.

### Quality

Anchors must be:

- **Defensible.** The lead can argue every score against the rubric.
- **Specific.** Vague anchors collapse the calibration.
- **Recent.** Anchors should be from production traffic in the last 30 days (PII-scrubbed). Old anchors drift.

---

## Common failure modes

### Failure 1: "We agree on average but not on specific cases"

This is the most common pathology. Aggregate per-rater scores look similar (mean 3.2 vs 3.4), but pairwise α is low because raters disagree on *which* cases are 3 vs 4.

**Fix:** anchors at the 2/3 and 3/4 boundary are insufficient. Add 4 more boundary anchors. The boundary is where calibration happens.

### Failure 2: "One rater is systematically harsh"

Rater A averages 2.8; Rater B averages 3.5. α is acceptable (they're correlated), but the absolute scale is off.

**Fix:** Re-anchor the "4 = excellent, ship it" definition. The harsh rater has a different mental model of "ship it." Discussion + 4 new "4" anchors usually fixes it.

### Failure 3: "Raters agree, but the lead disagrees with both"

This is the rubric being wrong, not the raters being wrong. The raters have converged on a different rubric than the one the lead intended.

**Fix:** Re-write the rubric. The lead's intuition is the spec; if raters can't reconstruct it from the document, the document is incomplete.

---

## How often to re-calibrate

- **After any rubric change.** Always.
- **Every 6 weeks if the rubric is stable.** Inter-rater agreement drifts even with no rubric change, especially on `naturalness` and `price_fairness`.
- **When a new rater joins.** Treat the new rater as one party in a fresh 2-session calibration with an existing calibrated rater.
- **When production scores diverge from prior model results unexpectedly.** Possibly the model changed; possibly the rater drifted. Re-calibrate before assuming the model.

---

## What to track

- **Per-rater calibration date and per-dimension α at sign-off.** Each rater's "calibrated since" date.
- **Aggregate inter-rater α on every production eval cycle.** If it drops below 0.7 mid-cycle, that cycle's scores are flagged.
- **Re-calibration frequency.** Should be 6–8 weeks. If more often, the rubric is unstable; if less often, the raters are drifting unmonitored.

---

## LLM-as-judge — when to use it, when not to

Calibrated humans are expensive and slow. LLM judges are cheap and fast. The tradeoff:

- **LLM judges are good for:** consistency at scale, regression detection on stable rubrics, ranking many model versions against each other, anything where the *relative* signal matters more than the *absolute* score.
- **LLM judges are bad for:** rubric design (the judge has the model's priors), naturalness in low-resource languages (no anchor in the judge's training data), price-fairness in markets without published prices (no anchor at all), anything where the judge's priors are correlated with the generator's priors.

In practice for jak.ma: LLM judge for the weekly regression check; calibrated humans for the bi-weekly deep eval and for any release-gate decision. The two signals are checked against each other — if they diverge, the human signal wins, but the divergence itself is information.

---

## How this connects to the rest

- **The rubric:** [`evaluation-rubric.md`](./evaluation-rubric.md)
- **Why disagreement matters:** [`verifier-philosophy.md`](./verifier-philosophy.md)
- **Adversarial set construction (uses the same calibration approach):** [`ernie-evaluation-notes/calibration_practice.md`](https://github.com/Samielakkad/ai-llm-evaluation-baidu-ernie/blob/main/calibration_practice.md)

---

**Sami EL AKKAD** · sam25@mails.tsinghua.edu.cn
