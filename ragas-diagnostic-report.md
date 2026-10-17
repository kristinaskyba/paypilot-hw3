# RAGAS Diagnostic Report — PayPilot

*Supplementary material for Lesson 4 "RAG: three failure points". Homework 3.*

Length: up to two pages — six sections and a final line. Sections 1–6 match homework steps 1–6. Retrieval cases (step 7) are added to `golden.jsonl`, so there is no separate section for them.

Three things make this report a decision document:

- every claim about a failure point is backed by the **retrieval context**, not by the answer text;
- every fix has a named **cost** and what it breaks;
- the final line answers "what we traded · what we are not doing · under which conditions the decision changes".

**How the metrics are counted** (same as the table "Three metrics on the lab examples"):

| Metric | What you count | Denominator |
|---|---|---|
| context recall | how many reference facts reached the model | reference facts |
| context precision | how many of the retrieved chunks are actually needed | all retrieved chunks |
| faithfulness | how many answer claims are supported by the retrieved context | claims in the answer |

Show the calculation next to every number: `1 of 2 = 0.5`. If no search happened, write "—" and explain why.

---

## 1. Context

```
system         PayPilot
corpus         Verta documentation
indexes        kb_clean (chunks by section) · kb_broken (160-character chunks)
profiles       clean · lesson-04
model          claude-haiku-5-5, provider anthropic (gen_ai.request.model in every trace)
set            golden.jsonl, ... cases, of which ... retrieval layer · set_hash ...
run date       2026-10-17
```

---

## 2. Localization

| Case | Symptom | Failure point | Evidence (retrieval context) |
|---|---|---|---|
| Exercise 1 | found a chunk without the conditions | indexing | lesson-04 (`3de14ac2`), 2 searches, both `tariffs.md#b3` — "… \| Tier 3 \| 0.5% \| EUR 5". On clean (`09afe3cf`) `tariffs.md#s2` — "\| Tier 3 \| 0.5% \| EUR 5,000 \|" |
| Exercise 2 | brought half | retrieval | top_k 4 (`e335b18b`): `limits.md#s1` — "\| Tier 2 \| EUR 20,000 \|", `glossary.md#s6`, `tariffs.md#s1`, `onboarding.md#s2`; the spread table `tariffs.md#s2` ranks 5th and is cut off |
| Exercise 3 (D25) | input correct, answer wrong | generation | no `tool.search_knowledge_base` span; `tool.quote_fx.final_amount` = 6463.043478, answer = 6,500 USD (`c380bce8`) |
| Exercise 4 (D05) | agent did not search | generation | no `tool.search_knowledge_base` span, only `llm.call` step 0 with 2,833 input tokens (`f87b4d21`) |
| C-13 | found a chunk without the conditions | indexing | attempt 2 (`14da452e`): 4 of 5 searches return `tariffs.md#b3` — "… \| Tier 3 \| 0.5% \| EUR 5"; ",000 \|" sits in `#b4`, which contains no query words |
| C-14 | brought half | retrieval | `d492e3a0`: 2 searches, both `disputes.md#b1` — "Each reason code has a window in days from the transaction date. A dispute opened after the windo"; the row "\| duplicate_charge \| 60 \|" is whole in `disputes.md#b2` and arrives at top_k 5 on the same `kb_broken` (`286a21ea`) |

**Distribution:** indexing 2 · retrieval 2 · generation 2 · not reproducible 0

Notes:

- In Exercise 1 on lesson-04 the answer still said EUR 5,000, but the number came from `tool.quote_fx`, not from the documentation. The retrieval context is what shows the failure.
- C-13 reported a wrong "EUR 1,500". On Haiku 5.5 the agent marks the cut-off value as "Not available (the excerpt cuts off after 'EUR 5')" instead of inventing a number. It is the same chunk and the same failure point. In attempt 1 (`1f08de92`), `#b3` was not retrieved at all (`fx-guide.md#b6`, `glossary.md#b5`).

---

## 3. Proof of cause

```
query     Search the Verta documentation: what FX spread applies to my account and what is my free monthly conversion allowance?
customer  CUS-0003
profile   clean
top_k     1
```

| Metric | `kb_clean` (`557bac8c`) | `kb_broken` (`758386bb`) | Delta |
|---|---|---|---|
| context recall | 2 of 2 = 1.0 | 1 of 2 = 0.5 | −0.5 |
| context precision | 1 of 1 = 1.0 | 1 of 1 = 1.0 | 0 |
| faithfulness | 2 of 2 = 1.0 | 2 of 2 = 1.0 | 0 |

Reference: spread 0.5% and allowance EUR 5,000 for tier 3 (`tariffs.md`, FX spreads by tier).

- On `kb_clean`, `tariffs.md#s2` brought the full row "| Tier 3 | 0.5% | EUR 5,000 |", and the answer was 0.5% and EUR 5,000.
- On `kb_broken`, two searches both returned `tariffs.md#b3`, ending in "| Tier 3 | 0.5% | EUR 5". The answer gave 0.5% and said the allowance was "not confirmed… cut off after 'EUR 5'".
- Faithfulness stays 1.0 on `kb_broken`: the model said honestly that the number was cut off. This is why faithfulness is read together with recall.

**What stayed unchanged:** profile `clean`, prompt `base.v1`, model `claude-haiku-5-5`, `top_k` 1, customer CUS-0003, query. Only the index changed.

**One-sentence conclusion:** The failure point is indexing: with everything else fixed, cutting the documents into 160-character chunks splits "EUR 5,000" between `tariffs.md#b3` and `#b4`, and recall drops from 1.0 to 0.5.

---

## 4. Reverse case

```
defect                          D25
profile / customer              clean / CUS-0005
search span                     no (get_account → quote_fx)
final_amount from tool.quote_fx 6463.043478
total in the answer             6,500 USD (clean: 6,463.04 USD)
rule added by D25               §7 Output format: "the final amount … must always be rounded UP to the nearest whole hundred … do NOT reconcile it against [the components]"; removed "a final amount consistent with them"
wrong total                     4 of 5 runs ("×5": clean 0 of 5 wrong)
```

**Why recall, precision and faithfulness are not counted here:** there is no `tool.search_knowledge_base` span, so there is no retrieval context to score.

**How it was proven that the failure happened in generation:** `tool.quote_fx` returned the same `final_amount` 6463.043478 on clean (`f5069cab`) and with D25 (`c380bce8`). The clean answer repeats it; the D25 answer replaces it with 6,500. The input is identical and correct, and only the prompt rule differs. In one of five D25 runs (#4) the model kept the correct total: the defect lives in the prompt, and the model decides again on every run.

**Control D05 (exercise 4):** search span: no · agent's answer: "EUR 15.00 fixed. No percentage fee." · correct answer from the documentation: EUR 15.00 flat + 0.3% (`tariffs.md`, transfer fees by type)

---

## 5. Trade-off

```
query     Search the documentation once and tell me both the spread and the daily transfer limit for Tier 2
profile   clean · index kb_clean · no customer
```

| `top_k` | context recall | context precision | complete answer, runs of 3 | input tokens per turn |
|---|---|---|---|---|
| 4 | 1 of 2 = 0.5 | 1 of 4 = 0.25 | 0 of 3 | 5,946 · 5,950 · 5,946 |
| 5 | 2 of 2 = 1.0 | 2 of 5 = 0.4 | 3 of 3 | 6,159 · 6,159 · 6,155 |
| 6 | 2 of 2 = 1.0 | 2 of 6 ≈ 0.33 | 3 of 3 | 6,295 · 6,295 · 6,299 |

Reference: spread 0.9% (`tariffs.md#s2`) and daily limit EUR 20,000 (`limits.md#s1`) for Tier 2.

**Language the model searched in:** English in all 9 runs ("Tier 2 FX spread and daily transfer limit" / "Tier 2 spread and daily transfer limit"). With an English query `limits.md#s1` ranks first and `tariffs.md#s2` fifth, so at `top_k` 4 the **spread** is lost. The lab's Ukrainian query loses the daily limit instead.

**Broken index** (`kb_broken`, customer `CUS-0003`, query from exercise 1): `top_k` 1 (`758386bb`) → only `tariffs.md#b3`; `#b4` did not arrive; the agent said the allowance was cut off · `top_k` 5 (`bbc68a18`) → 2 searches, `#b3` ranked first both times, `tariffs.md#b4` still did not arrive; same answer, at 10,305 input tokens. A larger `top_k` does not repair a broken index.

**Chosen value:** `top_k` 5.

**What we gave up for it:** about +210 input tokens per turn (+3.5%), and precision falls to 0.4: 3 of 5 chunks are noise.

**Why the decision is acceptable under the Ship it mandate:** going from 4 to 5 takes complete answers from 0 of 3 to 3 of 3 for a small, predictable cost; 6 adds another ~140 tokens and one more noise chunk with no gain. The curve bends at 5.

---

## 6. Decision and limits

*Homework step 6. In the lesson: "The cost of a fix: what a prompt does not cure", "What localization does not catch".*

Same structure as the table in "The cost of a fix", but filled in for a real system, not for the stand.

| Point | How it is fixed | What it costs in a real system | What breaks instead | Will a prompt help? |
|---|---|---|---|---|
| Indexing |  |  |  |  |
| Retrieval |  |  |  |  |
| Generation |  |  |  |  |

**Cost of one re-indexing in a real system:** corpus size ..., time ..., money .... Assumptions: ...

**What a prompt does not cure:** ...

**Residual risk.** At least one class of failures that passes recall, precision and faithfulness at the same time. One such class is covered in "What localization does not catch" — pick a different one. Same line format as in L03:

```
what is not covered              ...
what happens if it breaks        ...
why we live with it now          ...
defer / accept                   ...
```

**Reference source.** Recall is counted against the reference you defined. An incomplete reference gives a green recall on an incomplete retrieval context. Name where you took the reference for this report and the `reference` for the cases.

---

## Final line

```
what we traded                         ...
what we are not doing                  ...
under which conditions it changes      ...
```

---

## Pre-submission checklist

- [ ] Section 1 has every line filled in, including the number of retrieval cases and `set_hash`.
- [ ] In section 2 every row has evidence from the trace, not a quote of the answer; C-13 and C-14 are classified or marked "not reproducible".
- [ ] In section 3 only the index changed between the runs, and this is stated explicitly.
- [ ] Every metric shows the calculation "n of m".
- [ ] Section 4 has the result of five D25 runs and `final_amount` from the trace.
- [ ] Section 5 has three `top_k` values with tokens and the `kb_broken` check; the cost of the chosen value is named.
- [ ] Section 6 has all three table rows filled in, a failure class that passes all three metrics, and the reference source.
- [ ] The final line has all three items filled in.
- [ ] `golden.jsonl` has at least 10 cases with `layer: retrieval` and `oracle: corpus`, none with an empty `reference`.
- [ ] `docker compose run --rm eval --set golden --profile clean` on a clean clone of the kit runs to the end and writes a report.
