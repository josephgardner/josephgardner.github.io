---
layout: post
title: Continue from Boot
permalink: /continue-from-boot/
description: "How a one-line boot protocol preserves evidence, decisions, and bounded next work across fresh coding-agent sessions."
image: /public/continue-from-boot-social.png
image_type: image/png
image_alt: "Agent Session Protocol continuity loop from BOOT through STATE, QUEUE, NEXT, work, and checkpoint"
image_width: 1200
image_height: 630
comments: true
---

```text
> continue from boot
```

FastImg is a local image renderer designed and optimized for one machine: my
[2020 M1 MacBook Pro](https://support.apple.com/en-us/111893).

I began FastImg in August 2025, worked on it through September, and then left it
untouched for an unplanned nine months. When I returned in July 2026, I still
had the same laptop. The repository still contained the original
[Claude](https://claude.com/plugins/frontend-design)-built web interface,
[obligatory purple](https://www.newyorker.com/culture/infinite-scroll/the-ai-design-aesthetic-thats-taking-over-the-internet)
included.

By current image-generation standards, the 2020 M1 was a modest target. I wanted
to find the practical throughput ceiling of hardware I already owned.

The first clean benchmark averaged 26.25 seconds per image, or about 133 images
an hour. The lowest mean in the later samples was 6.05 seconds, or about 595
images an hour.

This was not a general Apple Silicon benchmark. FastImg has one production
target. The machine is a `MacBookPro17,1` with 16 GB of unified memory and an
8-core GPU. CPU fallback does not count. The goal is sustained local generation
on that machine.

### The six-second result

The first clean baseline used the one-step variant of
[SDXL-Lightning](https://huggingface.co/ByteDance/SDXL-Lightning) through
[Hugging Face Diffusers](https://huggingface.co/docs/diffusers/index) and
PyTorch's [Metal Performance Shaders (MPS) backend](https://docs.pytorch.org/docs/stable/notes/mps)
at 1024&times;1024. Later configurations used
[MLX](https://github.com/ml-explore/mlx), Apple's machine-learning framework for
Apple silicon, with [SDXL-Turbo](https://huggingface.co/stabilityai/sdxl-turbo).
Each configuration below was measured with five serial jobs. Here, `p50` is the
median render time, `p90` is the time within which 90 percent of renders
completed, and quantized configurations store model weights at lower numerical
precision to reduce memory use.

| Configuration | Size | Mean | p90 | Images/hour |
| --- | ---: | ---: | ---: | ---: |
| Diffusers/MPS, SDXL-Lightning | 1024 | 26.25s | 27.86s | 133 |
| MLX, SDXL-Turbo | 1024 | 20.94s | 24.87s | 172 |
| MLX, quantized SDXL-Turbo | 1024 | 12.58s | 15.64s | 286 |
| MLX, quantized SDXL-Turbo | 768 | 6.53s | 7.38s | 552 |
| MLX, quantized, style-constrained preset | 768 | 6.05s | 6.23s | 595 |

The 768px results trade resolution for throughput and are not directly
comparable to the 1024px results. The unquantized and quantized 1024px MLX runs
both reported about 17.1 GB of peak overall allocation on a machine with 16 GB
of unified memory. The 768px run brought that down to about 11.3 GB.

The 512px result had a 4.16-second median and a 23.44-second p90, so I kept
768px.

MLX also had to be loaded and used on the same thread, otherwise it could fail
with:

```text
There is no Stream(gpu, 0) in current thread.
```

FastImg now loads and renders through one dedicated worker thread. Jobs are
persisted to [SQLite](https://sqlite.org/docs.html) before they enter that queue.
Queued jobs resume in creation order after a restart; an interrupted processing
job is requeued once with the interruption recorded. Serialization also keeps
two render workloads from competing for the same 16 GB of memory.

### Two hundredths of a second was not worth shipping

The quickest preset reached a 6.23-second p90, then lost all four side-by-side
quality comparisons. The preset I shipped reached 6.25 seconds, passed ten
quality reviews, and completed a 21-image run without failure.

A single field named `current_model` could not represent these outcomes. FastImg
tracked three separate facts:

```text
fastest_measured_lane
quality_approved_lane
production_default_lane
```

Those values can point at the same configuration. They do not have to.

### Recovering the experiment

Nine months later, the benchmark record offered several plausible wrong turns.
An obsolete local runtime made MPS look unavailable. A 512px run looked faster
until its 23.44-second p90 was considered. The quickest preset lost its quality
comparison.

None of those facts was false; each was incomplete. The evidence that changed
their meaning was split across benchmark artifacts, per-image metadata, human
votes, live server state, commit history, and prior chat. A fresh agent could
recover one result faithfully and still choose the wrong next experiment.

The production command fit in a README. The measurement history fit in a
benchmark journal, although the journal eventually became too large to read at
the start of every session. Neither gave a fresh session a compact answer to
three immediate questions.

<aside class="pull-quote" aria-label="Continuity principle">
  <p>Continuity means knowing what is true, what was decided, and what comes
  next.</p>
</aside>

The first version was `START_HERE.md`, one resume file containing runtime facts,
benchmark state, recent commits, and the next task. I soon split it into
`BOOT.md` for recovery, `STATE.json` for current truth, and `NEXT.md` for the
next bounded step, while the benchmark journal kept the detailed evidence.

Those files were committed to FastImg first. Less than twenty minutes later, I
generalized them into the first Agent Session Protocol Gist.

Continued use exposed another problem. A project's durable priority can survive
many sessions; `NEXT.md` should describe only the next bounded step. I added a
queue to FastImg for the durable work and then carried that distinction back
into the Gist.

### A queue is not a session

The result is five file roles, separated by how often they change and the
question they answer:

```text
BOOT.md     How does a fresh session recover and work?
STATE.json  What is true now?
QUEUE       Which durable task owns attention?
NEXT.md     What bounded slice is authorized next?
TRACKER.md  What evidence and reasoning must survive?
```

For FastImg, the durable task might be "find the fastest acceptable local
renderer." That task can survive many sessions.

One `NEXT.md` slice might be much narrower:

```text
Benchmark quantized MLX SDXL-Turbo at 768px.
Use the fixed five-prompt set and seed 11.
Record mean, p50, p90, allocation, and failures.
Do not change the production default before human review.
```

When that slice finishes, the task remains active while `NEXT.md` advances to
the next evidence gap. The queue preserves strategic priority. `NEXT.md`
prevents that priority from becoming unlimited authority.

[![Agent Session Protocol continuity loop](https://gist.githubusercontent.com/josephgardner/5eb3359d1342e08d60ceba1993983cfe/raw/28c287d39f87f86d2f93442209602b86fff7bf2b/PROTOCOL_EXPLAINER.svg)](https://gist.githubusercontent.com/josephgardner/5eb3359d1342e08d60ceba1993983cfe/raw/28c287d39f87f86d2f93442209602b86fff7bf2b/PROTOCOL_EXPLAINER.svg)

[View the full-size explainer](https://gist.githubusercontent.com/josephgardner/5eb3359d1342e08d60ceba1993983cfe/raw/28c287d39f87f86d2f93442209602b86fff7bf2b/PROTOCOL_EXPLAINER.svg)
&middot; [Open the protocol Gist](https://gist.github.com/josephgardner/5eb3359d1342e08d60ceba1993983cfe)

### The candidate that won and still lost

I later tested a four-step
[FLUX.2 Klein](https://huggingface.co/black-forest-labs/FLUX.2-klein-4B)
configuration. The candidate ran at a 43.76-second generation p90 and a
49.40-second total-wall p90. That was inside the predeclared sub-minute limit
for a slower quality configuration. All twelve candidate renders completed,
metadata passed, and peak MLX memory held at 4.83 GB.

Then I reviewed twelve exact pairs against the production renderer:

```text
candidate wins:   9
candidate losses: 2
ties:             1
both bad:         0
```

The candidate passed its relative comparison gate. I still wasn't impressed by
the images.

"Usually better than the baseline" and "good enough to justify an
eightfold-slower quality configuration" are different questions. The experiment
stopped before confirmation and stability work, and production did not change.

The durable state recorded both outcomes:

```json
{
  "relative_gate_result": "passed",
  "product_owner_assessment": "not impressed",
  "decision": "rejected",
  "production_default_changed": false
}
```

If the next agent recovered only the 9-2-1 score, continuing the candidate would
look like the rational action. If it recovered only the rejection, the reason
would be lost and the benchmark could be repeated later. The evidence and the
decision both need to survive, along with the authority of the person or system
responsible for the gate.

The current protocol therefore keeps work status, evidence outcome, decision,
deployment state, and authority separate. Automated tests can own some gates.
Continuous integration (CI) or a deployment system can own others. Aesthetic
judgment remains human. Automated evidence does not satisfy a human-owned gate.

### Making the convention executable

Markdown files alone eventually drift, so the protocol includes a local
[JSON Schema](https://json-schema.org/) and a validator. The validator can check
things such as:

* the queue's active task, `STATE.json`, and `NEXT.md` all naming the same task;
* the active slice identity agreeing between state and next;
* referenced task, evidence, and workspace files existing;
* required acceptance checks and scope fences being present;
* recorded clean-worktree requirements matching reality;
* state and next staying inside their context budgets.

It cannot prove that a benchmark is honest or that a product decision is good.
It can keep a fresh session from starting with three conflicting versions of
the current task.

Detailed output does not get copied into `STATE.json`. State retains a compact
evidence reference: a run id, command, path, commit, timestamp, and short result.
The benchmark journal or generated artifact remains canonical. When working
context grows too large, old detail moves out and the reference stays behind.

The same rule applies across repositories. Each workspace records its role,
branch or commit when relevant, dirty-tree policy, and verification commands.
A clean primary repository is not a coherent checkpoint if a required sibling
contains an unexplained partial implementation.

At shutdown, the agent updates only the files whose truth changed, validates
the protocol, and commits one coherent checkpoint.

### Continue from boot

A fresh session starts with:

```text
> continue from boot
```

The agent reads the repository instructions and `BOOT.md`, runs the validator,
loads current state, reads the active queue item and bounded next slice, checks
the recorded workspaces, and opens the tracker only when it needs historical
evidence.

There is no continuity service. The files are ordinary Git artifacts and the
protocol is agent-agnostic. It has been pressure-tested primarily by this one
project, so I do not yet know whether these are the correct abstractions for
every long-running agent task. Five canonical files may still be too many.
Compaction still requires judgment. A validator catches structural drift, not
stale truth.

In FastImg, `continue from boot` now names a reproducible recovery procedure. A
fresh session can distinguish "this was fastest" from "this was approved,"
recover the exact Python runtime that makes the GPU work, avoid re-running the
512px tail-latency mistake, and continue the current experiment without being
given the previous conversation.

The [Agent Session Protocol Gist](https://gist.github.com/josephgardner/5eb3359d1342e08d60ceba1993983cfe)
contains the current specification, schema, validator, templates, and setup
prompt and remains the canonical source.
