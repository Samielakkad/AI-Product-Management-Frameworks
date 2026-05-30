# Evaluation Rubric — Five Dimensions for Conversational Marketplaces

> A reusable 5-dimension, 0–4 rubric for evaluating conversational systems in domains where "correct" is multi-dimensional. Developed for [jak.ma](https://jak.ma); methodology cross-applied during the Baidu ERNIE Mentor Program.

---

## Why five dimensions

A single "correctness" score collapses too many failure modes into one number. A response can be factually right and idiomatically wrong; it can be linguistically perfect and geographically nonsensical. A 5-dimension rubric forces raters to separate failure modes and gives the production team an actionable signal: "Pass 1 is fine; Pass 2 is regressing on price-fairness."

The five dimensions are not arbitrary. They map to the failure modes a conversational marketplace will hit in a low-resource market:

| Dimension | What it catches | Where in the pipeline |
|---|---|---|
| **Factuality** | The response misrepresents the retrieved data | Pass 2 |
| **Naturalness** | The response is grammatical but unnatural | Pass 2 |
| **Trade-fit** | The classifier picked the wrong category | Pass 1 |
| **Price-fairness** | The price band is wrong for the local market | Pass 2 + verifier |
| **Geographic** | The recommended worker is unreachable | Retrieval + verifier |

A regression on `factuality` is a generation problem. A regression on `trade-fit` is a classifier problem. A regression on `geographic` is a retrieval problem. The dimensions are diagnostic.

---

## The scale

Each dimension is scored 0–4. Anchors are written per-dimension below. A 4 is "would ship to a senior user." A 0 is "would damage user trust."

| Score | Meaning |
|---|---|
| 4 | Excellent — no concerns; would ship |
| 3 | Good — minor issues, would ship after a small fix |
| 2 | Acceptable — visible issues but recoverable |
| 1 | Poor — would erode user trust if shipped |
| 0 | Broken — would damage user trust immediately |

Aggregate: the dimension scores are averaged with equal weight. We report the aggregate but always alongside the per-dimension breakdown. A 3.7 aggregate with 2.1 on price-fairness is a different problem than a 3.7 aggregate with 3.4 on every dimension.

---

## Dimension 1 — Factuality

> Does the response accurately reflect the retrieved workers and their attributes?

**Anchors:**

- **4:** Every claim about a recommended worker is verifiable from the retrieval payload. Names, neighborhoods, years of experience, languages, ratings — all match.
- **3:** One minor detail is off (e.g. "He has 5 years experience" when the record says 4). Not user-damaging.
- **2:** A claim is misleading but not invented (e.g. emphasizing "verified" when the worker is not verified, but is approved).
- **1:** A claim is invented (e.g. "Hicham specializes in vintage plumbing" — not in the record).
- **0:** A worker is invented (cited by ID not in the retrieval payload). The verifier should have caught this; if it did not, that is also a `0` for the verifier.

**Calibration anchor (for raters):**

> Q: "bghit chi sba9 f Casa"
> Retrieved: [Hicham B. (Maarif, 4yrs, verified), Ahmed K. (Sidi Bernoussi, 7yrs, approved)]
> Response: "Wajadt-lik 2 sba9 mzyanin. Hicham B. f Maarif, 4 snin t9rib, verified. Ahmed K. f Sidi Bernoussi, 7 snin, approved."
> **Score:** 4 (every claim is verifiable in the retrieval).

---

## Dimension 2 — Naturalness

> Would a Moroccan reader find the Darija idiomatic and natural?

**Anchors:**

- **4:** Reads like a real Moroccan WhatsApp message. Appropriate code-switching, natural particles, no awkward MSA-Arabic constructions.
- **3:** Slightly bookish but acceptable. A native reader would not flag it.
- **2:** Noticeably MSA-leaning (uses "أريد" instead of "بغيت") or stilted. A native reader would notice but not be put off.
- **1:** Wrong dialect (e.g. Egyptian or Levantine Arabic constructions). A native reader would be put off.
- **0:** Not Darija (e.g. pure MSA or transliteration nonsense).

**Calibration anchor:**

> "Marhaba! Anaa sa-anasi3uka fee ijaadat sba9 jayed fee al-Dar al-Bayda'."
> **Score:** 1 (pure MSA, "anaa," "sa-anasi3uka" — not Darija at all).

---

## Dimension 3 — Trade-fit

> Does the system's classification match the user's actual intent?

**Anchors:**

- **4:** Classification is exact and unambiguous (user said "sba9" / plumber, system classified `plumbing`).
- **3:** Classification is correct but the system chose a broader category than ideal (user said "doorbell broken" — `electrical` is correct but `appliance_repair` would have been better).
- **2:** Classification is in an adjacent category (user said "leak in roof" — system classified `painting` instead of `tiling`).
- **1:** Classification is wrong but recoverable via follow-up (user said "AC not cold" — system classified `electrical`; user has to clarify).
- **0:** Classification is so wrong the conversation is unrecoverable.

**Calibration anchor:**

> Q: "kaynish chi pintur f Tetouan?"
> System: `trade_category: painting, city: tetouan`
> **Score:** 4 (exact).

---

## Dimension 4 — Price-fairness

> Is the price band within the local fairness range, as defined by the pricing taxonomy?

**Anchors:**

- **4:** Generated band is within ±10% of the rule-table midpoint for `(trade, city)`.
- **3:** Within ±20% (still passes V4, but at the edge).
- **2:** Within ±35% (would have been rejected by V4, but the magnitude is recoverable).
- **1:** Off by 35–100% (e.g. 2× the local rate). User-damaging.
- **0:** Off by >2× or unitless ("a few thousand"). The verifier should have caught this.

**Calibration anchor:**

> Q: "shhal taman sba9 f Sale?"
> Rule table: `plumbing.sale.mid = 400 MAD`
> Generated: `{low: 350, high: 480}` → midpoint 415 → 3.75% deviation.
> **Score:** 4.

---

## Dimension 5 — Geographic

> Are the recommended workers actually reachable for the user's city?

**Anchors:**

- **4:** Every recommended worker is in the user's city OR an adjacent commuter city in the adjacency table.
- **3:** All workers in the user's city, but the system did not surface adjacent-city workers when the in-city catalog was thin.
- **2:** One worker outside the commuter zone (e.g. Marrakech worker shown to a Casablanca user).
- **1:** Multiple workers outside the commuter zone.
- **0:** No workers in the user's region at all.

**Calibration anchor:**

> Q (user in Salé): "bghit chi sba9"
> Recommended: [worker in Salé, worker in Rabat (adjacent), worker in Temara (adjacent)]
> **Score:** 4 (all in commuter zone).

---

## How to run an evaluation cycle

### Weekly production cycle

1. **Sample 50 queries** stratified across trade categories (PII-scrubbed).
2. **Have 2 calibrated raters score all 50** across all 5 dimensions.
3. **Compute inter-rater Krippendorff α** per dimension. Target ≥ 0.7.
4. **If α < 0.7**, the raters need re-calibration before the scores can be trusted (see [`calibration-protocol.md`](./calibration-protocol.md)).
5. **Aggregate to per-dimension and overall scores.** Plot vs the rolling 4-week trend.
6. **If any dimension drops below 3.0**, cut a release-gate ticket. The team owns it before next release.

### Pre-release cycle

When a new prompt, new model, or new retrieval index is candidate for production:

1. Run the candidate against a frozen 100-query benchmark (`data/sample_queries.jsonl` in the eval suite).
2. Score against all 5 dimensions.
3. Candidate passes release-gate if **no dimension regresses by >0.3 from production baseline** and **aggregate does not regress.**
4. A regression on one dimension is acceptable if compensated by improvement on another; but >0.3 regression on any single dimension is auto-block.

---

## What this rubric does not capture

- **Latency.** Out of scope — separate SLO.
- **Cost.** Out of scope — separate SLO.
- **Adversarial robustness.** Adversarial queries are evaluated separately, see [`ernie-evaluation-notes/adversarial_pairs.md`](https://github.com/Samielakkad/ai-llm-evaluation-baidu-ernie/blob/main/adversarial_pairs.md).
- **Long-term user satisfaction.** Use NPS or retention, not this rubric.

---

## Transferability

Drop in replacements for non-marketplace domains:

| jak.ma dimension | Generic version |
|---|---|
| Factuality | Same |
| Naturalness | Same — but rewrite anchors for the target language |
| Trade-fit | Replace with "intent-fit" — does the classifier match the user intent? |
| Price-fairness | Replace with the equivalent verifiable constraint in your domain (e.g. "dosage-correctness" for a medical chatbot, "deadline-feasibility" for a scheduling assistant) |
| Geographic | Replace with the equivalent retrieval-grounding check |

The structural pattern — 5 dimensions, 0–4 scale, calibrated anchors per dimension, Krippendorff target ≥ 0.7 — transfers as-is.

---

## How this connects to the rest

- **Calibration:** see [`calibration-protocol.md`](./calibration-protocol.md)
- **Adversarial set construction:** see [`ernie-evaluation-notes/adversarial_pairs.md`](https://github.com/Samielakkad/ai-llm-evaluation-baidu-ernie/blob/main/adversarial_pairs.md)
- **Eval runner code:** see [`jak-ma-eval-suite/scripts/run_eval.py`](https://github.com/Samielakkad/ai-llm-evaluation-jakma/blob/main/scripts/run_eval.py)
- **System architecture:** see [`jak-ma-eval-suite/docs/architecture.md`](https://github.com/Samielakkad/ai-llm-evaluation-jakma/blob/main/docs/architecture.md)

---

**Sami EL AKKAD** · sam25@mails.tsinghua.edu.cn
