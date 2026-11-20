# Pattern playbooks

Read the section for the pattern chosen in [SKILL.md](SKILL.md) before starting any
agent. The dispatching rules in SKILL.md (worktrees, read-only, which agent,
self-contained prompts) apply to every section. In the prompt templates, replace
every `<…>`.

Contents: [Builder + Verifier](#builder--verifier) · [Lanes](#lanes) ·
[Best of N](#best-of-n) · [Council of models](#council-of-models) ·
[Team of reviewers](#team-of-reviewers) · [Sources](#sources)

## Builder + Verifier

Why: models are poor judges of their own work. They struggle to correct their own
reasoning without outside feedback and tend to favor their own output, so the check
has to come from a thread that did not write the change.

1. The builder (you, or one worker) implements the change.
2. Start ONE fresh, read-only verifier. Give it the user's original request word for
   word and the command that shows the whole change since the task started,
   uncommitted and new files included. Do not give it the builder's summary or
   reasoning.
3. It checks every requirement against the diff, runs the tests and build if they
   exist, and returns PASS or FAIL.
4. On FAIL, check the findings against the code. Send the confirmed ones to the agent
   that wrote that code, or fix them yourself, then verify again with a NEW verifier.
   After 3 failed rounds, stop and report what remains.

Verifier prompt:

> You are a read-only verifier: do not edit any file; do run tests and builds.
> The user asked for, verbatim: "<original request>".
> See the change with `<diff command>`.
> 1. List every requirement in the request as MET or UNMET, with evidence
>    (file:line or command output).
> 2. List regressions and risky changes the request did not ask for.
> 3. End with one line: PASS or FAIL.

## Lanes

Why: on a finance benchmark with independent parts, agent teams scored up to about
81% higher than one agent. On a planning benchmark with dependent steps they scored
39–70% lower, and in a coding study two agents on overlapping code succeeded about
30% less often than one agent doing both tasks. The gain exists only while the parts
stay independent.

1. List the lanes and each lane's paths. No file may appear in two lanes. A file two
   lanes need (shared types, routes, config, lockfiles) goes to one lane, or you
   change it yourself before the lanes start. Fold a part too small to be worth an
   agent into a neighbouring lane.
2. Read the existing code, then write the contract the lanes share — API routes and
   payloads, types, env vars, error format. Paste the same text into every lane's
   prompt.
3. Start one writer per lane in parallel, each in its own worktree. Give each lane
   "done" checks it can run alone: a lane that needs another lane's output builds
   against the contract with mocks or stubs. Checks that need two lanes belong to
   step 4.
4. When all have reported: merge the lane branches, run the full build and every
   test suite, and check the parts meet the contract (calls match routes, types
   match, migrations run).
5. Verify the integrated result with one fresh verifier (Builder + Verifier,
   steps 2–4). A confirmed finding goes back to the lane that owns the file, or you
   fix it.

Lane prompt:

> You own lane <name>. Edit only files under <paths>. If the task needs a change
> outside them, do not make it — report it.
> Contract, shared with the other lanes; do not change it: <contract>
> Goal: <goal>. Done when: <checks this lane can run alone>. Commit on your branch.
> Report: files changed, how to test your part, and every place you deviated from
> the contract or had to assume something about it.

## Best of N

Why: with a real check, extra attempts keep paying off. In one study a single model
went from 15.9% to 56% of real GitHub issues solved between 1 and 250 attempts.
Without a check, picking the winner by majority vote or with a scoring model stopped
improving around 100 attempts; that case belongs to the Council.

1. Fix the scoring command (a benchmark script, an eval, `npm test`) and any
   correctness command that must keep passing. Run both on the current code and
   record the baseline. For a benchmark, use the median of 3 runs, here and in
   step 3. If a command does not run, fix that first.
2. Start N writers with the SAME prompt on the same model, each in its own worktree:
   3 by default, 5 when earlier attempts already failed or the user asked for more.
   They must not see each other's work.
3. When all have finished, score every attempt yourself, with the same commands on
   the same machine. Do not use scores the workers report.
4. Put the baseline and all attempts in one table: attempt, score, correctness
   checks pass?, diff size.
5. The winner is the best score among attempts that pass the correctness checks;
   break ties by the smaller diff. If nothing beats the baseline, merge nothing. If
   the winner misses a target the user set, report the table and ask before
   merging. Otherwise merge only the winner, then run both commands once more on
   the merged branch.

Attempt prompt:

> Goal: <goal, and what earlier attempts already tried>. Edit only <paths>.
> Your result is scored with `<scoring command>`; the baseline is <baseline score>.
> Done when: <target score, or "better than the baseline">.
> `<correctness command>` must still pass.
> Work alone. Commit on your branch. Report: what you changed and why, and the
> output of both commands.

## Council of models

Why: combining answers from several models has scored higher than a single frontier
model, and a panel of different models has matched human judgment better than one
large judge.

Which models: the council needs different models. Use the ones the user named, or
the ones they chose when you asked in Step 3. With a single model, say the council
will be weaker and use independent fresh agents. You are the synthesizer: you write
the brief and merge, and you are neither a proposer nor a judge.

1. Read what you need of the project, then write the brief and the scoring criteria
   before starting anyone. Default criteria: fit to requirements, simplicity, risk,
   cost to build. Proposers and judges get the same brief and criteria.
2. Start 3 proposers on different models, in parallel. Each writes one complete
   proposal; none sees another's.
3. Remove model and agent names from the proposals and label them A, B, C.
4. Start 3 fresh judges, on different models where possible. Each scores every
   proposal on every criterion and lists the strongest ideas and the biggest
   weakness in each one. Names stay hidden so that no judge can favor its own
   model's proposal.
5. Add up the scores. The proposal with the highest total is the base (on a tie,
   the simpler one). Merge the strongest ideas from the others into it.
6. If the judges named a major weakness the merge did not fix, run a second and
   last round: three proposers improve the merged proposal using the judges' notes,
   then repeat steps 3–5 with fresh judges.
7. Present the final proposal, the score table, and which ideas came from which
   proposal, now with the model names. A design-only council ends here; the Step 2
   checks apply when the design is built.

Proposer prompt:

> Brief: <problem, constraints, what a complete answer contains>.
> It will be judged on: <criteria>.
> Write one complete proposal and state your assumptions. You may read the project
> to match its conventions; do not edit any file. Work alone.
> Reply with the proposal only. It will be judged anonymously: do not mention your
> model or tool name.

Judge prompt:

> You are one of three independent judges. Do not edit files.
> Brief: <brief>. Proposals A, B and C follow.
> Score each proposal 1–5 on each of: <criteria>, with one line of reasoning per
> score. Then, for each proposal, list its strongest ideas and its biggest weakness.

## Team of reviewers

Why: in a 1995 study of human inspections, teams whose members each looked for a
different kind of defect found about 35% more than teams whose members all used the
same general checklist. Panels of AI judges show the same direction: distinct roles
did better than one shared role.

1. Start 4 read-only reviewers in parallel, one concern each:
   - UX and product behaviour
   - Code simplicity and maintainability
   - Edge cases and error handling
   - Security: auth, input validation, injection, secrets, CORS, rate limits

   Swap in a specialist when a default concern does not apply or the change needs
   one: performance and DB queries, accessibility, API compatibility. Keep one
   concern per reviewer.
2. Each returns findings (file:line, severity, why it matters, suggested fix) and a
   verdict: CHANGES if it has any blocker or major finding, otherwise APPROVE.
3. Merge duplicate findings. Check every blocker and major against the code before
   acting on it; drop a wrong one and say why in the final report.
4. Send each confirmed blocker and major to the agent that wrote that code, or fix
   it yourself. Re-run the tests.
5. Repeat from step 1 with FRESH reviewers until all approve. After 3 rounds, stop
   and report what remains.

The single security reviewer from Step 2 follows the same steps with one reviewer.

Reviewer prompt:

> You are a read-only <concern> reviewer: do not edit any file.
> The user asked for, verbatim: "<original request>".
> See the change with `<diff command>`. Review it ONLY for <concern>; ignore other
> kinds of issue.
> For each finding: file:line, severity (blocker / major / minor), why it matters,
> suggested fix. End with one line: CHANGES if you have any blocker or major
> finding, otherwise APPROVE.

## Sources

The five patterns and the figures are from the Google Cloud article "5 agent
orchestration patterns backed by research":
<https://dev.to/googlecloud/5-agent-orchestration-patterns-backed-by-research-2l43>.
The chooser is adapted from its flowchart. This skill's own additions, not findings
from the studies: first-match row order with Builder + Verifier as the fallback, the
verifier after Lanes and Best of N, the round limits (3 for verifier and reviewers, 2
for the council), the A/B/C labels, the "about 10+ files" threshold, the single
security reviewer for smaller changes, and the prompt templates.

| Used for | Study |
|---|---|
| Self-review is biased | Huang et al. 2023, [arXiv:2310.01798](https://arxiv.org/abs/2310.01798); Panickssery et al. 2024, [arXiv:2404.13076](https://arxiv.org/abs/2404.13076) |
| Overlapping code, about 30% less often | Khatua et al. 2026, CooperBench, 652 tasks, [arXiv:2601.13295](https://arxiv.org/abs/2601.13295) |
| Up to 80.8% higher / 39–70% lower | Kim et al. 2026, Nature Machine Intelligence, [doi:10.1038/s42256-026-01268-y](https://doi.org/10.1038/s42256-026-01268-y) |
| Parallel subagents did not help small bug fixes | Li et al. 2026, [arXiv:2610.10263](https://arxiv.org/abs/2610.10263) |
| 15.9% → 56%; plateau near 100 without a check | Brown et al. 2024, [arXiv:2407.21787](https://arxiv.org/abs/2407.21787); AlphaCode, [arXiv:2203.07814](https://arxiv.org/abs/2203.07814) |
| Several models beat one; juries of models | Wang et al. 2024, [arXiv:2406.04692](https://arxiv.org/abs/2406.04692); Verga et al. 2024, [arXiv:2404.18796](https://arxiv.org/abs/2404.18796) |
| Distinct reviewer roles, about 35% more | Porter, Votta & Basili 1995, [doi:10.1109/32.391380](https://doi.org/10.1109/32.391380); Chan et al. 2023, [arXiv:2308.07201](https://arxiv.org/abs/2308.07201) |
