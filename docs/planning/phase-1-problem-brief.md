# Phase 1: Problem Brief

**Project:** train a small open model (about 0.5 to 1.5 billion parameters)
to read Indonesian money messages, and run it on Arthur's own server. It is
Project 9's fallback for messages the rules parser can't settle, so no
financial message leaves the server. Training is a LoRA fine-tune with MLX
on Arthur's Mac, at $0. It is the fourteenth portfolio piece.

The repo is `small-model-training`. A second, later slice reuses the same
setup to shrink Juku's chat (ADR-012). The name says what the repo does for
both.

## Problem

**For Project 9's users:**
- **The fallback is off, and the reasons are money and privacy.** Project 9's
  ADR-007 keeps the model fallback off in v1: the balance is $0, the
  provider's data terms couldn't be found, and Meta's terms forbid letting
  WhatsApp data train a provider's models. A model on Arthur's own server
  removes all three.
- **Asking costs a tap; a wrong amount costs trust.** The rules ask whenever
  a message has two possible amounts ("beli iphone 15 pro 17jt"). A model
  that points at the right one turns a "which amount?" question into a
  one-tap **Ya**.

**For the portfolio:**
- Every earlier project calls someone else's model. None trains one, and
  none measures what training bought.
- A reviewer can't yet see how Arthur handles training data: where it came
  from, how much is synthetic, and how the test set was kept clean.
- Most fine-tuning demos report a score. This one has a hard safety rule a
  reviewer can check: **zero wrong amounts**, and a rules engine that stays
  the safety net.

## What Project 9's numbers already say (read 2026-11-03)

Project 9's rules, on its 100-message held-out set (`docs/parser-accuracy.md`
on `feat/amount-parser`, commit `16a324b`): 0 wrong amounts, **coverage
96.5%** (82 of 85). The 3 misses are all spelled-out numbers:
"setengah juta", "lima puluh ribu", "satu juta dua ratus ribu".

**Under Project 9's own rule, a model can't recover those 3.** ADR-007 says
the model may only point at the words that hold the amount, and code reads
them with `parse_amount`. A probe of `parse_amount` at that commit
(2026-11-03) returned `None` for "setengah juta" and "lima puluh ribu": the
parser can't read the words even when the model finds them. So on Project 9's
held-out set, **the model's coverage gain is zero by construction**.

What the model *can* add, under the same rule:
- **Turn questions into one-tap confirmations.** "beli iphone 15 pro 17jt"
  asks today with candidates 15, 15,000 and 17,000,000. The model points at
  "17jt", which `parse_amount` reads as 17,000,000.
- **Fix direction and category.** The rules wrote "jual sepeda bekas 1,2jt"
  and "freelance desain dibayar 750rb" as spending, and put "martabak manis
  45k" in Lainnya.
- **Catch real messages the written corpus doesn't have.** 96.5% is on
  Claude-written text. Real messages will be messier, which is why this
  project waits for Project 9's real-data gate.

Spelled-out numbers are a rules job: a deterministic reader in Project 9's
parser, made by Project 9's session. Phase 5 ADR-003 settles how the gain is
measured instead.

## Users

| Party | Who | Wants |
|---|---|---|
| **Project 9's people** | Arthur and up to 4 invited people | Fewer questions, no wrong amounts, nothing sent off the server |
| **Arthur (builder)** | Trains and reviews | A setup he can rerun for Juku, at $0 |
| **Reviewer / hiring team** | AI and backend engineers | Honest data provenance, a clean test set, a measured gain and a hard safety gate |

## Positioning: what makes this one different

1. **The model points; it never computes.** It returns the words that hold
   the amount. Project 9's parser turns them into an integer, and a person
   confirms. Training can't teach it to invent a number, because a number
   it invents fails the parser.
2. **Zero wrong amounts is a hard gate, not a target.** One wrong amount on
   the held-out sets fails the model, whatever its other scores.
3. **The data's origin is on every line.** Each training example says where
   it came from. The report states the synthetic share.
4. **What is served is what was scored.** The quantised file that runs on
   the server is the one the held-out sets measured, by its hash.
5. **No message leaves the server.** Training is on the Mac, and serving is a
   container with no internet access.

## Goals and how they're measured

| # | Goal | Target | Measured by |
|---|---|---|---|
| G1 | No wrong amount is proposed | **0** on every held-out set (hard gate) | Held-out scoring, on the served file |
| G2 | The model settles what the rules can't | Resolved rate on the hard-tail set (ADR-003); first guess ≥ 60%, set after the dev run | Held-out scoring |
| G3 | Rules plus model never does worse than rules alone | No metric worse on Project 9's held-out set | Paired comparison |
| G4 | It fits the shared server | Under 1 GB of memory; p95 latency under 5 s on the server's CPU (guesses, spike 3 measures) | Measured in a capped container |
| G5 | No message leaves the server | The model container has no route out; no message text in its logs | Network and log tests |
| G6 | The data is honest | Every example carries its source; synthetic share reported; Arthur's review sample graded | Data checks in CI; review sheet |
| G7 | It costs nothing | $0 for training, data and serving | No paid calls in any slice |
| G8 | The setup is reusable | Juku's slice runs on it without changes to the training or scoring core | ADR-012, when its gates open |

## Non-goals

- **The model supplying a number.** It points at text, always.
- **Replacing the rules.** The rules run first, and stay the safety net.
- **A general Indonesian assistant.** One narrow task.
- **Reading spelled-out numbers by model.** That's a rules change in
  Project 9 (ADR-003).
- **Training on anyone's real messages without consent.** Only Arthur's own,
  and only after he rewrites them (ADR-006).
- **GPU servers, cloud training or paid APIs.**
- **Combining the finance model and Juku's model.** Separate slices,
  separate models.

## Cost of doing nothing

Project 9 keeps its fallback off, and the rules' questions stay as they are.
That is safe. The portfolio keeps lacking any project that trains a model,
which is the gap reviewers in AI roles look for first. The project waits its
turn behind Project 9's real-data gate either way.
