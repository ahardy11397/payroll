# payroll — a small-business payroll CLI written in Memelang

A complete small-business payroll program, 100% Memelang (Structured
English, compiled by the offline `heuristic` parser). It manages employees,
monthly timecards, and payslips, and emits identical behavior on all six
backends: **Python, Rust, C++, Go, C, and TypeScript**.

Written as a beta demonstration: what a CS graduate would build solo for a
real payroll need — typed records, modules, `Optional<T>` error channels,
JSON persistence, deterministic calendar math, and a cross-target test
suite.

## Source layout

| File | Role |
|---|---|
| `main.meme` | argv router only (loads `payroll_model` + `payroll_logic` modules) |
| `payroll_model.meme` | domain: `Employee`/`Timecard`/`Payslip` records, JSON loaders/savers, lookup, `apply_add_time` mutator (explicit element-wise copy, value semantics on every backend) |
| `payroll_logic.meme` | pay math (overtime, retirement, marginal tax brackets), payslip building + rendering, validate functions, UTC calendar conversion, exact money/hours text |
| `tests/meme/*.meme` | 51 test cases (`meme test` discovers them) |
| `tests/meme/payroll_model.meme`, `tests/meme/payroll_logic.meme` | symlinks to the root modules (modules resolve relative to the importing file's directory) |

## CLI

```
payroll add-employee <id> <name> <hourly-rate-cents> <single|married>
payroll add-time <employee-id> <hours-x100> [yyyy-mm]
payroll run <yyyy-mm>
payroll payslip <employee-id> <yyyy-mm>
payroll history <employee-id>
```

- `<hours-x100>` is hours in hundredths: `3750` = 37.50h. Repeated
  `add-time` for the same employee and month **accumulates** hours.
  `[yyyy-mm]` is optional and defaults to the current UTC month; pass an
  explicit month to record time for any month (past months are the normal
  payroll workflow).
- `run <yyyy-mm>` pays every employee who has time recorded for the month
  and stores one payslip per employee with all line items. Running a month
  twice is rejected, and after a run the month's timecards are **locked**
  (further `add-time` for that month is rejected).
- `payslip` renders one stored payslip; `history` renders all of an
  employee's payslips sorted by month.
- `add-employee` requires a unique id, a positive integer hourly rate in
  cents **capped at 100000000** ($1,000,000.00/h), and filing status
  `single` or `married`. `add-time` requires a positive `hours-x100`
  **capped at 74400** (744.00h = every hour of a 31-day month), applied to
  the accumulated monthly total; the caps keep every money product inside
  the 64-bit integer range identically on all six backends.

Validation failures print one clear line to **stderr** and exit with code
`1`. Unknown commands print the usage text to stderr and exit `1`. Reports
go to stdout.

## Payroll rules

All money is **integer cents**; hours are hundredths of an hour. Fractional
cent results round **half-up on magnitude**. No binary floats touch the
money math (see design notes).

| Line item | Rule |
|---|---|
| Regular hours | up to 160.00h/month at the hourly rate |
| Overtime | hours beyond 160.00h at 1.5x = straight pay + half-rate premium |
| Gross | every recorded hour at the straight rate |
| Overtime premium | extra half rate on overtime hours, computed as half of the *rounded* straight pay of the overtime hours (differs from rounding the exact half by at most 1 cent, in the employee's favor) |
| Retirement | 5% of gross, half-up |
| Taxable gross | gross + overtime premium − retirement |
| Tax | marginal brackets on taxable gross (table below) |
| Net | gross + overtime premium − retirement − tax |

Marginal tax brackets (annual-equivalent thresholds applied to this
month's taxable gross):

| Filing status | 10% up to | 22% beyond |
|---|---|---|
| single | 250000 cents ($2,500.00) | 250000 cents |
| married | 500000 cents ($5,000.00) | 500000 cents |

Example: 160.01h at $25.00/h, single — gross $4000.25, premium $0.13,
retirement $200.01, tax $536.08, net $3264.29.

## Data files

All persisted state is JSON under `data/` (created on demand, gitignored):

- `data/employees.json` — `List<Employee>` (`id`, `name`, `rate_cents`,
  `filing_status`)
- `data/timecards.json` — `List<Timecard>` (`employee_id`, `month`,
  `hours_x100`), one accumulated timecard per employee per month
- `data/payslips.json` — append-only `List<Payslip>`; each payslip stores
  the full line items (`gross_cents`, `overtime_premium_cents`,
  `retirement_cents`, `tax_cents`, `net_cents`) plus an employee snapshot
  (`name`, `filing_status`, `rate_cents`), so payslips and history render
  from stored data alone

A missing file is the normal first-run state and degrades to empty state
through the Optional JSON readers. A file that **exists** but is corrupt,
wrong-shaped, or holds any record missing a required field fails loudly
with one stderr line and exit 1 (see design notes) — it is never silently
treated as empty state, and never a traceback.

## Running

The compiler lives at `/mnt/ssd/projects/meme` (run everything from this
repo; `--provider heuristic` is the only parser used):

```bash
# compile to Python and run the CLI
uv run --project /mnt/ssd/projects/meme meme compile main.meme \
    --provider heuristic --target python -o payroll.py
python3 payroll.py add-employee e1 "Ada Lovelace" 2500 single

# native binary (C11 backend)
uv run --project /mnt/ssd/projects/meme meme build main.meme -o payroll_bin \
    --provider heuristic
./payroll_bin run 2026-11

# other targets: --target rust|cpp|go|c|ts
```

## Tests

```bash
uv run --project /mnt/ssd/projects/meme meme test              # python (default)
uv run --project /mnt/ssd/projects/meme meme test --target rust
uv run --project /mnt/ssd/projects/meme meme test --target cpp
uv run --project /mnt/ssd/projects/meme meme test --target go
uv run --project /mnt/ssd/projects/meme meme test --target c
uv run --project /mnt/ssd/projects/meme meme test --target ts
```

58 cases across five files (all six targets green):

- `01_model_test.meme` — 16 cases — record JSON roundtrips (all fields),
  missing/corrupt/partial-record/wrong-shape file tolerance, lookup helpers,
  `apply_add_time` accumulation + value semantics, strict month validation,
  rate-cap and corrupt-data-file gates
- `02_pay_math_test.meme` — 8 cases — overtime split exactly at/just past
  160h, half-up rounding on fractional cents (straight pay, overtime
  premium, retirement, tax rates)
- `03_tax_test.meme` — 10 cases — bracket boundaries at 250000/500000
  cents exact, one cent past, well past; both filing statuses
- `04_payslip_test.meme` — 12 cases — full payslip math at exactly 160h vs
  160.01h, a married 320h overtime month, zero hours, the net identity
  asserted per line over a random-ish 35-cell batch, stored snapshot
  fields, `validate_add_time` rejections (unknown employee, bad hours,
  bad month, locked month, hours cap, accumulated-hours cap)
- `05_operations_test.meme` — 12 cases — duplicate-run rejection (same and
  other employee), no-timecard month, ghost-employee timecard, timecard
  lock, history month sorting, full two-month flow, exact money/hours
  text, fixed-epoch month conversion

Notes:

- Tests write scratch JSON with per-test unique filenames into the
  compiler-managed test working directory, which is deleted automatically
  after each run.
- Every test file parses with the offline `heuristic` provider only.
- Test assertions use fixed epochs and month strings only — the wall clock
  is never in an assertion (`epoch_to_month_text` itself is tested with
  fixed epoch constants).
- Generated `.meme.lock` sidecars and `build.lock` are gitignored build
  artifacts; they are safe to delete and regenerate. If a root `build.lock`
  exists, delete it before running `meme test` (the test runner can pick it
  up as a lockfile cache).

## Design notes

- **Error channel:** all operation validation returns `Optional<String>`
  (`nothing` = ok, `some` = error message). `main.meme` prints the message
  to stderr and exits 1. The language has no stderr print, so error lines
  are written with `append ... to file "/dev/stderr"` (produces real
  stderr bytes with identical exit codes on all six backends).
- **Value semantics:** mutators (`apply_add_time`) return a *new* list
  built with an explicit element-wise copy — the plain `define list out as
  src` copy idiom aliases backing storage on the Go, C, and TypeScript
  backends — so the caller's list is never mutated and all six backends
  behave identically.
- **Integer money only:** cents are `Int` everywhere; the only fractional
  step (hours_x100 × rate_cents ÷ 100) rounds half-up via `(x + 50) / 100`
  integer division. The dollar column is rendered by exact integer
  `dollars_text` (1234 → "12.34"), hours by `hours_text` (16001 →
  "160.01"). Memelang has no `Int`→`Float` conversion (arithmetic is
  same-type only and `format <f> with N decimals` requires a `Float`
  target), so `format ... with 2 decimals` appears only in a documented
  shim (`dollars_float_preview`) that keeps the runtime float helper linked
  for the JSON serializer; it never participates in money math.
- **Determinism:** history sorts by month string (`yyyy-mm` sorts
  lexicographically = chronologically); the only wall-clock use is
  `add-time`'s default month, computed by walking-calendar UTC conversion
  from `current epoch seconds` — never in assertions.
- **Data-integrity gate:** at startup the program checks each data file
  with a raw text read plus the typed record reader; a file that exists
  but cannot be parsed as its record list (including a valid JSON array
  holding any record with a missing required field) fails with one stderr
  line and exit 1 instead of degrading to empty state — silently
  continuing would wipe stored payslips on the next save and defeat the
  duplicate-run lock.
- **No cross-process locking:** concurrent CLI invocations race on the
  JSON files (last save wins). Single-user small-business scope: run one
  command at a time.
- **Append-only payslips:** `run` checks the stored payslip list before
  computing, so a month can never be paid twice and a paid month's
  timecards are locked; partial failures cannot double-pay because the
  whole run is validated before any payslip is written.
