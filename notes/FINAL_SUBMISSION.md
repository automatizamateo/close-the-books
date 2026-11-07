# FINAL submission record

## What was submitted

| Item | Value |
|---|---|
| File | `v8c_pf130` CSV (`sub_final_v8c_pf130.csv.gz`), not stored in this repo |
| shasum (SHA-1) | `eac9f95b86b4b8024c10f8870016eebc78a7aa3f` |
| Code | team repo, branch `model/v8b-pf`, commit `700bf5f` (https://github.com/jorgecarnicero/talky-x-claude) |
| Code in this repo? | **No.** It was built by a teammate and is not copied here. |
| FINAL score | **94.24** (platform, after the reveal) |
| LIVE score of the method | 94.67 (v8b-pf, same line without the ZA01 pattern); gap about -0.4 |

## Validation of the file (before upload)

`solution/tools/validate.py <file> --phase final`:

- 820,974 rows, 220,256 / 220,256 bank movements answered, 0 missing;
- errors: none; "OK to upload";
- row kinds: 798,333 item rows plus 22,641 category rows (`BANK_FEES` 10,043, `UNIDENTIFIED` 6,633, `CASH_POOL` 1,552,
  `PAYROLL` 1,387, ...).

I did **not** run the allocated-sum-versus-amount check or a row-by-row comparison against another candidate: the
candidate I wanted to compare against never finished (see below).

## How it was produced

Per the teammate's launcher `RUN_FINAL_v8c.sh`:

- Environment: `AR_DUST=1 EXC_R5=1 PAYER_DIR=1 AR_GUESS=1` (`PYTHONHASHSEED` is not pinned in that script).
- Passes, in order: `batches, remittance, ap_match, nonitem, ar_multi, ar_pdf, v3import_strong, ar_match,
  ar_multi_late, ar_bigpool, ar_dispute, ar_guess, v3import, v3rest, exc_fix, cancel_guard`.
- Then `postfix.py --phase final --win 130`: rewrites receipts answered `UNIDENTIFIED` with a best-guess invoice
  subset of an identified payer, using invoices issued in the previous 130 days.

## Pieces combined and their evidence on LIVE

All numbers are LIVE scores from the team log; none was tuned on FINAL.

| Piece | Why | LIVE evidence |
|---|---|---|
| Cascade of precise passes over the v3 backbone | precision first, backbone fills the rest | v3 82.25 to v3.5 91.92 |
| `v3import_strong` before our matchers | v3's referenced / single-exact answers were better than ours on those segments | v7d 93.89 to v8b 94.58, **together with** `ar_guess` (the log does not separate them) |
| `ar_guess` calibrated best guess | an extra pair that is right with probability above F1/2 (about 0.47) raises F1 | see the previous row |
| `postfix` best guess on remaining `UNIDENTIFIED` | same argument | v8b 94.58 to v8b-pf 94.67; v7d 93.89 to v7d-pf 94.25 |
| ZA01 payer pattern (`EFT CREDIT <name>`) | new South African entity in FINAL: payer extraction 15 % to 99.6 % (as stated in the teammate's commit message; I did not re-measure it) | no LIVE evidence possible (ZA01 does not exist in LIVE) |

The last row is the one piece that rests on a measurement on FINAL **inputs** (the extraction rate), not on a score, and
that measurement is the teammate's.

## Known problems

- **About 766 false `DUPLICATE_PAYMENT` rows** on card settlements. Card order numbers repeat across entities in FINAL
  and the order cache was keyed by number only.
- **Variant C never finished.** I built `model/v9-final` (the submitted line plus `exc_overpay` and the FINAL
  guards) and ran it; I stopped it after `v3rest`, before `exc_fix`, `exc_overpay` and `cancel_guard`. No output, never
  scored. `exc_overpay` has LIVE evidence only on the older v7 line (93.60 with it, 93.42 without, about 94 movements
  moved from `UNIDENTIFIED` to `OVERPAYMENT`); its effect on the submitted line is unknown.
- The laptop slept twice during the C run, pausing the process.

## Anti-overfitting method

See [`ANTI_OVERFIT.md`](ANTI_OVERFIT.md). In short: dev decides, LIVE confirms, a sealed window is read once; gates G1-G7;
no LLM in the pipeline; one scored FINAL submission, validated before upload.
