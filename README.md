# L04 · RAGAS Diagnostic Report — PayPilot
Set from HW_2: https://github.com/kristinaskyba/PayPilot/tree/main/lab03

## Run
Put `golden.jsonl` into `~/paypilot/l03/sets/` and run:

    docker compose run --rm eval --set golden --profile clean

`set_hash`: 75ef2194fec3

## Retrieval cases
| id | Source | Corpus file and row |
| --- | --- | --- |
| RET-H01 | complaint C-13 | tariffs.md, FX spreads by tier — `\| Tier 3 \| 0.5% \| EUR 5,000 \|` |
| RET-H02 | complaint C-14 | disputes.md, reason codes table — `\| duplicate_charge \| 60 \|` |
| RET-H03 | lab, exercise 1 | tariffs.md, FX spreads by tier — `\| Tier 3 \| 0.5% \| EUR 5,000 \|` |
| RET-H04 | lab, exercise 2 | tariffs.md, FX spreads by tier — `\| Tier 2 \| 0.9% \| EUR 1,000 \|` |
| RET-H05 | corpus | tariffs.md, transfer fees — `\| SWIFT \| EUR 15.00 \| 0.3% \|` |
| RET-H06 | corpus | limits.md, limits by tier — `\| Tier 1 \| EUR 5,000 \| EUR 50,000 \|` |
| RET-H07 | corpus | limits.md, limits by tier — `\| Tier 3 \| EUR 100,000 \| EUR 1,000,000 \|` |
| RET-H08 | corpus | disputes.md, reason codes table — `\| unauthorized_debit \| 56 \|` |
| RET-H09 | corpus | disputes.md, reason codes table — `\| fraud_card_not_present \| 120 \|` |
| RET-H10 | corpus | tariffs.md, transfer fees — `\| Card payment \| EUR 0.00 \| 1.2% \|` |
| RET-H11 | corpus | disputes-process.md, provisional credit — "…a provisional credit is applied within two business days." |
| RET-H12 | corpus | fx-operations.md, allowance accounting — "…resets on the first calendar day of each month." |

## Runs
| Run | Profile | Result | Report |
| --- | --- | --- | --- |
| Baseline | `clean` | 40/41 (retrieval 11/12) | `reports/golden-clean-20261017T151008.json` |
| Lesson profile | `lesson-04` | 32/41 (retrieval 3/12) | `reports/golden-lesson-04-20261017T151421.json` |

Prediction before the `lesson-04` run: on `lesson-04` (kb_broken, top_k forced to 1, D05) these retrieval cases will fail:
- RET-H01, RET-H03 — chunking: "EUR 5,000" is split between `tariffs.md#b3` and `#b4`;
- RET-H02, RET-H09 — top_k 1: `disputes.md#b1` (the intro) outranks `#b2`, where the 60 and 120 rows are whole;
- RET-H08 — the same top_k 1 problem: the row is whole in `disputes.md#b3` but the intro ranks first;
- RET-H05, RET-H10 — D05: the agent answers transfer fees from the stale list in the prompt without searching (and the SWIFT row is also split on `kb_broken`);
- RET-H04 — top_k 1 brings one chunk, the limits table, so the spread is missing.

Expected to pass: RET-H06, RET-H07 (the limits rows are whole in `limits.md#b3`), RET-H11, RET-H12 (a single prose sentence in one chunk); uncertain, because with top_k 1 the first chunk must be exactly the right one.
Also on `clean`: RET-H04 may fail even on the baseline — at top_k 4 the English search query loses the spread table (0 of 3 in report section 5).

Commit with the prediction: `abc2dbb`
What failed on `lesson-04`: RET-H01, RET-H02, RET-H03, RET-H05, RET-H06, RET-H07, RET-H08, RET-H11, RET-H12. On `clean`: RET-H04.
Difference from the prediction (6 of 12):
- RET-H04 passed: top_k 1 returned `tariffs.md#b3`, which holds the Tier 2 spread; the daily limit was missing, but the case checks only "0.9%" — a one-needle case cannot see the missing second fact.
- RET-H09 passed: the agent's second search used the code name `fraud_card_not_present` and `disputes.md#b2` ranked first.
- RET-H10 passed: D05 skipped the search, but the stale fee list in the prompt has the card fee right (1.2%); only the SWIFT fee is wrong there.
- RET-H06 failed: top_k 1 returned `tariffs.md#b3` instead of `limits.md#b3`.
- RET-H07 failed: top_k 1 returned `glossary.md#b5`, then the limits intro `limits.md#b1`; the table chunk `#b3` never ranked first.
- RET-H11, RET-H12 failed: the answer sentence is cut between chunks — the start with the query words is in `disputes-process.md#b4` / `fx-operations.md#b3`, the answer phrase in `#b5` / `#b4`. Search returns the chunk with the query words, not the one with the answer. By the trace these are chunking failures, not top_k as labelled in `golden.jsonl`.
