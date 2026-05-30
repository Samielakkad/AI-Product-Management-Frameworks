# Multimodal Decision Matrix — Where to Put Image Classification

> When to do image classification client-side vs through a vision API vs not at all. The tradeoff is latency × cost × privacy × accuracy, and the right answer depends on which one is binding.

---

## The four placements

```
Option A: Client-side classifier         Option B: Server-side classifier
   (TensorFlow.js, ONNX Web)                (Python service, custom model)
   ┌─────────────┐                          ┌──────────────┐
   │   Browser   │                          │   Browser    │
   │  Image → ML │                          │   Image      │
   │  → label    │                          │     ↓        │
   └─────────────┘                          │  Server      │
                                            │  Image → ML  │
                                            │  → label     │
                                            └──────────────┘

Option C: Vision API                      Option D: No classifier
   ┌─────────────┐                          ┌─────────────┐
   │   Browser   │                          │   Browser   │
   │  Image      │                          │  Image →    │
   │     ↓       │                          │  ignore     │
   │  Vision API │                          │             │
   │  → label    │                          └─────────────┘
   └─────────────┘
```

---

## The four constraints

| Constraint | Description | When it dominates |
|---|---|---|
| **Latency** | Time from "user submits image" to "system has label" | Real-time chat |
| **Cost** | $ per image classification at production scale | High-volume products |
| **Privacy** | Does the image leave the device? | Regulated industries, image-sensitive markets |
| **Accuracy** | Recall and precision on the target taxonomy | Domains where misclassification is expensive |

A product rarely has all four binding. Identify which one(s) bind for your product, then pick the placement.

---

## Decision matrix

| Placement | Latency | Cost at scale | Privacy | Accuracy | Best for |
|---|---|---|---|---|---|
| **A. Client-side** | 50–300ms | Free | Excellent (image never leaves) | Medium (small model) | Privacy-first products, high-volume products with cost pressure |
| **B. Server-side** | 100–500ms | Low (own infra) | Medium (image transits) | High (full-size model) | Mid-scale products with custom taxonomy |
| **C. Vision API** | 200–800ms | High ($0.005–0.05/image) | Low (image leaves your control) | Very high (frontier model) | Low-volume products, prototypes, broad-domain classification |
| **D. None** | 0ms | $0 | Excellent | n/a | Products where the text signal is sufficient |

---

## The jak.ma path

We landed on a hybrid: **Option C in production today, Option A on the roadmap.**

### Today: Grok-2-Vision (Option C)

- About 12% of queries arrive with an image.
- Cost: $0.005 per image classification.
- Latency: 200–400ms median.
- We get high accuracy across a broad domain (the model knows what a faucet, a broken AC, a cracked tile look like).
- The image transits to the API; we have legal language in the ToS but it is not zero-trust.

This works because volume is moderate (~400 image classifications/day) and accuracy matters more than cost. At our current pricing, image-bearing queries cost ~$2/day in vision-API fees.

### Tomorrow: client-side MobileNetV3 (Option A)

The break-even point is at ~50,000 image queries/day, where vision-API costs reach $250/day. Before that, we keep Option C; after that, we switch to Option A.

The transition plan:

1. **Collect labeled images.** Need 50–100 labeled images per trade × 12 trades = 600–1200 labeled examples.
2. **Train MobileNetV3 on the catalog.** ~4 hours on a single A100, ~$15 in compute.
3. **Convert to TensorFlow.js.** Adds 5–10MB to the SPA bundle. Acceptable for our market (avg mobile data plan is good).
4. **A/B test in production** against the vision API. Switch when client-side reaches 90% of API recall at acceptable precision.

**Why MobileNetV3 specifically:**

- ~250ms inference on a mid-tier Android device.
- Pre-trained on ImageNet, fine-tunes well on small custom taxonomies.
- TFJS port is mature and well-documented.
- Quantized model is ~10MB — fits the bundle budget.

### Why not Option B (server-side custom model)

For jak.ma specifically, we evaluated it and skipped:

- Adds an inference service to maintain (we currently have zero).
- Adds latency vs Option A (image still transits to the server).
- Privacy story is worse than Option A (image leaves device).
- Cost is only marginally better than Option A once at scale (Option A is $0; Option B has fixed infra cost).

Option B makes sense for products with a custom taxonomy where the client-side model is too small to achieve target accuracy. Not jak.ma's case.

### Why not Option D (no classifier)

We tried — text-only classification for image-attached queries. Eval-score on `trade_fit` dropped 0.4 on the image-bearing slice. The image evidence really does help disambiguate "leak" (plumbing) from "stain" (cleaning) from "crack" (tiling).

---

## When Option A would be wrong for jak.ma

If our taxonomy were broader (instead of 12 closed trades, say 200 open categories), MobileNetV3 trained on 12 × 100 images would not have the capacity. We would need a larger model (which a browser cannot reasonably ship) or stay on Option C.

If our market were privacy-regulated (medical imaging, child safety classifications), Option A's "image never leaves device" would shift from "nice" to "required." We are not in that regime.

---

## The honest answer most teams should give

Most teams reaching for vision in their LLM product should run **Option D first** for ~2 weeks. Strip the image, classify on text only, measure the eval-score gap. If the gap is small, image classification is decoration, not signal — skip it.

Most teams who genuinely need image classification should default to **Option C** as a prototype. Cost is irrelevant at prototype scale. Latency is acceptable. Accuracy is high. Migrate to Option A only when scale forces it.

Most teams jumping to Option B as their first move are over-engineering. Server-side custom-model inference is justified only by specific custom-taxonomy or accuracy-budget pressures.

---

## A worked latency budget for Option A

If you choose Option A, here is what the latency budget looks like for jak.ma-sized requests:

| Stage | Budget | Comment |
|---|---|---|
| Image capture + downscale | 50ms | Use a Canvas resize to 224×224 before inference |
| Inference | 200ms (mid-tier Android) | MobileNetV3 quantized |
| Result available for Pass 1 | within 250ms total | Falls within the 250–400ms Pass 1 latency budget |

Inference dominates. The trick is to start inference *as soon as the image is captured*, not when the user hits send. By the time the user types their query and hits send, the label is already available.

---

## How this connects to the rest

- **System architecture (where the image fits):** [`jak-ma-eval-suite/docs/architecture.md`](https://github.com/Samielakkad/AI-LLM-Evaluation-JakMa/blob/main/docs/architecture.md)
- **Cost model:** [`jak-ma-eval-suite/docs/cost_model.md`](https://github.com/Samielakkad/AI-LLM-Evaluation-JakMa/blob/main/docs/cost_model.md)
- **Latency budget:** [`jak-ma-eval-suite/docs/latency_budget.md`](https://github.com/Samielakkad/AI-LLM-Evaluation-JakMa/blob/main/docs/latency_budget.md)

---

**Sami EL AKKAD** · sam25@mails.tsinghua.edu.cn
