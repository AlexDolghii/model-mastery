# Model Mastery workshop, my TrailMate build log

I ran this on 19 September 2026 in Microsoft Foundry, Sweden Central.
Project `foundry-workshop-hava7qakpi2lu`, resource group `rg-model-masterylod65269309`.

## What I built

TrailMate, a product expert agent for a made up outdoor gear company called Contoso Outdoors. It answers questions about their gear by searching 10 product manuals, and it is supposed to say it does not know rather than guess when the manuals do not cover something.

An agent here is three things stuck together: a model, a set of instructions, and a tool. My tool was file search over the manuals.

## The setup, and the bug I hit

I had six model deployments to work with: gpt-5.4, gpt-5.4-mini, model-router, MAI-Image-2.5-Pro, claude-sonnet-4-6 and claude-haiku-4-5. Plus a Foundry project, Application Insights and a Log Analytics workspace.

The setup script wrote me a broken `.env`. Here is what it put in as my Application Insights connection string:

```text
APPLICATIONINSIGHTS_CONNECTION_STRING=The command requires the extension application-insights. Do
```

That is not a connection string. It is an error message. The Azure CLI wanted an extension called `application-insights` that was not installed yet, printed a prompt asking about it, and the script grabbed the prompt text. I installed the extension and re-ran `setenv.sh`. If I had not caught that, every trace in lab 02 would have quietly gone nowhere and I would have spent the rest of the day wondering why my observability tab was empty.

## Lab 1, picking a model

The point of this lab is that "which model is best" is the wrong question. The real one is which model is best for this specific job, at what cost and what speed.

### Frontier against mini

Same task for both, turning a tent description into structured JSON.

| asked | served | latency | tokens |
|---|---|---|---|
| gpt-5.4 | gpt-5.4 | 6.13s | 122 |
| gpt-5.4-mini | gpt-5.4-mini | 3.40s | 123 |

Same answer, same token cost, and the mini was nearly twice as fast. On a job this well defined, paying for the big model bought me nothing.

### The reasoning dial

Then an open ended planning question at low and high reasoning effort.

| effort | latency | tokens | answer chars |
|---|---|---|---|
| low | 34.23s | 3308 | 14090 |
| high | 35.04s | 3785 | 13744 |

High effort burned 14% more tokens and gave me a shorter answer. The extra tokens went into thinking I never get to see, and on a question like this there was nothing in there worth paying for. I would treat that dial as something to test, not something to turn up by default.

### The router

Two prompts, one `model-router` endpoint.

| ask | served model | latency | tokens |
|---|---|---|---|
| simple | gpt-5-mini-2025-08-07 | 4.97s | 43 |
| complex | grok-4-1-fast-reasoning | 21.79s | 3337 |

43 tokens against 3,337. And the router did not just reach for a bigger GPT on the hard one, it went to Grok, which is xAI's model. I called one endpoint and Foundry picked across vendors for me.

One other thing from this lab stuck with me. I asked gpt-5.4 to generate an image. It cannot, it is a language model. But it never said so. It handed me back a text prompt instead and carried on, which is a sneaky way to fail if you are not paying attention.

## Lab 2, the hill climb

This is the part I actually came for. Build the agent, measure it, change one thing, measure again.

My yardstick was 14 test cases plus a rubric graded by an LLM, frozen so every version got judged the same way.

| Version | What I changed | Pass rate | Elapsed | Tokens |
|---|---|---|---|---|
| v1 Basecamp | nothing, this was the start | 0.86 (12/14) | 310.4s | 577,600 |
| v2 Clearpath | instructions, rewritten by hand | 1.00 (14/14) | 310.8s | 579,694 |
| v3 Trailfinder | model, swapped to model-router | 0.79 (11/14) | 191.4s | 169,350 |
| v4 Summit | instructions, written by Agent Optimizer | 0.93 (13/14) | 351.7s | 609,772 |

Look at the errored column before you read any of those scores. Mine was zero on all four runs, so all 14 cases got graded every time and the comparison actually holds. The portal's pass rate only counts items it managed to grade, so anything that errors out just vanishes, and a version can look great while being judged on a shrinking slice of the test set.

Rewriting the instructions took v1 from 12 out of 14 to a clean sweep and cost me 2,014 extra tokens. That is a third of a percent. Instructions were the cheapest quality I bought all day, by a mile.

The router was the surprise. It cut 408,330 tokens, around 70%, and took 119 seconds off the run. It also lost me three test cases. The lab notes say this step should cut cost while holding quality. Mine did not hold quality, so I did not promote it.

Then I handed the job to Agent Optimizer. Two candidates, 26 minutes, 1.1 million tokens. It scored my v2 baseline at 0.692, its first candidate at 0.716, its second at 0.739. I promoted the second one as version 4.

## The thing I actually learned

v4 beat my v2 on the optimizer's own scale, 0.739 against 0.692. On pass rate it lost, 0.93 against my 1.00. It was slower and more expensive too.

Two things fell out of that.

My perfect score was hiding something. v2 passed all 14 cases but only earned about 69% of the quality the rubric was willing to hand out. A pass or fail line cannot see partial credit, so "14 out of 14, done here" and "0.692, there is a third of it still sitting on the table" were both true about the same agent at the same moment.

And the optimizer did exactly what I asked. It maximized the score I pointed it at, which turned out not to be the score I cared about. It picked up partial credit across a lot of cases, and in doing that it pushed one case over the fail line.

So my hand written v2 won. Worth adding that v4's answers read better than v1's, cleaner and more sure of themselves, and still scored worse. You cannot tell which version is better by reading a couple of chat replies. That is the whole reason the frozen rubric is there.

## What the optimizer changed in my instructions

I wrote v2 as a numbered list of rules in plain prose. The optimizer broke it into labelled sections, Your task, How to work, Response style, Grounding and accuracy rules, and turned my principles into things you can actually check against:

"Do not use memory, outside knowledge, or unstated assumptions"

"Do not mention file search, tools, backend systems, or citations unless the user explicitly asks"

"Start with the direct answer"

"Do not leave trailing or unfinished text"

I had written one rule about not calling something waterproof when the manual says water resistant. It turned that into a whole family of them. Light rain is not heavy rain. Cold nights are not extreme cold. That is the move it makes, taking something vague and making it specific enough to test.

## Where everything is

| Path | What is in it |
|---|---|
| `foundry/agent-builder/labs/core/00-validate-setup.ipynb` | Environment checks, outputs saved |
| `foundry/agent-builder/labs/core/01-model-selection.ipynb` | The model comparisons above, outputs saved |
| `foundry/agent-builder/labs/core/02-agent-optimization.ipynb` | The full climb, outputs saved |
| `foundry/agent-builder/src/agent/exported-versions/` | Instructions and config for all four of my versions |
| `foundry/agent-builder/src/data/evaluation-cases.jsonl` | My 14 test cases |
| `foundry/agent-builder/src/data/evaluators/` | The rubric |
| `foundry/agent-builder/src/data/` | The 10 product manuals |
| `foundry/agent-builder/scripts/provision.sh` | Provisioning |

## Rebuilding it

The Foundry project only lives as long as the lab session does. My agent, its four versions, the vector store and every evaluation run went out with the resource group.

What is in here is enough to build the whole thing again somewhere else: the manuals, all four instruction sets, the test cases, the rubric and the provisioning script.

## The method

Measure where you are. Change one thing. Measure again against the same yardstick. Keep it only if the numbers say so.
