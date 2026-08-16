# Variance dashboard design system

A design system and Claude Code skill for dashboards whose job is **"what
changed, and why"** — built for exception counts compared across two dates,
with an LLM (Snowflake Cortex Analyst) explaining each variance in natural
language.

## What's here

| Path | What it is |
|---|---|
| [`.claude/skills/variance-dashboard-design/`](.claude/skills/variance-dashboard-design/) | The Claude Code skill, and the reference docs both assistants read. |
| [`.github/`](.github/) | The GitHub Copilot equivalent — instruction and prompt files. |
| [`prototype/exception-variance-dashboard.html`](prototype/exception-variance-dashboard.html) | A working, self-contained prototype. Open it in a browser — no build step. |

The reference docs under `.claude/skills/.../references/` are the single source
of truth. The Copilot files link to them rather than duplicating, so the rules
can't drift apart — **copy both directories** into the target app.

## The reference docs

- **`references/tokens.css`** — the complete token set, drop-in, correct in
  light, dark, and the default "system" theme state
- **`references/components.md`** — React + TypeScript source for every
  component, plus adapters for CSS Modules, Tailwind, and styled-components
- **`references/rules.md`** — the reasoning behind each rule, the ranking
  math, and the anti-pattern checklist

All tokens are prefixed `--vd-` so they can't collide with the host app's own
variables.

## Using it with Claude Code

Copy the skill folder into the target React repo:

```
<your-react-app>/.claude/skills/variance-dashboard-design/
```

Then ask Claude Code to apply it — the skill triggers on requests like
"restyle this metrics dashboard", "make this dashboard less boring", or
"add a reason panel to these metric cards". You can also invoke it directly
with `/variance-dashboard-design`.

## Using it with GitHub Copilot

Copy `.github/` into the target React repo alongside `.claude/`. It provides:

| File | What it does |
|---|---|
| `.github/prompts/variance-dashboard-design.prompt.md` | Type `/variance-dashboard-design` in Copilot Chat to apply the system. |
| `.github/prompts/variance-dashboard-review.prompt.md` | Type `/variance-dashboard-review` to audit existing dashboard code against the anti-pattern checklist. |
| `.github/instructions/variance-dashboard-design.instructions.md` | Always-on rules, auto-applied to files matching its `applyTo` glob. |

**Narrow the `applyTo` glob** in the instructions file to the paths where your
dashboard code actually lives. Copilot loads instruction files in full for
every matching request — unlike a Claude skill it has no progressive
disclosure, so scoping it tightly matters.

Prompt files are a VS Code Copilot Chat feature; enable
`chat.promptFiles` if the slash commands don't appear.

## The prototype

Open `prototype/exception-variance-dashboard.html` directly in a browser.
Everything is clickable: change the two dates, switch the ordering, and expand
any card to see the breakdown by reason code and the streamed explanation.

The demo data tells one deliberate story — a vendor master migration on 12 Aug
breaks invoice matching and cascades into payment reconciliation, an unrelated
access recertification opens on 14 Aug, and KYC quietly improves. Two causes
plus one improvement, because a human scanning the grid would likely blame
everything on a single incident.

Exception counts, teams and reason codes are illustrative.

## The ideas that carry it

- **Two observations are not a trend.** With exactly two dates, a sparkline
  invents data you didn't measure. A dumbbell (● ——— ●) states only what was
  measured, and its connector length *is* the variance.
- **Absolute change leads; percentage follows.** On counts a percentage alone
  lies — 2→4 is "+100%".
- **Severity is signed.** A large move in the good direction is just as far
  outside normal and should rank the same, but it is never a warning.
- **Rank by deviation from normal**, not raw magnitude — otherwise the
  highest-volume metric sits on top forever and the small process that tripled
  is buried.
- **Lead with the answer.** The page opens with a written summary, not a grid.
- **Explain the top movers before you're asked.**

See [`references/rules.md`](.claude/skills/variance-dashboard-design/references/rules.md)
for the full reasoning.
