# TrailMate agent versions (v1 to v4)

Exported from the Microsoft Foundry project `foundry-workshop-hava7qakpi2lu`.
Each version changed exactly one thing from the one before it.

| Version | Instructions | Model | Lever changed | Pass rate |
|---|---|---|---|---|
| v1 - Basecamp | `v1-instructions.md` | gpt-5.4 | starting point (deliberately vague) | 12/14 = 0.86 |
| v2 - Clearpath | `v2-instructions.md` | gpt-5.4 | instructions -> hand-optimized | 14/14 = 1.00 |
| v3 - Trailfinder | `v3-instructions.md` | model-router | model -> router | 11/14 = 0.79 |
| v4 - Summit | `v4-instructions.md` | gpt-5.4 | instructions -> Agent Optimizer | 13/14 = 0.93 |

All four used the same file-search tool over the same vector store of 10 Contoso
Outdoors product manuals, and were scored with the same frozen rubric and 14-case
test set (`../../data/evaluation-cases.jsonl`).

Best result: v2. The automated optimizer raised its own task-weighted score
(0.692 -> 0.739) but scored lower on pass rate than the hand-written v2.

`versions.json` holds the model and tool configuration for each version.
