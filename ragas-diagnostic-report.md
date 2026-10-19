# RAGAS Diagnostic Report — PayPilot

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
set            golden.jsonl, 46 cases, of which 12 retrieval layer · set_hash 75ef2194fec3
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

| Point | How it is fixed | What it costs in a real system | What breaks instead | Will a prompt help? |
|---|---|---|---|---|
| Indexing | Re-chunk by section and keep tables whole (no cut inside a row), keep the section heading in every chunk, then re-index | Re-embedding the whole corpus plus a full regression run of the golden set, and engineer time to check chunk boundaries. Time to implement: ≈ 1 hour of embedding and index swap plus 1–2 engineer days (see the estimate below) | All chunk ids change, so every `reference`/`corpus_source` must be re-pinned; bigger chunks bring more tokens and more noise per hit | No. The model never sees ",000" from `#b4`; a prompt can only make it say "cut off" honestly, as Haiku 5.5 already does |
| Retrieval | `top_k` 5 instead of 4 (later: a reranker or hybrid search) | About +210 input tokens (+3.5%) on every turn that searches, multiplied by all traffic; a reranker would add a call and latency. Time to implement: `top_k` 4 → 5 is a config change, ≈ 1 hour including one regression run of the golden set (the 41-case clean run took ~3 min on the stand; a 200-case set ≈ 15 min), plus ½ day to check the token budget and watch the first day of traffic; a reranker or hybrid search ≈ 1–2 weeks (new component, tuning, full regression) | Precision falls to 0.4: 3 of 5 chunks are noise, more text for the model to pick the wrong row from | Partly. "Search again with other words" sometimes helps (C-14 needed 3 searches at `top_k` 5), but at `top_k` 1 two searches returned the same `disputes.md#b1` |
| Generation | Fix the prompt (remove the D25 rounding rule and the D05 stale fee list) and add a deterministic check: the stated total must equal `tool.quote_fx.final_amount` | Prompt review plus a regression run with repeats (×5), because the defect fires only 4 of 5 times. Time to implement: removing the D25 rule and the D05 fee list ≈ 1 day (1-hour edit, prompt review, regression run with ×5 repeats — one run proves nothing for a 4-of-5 defect); the deterministic check (stated total = `tool.quote_fx.final_amount`) ≈ 2–3 engineer days to write, test and add to the gate | Any prompt change can shift other behaviour, so it needs the same regression run every time | Yes. Both D25 and D05 are prompt-caused, so this is the only point a prompt fixes; the deterministic check is the backstop |

**Cost of one re-indexing in a real system:** corpus size ≈ 5M tokens (≈ 1,000× the stand's ~5,000-token corpus of 21 files, e.g. ~2,000 documents of ~2,500 tokens), time ≈ 1 hour for embedding and index swap plus 1–2 engineer days to review chunk boundaries and re-pin references, money ≈ $0.10 for embeddings plus ≈ $0.15 for one regression run of a 200-case set. Assumptions: embedding price ~$0.02 / 1M tokens; ~6,000 input tokens per eval case (measured in section 5); claude-haiku-5-5 at $0.10 / 1M input (stand `pricing.py`). The stand itself uses word matching without embeddings, so re-indexing there is free; the real cost is people's time and re-pinning broken references, not compute.

**What a prompt does not cure:** a number cut in half by chunking (Exercise 1, C-13), a fact that never enters the top_k (Exercise 2, C-14 at `top_k` 1), and a top_k forced by the profile (D17). A prompt cannot make the model read a chunk that search did not return.

**Residual risk.**

```
what is not covered              a faithful answer that omits a mandatory action: e.g. a dispute blocked by a customer-level restriction, where the agent correctly explains why it is blocked but does not escalate to a human agent (disputes.md, "Escalation on blocked disputes"); recall 1.0 and precision 1.0 because the rule arrived, faithfulness 1.0 because every stated claim is in the context, so the missing step is invisible to all three metrics
what happens if it breaks        the customer is left with a blocked dispute and no next step; the case is never handled within the reason-code window, a process breach and a C-17-type complaint ("no next step")
why we live with it now          the three metrics score only what the answer says, not what it should have done; the escalation is visible only in the trace as a missing tool.escalate_to_human call
defer / accept                   defer: add cases that assert the escalate_to_human call in the trace (a tool-level check, not a retrieval metric) to the L13 gate
```

**Reference source.** The reference for this report and the `reference` of the retrieval cases come from the Verta documentation in the stand, `app/rag/corpus/*.md` (`tariffs.md`, `limits.md`, `disputes.md`), read directly from the files and never from agent answers. Limit: the corpus itself is the reference. If a document is outdated, recall stays green while the answer is wrong for the customer.

---

## Final line

```
what we traded                         +~3.5% input tokens per search turn and precision 0.4, for complete two-fact answers at top_k 5
what we are not doing                  not fixing indexing with top_k or the prompt (re-chunking is a separate change), and not switching to a more expensive model: none of the six failures was the model's fault
under which conditions it changes      questions that routinely need 3+ chunks, a token or latency budget being hit, or re-chunking by section (then re-measure top_k 3–5)
```
