---
name: agent-orchestration-patterns
description: >-
  Chooses whether a coding task needs more than one agent and which of five team
  patterns fits: builder + verifier, lanes, best of N, council of models, team of
  reviewers. Use when the user wants to split, parallelize, cross-check or review
  coding work with several agents or models, or says "use a verifier", "split into
  lanes", "best of N", "council of models", "team of reviewers", "แยกคนตรวจ",
  "แบ่งงานให้หลาย agent", "รันหลายรอบเลือกอันดีสุด", "ให้หลายโมเดลเสนอ", "ทีมรีวิว".
  Load before dispatching: this skill picks the pattern; the host's dispatch skill
  (in Orca, `orchestration`) starts the agents.
license: MIT
metadata:
  version: "2.0.0"
  source: "https://dev.to/googlecloud/5-agent-orchestration-patterns-backed-by-research-2l43"
---

# Agent Orchestration Patterns

A team of agents beats one agent only when the task has the shape a pattern needs; otherwise it
costs more and does worse. This skill decides whether to use several agents, and which pattern.

## Step 1 — Choose how the work gets built

Take the FIRST row that fits; if the work later stops fitting it, choose again. If the user asked
for one of these patterns by name, use it — unless the task lacks what it needs (Best of N with no
scoring command, Lanes over shared files): then say what is missing and offer the row that fits.

| The task is… | Build it with |
|---|---|
| Simple: a small, low-risk change or small bug fix — a few files, one obvious approach, no security-sensitive code | **Single thread**: do it yourself, no extra agents, skip Step 2 |
| Independent parts, each a real chunk of work in its own files or folders | **Lanes** |
| Difficult (one attempt may well miss) and a command can score the result: tests, eval, benchmark | **Best of N** |
| Difficult and open-ended, nothing can score it: architecture, schema, API or visual design, naming | **Council of models** |
| Anything else that matters | **Builder + Verifier** |

Keep work in ONE thread, not Lanes, when:
- two parts would edit the same files — in one study two agents on overlapping code succeeded
  about 30% less often than one agent doing both tasks;
- each step needs the result of the one before — agent teams scored 39–70% lower than one agent on
  such a planning benchmark. Parts joined only by a contract you write first stay independent.

## Step 2 — Add checks

Apply every line that is true. These add to Step 1 and apply once there is code to check.

- Built with Lanes → one fresh verifier on the integrated result.
- Best of N scored by a command that does not check correctness → one fresh verifier before merging.
- Large change, about 10+ files or several subsystems → finish with **Team of reviewers**.
- Smaller change that touches security-sensitive code (auth, permissions, payments, secrets, user
  data) → one read-only security reviewer instead of the team.

## Step 3 — Announce, then read the playbook

Tell the user in one line which pattern(s), why, and how many agents that starts — also for Single
thread — then continue without waiting. Two cases wait for the user's answer: the offer from
Step 1, and a Council whose models the user did not name (ask which agents or models to use,
listing what is available). Unless it is Single thread, read the chosen pattern's section in
[PATTERNS.md](PATTERNS.md) before starting an agent.

| Pattern | Starts | The rule that makes it work |
|---|---|---|
| Builder + Verifier | 1 fresh read-only verifier | It gets the user's original request and the diff, never the builder's summary |
| Lanes | 1 writer per lane | No file belongs to two lanes; the shared contract is agreed first |
| Best of N | 3–5 writers on the same task | One scoring command for all attempts, run by you, not by the workers |
| Council of models | 3 proposers + 3 judges, different models | Proposals are independent; judges score them with names hidden |
| Team of reviewers | 4 read-only reviewers | One concern each; fix, then re-review with fresh reviewers, 3 rounds at most |

## Dispatching

- **Parallel writers** (lane workers, Best-of-N attempts) each get their own worktree and branch,
  and commit there. You merge, into the branch you were working on.
- **Read-only agents** (verifier, reviewers, judges, design proposers) need a fresh context, not a
  worktree: one that has not seen this conversation or the builder's reasoning. Read-only means no
  file edits; running tests, builds and the app is expected.
- **Which agent:** your own agent and model unless the user named another; only the Council differs.
- **In Orca:** load the `orchestration` skill and take every command from its guide; never write
  Orca commands from memory. Use Orca workers for parallel writers and for any agent on a
  different CLI or model. Read-only agents on your own model may be your own subagents unless the
  user asked for Orca workers. (Not installed? `npx skills add stablyai/orca`.)
- **Elsewhere:** the host's subagent tool, with worktree isolation for parallel writers.
- **Every prompt is self-contained**, because the agent has not seen this conversation: goal,
  files it may edit (or "edit nothing"), what done means, and the report format.

## Combining

Choose per phase: a design phase that is itself difficult and open-ended gets a Council before the
build. Big feature: Council (design) → Lanes (build) → verifier → Team of reviewers. Add a pattern
only when its row or check line is true; each one costs agents.

## Common mistakes

- Orchestrating a simple change because this skill was loaded.
- Acting on a verifier's or reviewer's finding without checking it against the code.

## Final report

Short, in the user's language: pattern(s) used and why, agents run, results (test output, score
table for Best of N, verdicts for verifier and reviewers), what was merged, what is unresolved.
