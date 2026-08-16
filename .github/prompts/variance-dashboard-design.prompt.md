---
mode: agent
description: Apply the variance dashboard design system to a React dashboard — anomaly-ranked metric cards, dumbbell marks for two-date comparisons, signed severity, and a streaming reason panel.
---

# Apply the variance dashboard design system

Restyle or build a metrics dashboard in this React app using the design system
documented in this repo. It is built for dashboards whose job is **"what
changed, and why"** — not "here are some numbers".

## Source of truth

Read these before writing code. They are the specification; this prompt is only
the procedure.

- [Tokens](../../.claude/skills/variance-dashboard-design/references/tokens.css) —
  the complete `--vd-` token set, all three theme states
- [Components](../../.claude/skills/variance-dashboard-design/references/components.md) —
  React + TypeScript source for every component, plus adapters for CSS Modules,
  Tailwind, and styled-components
- [Rules & anti-patterns](../../.claude/skills/variance-dashboard-design/references/rules.md) —
  why each decision exists, the ranking math, the checklist

## Before writing anything

1. **Read the host app first.** Find its existing theme file, token set, or
   Tailwind config. If the app already has a design system, this one *fills
   gaps and never overrides*. Precedence: the user's words → the app's existing
   system → this system.
2. **Confirm the styling layer** if the repo doesn't make it obvious — CSS
   Modules, Tailwind, styled-components/emotion, or plain CSS. The tokens work
   in all four; only the component syntax changes.
3. **Never rename the `--vd-` prefix.** It exists so these tokens cannot
   collide with the host app's variables.

## Procedure

Do these in order. Steps 1–2 are the whole visual identity.

1. **Install the tokens.** Copy `tokens.css` into the app (e.g.
   `src/styles/variance-tokens.css`) and import it once at the root. Do not
   edit its structure — the three-state theme pattern is load-bearing and is
   the part people break.
2. **Set the type roles.** Two roles, no webfonts. `--vd-sans` carries
   everything read as data, including the hero figure. `--vd-mono` carries
   eyebrows, rank badges, z-scores, axis ticks, and SQL. Never put a display or
   serif face on a hero figure.
3. **Build the primitives in this order** — later ones compose earlier ones:
   `DeltaChip` → `StatusChip` → `Dumbbell` → `MetricCard` → `ReasonPanel` →
   `BriefingBand`.
4. **Wire the ranking.** Cards are ordered by how far each metric sits outside
   its own normal movement, not by a fixed list. This is the single change that
   makes the page feel alive. It needs a per-metric `sigma` — see the Ranking
   section of the rules file; it is the one piece of real backend work.
5. **Check the anti-pattern list** in the rules file, then render it and
   actually look at it. The checklist catches encoding mistakes, not layout
   collisions.

## The rules that carry the design

Load-bearing. Full reasoning is in the rules file.

- **Two observations → a dumbbell, never a sparkline.** A line through two
  points invents a trend that isn't in the data.
- **Absolute change leads; percentage is secondary.** On counts a percentage
  alone lies — 2→4 is "+100%".
- **Severity is signed.** A large move in the good direction ranks the same but
  is never a warning — it gets its own `improved` state.
- **Rank by deviation from normal**, not raw magnitude.
- **Status never rides on color alone** — always glyph + word.
- **Lead with the answer** — a written summary above the grid, not a bare grid.
- **Counts baseline at zero.**
- **Pre-fetch reasons for the top three movers**; the rest explain on click.
- **Hold the frame on refetch** — previous render at reduced opacity, never a
  skeleton flash.

## Output

Report which files you created or modified, which styling layer you targeted,
and anything from the checklist you could not satisfy and why.
