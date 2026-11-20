# PLAN — payroll: a Memelang graduate-level demo

Build a complete small-business payroll CLI entirely in Memelang. This is a
beta demonstration program: it must look like what a CS graduate would
build solo for a real payroll need.

Language docs: /mnt/ssd/projects/meme/docs/WRITING_GUIDE.md and
docs/TUTORIAL.md (read both first). Compiler commands run via
`uv run --project /mnt/ssd/projects/meme meme ...` from this repo.

## Ground rules (non-negotiable)

- Work ONLY in this repo. NO git commits — leave changes for review.
- 100% Memelang for program logic; no other languages.
- All money in integer cents; floats only at print time via
  `format <x> with 2 decimals`.
- All persistence as JSON files in `data/` (created on demand) using the
  Optional record readers/writers; corrupt/missing file = clean Optional
  handling, never a traceback.
- Every construct must parse with `--provider heuristic`.
- Run all test suites in the FOREGROUND only.

## Domain and CLI (argv-routed thin main + logic modules; unknown command →
usage + exit 1; every error = one stderr line + exit 1)

    payroll add-employee <id> <name> <hourly-rate-cents> <filing-status>
                           # filing status: single | married
    payroll add-time <employee-id> <hours-x100>   # e.g. 3750 = 37.50h
    payroll run <yyyy-mm>          # pay every employee for the month's timecards
    payroll payslip <employee-id> <yyyy-mm>
    payroll history <employee-id>  # all past payslips, sorted by month

Payroll rules (tax brackets are marginal, applied per filing status):
- Regular hours: up to 160h/month at the hourly rate; hours beyond 160 are
  overtime at 1.5x.
- Pre-tax deduction: retirement contribution = 5% of gross (integer cents,
  rounded half-up on magnitude).
- Marginal tax brackets (annual-equivalent, applied to this month's taxable
  gross): single — 10% up to 250000 cents, 22% beyond; married — 10% up to
  500000 cents, 22% beyond.
- Net = gross + overtime premium − retirement − tax. Store every payslip
  with its line items (gross, overtime, retirement, tax, net) so payslips
  and history render from stored data.
- Duplicate `run` for the same month+employee is rejected; editing
  timecards after a run for that month is rejected.

Suggested data model: `Employee`, `Timecard`, `Payslip` records with JSON
persistence; `Map<Int, Employee>`; payslip rendering with `format ... with
2 decimals` for the dollar column only.

## Tests — tests/meme/*.meme, `meme test`

Cover at minimum: bracket math at exact boundaries (250000/500000 cents,
one cent past, well past); overtime at exactly 160h vs 160.01h; retirement
rounding half-up on magnitude; net = gross − deductions identity for a
random-ish batch (assert per-line); duplicate run rejection; timecard
edit-after-run rejection; unknown employee; corrupt data file tolerance.
Use fixed epochs/months — never the wall clock in assertions.

## Acceptance ("done means")

1. `uv run --project /mnt/ssd/projects/meme meme test` — 0 failed (python).
2. Same with `--target rust`, `cpp`, `go`, `c`, `ts` — ALL SIX green.
3. `meme build main.meme -o payroll_bin` — run an end-to-end demo (3
   employees incl. one overtime month, run payroll, print one payslip and
   a history) and include the transcript in your final summary.
4. README.md: CLI usage, tax rules table, data files, test instructions.
5. Final message: files, test count per target, demo transcript,
   deviations (if any) with reasons.
