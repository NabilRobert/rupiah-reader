# Slice Plan and Cost Model

## Prerequisites (outside this repo)

| # | Prerequisite | Needed for | Who |
|---|---|---|---|
| P1 | Arthur OKs Phases 1, 2 and 5 | Everything | Arthur |
| P2 | ~~Arthur OKs mlx-lm and llama.cpp~~ Done 2026-11-03 | Spikes, slice 1 | Arthur |
| P3 | Downloading the two finalists, about 2 GB each: agreed for when the spikes run, after P1 | Spikes 1 and 2 | Arthur |
| P4 | ~~Project 9's parser merged, so this repo pins it by commit~~ Done 2026-11-03: pin `ab169ba` (parser, spelled numbers, `tracker.corpus`) | Slice 1 | Project 9's session, Arthur merges |
| P5 | ~~Project 9's card-number filter (PR3) merged~~ Done: `looks_sensitive` is in `ab169ba` | Slice 1's data checks | Project 9's session |
| P6 | **Project 9's real-data gate:** its slice 5 dogfood done, misses counted | Starting slice 1 (given) | Arthur, Project 9's session |
| P7 | A GitHub repo for this project | Pushing anything | Arthur |
| P8 | Project 9 deployed, and the server checked for room | Slice 2 | Project 9's session, Servers & security session |
| P9 | Juku's Q1 and MEM1 at 90% or more; provider terms read; a budget | Slice 4 | Arthur, Juku's session |

## Spikes (local, $0, each after P2 and P3)

They answer planning questions, so they may run before P6.

| # | Question | Method | Pass | Time |
|---|---|---|---|---|
| 1 | Does the toolchain work for each finalist? | 100 lines, 50 LoRA steps in mlx-lm, fuse, convert with llama.cpp, Q8_0, ask `llama-server` for schema-constrained JSON | Both finalists go end to end, or a failing one is swapped for Qwen3-0.6B | 1 h each |
| 2 | Memory and time on the M2 | Peak memory and tokens per second from mlx-lm's output, at batch 4 and 256 tokens | A full run fits under 12 GB and an evening; ADR-004's guesses replaced | 30 min |
| 3 | Does serving fit, and is it quiet? | `llama-server` in Docker on the Mac, `--cpus=1`, 1 GB cap: latency on 200 messages; what each log level prints; `/slots` and the web UI off; `response_format` | p95 under 5 s as a proxy; no message text in any log; settings confirmed | 1 h |
| 4 | Does greedy output agree across machines? | 200 messages through Metal and a CPU-only build (`--device none`) | Identical answers, or the differences listed and scored | 30 min |

Spike 3's real numbers come from the server itself, in slice 2.

## Slices

**Evaluation-driven order, as in Project 9:** the checks, held-out sets and
scorer come first. Each new test is mutation-checked once. Each slice
updates `docs/guide.md`.

| Slice | Delivers | Done when | Cost |
|---|---|---|---|
| **1. The finance model on the Mac** | Scaffold, CI, data checks, generator, T2, Inspect task and scorer, bake-off, served file, report | See below | $0 |
| **2. Serve it as Project 9's fallback** | Container file; hardening tests (no route out, no prompts in logs, hash check); latency on the server; hand-over to Project 9's session, which flips its setting; T3 scored once Arthur has rewritten his misses | G4 and G5 measured on the server; Project 9's combined system passes G3; T3 reported | $0 |
| **3. Evidence and write-up** | Results from run records; JOURNEY; portfolio entry (with the portfolio session); publishing if Arthur agrees | Every number traces to a run record; a privacy check passes | $0 |
| **4. Juku's chat distillation (separate)** | Shiori's answers collected; `tools`-format data; training on the same core; scoring through Juku's runner | All three gates of ADR-012 met first; then a measured comparison with Shiori | Collecting answers is paid; budget not set |

## Slice 1 in detail: the first slice

**Scope:** the Mac only. No server, no Project 9 change, no paid call.

**Delivers:**
- `pyproject.toml` (uv), ruff with the complexity rules, pyright strict;
  `.github/workflows/ci.yml`; a README with a layout section;
  `docs/guide.md`;
- `core/`: the provenance model, data checks (D3, D4), the leak check (E2),
  run records, the served-file build, the report generator;
- `tasks/finance/`: word lists, the seeded generator, the output schema, the
  scorer (Project 9's parser plus exact match);
- `evals/finance.py`: the Inspect AI task;
- `configs/`: one YAML per training run;
- `data/holdout/t2.jsonl`, frozen and hashed; T1 and the dev set read from
  Project 9's `tracker.corpus` at `ab169ba`, with the dev texts that are also
  in T1 dropped;
- `docs/model-report.md`, generated.

**Order:**
1. Scaffold. Write the data checks and the leak check, with failing tests.
2. Write the word lists. **Arthur reviews them** (about 30 minutes).
3. Write the generator and the dev split. Check every line.
4. Have T2 written by a separate agent, filtered by the rules. **Arthur
   reviews 100 lines** (about 45 minutes). Freeze and hash it.
5. Write the Inspect task and scorer. Score three baselines on the dev
   split: rules alone, and each finalist **untrained**. Training has to beat
   those, and the report shows it.
6. Train both finalists with one config. Pick on the dev split (ADR-005).
7. **Arthur reviews 100 generated lines** (about 45 minutes). Over 5
   unnatural lines: fix the lists, retrain, redraw.
8. Build the served file. Score T1 and T2 **once**, on its hash.
9. Generate the report. Then a pass trying to break it: prompts inside
   messages, 4,096 characters, emoji only, digits outside 0 to 9, two real
   amounts, an amount glued to a word.

**Done when:**
- wrong amounts: **0** on T1 and T2 (hard gate, ADR-008);
- resolved rate on T2 measured, and G2's 60% guess confirmed or changed, in
  writing;
- the combined system on T1 is no worse than rules alone on any measure;
- the synthetic share and Arthur's review counts are in the report;
- CI is green, and the new tests are mutation-checked.

**Branches:** `feat/data-checks`, `feat/finance-generator`,
`eval/finance-holdout`, `feat/finance-training`, from `main` once
`docs/planning` is merged.

**Arthur's time:** about 2 hours in all, in three sittings.

## Cost model

| Item | Cost |
|---|---|
| Training on the Mac | $0 (electricity only) |
| Data: generator, Project 9's dev set, Arthur's rewrites | $0 |
| T2, written by an agent in a Claude Code session, as Project 9's corpus was | $0 extra |
| Scoring with Inspect AI against a local server | $0 |
| Serving on the existing server | $0 |
| Disk on the Mac | About 10 GB: two base models, fused copies, GGUF files (34 GB free) |
| Juku's slice | Not budgeted; needs Arthur's OK |

**Nothing in slices 1 to 3 costs money.** For Project 9, the model fallback
would go from "about $0.003 to $0.017 a month, not funded" (its ADR-012) to
$0, with no provider in the path.
