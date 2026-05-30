# AI + Product Management · pm-frameworks-darija

> Reusable PM frameworks extracted from shipping a Darija-language conversational marketplace ([jak.ma](https://jak.ma)) and from the Baidu ERNIE Mentor Program (October–December 2025). Pricing taxonomy, evaluation rubric, rater calibration, verifier philosophy, multimodal decision matrix.

---

## Why this exists

Most PM frameworks you find online are written for B2B SaaS in English-speaking markets with deep incumbents and clean data. Almost none survive contact with a low-resource dialect, a market without published price discovery, and an AI stack that will fabricate confidently in a language you cannot easily evaluate.

This repository is the set of frameworks I had to invent to ship one specific product. I publish them so the next person does not have to invent them again.

The frameworks here have been load-bearing on:

- **[jak.ma](https://jak.ma)** — production Moroccan service marketplace, 3,400 daily queries, 0.7% verifier rejection rate
- **Baidu ERNIE Mentor Program** — three months of evaluation work on a frontier Chinese LLM (October–December 2025)

If you are building in a low-resource language, an unstandardized market, or a domain where LLMs will fabricate plausibly, the frameworks should transfer with minimal adaptation.

---

## What's inside

| File | What it is | When to use it |
|---|---|---|
| [`pricing-taxonomy.md`](./pricing-taxonomy.md) | How to build a verifiable price table from scratch in a market with no published prices | Any marketplace where the model will be asked "how much does X cost?" |
| [`evaluation-rubric.md`](./evaluation-rubric.md) | Five-dimension rubric (factuality, naturalness, trade-fit, price-fairness, geographic) with 0–4 anchors | Any conversational system where "correct" is multi-dimensional |
| [`calibration-protocol.md`](./calibration-protocol.md) | How to calibrate human raters to Krippendorff α ≥ 0.7 in two sessions | Before you trust any rater scores |
| [`verifier-philosophy.md`](./verifier-philosophy.md) | When to use a deterministic verifier vs LLM-as-judge | Designing the contract between your model and your users |
| [`multimodal-decision.md`](./multimodal-decision.md) | Decision matrix: when to push image classification client-side vs server-side vs through a vision API | Any product where images are part of the input |

Each file is a standalone reference. You can read them in any order.

---

## The thread

There is one idea connecting all five: **the value of a constraint is that it disagrees with the model on a different axis.**

A model trained on global text will agree with itself about the price of a faucet repair in Salé. A rule table built from a Moroccan survey will disagree, and that disagreement is the value.

A rater who shares the model's priors will agree with the model's mistakes. A calibration protocol that anchors the rater to ground-truth examples will disagree, and that disagreement is the value.

A verifier that uses an LLM to check an LLM will collapse into shared bias. A verifier that uses Python rules will diverge, and that divergence is the value.

The frameworks here are five different ways to make the disagreement happen on purpose.

---

## How they fit together

```
                User query
                    │
                    ▼
            ┌───────────────┐
            │ Classification│ ← evaluation-rubric.md (trade_fit)
            └───────┬───────┘
                    │
                    ▼
            ┌───────────────┐
            │   Retrieval   │ ← evaluation-rubric.md (geographic)
            └───────┬───────┘
                    │
                    ▼
            ┌───────────────┐
            │   Generation  │ ← evaluation-rubric.md (factuality, naturalness)
            └───────┬───────┘
                    │
                    ▼
            ┌───────────────┐
            │   Verifier    │ ← verifier-philosophy.md
            │               │ ← pricing-taxonomy.md (price-fairness check)
            └───────┬───────┘
                    │
                    ▼
                 Response

   Across the loop:                  Across the data path:
   calibration-protocol.md           multimodal-decision.md
   (rater training, eval set)        (when to use vision)
```

---

## Provenance

These frameworks were stress-tested against:

- **~3,400 daily production queries** across 12 trade categories and 12 Moroccan cities, with a 200-worker initial pricing survey
- **Eight weekly eval cycles** with calibrated human raters at jak.ma
- **The ERNIE Mentor Program** — methodology peer-reviewed against Baidu's internal evaluation infrastructure, with adversarial-pair construction protocols cross-applied

Where a framework's design was changed by production data, the file documents the original assumption and the change.

---

## What's NOT in here

- **Model architecture details.** See [jak-ma-eval-suite/docs/architecture.md](https://github.com/Samielakkad/ai-llm-evaluation-jakma/blob/main/docs/architecture.md).
- **Production code.** See [jak.ma](https://jak.ma).
- **Baidu-confidential details.** The ERNIE-program methodology is the public version; specifics are not in this repo.
- **A general PM curriculum.** This is opinionated, narrow, and applied. If you want broad PM frameworks, look at Reforge or Lenny's Newsletter.

---

## Who this is for

- **PMs shipping LLM products in low-resource languages.** This is the manual you wish existed.
- **Engineers designing verifiers.** [`verifier-philosophy.md`](./verifier-philosophy.md) is the most engineering-dense file.
- **Evaluators / red-teamers.** [`calibration-protocol.md`](./calibration-protocol.md) is the most reusable file across domains.
- **Founders building marketplaces in markets without price discovery.** [`pricing-taxonomy.md`](./pricing-taxonomy.md) is the most product-dense file.

---

## License

MIT. Use these in your product, your blog post, your PRD, your interview. Citation appreciated, not required.

---

## Related repositories

- **[jak-ma-eval-suite](https://github.com/Samielakkad/ai-llm-evaluation-jakma)** — verifier spec, prompt set, sample queries, eval runner
- **[jak-ma-case-study](https://github.com/Samielakkad/ai-product-jakma-case-study)** — production narrative: decisions, tradeoffs, what broke
- **[ernie-evaluation-notes](https://github.com/Samielakkad/ai-llm-evaluation-baidu-ernie)** — the calibration methodology from the Baidu ERNIE Mentor Program
- **[darija-nlp-resources](https://github.com/Samielakkad/ai-nlp-darija-resources)** — public corpora, papers, tools for Moroccan-Arabic NLP

---

**Sami EL AKKAD** · Tsinghua SIGS, AI MSc · sam25@mails.tsinghua.edu.cn · [jak.ma](https://jak.ma)
