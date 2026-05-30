# Pricing Taxonomy

> How to build a verifiable price table from scratch in a market with no published price discovery.

---

## The problem

A conversational system that surfaces prices will be wrong about prices, and the user will not know it is wrong. In a market with no published price benchmark (Moroccan plumbing, Tunisian electrical, Egyptian tiling — any service market across the MENA region, plus most of South and Southeast Asia), the model has no reliable prior to anchor on. It will produce a number that sounds plausible in English-language reasoning and is wrong by a factor of 3–10 in local terms.

This document is the operational playbook for building the rule table that catches that. It is the framework behind the [`V4` price-fairness check](https://github.com/Samielakkad/ai-llm-evaluation-jakma/blob/main/VERIFIER_SPEC.md) in the jak.ma verifier.

---

## The taxonomy structure

The table is two-dimensional: `(trade_category, city) → PriceBand`. Each band is four numbers:

| Field | What it is | How to derive it |
|---|---|---|
| `low` | Bottom of the survey range, 10th percentile | From the survey |
| `mid` | Median observed for typical job | From the survey |
| `high` | Top of the survey range, 90th percentile | From the survey |
| `urgent_premium` | Multiplier for `urgency = now` | From the survey |

Plus one configuration:

| Field | What it is |
|---|---|
| `fairness_margin` | The verifier accepts any generated band whose midpoint lies within `mid × (1 ± fairness_margin)`. Default 0.20 (20%). |

The verifier (V4) rejects any generation outside this band. The product behavior on rejection is to substitute the rule-table band with a templated message.

---

## How to build the table

### Step 1: Define the trade categories

For jak.ma, we landed on 12 categories: plumbing, electrical, tiling, painting, carpentry, welding, locksmithing, AC repair, appliance repair, gardening, moving, cleaning.

Heuristics for picking categories:

- **Mutual exclusivity.** A worker should be in exactly one (or one primary). If they fall into two, split the category.
- **Granularity at the user query level, not the worker level.** Users ask for "a plumber," not for "a residential gas-line specialist." Categories should match user vocabulary.
- **Survey feasibility.** If you cannot survey 15–20 workers per category × city cell, the category is too granular.

### Step 2: Define the cities (or geographic units)

For jak.ma, 12 metros covering ~70% of urban Morocco. The taxonomy is one row per city — no neighborhoods, no regions.

Heuristics:

- **One-hop adjacency table separately.** Salé and Rabat are 5km apart; Mohammedia and Casablanca are 20km. These are separate rows in the price table but are linked in an adjacency table so the verifier (V3 — city plausibility) treats them as substitutable for "reachable for the job."
- **Skip rural areas in v1.** The catalog there is too thin to support a price band.

### Step 3: Run the survey

Operationally, this is 80% of the work. The remaining 20% is filing it correctly.

**Sample size:** target 15–20 workers per (category, city) cell. For 12 × 12 = 144 cells, that's roughly 200–300 worker interviews.

**Interview format:**

1. "What is your most common job in this category?" — defines the "typical job" for the cell.
2. "What did your last three jobs of that type charge?" — anchors the band.
3. "If a customer called you for an emergency on a Sunday night, what would you charge?" — anchors `urgent_premium`.

**Don't:** survey via online form. Workers in the long tail of this market do not respond to forms. We did in-person and phone interviews. Total field cost: ~$1,200 in survey-coordinator time across two weeks.

**Don't:** trust web-scraped prices. Yelp-equivalents in this market are sparse and adversarially gamed.

### Step 4: Compile to the rule table

The output is a TypeScript-like structure (in jak.ma's `priorityService.ts`):

```typescript
const Pricing: Record<Trade, Record<City, PriceBand>> = {
  plumbing: {
    casablanca: { low: 200, mid: 450, high: 1200, urgent_premium: 1.4 },
    rabat:      { low: 180, mid: 400, high: 1100, urgent_premium: 1.4 },
    marrakech:  { low: 220, mid: 500, high: 1300, urgent_premium: 1.5 },
    // ...
  },
  // ... 12 trades
};

const FairnessMargin = 0.20;
```

### Step 5: Wire it to the verifier

The verifier check (in pseudocode):

```python
def check_price_band(generated_band, trade, city, urgency):
    rule = Pricing[trade][city]
    expected_mid = rule.mid * (rule.urgent_premium if urgency == "now" else 1.0)
    generated_mid = (generated_band.low + generated_band.high) / 2
    deviation = abs(generated_mid - expected_mid) / expected_mid
    return deviation <= FairnessMargin
```

If `check_price_band` returns `False`, the verifier rejects. The fallback path substitutes the rule-table band with a templated Darija message: "Hadshi taman l3am f had l-khedma huwa bin {low} u {high} dirham. Ila bghiti tafsil ktar, qoul-li."

---

## Maintenance cadence

- **Quarterly revision.** Re-survey 10–15% of the cells (sampled by recency of last update). Update the bands.
- **Drift alert.** If the median observed `price_band_mad` in production drifts >15% from the rule table for any cell, trigger a re-survey of that cell within two weeks.
- **New city onboarding.** Survey ~50 workers (4 per trade × 12 trades) before the city goes live in the chat. This was the bottleneck for Tangier and Agadir; budget for the survey, not just the deployment.

---

## What the table does not capture

- **Material costs vs labor costs.** The band is total job cost. If a customer brings their own materials, the band is wrong. This is rare enough in this market that we have not split it.
- **Time-of-day variance beyond `urgent_premium`.** A 9pm callout costs differently from a Sunday morning callout. The single multiplier is a deliberate simplification.
- **Long-tail rare jobs.** The band is for the "most common job in this category." A water-heater install in plumbing is typical; a sewage-line replacement is not. The verifier accepts any generation within the band, even if the actual job is rare-and-expensive. The Pass 2 generation is instructed to flag rare jobs via `urgency_note`.

---

## Transferability to other markets

The taxonomy structure is portable. The numbers are not.

To port to a new market:

1. **Trade taxonomy will be similar** — most service-marketplace verticals share ~80% of categories across cultures (plumbing, electrical, painting are universal; some categories vary, e.g. "tiler" is a separate trade in Morocco but bundled with "general contractor" in the US).
2. **Cities will need a fresh adjacency table.** Geographic plausibility is local.
3. **The survey is irreducible.** There is no shortcut. The whole point is that the rule table disagrees with the model's global prior. If you skip the survey, you have a rule table that agrees with the model — and there is no verifier.

The price taxonomy applied to a Tunisian market would have ~80% category overlap, ~0% number overlap, and ~30% adjacency-table overlap.

---

## How this connects to the rest of the system

- **Verifier:** uses the table directly via V4 — see [`verifier-philosophy.md`](./verifier-philosophy.md)
- **Evaluation rubric:** the `price_fairness` dimension is anchored to the table — see [`evaluation-rubric.md`](./evaluation-rubric.md)
- **System architecture:** see [`jak-ma-eval-suite/docs/architecture.md`](https://github.com/Samielakkad/ai-llm-evaluation-jakma/blob/main/docs/architecture.md)

---

**Sami EL AKKAD** · sam25@mails.tsinghua.edu.cn
