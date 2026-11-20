# agent-orchestration-skills

Agent skills for running coding agents as a team — 5 research-backed orchestration
patterns, tuned for [Orca](https://github.com/stablyai/orca) but usable with any agent
that supports skills (Claude Code, Codex, Cursor, …).

Based on Google Cloud's post
["5 agent orchestration patterns backed by research"](https://dev.to/googlecloud/5-agent-orchestration-patterns-backed-by-research-2l43).

## Skills

| Skill | What it does |
|---|---|
| [`agent-orchestration-patterns`](skills/agent-orchestration-patterns/SKILL.md) | Decides whether a task needs more than one agent, and which pattern fits: Builder + Verifier, Lanes, Best of N, Council of models, or Team of reviewers |

### How it chooses

| The task is… | Pattern |
|---|---|
| Simple: small, low-risk change or bug fix | Single thread — no extra agents |
| Independent parts in separate files or folders | Lanes |
| Difficult, and a command can score the result | Best of N |
| Difficult and open-ended, nothing can score it | Council of models |
| Anything else that matters | Builder + Verifier |

Then it adds checks: a verifier on integrated or partly scored work, a Team of
reviewers for large changes, and a single security reviewer for smaller changes that
touch security-sensitive code.

### Files

```
skills/agent-orchestration-patterns/
├── SKILL.md      # chooser, dispatching rules, pattern summary (loaded first)
└── PATTERNS.md   # steps and prompt templates per pattern, plus the studies cited
```

The skill only picks the pattern. Starting the agents is left to the host: in Orca,
the `orchestration` skill; elsewhere, the agent's own subagent tool.

## Install

```bash
# all skills in this repo
npx skills add Ruchkung/agent-orchestration-skills

# just this skill, globally, for Claude Code
npx skills add Ruchkung/agent-orchestration-skills --skill agent-orchestration-patterns -g -a claude-code
```

For Orca, also install Orca's own skills so workers can be dispatched:

```bash
npx skills add stablyai/orca
```

## ภาษาไทย

Skill สำหรับจัดทีม AI coding agent: ช่วยตัดสินว่างานนี้ควรใช้ agent ตัวเดียวหรือหลายตัว
และถ้าหลายตัวควรใช้รูปแบบไหนจาก 5 แบบ — แยกคนสร้าง/คนตรวจ, แบ่งเลน, รันหลายรอบเลือกอันดีสุด,
สภาโมเดล, ทีมรีวิวเวอร์ — งานเล็กที่ไม่คุ้มจะไม่ถูกแตกให้หลาย agent

## Sources

The studies behind each pattern are listed in
[PATTERNS.md](skills/agent-orchestration-patterns/PATTERNS.md#sources). Round limits,
file-count thresholds and prompt templates are this skill's own additions.

## Add a new skill

```bash
mkdir -p skills/<new-skill> && $EDITOR skills/<new-skill>/SKILL.md
```

`SKILL.md` needs frontmatter with `name` (lowercase, hyphens, same as the folder) and
`description` (what it does, then "Use when …"). Keep it under 100 lines; move detail
into one reference file next to it.

## License

MIT
