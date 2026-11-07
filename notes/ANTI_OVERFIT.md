# Anti-overfitting discipline

The hackathon gives three phases: TRAIN (labelled), LIVE (scored on a leaderboard, we may submit repeatedly) and
FINAL (one scored submission). The only way to lose a lot on FINAL is to tune to what we can see. This note is the
process we followed and the tools that enforce it.

## The loop

**dev decides, LIVE confirms, a sealed window is read once.**

- *dev*: two held-out months of TRAIN, rebuilt to look like a LIVE phase (`solution/tools/make_dev_dataset.py`,
  `solution/tools/dev.py`). The unchanged pipeline runs on it with `--phase live` and is scored with the official scorer.
- *LIVE*: one full CSV per accepted candidate. We read the leaderboard, never tune against it row by row.
- *sealed window*: a third TRAIN window (`dev.py --sealed`) that is looked at once per accepted candidate, and only the
  **sign** of the change is read.

Typical gains of a candidate are 30 to 300 bank movements out of ~70k on dev, i.e. well inside the noise of a score
that is partly a macro-F1 over rare exception classes. A few rows of a rare class can move the score by half a point,
so the gates below count **movements**, not score points.

## What can go wrong (and what we did about it)

1. *Holdout reuse.* Every accept/reject decision on dev is an adaptive query. Mitigation: statistical gates (G1-G3) and
   the sealed window (G4).
2. *Distribution shift.* dev, LIVE and FINAL are different months with different customers, banks and formats.
   Dev scored consistently above LIVE (about 0.6-0.9 points). Mitigation: `solution/tools/shift_report.py` compares two
   phases (entities, accounts, currencies, payers without history, reference rates, invoice-to-receipt delay,
   statement/advice formats) and alerts when a measure leaves a calibrated band.
3. *Analyst degrees of freedom.* A literal copied from dev (a date, an id, a name, an amount, a threshold) is a
   selected variable. Mitigation: G5.
4. *Composite metric.* See above: gates are computed on movements.

## The gates

| Gate | Rule | Where |
|---|---|---|
| G1 paired McNemar | c = wrong->right, b = right->wrong; z = (c-b)/sqrt(b+c) >= 3 and c-b >= 30 | `significance.py` |
| G2 sign test per payer | movements of one payer are not independent: exact one-sided binomial over payers (p <= 0.01) and no payer carries more than 25 % of c-b | `significance.py` |
| G3 balance by month and entity | c-b > 0 in every month of the window and >= 0 in every large entity | `significance.py` |
| G4 sealed window | one run per accepted candidate; only the sign of the change is read and it must agree with dev | `dev.py --sealed`, by hand |
| G5 no phase literals | sweep the diff for dates, ids, names, amounts or thresholds taken from the dev window | `git diff` + grep |
| G6 LIVE confirmation | one full CSV per accepted candidate; LIVE change must have the same sign as dev | leaderboard log |
| G7 one switch per pass, soft failure | each pass is enabled by its own variable; if it raises, it is skipped and the trunk output is untouched | code review |

`significance.py` returns ACCEPT / NEUTRAL / REJECT (exit code 0 / 1 / 2). NEUTRAL means no evidence either way:
do not merge and do not spend a LIVE upload.

## Other rules we applied

- No LLM or external API anywhere in the pipeline: every answer is reproducible from the data with fixed seeds
  (`PYTHONHASHSEED=0`).
- One scored FINAL submission, produced by a specific commit and verified with `solution/tools/validate.py` before upload.
- `solution/tools/final_check.py` verifies the environment before the run (masters, pdftotext, env variables, pass list).
- The FINAL data is read only as input (no labels exist for it); nothing is tuned on it.

## Background reading

Blum & Hardt, *The Ladder: A Reliable Leaderboard for Machine Learning Competitions* (2015); Dwork et al.,
*Generalization in Adaptive Data Analysis and Holdout Reuse* (2015); Roelofs et al., *A Meta-Analysis of Overfitting in
Machine Learning* (NeurIPS 2019); McNemar's test and the sign test for paired binary outcomes.
