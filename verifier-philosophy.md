# Verifier Philosophy — Why Determinism Beats LLM-as-Judge in Production

> When to use a deterministic Python verifier vs an LLM-as-judge for production grounding. The argument is structural: shared priors collapse into shared mistakes.

---

## The one-sentence version

**The value of a verifier is that it disagrees with the generator on a different axis.** If the verifier and generator share priors, they will agree on the generator's mistakes, and the verifier adds zero protection.

---

## Two architectural patterns

### Pattern A: LLM-as-judge

```
Generator (LLM) → Output → LLM-as-judge → pass / fail
```

The judge is itself an LLM. It reads the output and produces a verdict — usually a score against a rubric, sometimes a structured pass/fail.

**Used by:** OpenAI eval frameworks, most academic LLM benchmarks, most "AI safety" claim layers in industry.

### Pattern B: Deterministic verifier

```
Generator (LLM) → Output → Python rules → pass / fail
```

The verifier is hand-written code that checks structural properties of the output against pre-computed truth (rule tables, retrieval payloads, schemas).

**Used by:** jak.ma, code-generation pipelines that check compile/test, OpenTable-style booking pipelines that check availability.

---

## When each pattern wins

LLM-as-judge wins when:

- **The rubric is subjective or qualitative.** "Is this poem moving?" — no Python check exists.
- **The ground truth is in the judge's training data.** "Is this fact about Wikipedia entries correct?" — the judge has seen Wikipedia.
- **You need to evaluate many model versions against each other quickly.** Relative ranking signal is fast and cheap.
- **The judge's priors and the generator's priors are uncorrelated.** Different model family, different training data.

Deterministic verifier wins when:

- **The constraint is verifiable structurally.** Schema validity, retrieval grounding, price band membership, geographic plausibility, format compliance.
- **The ground truth is local and external to the model.** Local price tables, recent inventory state, specific factual data the model has not seen.
- **The cost of a false positive is high.** Production user trust, regulatory compliance, financial transactions.
- **The judge's priors are correlated with the generator's priors.** Same model family or shared training data — the judge will rubber-stamp the generator.

---

## Why jak.ma uses a deterministic verifier

Three reasons specific to this product, each generalizable:

### 1. The judge would have the generator's biases

Pass 2 of jak.ma runs on Grok-3-mini. If we ran an LLM-as-judge against the output, the cheapest option would be Grok-3-mini or a sibling. Grok's prior over Moroccan plumbing prices is biased high (English-language reasoning anchors). The judge would agree with the generator's biased price, and the verifier would add no signal.

This is the **shared-priors collapse**. It is the dominant failure mode of LLM-as-judge in low-resource markets.

### 2. The ground truth is not in any model

The fairness band for plumbing in Salé is a number derived from a 200-worker survey conducted in October 2025. It is not on the public internet. It is not in any training set. The only way the model could agree with it is if we put the number in the prompt — at which point the verifier is the prompt, and a deterministic check is faster, cheaper, and more reliable.

### 3. Determinism is debuggable

When the verifier rejects an output, the rejection is annotated: `V4_price_band_failure: generated 1200, expected 200-900 for plumbing in Salé`. We can show this to the team and the team can act on it.

When LLM-as-judge rejects an output, the rejection is "the judge gave it a 2." Why 2? The judge does not know. The rejection is not actionable.

---

## Six checks the jak.ma verifier runs

The deterministic verifier has six independent checks. Each is < 1ms to compute. The full spec is in [VERIFIER_SPEC.md](https://github.com/selakkad2003/jak-ma-eval-suite/blob/main/VERIFIER_SPEC.md).

| ID | Check | Why deterministic |
|---|---|---|
| V1 | Schema validity | Pydantic parse — trivially deterministic |
| V2 | Worker grounding | Set membership — constant-time check |
| V3 | City plausibility | Adjacency table lookup |
| V4 | Price band | Rule-table lookup + range check |
| V5 | Script purity | Regex over Unicode blocks |
| V6 | Toxicity / PII | Blocklist match + phone-number regex |

None of these benefit from an LLM. All of them benefit from being fast, explainable, and free.

---

## When to add a hybrid layer

A verifier can have an "LLM sanity check" as an OPTIONAL last layer:

```
LLM output → V1..V6 (Python) → if all pass → LLM-as-judge for tone → user
```

This is reasonable when:

- The deterministic checks catch the high-severity failures.
- The LLM judge catches *tone* or *politeness* — qualitative properties the deterministic verifier cannot encode.
- The LLM judge's rejection rate is independently low (so it does not introduce a new bottleneck).

jak.ma does not currently run this hybrid layer in production. Tone failures are rare in our domain (the responses are short and informational) and the cost is real.

---

## The anti-pattern: an LLM verifier checking an LLM generator with shared training

This is the dominant industry mistake. A team trains a generator. They want a "safety layer." They prompt the same model (or a sibling from the same family) to check the generator's output.

The shared-priors collapse means:

- The verifier accepts the generator's plausible-but-wrong outputs.
- The team observes "99% acceptance rate" and concludes the system is good.
- Users encounter the 99% — including the plausible-but-wrong slice — and lose trust.

The acceptance rate is high *because* the verifier shares the generator's blind spots. The metric is misleading.

The fix is structural: introduce a check that *cannot* share the generator's priors. Python rules. A different model family. Retrieved-from-external-source ground truth. A human rater. Anything but the same LLM.

---

## The asymmetric-disagreement principle

The strongest verifier is one whose failure modes are **uncorrelated** with the generator's failure modes.

- A generator that fabricates prices, paired with a verifier that checks against a hand-curated rule table → uncorrelated → strong protection.
- A generator that mis-classifies adjacent cities, paired with a verifier that uses an adjacency table → uncorrelated → strong protection.
- A generator that produces unsafe content, paired with a verifier that uses the same model to detect unsafe content → correlated → weak protection.

When designing a verifier, the question is not "what does the model do well?" — it is "where does the model fail in a way I can detect with a different method?"

---

## Practical guidance

If you are building a verifier today:

1. **List the failure modes you observe.** Be specific. "Price fabrication" not "hallucination."
2. **For each failure mode, ask: is there a deterministic check?** Schema? Set membership? Range? Regex?
3. **If yes, use the deterministic check.** It is faster, cheaper, and more reliable.
4. **If no, ask: can I introduce a check with uncorrelated priors?** Different model family. Retrieved ground truth. A different rubric.
5. **Only if neither works, fall back to LLM-as-judge with the same model family.** Document that this check has lower confidence and use it for ranking, not gating.

---

## How this connects to the rest

- **The actual checks:** [`jak-ma-eval-suite/VERIFIER_SPEC.md`](https://github.com/selakkad2003/jak-ma-eval-suite/blob/main/VERIFIER_SPEC.md)
- **Pricing rule table used by V4:** [`pricing-taxonomy.md`](./pricing-taxonomy.md)
- **5-dim rubric for evaluation:** [`evaluation-rubric.md`](./evaluation-rubric.md)
- **System architecture:** [`jak-ma-eval-suite/docs/architecture.md`](https://github.com/selakkad2003/jak-ma-eval-suite/blob/main/docs/architecture.md)

---

**Sami EL AKKAD** · sam25@mails.tsinghua.edu.cn
