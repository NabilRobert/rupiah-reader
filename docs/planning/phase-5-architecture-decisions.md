# Phase 5: Architecture Decisions

Short ADRs, 2026-11-03. **Given** means it came in Arthur's brief. **Settled**
means decided with Arthur in this planning pass. The rest are
recommendations, open until he agrees. The summary table, the first slice
and the open questions are at the end.

```
Arthur's Mac (offline, $0)                         Shared server (no route out)
──────────────────────────                         ────────────────────────────
generator + P9 dev set + Arthur's rewrites         Project 9 tracker
        │  label checks (P9 parser, PR3, leaks)        │ rules couldn't settle it
        ▼                                              ▼
mlx_lm.lora ─► adapter ─► fuse ─► GGUF ─► hash ──► llama-server (same file)
                                    │                  │ {"amount_text": ...}
                     llama-server (local)              ▼
                                    │            P9 parser ─► person taps Ya ─► ledger
             Inspect AI task over T1, T2, T3
                                    │
                             model report
```

## Reuse map (the folder's "reuse before building" rule)

Arthur's repos were searched first (`grep` for mlx-lm, lora, fine-tune,
gguf, llama.cpp and llama-server across the folder, 2026-11-03). The only
hits are planning prose and Arthur's checkout of Inspect AI's own tests. No
repo trains or serves a model, so the training side has nothing to depend
on. The task side does.

| Part | Reused | Built (and why) |
|---|---|---|
| Reading an amount | **Project 9's `parse_amount`**, as a git dependency pinned to a commit | — |
| Labelling rules, categories, corpus format | **Project 9's corpus README and dev set** | The `amount_text` span, derived by code (ADR-006) |
| The "model points, code decides" contract | **Project 9's ADR-007** (from Pitstop, via Juku) | — |
| Card-number filter | **Project 9's PR3** check, when it lands | — |
| LoRA training | **mlx-lm** (ADR-004) | A YAML config per run |
| GGUF conversion, quantising, serving | **llama.cpp** (ADR-009) | A container file |
| Held-out scoring | **Inspect AI**, Arthur's eval standard, through its OpenAI-compatible provider (ADR-011) | One task and one scorer |
| Juku's slice scoring | **Juku's `run_suite.py`** and **agent-eval-harness** graders (ADR-012) | A model setting |
| Traces, alerts at deploy | **agent-observability**, **systems-monitoring**, as Project 9 already plans | Config entries in their repos |
| CLI, validation, tests | Typer, Pydantic, pytest, uv, ruff, pyright | — |
| The message generator | No open dataset of informal Indonesian money messages ("25rb", "goceng") was found (ADR-006) | **Built:** labels exact by construction, and it is what makes the data story honest |
| The model report | Hugging Face's model card layout | The generator that fills it from run records |

## ADR-001: One narrow task first, a reusable core, Juku later (given)

**Decision:**
- Slice 1 trains one model for one task: reading Project 9's messages.
- The training, served-file build, scoring and report code are task-blind,
  so Juku's slice reuses them (ADR-011).
- Juku's chat distillation is a separate, later slice with its own gates
  (ADR-012). The two models are never combined.
- The build starts after Project 9's real-data gate.

**Why:** the finance task has exact labels and a hard rule, so it's the
right place to prove the setup. Juku's task has fuzzy answers and tool
calls, which would hide setup bugs.

**Rejected:**
- *Both tasks in one model:* two failure modes in one score, and Juku's data
  isn't allowed yet.
- *Juku first:* its gates are closed (89% and 77% against 90%).

## ADR-002: The model points at text; it never supplies a number (settled)

**Decision:** the output is `{"amount_text", "direction", "category"}`.
- `amount_text` is an exact substring of the message, or `null`.
- Project 9's parser turns it into rupiah. A person confirms.
- No description: Project 9's code already writes it from the message.
- Constrained decoding (JSON schema) plus Pydantic validation.

**Why:** this is Project 9's ADR-007, unchanged. It means a training bug, a
bad example or a prompt in a message can't produce an amount the parser
didn't read from the text.

**Rejected:**
- *The model outputs the integer:* one bad number breaks Project 9's central
  claim.
- *The model normalises spelled-out words to digits:* a number the model
  wrote, by another name.

## ADR-003: The gain is settled questions and right labels, not raw coverage (settled)

**Finding (2026-11-03):** Project 9's 3 held-out misses are spelled-out
numbers. `parse_amount` returns `None` for "setengah juta" and "lima puluh
ribu", so a pointing model can't recover them. Coverage on Project 9's
held-out set can't rise. See Phase 1.

**Decision:** measure what the model can change, on the rules' failures:

| Measure | Definition | Gate |
|---|---|---|
| **Wrong amount** | A proposal whose `amount_text`, read by the parser, differs from the label | **0 (hard)** |
| **Resolved** | Of messages labelled `entry` that the rules missed or asked about, the share where the proposal gives the right amount | G2, set after the dev run |
| **Right abstain** | Of messages labelled `ask` or `unparsed`, the share where the model returns `null` or a span the parser still finds ambiguous | Reported |
| **Direction / category** | Exact match on `entry` messages | Reported, against the rules |
| **Combined system** | Rules first, model on their failures, on T1 | No measure worse than rules alone (G3) |
| **Latency** | p50 and p95 per message on the server's CPU, in a capped container | G4 |

**And a separate change for Project 9's session:** a deterministic reader for
spelled-out numbers ("setengah juta", "lima puluh ribu") in the rules. That
is where those messages belong. It's reported to Project 9, not built here.

**Rejected:**
- *Report coverage alone:* it would show zero gain on T1 and hide the real
  value, or tempt a model that writes numbers.
- *A model-graded score:* the scorer is code (E5).

## ADR-004: mlx-lm for LoRA training on the Mac (settled)

**Decision:** `mlx-lm` 0.32.x (MIT, 0.32.2 released 2026-10-16; two PyPI
maintainers, Apple's MLX team), LoRA with `--mask-prompt`, from a committed
YAML config. Data in its `chat` format.

**What its docs say** ([LORA.md](https://github.com/ml-explore/mlx-lm/blob/main/mlx_lm/LORA.md)):
- fine-tune types `lora` (default), `dora` and `full`; a quantised base
  model switches to QLoRA;
- formats `chat`, `tools`, `completions` and `text`; unknown keys are
  ignored, so the `provenance` field is safe;
- `--mask-prompt` trains on the answer only (`chat` and `completions`);
- memory levers: `--batch-size`, `--num-layers`, `--grad-checkpoint`;
- `mlx_lm.fuse` merges the adapter, but `--export-gguf` covers only Llama,
  Mistral and Mixtral. So conversion uses llama.cpp (ADR-009).

**Memory and time on the M2 (16 GB): estimates, spike 2 measures.** A 1B
model in bf16 is about 2 GB of weights. LoRA at batch 4 and 256 tokens should
peak well under 8 GB. About 5,000 examples of about 120 tokens, for 3 epochs,
is under 2 million tokens: a guess of 15 to 45 minutes. No number here is
used until spike 2 replaces it.

**Rejected:**

| Option | Why not |
|---|---|
| Hugging Face TRL + PEFT on PyTorch MPS | Works on a Mac, but heavier to install, and not measured here; mlx-lm is built for Apple silicon |
| Unsloth, Axolotl | Built around NVIDIA GPUs; Arthur has none |
| A cloud GPU (Colab, rented) | Money or free-tier limits, and training data leaves the Mac |
| A provider's fine-tuning API | Money, data leaves, and the model can't run on the server |
| Full fine-tuning | More memory for no expected gain on a narrow task; the adapter is a few MB |
| Plain PyTorch loop | Hand-rolling what mlx-lm does |

## ADR-005: Base model: a two-model bake-off on the dev split

**Decision:** fine-tune both finalists with the same config and data, and
pick on the **dev split only**, by wrong amounts (must be 0), then resolved
rate, then server latency. Lock the choice before any held-out scoring.

| Model | Params (HF metadata) | Licence | Indonesian | Tooling | Verdict |
|---|---|---|---|---|---|
| **Sailor2-1B-Chat** (sail) | 0.99B | Apache-2.0; asks to cite the report | Yes: SEA-focused, Indonesian listed; continued pretraining on 500B tokens from Qwen2.5-0.5B | `Qwen2ForCausalLM`: in mlx-lm and llama.cpp's converter | **Finalist:** made for this region's languages |
| **Qwen3.5-0.8B** (Qwen) | 0.87B | Apache-2.0 | "201 languages" claimed, not listed on the card | `qwen3_5` in mlx-lm and llama.cpp; a vision-language hybrid (Gated DeltaNet), used text-only | **Finalist:** newest; risk that a new architecture trips the tools (spike 1) |
| Qwen3-0.6B / 1.7B | 0.75B / 2.0B | Apache-2.0 | Yes, listed in the Qwen3 blog | Mature | Reserve, if a finalist fails spike 1 |
| Qwen2.5-0.5B / 1.5B-Instruct | 0.49B / 1.54B | Apache-2.0 | Not listed | Mature | Rejected: Sailor2 is this, further trained for Indonesian |
| Llama-3.2-1B-Instruct | 1.24B | Llama 3.2 Community Licence | Not in its 8 listed languages | Mature | Rejected: a fine-tune's name must begin with "Llama", "Built with Llama" must be shown, and Indonesian isn't supported |
| Gemma 3 1B | 1.0B | Gemma Terms of Use | Multilingual | Mature | Rejected: custom terms whose use restrictions must be passed on, and Google may restrict use remotely |
| Gemma 4 E2B | 5.1B stored | Apache-2.0 | Multilingual | Supported | Rejected: too big for 2 vCPUs |
| LFM2-1.2B (Liquid AI) | 1.17B | LFM Open Licence v1.0: commercial use only under $10M revenue | Not in its list | Supported | Rejected: revenue-capped licence, no Indonesian |
| Granite-4.0-1B (IBM) | 1.63B | Apache-2.0 | Not in its list | Supported | Rejected: no Indonesian |
| SmolLM3-3B | 3.1B | Apache-2.0 | Not in its list | Supported | Rejected: too big, no Indonesian |

**Downloads:** about 2 GB per finalist. Each needs Arthur's OK.

## ADR-006: Data: mostly generated by code, labels checked by the parser (settled)

**Decision:**

| Source | Size (guess) | Synthetic? | Labels |
|---|---|---|---|
| **Generator** | 3,000 to 5,000 | Yes, by code | Exact by construction: it picks the amount, writes it in one of Project 9's forms, and records the span |
| **Project 9 dev set** | 205 | Yes, written by Claude | Project 9's labels; the span derived by code |
| **Arthur's rewrites** | Tens, after the real-data gate | No: written by Arthur from his own misses | Arthur's; checked by code |

- **The generator** composes messages from word lists: items, merchants,
  verbs, income words, slang amounts, typos, emoji, word orders, and
  distractor numbers ("iphone 15", "warung 88", "2 porsi"). Arthur reviews
  the lists, not every line. It also writes negatives the model must abstain
  on: plans, questions, balances, commands, two real amounts.
- **Every label passes D3:** the span is in the text and `parse_amount` reads
  the labelled amount from it. Lines that fail are dropped and counted.
- **Arthur's review:** 100 random generated lines, graded natural or not, and
  label right or not. Over 5 unnatural lines, the lists change and the
  sample is redrawn.
- **The report states:** the synthetic share (expected above 95% before T3),
  the generator version, and the review result.

**Why mostly code, not a model:** labels are right by construction; it costs
nothing; and it avoids one question. Anthropic's consumer terms (effective
2025-10-08) bar using the services "to develop any products or services that
compete with our Services, including to develop or train any artificial
intelligence or machine learning algorithms or models". The commercial terms
(2025-06-17) bar training "competing AI models". A narrow money-span reader
for a personal tracker doesn't compete with a general assistant, so the plan
treats Project 9's Claude-written dev set as usable. That's engineering
judgement, not legal advice, and the generator keeps the exposure small.
Arthur agreed on 2026-11-03: the dev set is used, and the report discloses
it.

**Rejected:**
- *Real WhatsApp messages from friends or public groups:* no consent; personal
  financial data under UU PDP.
- *Model-written training data in bulk:* labels need checking anyway, and the
  terms question grows with the share.
- *An open Indonesian dataset:* a search found none with informal money
  messages and exact amounts. The closest is IPerFEX-2023 (Journal of Big
  Data, 2024), personal-finance entity extraction with IndoBERT; its sample
  writes "Rp 1.453.000", and its data availability and licence couldn't be
  read (the publisher's page needs a login). Worth citing as related work;
  to check before any use.
- *Translating English finance messages:* the slang and forms are the point.

## ADR-007: Three held-out sets, written apart from the training data

**Decision:**
- **T1, Project 9's held-out set (100):** never trained on. Used for the
  combined-system comparison (G3). Training reads Project 9's `dev.jsonl`
  only. If Project 9 replaces its held-out set (for example after building
  the spelled-number reader, which burns the 3 misses already read), T1
  follows the new set. Agreed with Project 9's session, 2026-11-03.
- **T2, the hard tail (about 300):** messages that defeat the rules. A
  separate Claude agent writes them from Project 9's labelling rules and a
  list of what the rules can't do. It never sees the generator, its lists or
  the training data. The rules run on each one, and only those they miss or
  ask about are kept. Arthur reviews 100.
- **T3, Arthur's misses (later):** his own unparsed messages from
  dogfooding, rewritten by him, kept in `data/private/`. Only counts are
  published.
- **The leak check** (E2) runs in CI on normalised text.
- **Scored once per frozen model** (E3), like Project 9's held-out set.

**Why:** T1 alone has too few failures (3 misses, 2 needless questions) to
measure anything. A generated test set would share the generator's quirks
and flatter the model. T3 is the only real-world check, so it is reported
apart, however small.

**Statistics:** zero wrong amounts in about 300 proposals is consistent with
a true rate of up to about 1% (rule of three, 95% confidence). The report
says so.

**Rejected:**
- *A random split of the generated data as the test set:* leakage by
  template.
- *Scoring the held-out sets during tuning:* tuning on the test.

## ADR-008: Zero wrong amounts is a hard gate, with a fixed procedure

**Decision:** one wrong amount on any held-out set fails the model, whatever
its other numbers. Then, as Project 9 does:
1. Report it as found, with the first number.
2. Fix the cause: the data, the labels or the prompt, not the one message.
3. Move every held-out message that was read into the dev split.
4. Have a fresh held-out set written the same way, and score it once.

A proposal the person would confirm counts as a write, because people tap
**Ya** without reading closely. That's why the gate is on proposals, not on
confirmed entries.

**Rejected:** *a small allowed error rate:* Project 9's whole claim is zero.

## ADR-009: Serve with llama.cpp's `llama-server` (framework settled)

**Decision:**
- Fuse the adapter, convert with llama.cpp's `convert_hf_to_gguf.py`
  (registers `Qwen2ForCausalLM` and `Qwen3_5ForCausalLM`, checked
  2026-11-03), quantise to Q8_0 (Q4_K_M if spike 3 needs it), and hash.
- `llama-server` (MIT) from a pinned image digest. Score that exact file.
- Hardening, from its README: the web UI is **on by default**
  (`--no-webui`), the slots endpoint is **on by default** (`--no-slots`),
  `--api-key`, no `--log-prompts-dir`, `-np 1`, a small `-c`.
- Its JSON-schema constrained output (`json_schema` on `/completion`, `-j`
  on the command line) enforces K3. Whether the OpenAI-style
  `response_format` works the same is checked in spike 3.
- Container: memory cap 1 GB, CPU limit 1.0, internal network only.

**Why:** one small C++ binary, CPU-first, the established choice for running
GGUF files. It speaks the OpenAI chat API, which PydanticAI and Inspect both
use, so nothing else changes.

**Rejected:**

| Option | Why not |
|---|---|
| Ollama | Wraps llama.cpp with a model store and its own daemon; more surface for no gain |
| vLLM | Built for GPUs; heavy on a 2-vCPU server |
| PyTorch + Transformers on CPU | Gigabytes of dependencies and slower on CPU |
| MLX on Linux (`mlx[cpu]`) | Exists per PyPI's extras, but undocumented there and not the established CPU server |
| ONNX Runtime GenAI | Possible, but another conversion path to verify |
| Serving from the Mac | The Mac isn't always on, and messages would leave the server |

## ADR-010: Plugging into Project 9 changes a setting, not its rules

**Decision:**
- Project 9's ADR-007 fallback already uses PydanticAI with a typed proposal.
  PydanticAI can point at any OpenAI-compatible endpoint, so the change is a
  base URL to the local `llama-server`, plus the trigger in I2.
- Everything in Project 9's M1 to M5 stays: substring, parser, **Ya**,
  expiry, timeout to "belum paham", daily cap.
- **Project 9's session makes the change, in its repo.** This repo hands over
  the GGUF file's hash, the container file and the report.
- It reopens one line of Project 9's ADR-007: "A local model: too heavy for a
  small shared server." Spike 3 measures whether that still holds.

**What it removes:** the three reasons the fallback is off (no budget,
unread provider terms, Meta's section 4.7 on training). Nothing leaves the
server, and no provider sees a message.

**Rejected:**
- *The model inside Project 9's container:* its cap is 256 MB, and a model
  crash would take the webhook with it.
- *The model before the rules:* slower and costlier for messages the rules
  already settle.

## ADR-011: A task-blind core, scored with Inspect AI

**Decision:** the layout separates the core from each task:

```
core/        data checks, provenance, train config, served-file build, report
tasks/
  finance/   generator, word lists, output schema, scorer (uses P9's parser)
  juku_chat/ later (ADR-012)
evals/       Inspect AI tasks, one per task
configs/     one YAML per training run
```

- **Scoring:** an Inspect AI task per task, run against the local
  `llama-server` through Inspect's `openai-api/<provider>/<model>` provider
  (base URL and key from environment variables, per Inspect's docs). The
  scorer is plain code.
- **Typer** CLI: `build-data`, `train`, `build-served-file`, `score`,
  `report`. Each writes a run record the report reads.

**agent-eval-harness doesn't fit the finance task:** it grades agents
calling MCP tools, and this model has no tools (Project 9's ADR-014 found
the same). It fits Juku's slice, through Juku's own runner.

**Rejected:** *one script per step:* no shared records, so the report would
be typed by hand.

## ADR-012: Juku's chat distillation: a separate slice, gated three ways (given)

**Decision:** train a small model to copy Shiori's answers, on the same
core. It starts only when all three hold:
1. **Juku's chat gate (Q1) and memory gate (MEM1) pass.** Now 89% and 77%
   against 90% (paid run, 2026-11-03). Copying a teacher that misses its
   gates copies the misses.
2. **The provider's terms allow it.** Shiori runs on gpt-5-mini through
   Sumopod. OpenAI's terms list "use Output to develop models that compete
   with OpenAI" as prohibited (found by search; the live page refused the
   fetch, so it's to be read in a browser). Sumopod's own terms couldn't be
   found in Project 9's planning. Arthur decides after reading both.
3. **A budget Arthur approves.** Collecting Shiori's answers is a paid run.

**What changes for Juku's task:** mlx-lm's `tools` format, since Shiori
calls 8 read tools; scoring through Juku's `run_suite.py` with a model base
URL, so `agent-eval compare` works against the teacher. A 1B model calling
tools well is an open risk, measured, not assumed.

**Rejected:** *training on Shiori's answers now:* all three gates are
closed.

## ADR-013: The model report, and whether to publish weights

**Decision:** a generated `docs/model-report.md` per frozen model, laid out
like a Hugging Face model card: base model, revision and licence; data
sources and synthetic share; Arthur's review counts; every held-out number
per set; latency; the scored file's SHA-256; and "what this can't claim".

**Recommended:** publish the adapter and the generator's output on Hugging
Face under Apache-2.0, after the gates pass. It lets a reviewer rerun the
held-out scoring. Arthur decides after the gates pass (2026-11-03);
nothing is published without his OK.

## ADR-014: Standard stack for the glue

uv, Python 3.12, Pydantic, Typer, pytest, ruff with the complexity rules,
pyright strict, and a CI workflow running lint, types, tests and the data
checks. No training runs in CI: it has no Apple silicon, and training isn't
a check. The generator is plain Python: a templating library would be
heavier than the problem.

## Decisions

| # | Decision | Rejected | Status |
|---|---|---|---|
| 001 | One task first; task-blind core; Juku later; after P9's real-data gate | Both at once; Juku first | **Given** |
| 002 | Model points at text; never a number | Integer output; normalised digits | **Settled** |
| 003 | Gain = settled questions and right labels; zero-wrong gate; spelled numbers go to P9's rules | Coverage alone; model grading | **Settled** |
| 004 | mlx-lm LoRA on the Mac | TRL/PEFT, Unsloth, Axolotl, cloud GPU, provider fine-tuning, full fine-tune | **Settled** |
| 005 | Bake-off: Sailor2-1B-Chat vs Qwen3.5-0.8B, on dev only | Llama 3.2, Gemma 3, Gemma 4 E2B, LFM2, Granite, SmolLM3 | Open |
| 006 | Mostly generated data; labels checked by the parser; Arthur samples | Real messages, bulk model-written data, translation | **Settled** (P9 dev set included, disclosed) |
| 007 | T1, T2 (separate author), T3 (Arthur); leak check; score once | Random split; tuning on held-out | Open |
| 008 | Zero wrong amounts, hard gate, fixed procedure | An allowed error rate | Open |
| 009 | llama-server, Q8_0 GGUF, hardened, capped | Ollama, vLLM, PyTorch CPU, MLX on Linux, ONNX, the Mac | **Settled** (framework); settings open until spike 3 |
| 010 | P9 points its PydanticAI fallback at the local server; P9's session changes it | Model in P9's container; model first | Open |
| 011 | Task-blind core; Inspect AI scoring; Typer CLI | One script per step | Open |
| 012 | Juku's slice after its gates, terms check and budget | Training on Shiori now | **Given** |
| 013 | Generated model report; publish adapter after gates | Hand-written report | Report: open; publishing: decided after the gates |
| 014 | Standard stack; no training in CI | — | Open |

## The first slice

**Slice 1: the finance model, end to end on the Mac, at $0.** Data, the
bake-off, the served file and held-out scoring on T1 and T2. No server, no
Project 9 change. See the [slice plan](slice-plan.md).

## Settled with Arthur (2026-11-03)

1. **The gain (ADR-003):** measure settled questions, direction and
   category on the hard tail, with zero wrong amounts as the hard gate. Raw
   coverage on Project 9's held-out set isn't the measure. Spelled-out
   numbers go to Project 9's rules.
2. **New frameworks:** mlx-lm for training and llama.cpp's `llama-server`
   for serving are approved.
3. **Claude-written data (ADR-006):** Project 9's dev set goes into training,
   and the report states its share and the reasoning about Anthropic's terms.
4. **Downloads:** none during planning. Both finalists are downloaded when
   the spikes run, after Arthur OKs the phases.
5. **Review time:** about two hours, as in the slice plan.
6. **Publishing (ADR-013):** decided after the gates pass. The licence and
   model card are planned so it stays possible.

## Open questions

**For Arthur, later:**
- OK to the phases, which starts the spikes and their downloads (P1, P3).
- Publishing, once slice 1's results are in.
- Juku's slice: the provider terms and a budget (ADR-012).

**For Project 9's session (reported, not asked of Arthur):** a spelled-out
number reader in the rules; export the corpus loader and labelling schema as
importable code, so this repo depends on them instead of copying.

**For the spikes:** Qwen3.5's hybrid architecture through mlx-lm and
llama.cpp; memory and time on the M2; latency on 2 vCPUs; what each log
level prints; `response_format` in `llama-server`; Mac and server agreeing
on greedy output.

## Sources (read 2026-11-03)

**Primary:**
[mlx-lm LORA.md](https://github.com/ml-explore/mlx-lm/blob/main/mlx_lm/LORA.md) ·
[mlx-lm on PyPI](https://pypi.org/project/mlx-lm/) ·
[mlx-lm model list](https://github.com/ml-explore/mlx-lm/tree/main/mlx_lm/models) ·
[llama.cpp server README](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md) ·
[llama.cpp converters](https://github.com/ggml-org/llama.cpp/tree/master/conversion) ·
[Sailor2-1B-Chat](https://huggingface.co/sail/Sailor2-1B-Chat) ·
[Qwen3.5-0.8B](https://huggingface.co/Qwen/Qwen3.5-0.8B) ·
[Qwen3 blog](https://qwenlm.github.io/blog/qwen3/) ·
[Llama 3.2 licence](https://huggingface.co/meta-llama/Llama-3.2-1B-Instruct/blob/main/LICENSE.txt) ·
[Gemma terms](https://ai.google.dev/gemma/terms) ·
[LFM2 licence](https://huggingface.co/LiquidAI/LFM2-1.2B/blob/main/LICENSE) ·
[Hugging Face model API](https://huggingface.co/api/models) (sizes, licence tags) ·
[Inspect AI providers](https://inspect.aisi.org.uk/providers.html) ·
[Anthropic consumer terms](https://www.anthropic.com/legal/consumer-terms) ·
[Anthropic commercial terms](https://www.anthropic.com/legal/commercial-terms) ·
Project 9: `docs/parser-accuracy.md` and `tracker/parsing/amount.py` at
`21a307f` (branch `feat/amount-parser`) ·
Juku: `evals/paid-run-results.md` at `0eeff4d`

**Secondary, used only where marked:**
[OpenAI terms, via search](https://openai.com/en-GB/policies/terms-of-use/) (the live page refused the fetch)
