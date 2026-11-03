# Agent-Hunter HW7: benchmark evidence

Execution evidence for the HW7 benchmark of **Agent-Hunter: Trial-then-Hire** (MIT AI Studio, Fall 2026;
Chris Ackerman and Nathalie Abou Harb). The claim: an agent that hires an invoice-extraction provider by
running a short trial with known answers and scoring it (Trial-then-Hire) picks a better provider than an
agent that picks by reading the providers' public profiles (agent cards).

Everything here is copied unchanged from the project's (private) code repo, except the redactions listed
under [What's left out](#whats-left-out). The test plan in `PREREG.md` was merged before the first real run.

## Files

| Path | What it is |
| -- | -- |
| `PREREG.md` | The pre-registered test plan: claim, cases, metric, pass/fail rules, settings. Merged before any run. |
| `jobs/manifest.json` | The 12 benchmark jobs: type, split (tuning j01–j08, fresh j09–j12), source, generator seeds, held-out invoice IDs. |
| `jobs/jNN.json` | Each job posting as the requester sends it. |
| `jobs/samples/jNN/sample_{1,2,3}.pdf` | The 3 sample invoices attached to each posting. |
| `rubrics/jNN.json` | The rubric built once per job (field weights, tolerances, required fields, consistency checks), reused by every run of that job. |
| `runs/<run_id>/summary.json` | Per run: code commit, providers commit, settings, wall clock, spend (provider vs. platform), call counts, failures by job type, errors. |
| `runs/<run_id>/submissions.jsonl` | One row per (provider, invoice): job, repeat, arm (`trial` or `heldout`), status, extracted JSON (`output`), doc score, per-field scores, sum check, latency, cost. |
| `runs/<run_id>/picks.jsonl` | One row per pick: method `tth` (Trial-then-Hire, per repeat), `profile` (AI reads the agent cards, 5 runs) or `cheapest`; picked provider, tie status, reason, selection cost and time, per-step timings. |
| `runs/<run_id>/llm_calls.jsonl` | Every platform model call of the run: component, model, tokens, cost, job ID. |
| `runs/<run_id>/trials/jNN-rR.json` | The trial for job NN, repeat R: its 12 invoice IDs and where each came from (sample or synthetic). No answer keys. |
| `results/findings.md` | **Start here:** the headline result, the failure we found, and the change we propose. |
| `results/change-on/` | The same tables and charts with the AGE-39 change on (accuracy-first tie-break), recomputed from the same runs: `analyze --runs tuning-r1 fresh-r1 --change on`. |
| `results/main_table.md` / `.csv` | The main table: mean selection quality (SQ) with a 95% CI, hit rate, bad picks, cost per correct invoice and time to hire for Trial-then-Hire (6 invoices), profile-reading, cheapest and random, plus the PREREG §7 verdict. |
| `results/report.md` | Every table in one page: trial-size sweep, per job, by job type, real vs synthetic, run-to-run noise, ground truth, exploratory diagnostics and checks. |
| `results/*.csv` | Machine-readable versions: `sweep`, `ground_truth`, `per_job`, `noise`. |
| `results/sq_vs_trial_size.png`, `results/tie_rate_vs_trial_size.png` | The two charts. |

### Runs

| Run | What | Notes |
| -- | -- | -- |
| `j01-check` | Harness check, job j01 only, 1 repeat. | Not part of the analysis; held-out answers from it are reused (`cached_from`) by later runs. |
| `tuning-r1` | **Tuning round:** jobs j01–j08, Trial-then-Hire 2 repeats (r1, r2), held-out pile once, profile pick 5×, cheapest pick. | 8/8 jobs done, no errors. $6.50 total ($6.49 providers, $0.011 platform), 1,932 provider calls, 27.6 min. |
| `fresh-r1` | **Fresh round:** jobs j09–j12, same design as `tuning-r1`, code with the AGE-39 change merged (off by default; it only changes selection, so `analyze --change on` re-ranks these same logs). | 4/4 jobs done, no errors. $3.12 total ($3.11 providers, $0.006 platform), 1,008 provider calls, 20.0 min. |

`tuning-r1` notes: the first start stopped after 2 platform calls ($0.0002) because the PDF renderer
couldn't find its system libraries; it was resumed under the same run ID, so `llm_calls.jsonl` holds those 2
calls too; `wall_clock_s` (1,657 s) is the resumed part, `wall_clock_s_total` (1,663 s) includes the first start. Platform calls ran on Chris's Parley account
(`--platform-account chris`) instead of Nathalie's; the account only affects billing.

`fresh-r1` notes: j09's first try at repeat r1 ran into network trouble (85 timeouts). Those 84 rows and
the summaries of the stopped tries are kept in `runs/fresh-r1/discarded/`, and the 84 (provider, invoice)
pairs were sent again. The resend was blocked by Parley's 402 `INSUFFICIENT_TOKENS` (Chris's account had
used up its monthly free-model token allowance), so its last 63 calls ran with the providers service on
Nathalie's Parley account. Code and providers commit are unchanged; the account only affects billing.
`wall_clock_s_total` (1,198 s) adds up all tries, including the stopped ones.

## What's left out

- **Real invoices (DocILE, research-use licence), jobs j02 and j09:** their postings and sample PDFs are
  left out, and the `output` field of their rows in `submissions.jsonl` is set to `null`. Their IDs, scores,
  field scores, cost and latency are kept. The DocILE docs are chosen by a seeded rule in `PREREG.md` §2.
- **Held-out and trial invoice PDFs and answer keys:** they regenerate exactly from the seeds in
  `jobs/manifest.json` and the code repo.
- **The providers' code** and all keys.

## How to reproduce

You need access to the code repo (`agent-hunter`) and the providers service. From the `agent-hunter` root:

```bash
uv sync && cp .env.example .env     # fill in PARLEY_API_KEY, PARLEY_ACCOUNT
uv run python -m experiments.hw7.make_jobs          # postings, samples, held-out piles (DocILE in data/docile/)

# providers service, at the commit the agent cards are frozen on
cd ../agent-hunter-providers && git checkout 5d9536a && uv run uvicorn service:app --port 8100

# on Apple Silicon the PDF renderer needs:  export DYLD_FALLBACK_LIBRARY_PATH=/opt/homebrew/lib
uv run python -m experiments.hw7.run_benchmark --mock --only j01,j02 --repeat 2   # dry run, no cost
uv run python -m experiments.hw7.run_benchmark --only j01 --repeat 1 --run-id j01-check
uv run python -m experiments.hw7.run_benchmark --jobs tuning --repeat 2 --max-spend-usd 12 --run-id tuning-r1
```

Rerunning with the same `--run-id` resumes: (provider, invoice) pairs and picks already logged are
skipped. Model outputs aren't deterministic, so a rerun gives new rows, not identical ones.

The analysis makes no model or provider calls. It reads the logs only, so it gives exactly the same
`results/` every time (seeded bootstraps and draws):

```bash
uv run python -m experiments.hw7.analyze --runs tuning-r1 fresh-r1
# or against this repo's copy of the logs:
uv run python -m experiments.hw7.analyze --runs tuning-r1 fresh-r1 \
    --runs-dir ../agent-hunter-hw7/runs --manifest ../agent-hunter-hw7/jobs/manifest.json --out /tmp/results
```
