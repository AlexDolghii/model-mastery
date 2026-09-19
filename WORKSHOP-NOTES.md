# Model Mastery Workshop - TrailMate build log

**Date:** 19 September 2026
**Platform:** Microsoft Foundry (Sweden Central)
**Project:** `foundry-workshop-hava7qakpi2lu`
**Resource group:** `rg-model-masterylod65269309`

## What was built

TrailMate - a grounded product-expert agent for Contoso Outdoors. It answers gear
questions from 10 product manuals through a file-search tool, and refuses to
invent anything the manuals do not state.

An agent here is: **model + instructions + a tool**.

## Environment

Six model deployments, all Global Standard:
`gpt-5.4`, `gpt-5.4-mini`, `model-router`, `MAI-Image-2.5-Pro`,
`claude-sonnet-4-6`, `claude-haiku-4-5`.

Resources: Foundry account, Foundry project, Application Insights, Log Analytics.

### Setup bug found and fixed

`setenv.sh` captured an Azure CLI error message as the App Insights connection
string:

```text
APPLICATIONINSIGHTS_CONNECTION_STRING=The command requires the extension application-insights. Do
```

Cause: the `application-insights` CLI extension was not installed when the script
ran. Fix: install the extension, re-run `setenv.sh`. Left alone, every trace in
lab 02 would have silently produced nothing.

## Lab 1 - Model selection

The question is never "which model is best", it is "best for this task, at what
cost, latency and quality".

### Frontier vs mini (identical JSON extraction task)

| asked | served | latency | tokens |
|---|---|---|---|
| `gpt-5.4` | `gpt-5.4` | 6.13s | 122 |
| `gpt-5.4-mini` | `gpt-5.4-mini` | 3.40s | 123 |

Same answer, same token cost, mini 1.8x faster. On a well-specified task the
frontier model earned nothing. Buy the capability the task needs, not the
capability that exists.

### The reasoning-effort dial

| effort | latency | tokens | answer chars |
|---|---|---|---|
| low | 34.23s | 3308 | 14090 |
| high | 35.04s | 3785 | 13744 |

High effort spent 14% more tokens and returned a **shorter** answer. The extra
tokens went into hidden reasoning that had nothing to unlock. The dial is a
hypothesis to test, not a default to set.

### Model Router

| ask | served model | latency | tokens |
|---|---|---|---|
| simple | `gpt-5-mini-2025-08-07` | 4.97s | 43 |
| complex | `grok-4-1-fast-reasoning` | 21.79s | 3337 |

77x token spread from one endpoint, and the router selected across vendors
(xAI) without being told to.

### Capability boundary

Asked to generate an image, `gpt-5.4` did not refuse - it quietly returned a
text prompt instead. That silent pivot is the boundary revealing itself.

## Lab 2 - The hill climb

Measurement: a 14-case test set plus a frozen LLM-judge rubric, identical for
every version. Change one lever, re-measure, keep only what the evidence
supports.

| Version | Lever changed | Pass rate | Elapsed | Tokens |
|---|---|---|---|---|
| v1 Basecamp | starting point (vague instructions) | 0.86 (12/14) | 310.4s | 577,600 |
| v2 Clearpath | instructions -> hand-optimized | **1.00 (14/14)** | 310.8s | 579,694 |
| v3 Trailfinder | model -> `model-router` | 0.79 (11/14) | 191.4s | 169,350 |
| v4 Summit | instructions -> Agent Optimizer | 0.93 (13/14) | 351.7s | 609,772 |

`errored = 0` on all four runs, so all 14 cases were graded every time. Check
that column before reading any score: the portal's pass-rate percentage only
counts graded items, so errored items vanish silently and a version can look
strong while being graded on a shrinking slice.

### What each lever did

- **v1 -> v2 (instructions):** +0.14 pass rate for +2,014 tokens, a 0.35% cost
  increase. Instructions are nearly free quality and the highest-leverage lever
  available.
- **v2 -> v3 (model):** the router cut 408,330 tokens (-70%) and 119 seconds
  (-38%) but lost 0.21 pass rate. The lab text predicted quality would hold. It
  did not. v3 was therefore **not** promoted.
- **v3 -> v4 (Agent Optimizer):** 2 candidates, 26m 6s, 1.1M tokens.
  Baseline 0.692 -> candidate_1 0.716 -> candidate_2 0.739 (+0.047).
  candidate_2 promoted as version 4.

## The central finding

v4 improved the optimizer's **own** metric (task-weighted average 0.692 ->
0.739) while getting **worse** on pass rate (1.00 -> 0.93), and was slower and
more expensive.

Two things follow:

1. **A 100% pass rate hid real headroom.** v2 passed every case but earned only
   ~69% of available rubric quality. A binary threshold cannot see partial
   quality, so "14/14, we are done" and "0.692, a third is still on the table"
   were both true of the same agent.
2. **An optimizer maximizes the metric it is given, faithfully** - including
   when that is not the metric that matters. It bought partial credit across
   many cases while pushing one case over the fail line.

Best version by the metric that mattered: **v2, hand-written.**

Also worth noting: v4's answers *read* cleaner and more confident than v1's,
and still scored lower. You cannot judge a version by eyeballing two chat
replies - that gap is exactly why the frozen rubric exists.

## What Agent Optimizer actually changed

v2 was prose rules. candidate_2 restructured them into labelled sections
(Your task / How to work / Response style / Grounding and accuracy rules) and
turned principles into checkable prohibitions:

- "Do not use memory, outside knowledge, or unstated assumptions"
- "Do not mention file search, tools, backend systems, or citations unless the
  user explicitly asks"
- "Start with the direct answer"; "Do not leave trailing or unfinished text"
- generalized one water-resistant vs waterproof example into a family: light
  rain vs heavy rain, cold nights vs extreme cold

The characteristic move is turning vague principles into specific, verifiable
rules with concrete instances.

## Where everything lives in this repo

| Path | Contents |
|---|---|
| `foundry/agent-builder/labs/core/00-validate-setup.ipynb` | Environment validation, all outputs saved |
| `foundry/agent-builder/labs/core/01-model-selection.ipynb` | Model comparison tables, all outputs saved |
| `foundry/agent-builder/labs/core/02-agent-optimization.ipynb` | Full v1-v4 climb, all outputs saved |
| `foundry/agent-builder/src/agent/exported-versions/` | Instructions and config for v1, v2, v3, v4 |
| `foundry/agent-builder/src/data/evaluation-cases.jsonl` | The frozen 14-case test set |
| `foundry/agent-builder/src/data/evaluators/` | The rubric used to grade every run |
| `foundry/agent-builder/src/data/` | The 10 Contoso Outdoors product manuals |
| `foundry/agent-builder/scripts/provision.sh` | Infrastructure provisioning |

## Rebuilding TrailMate

The Foundry project is provisioned per lab session and does not survive it -
the agent, its four versions, the vector store and the evaluation runs all
disappear with the resource group.

This repo holds everything needed to recreate it on another Foundry project:
the 10 manuals, the instruction sets for all four versions, the frozen test
set, the rubric, and the provisioning script.

## The method, in one line

Measure a baseline, change one lever, re-measure against the same frozen
yardstick, and promote only what the evidence supports.
