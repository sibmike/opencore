# OpenCore

> The shareable engineering playbook your AI development actually needs.
> Bigger than `CLAUDE.md`. Synced across every project and every teammate's AI. Evolves as you ship.

---

## The problem

### Your dev playbook is glued to one project

`CLAUDE.md` is not the playbook. The playbook is many files — how you debug, how you ship, how you test, how you do infra, how you handle security, when you plan, when you don't.

Today, that playbook lives glued to one project:

```
your-startup/
├── CLAUDE.md                       300 lines, project-specific
├── docs/
│   ├── conventions.md
│   ├── debugging.md
│   ├── deployment.md
│   ├── post-deploy-checklist.md
│   ├── architecture.md
│   ├── workflows.md
│   └── ...                          20+ files
└── ERRATA.md                       47 hard-won lessons

weekend-project/   ──  CLAUDE.md  (rewrote from memory)
client-work/       ──  CLAUDE.md  (copy-pasted from somewhere)
```

Your startup playbook is dozens of files and dozens of lessons — and **none of it transfers**. It is glued to startup-specific tables, services, terms. You have no disciplined way to sync what worked there with what is happening here. Every new project starts fresh. Your practice never compounds.

### And when you bring on a teammate, the AI chaos begins

Now there are two AIs in the repo. Yours follows your user-guidelines. Theirs follows theirs. Your AI commits one convention; theirs commits the opposite. Tuesday's commit undoes Monday's. Wednesday's PR undoes Tuesday's. The repo never settles. It is a Greek tragedy with multiple AIs and many hands.

And the deeper pain: *how to work with AI* — when to plan, when to commit, when to ask, what to never let the agent touch — is tacit knowledge that lives in each senior dev's head. **It does not transfer to your junior.** Every AI-using dev on the team is reinventing AI-collaboration from scratch. When the senior leaves, that knowledge goes with them.

## The fix

One playbook. Forked once. Read by every repo. Evolves as you ship.

```
your-startup/      .opencore/    ← forked playbook: 10 practices + evolution loop + stack warm-start
                   CLAUDE.md     ← 3 lines: "read .opencore/CORE.md"
                   docs/         ← only what is truly project-specific stays

weekend-project/   .opencore/    ← same fork
                   CLAUDE.md     ← same 3 lines

client-work/       .opencore/    ← same fork
                   CLAUDE.md     ← same 3 lines
```

Project-specific things stay in the project. Universal practices live in the fork — **in the repo, not in anyone's head**. Your AI reads it. Your cofounder's AI reads it. Your new hire's AI reads it. Same rules for all of them. A sharper debugging step you discover today updates the fork tomorrow — and every repo, every teammate, every AI gets it.

---

## Install

```bash
# In any project:
git clone https://github.com/<TBD>/opencore.git .opencore
echo "Read .opencore/CORE.md before doing anything." > CLAUDE.md
```

That is the entire setup.

---

## Why fork it

**End the Greek tragedy with your team's AIs.** Two devs ship to the same repo with AI assistance and their agents fight each other — one follows your conventions, the other follows theirs, Tuesday's commit undoes Monday's. The deeper pain: *how to work with AI* (when to plan, when to commit, when to never let the agent touch something) is tacit knowledge in each senior dev's head that **does not transfer to your junior**. OpenCore in the repo gives every contributor's AI the same rules — and gives the team a durable place to codify the meta-skill of working with AI, instead of letting it die in private user-preference files.

**Stop being afraid of your cofounder's AI.** Right now you do not want their agent anywhere near your codebase — it has no clue about your repo's rules and will mess things up. With OpenCore in the repo, their agent reads the same playbook yours does. The fear evaporates because the rules stop living in your head.

**Code like a pro.** If you are a vibe coder, founder, or weekend hacker, this is the engineering playbook senior devs actually use — debugging, git discipline, testing, infra, security, planning. Fork it. Your AI agent reads it. You ship without the rookie mistakes.

**Make your seniority compound.** If you are already a senior engineer, this is how you align your coding agent to your preferences in a durable way. Codify what you know once. Sync across every repo. Stop re-teaching your agent from scratch every project.

---

## What is in it

10 practices, lifted from real production codebases:

| Practice | What it covers |
|---|---|
| [`practices/debugging.md`](practices/debugging.md) | Observe → hypothesize → verify → plan → implement. No trial-and-error. |
| [`practices/git-workflow.md`](practices/git-workflow.md) | Branching, commits, merges. Discipline that prevents history rot. |
| [`practices/session-protocol.md`](practices/session-protocol.md) | What to read at session start. How to end a session. |
| [`practices/anti-patterns.md`](practices/anti-patterns.md) | The mistakes everyone makes once. Listed so you make them zero times. |
| [`practices/testing-principles.md`](practices/testing-principles.md) | What tests prove. What they do not. Coverage philosophy. |
| [`practices/infra-management.md`](practices/infra-management.md) | Verify after deploy. Sequence changes. Do not drift. |
| [`practices/plan-mode.md`](practices/plan-mode.md) | When planning is mandatory. What makes a plan adequate. |
| [`practices/feature-lifecycle.md`](practices/feature-lifecycle.md) | The doc trail from spec to archive. |
| [`practices/security-principles.md`](practices/security-principles.md) | Auth layering, secrets, least-privilege defaults. |
| [`practices/architecture-principles.md`](practices/architecture-principles.md) | Purpose-fit storage. Layering. Typed exceptions. |

Plus 3 documentation standards ([`documentation/`](documentation/)) and the evolution loop that keeps the rest of this list calibrated to reality:

| Evolution | What it does |
|---|---|
| [`evolution/dreaming.md`](evolution/dreaming.md) | End-of-session reflection. Plan vs. reality vs. lesson. |
| [`evolution/delta-extraction.md`](evolution/delta-extraction.md) | Batch-process dreams into proposed updates. |
| [`evolution/core-update-gate.md`](evolution/core-update-gate.md) | The bar for promoting a lesson upstream. |

**Plus a stack-specific warm start.** If you are building a full-stack web app on AWS, fork [`stack-modules/full-stack-web-aws/v0.1.0/`](stack-modules/full-stack-web-aws/v0.1.0/) alongside CORE. It ships ready-made practices for AWS deploy sequencing (App Runner / Lambda / Amplify), CORS and frontend-backend wiring, IAM ordering, Python venv + conda + npm interop, UI review checklists. Knowledge you would otherwise pay for in 6 months of production incidents. Other archetypes (CLI tools, data pipelines, mobile) get scaffolded as the community contributes them.

Current version: **CORE v0.1.2**, **stack/full-stack-web-aws v0.1.0**. See [`CORE.md`](CORE.md) for the full index and [`CORE-CHANGELOG.md`](CORE-CHANGELOG.md) for lineage.

---

## Three ways to use it

| You | Do this | Get this |
|---|---|---|
| **Solo coder** | Fork once. Drop `.opencore/` into every repo. | One playbook your AI agent reads in every project. Lessons compound across your work. |
| **Small team** | Team forks. The fork lives in the shared repo. Everyone's AI reads it. | Convention chaos disappears — every contributor's agent follows the same rules regardless of which tool or user-prefs they use. Convergent lessons (same mistake in two or more projects) get promoted. |
| **OSS contributor** | Use upstream OpenCore directly. PR a delta when a lesson held up. | Your hard-won lesson becomes the default for the next person who forks. |

---

## How it works

Two loops keep your fork honest:

- **Loop 1** moves lessons between your sessions inside a single project.
- **Loop 2** moves lessons across projects — yours to other people's.

### Loop 1 — Your work compounds across sessions

Every session ends with a [*dream*](evolution/dreaming.md): a short reflection your agent writes about what you planned, what actually happened, and what changed. Dreams pile up. After five, a [*delta-extraction*](evolution/delta-extraction.md) pass looks for **convergence** — the same lesson surfacing in multiple independent sessions. Convergent lessons become PRs against your fork. The next session reads the updated practice. Same agent, same project — better playbook every week.

```
session ─► dream ─► [5 dreams] ─► delta ─► PR ─► your fork updates
                                                      │
                                                      ▼
                                              next session reads it
```

### Loop 2 — Your lessons compound across projects

When a drift in your fork held up across multiple of your projects (or your team's projects), it is a candidate for upstream. Pass the [gate](evolution/core-update-gate.md), it lands in CORE. The next person who forks gets your lesson baked in — without paying the tuition you paid.

```
your project's drift ─► review ─► upstream PR ─► CORE v0.1.3
                                                      │
                                                      ▼
                                  next person forks and starts smarter
```

### The rules

> **Individual dreams are noisy — convergence across dreams is signal.**
> — [`evolution/delta-extraction.md`](evolution/delta-extraction.md)

> **Nothing auto-applies. Everything is a PR.**
> — [`evolution/core-update-gate.md`](evolution/core-update-gate.md)

---

## FAQ

**How is this different from a `CLAUDE.md` / `.cursorrules` / `AGENTS.md`?**
Those are one file per repo, per tool. OpenCore is a full playbook — 10 practices, an evolution loop, and templates — that you fork once, share across every repo, and evolve through PRs.

**Does my code leave my machine?**
No. OpenCore is files in a git repo. Nothing phones home.

**Do I need a specific AI tool?**
No. The practices are agent-agnostic. Claude, Cursor, Copilot, plain humans — all read Markdown.

**Two of us code on the same repo with different AI tools and different user-preferences. How does this help?**
Both agents read the same `.opencore/` in the repo. Same conventions, same debugging protocol, same architecture rules — regardless of which agent or whose user-prefs file. The cross-agent chaos disappears at the repo boundary.

**I run 50 engineers across 12 repos. Should I use this?**
Probably not alone. OpenCore is built for solo and small-team workflows where one human can review every PR.

---

## License

MIT. Use it commercially. Fork it. Ship it.

## Contribute

Open a PR. Reference the project where the lesson held up. See [`evolution/core-update-gate.md`](evolution/core-update-gate.md) for the promotion bar.

---

If OpenCore saved you from rewriting the same playbook for the fifth time, star the repo. It is how other people find it.
