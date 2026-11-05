# rupiah-reader

A small open model (about 0.5 to 1.5 billion parameters), fine-tuned with
LoRA on a Mac, that reads Indonesian money messages such as "beli iphone 15
pro 17jt". It is the planned local fallback for
[Project 9's finance tracker](https://github.com/NabilRobert/financed-the-tracker):
when the rules parser can't settle a message, the model points at the words
that hold the amount, the parser reads them, and a person confirms. It runs
on the tracker's own server, so no financial message leaves it, and it costs
$0.

## Status: planned, paused 2026-11-05

The plan is written and reviewed. No code exists yet.

**Why it's paused:** it waits on Project 9's real-data gate (real messages
used and the misses counted) and on its labelled corpus. Training on
written messages alone would measure the wrong thing.

**The first slice, when it starts:** train two candidate models on the Mac
from generated data plus Project 9's dev set, then score the served file
once on held-out messages the training never saw. The hard gate is **zero
wrong amounts**, checked by Project 9's rules parser. One wrong amount fails
the model. The gain is measured as rules questions turned into right
answers, and the rules stay the safety net. The report states the share of
synthetic data.

**Later, separately:** the same setup may distil Juku's chat model, once
Juku's own quality gates, the provider's terms and a budget allow it.

## Layout

| Path | What it holds |
|---|---|
| `docs/planning/phase-1-problem-brief.md` | The problem, goals and non-goals |
| `docs/planning/phase-2-requirements-constraints.md` | Requirements, threats and the design diagram |
| `docs/planning/phase-5-architecture-decisions.md` | Decisions, with the options rejected |
| `docs/planning/slice-plan.md` | Prerequisites, spikes, slices and costs |

There is no code yet, so there is no CI workflow and no `docs/guide.md`.
Both come with the first slice.
