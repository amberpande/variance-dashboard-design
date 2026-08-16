---
name: variance-dashboard-design
description: Apply the Exception Variance briefing design system to a React dashboard — anomaly-ranked metric cards, dumbbell marks for two-date comparisons, signed severity, and an AI reason panel that streams in place. Use when restyling or building a metrics/variance/exceptions dashboard in React, when asked to make a dashboard "less boring", when adding a Cortex Analyst / LLM "why did this move" explanation panel, or when building metric cards, delta chips, status chips, sparkline or dumbbell marks, or a two-date comparison control.
---

# Variance dashboard design system

A design system for dashboards whose job is **"what changed, and why"** — not
"here are some numbers". It was built for exception counts compared across two
dates with an LLM explaining each variance, and it generalises to any
metric-plus-explanation dashboard.

Everything is CSS custom properties prefixed `--vd-`, so it drops into an
existing React app without fighting whatever styling the app already uses.

## Before you write anything

1. **Read the host app first.** Find its existing theme file, token set, or
   Tailwind config. If the app already has a design system, this one *fills gaps
   and never overrides*. Precedence is: the user's words → the app's existing
   system → this skill.
2. **Ask which styling approach the app uses** if it isn't obvious from the
   repo: CSS Modules, Tailwind, styled-components/emotion, or plain CSS. The
   tokens work in all four; only the component syntax changes. See
   `references/components.md` § Adapting to your styling layer.
3. **Never rename the `--vd-` prefix** to something shorter. It exists so these
   tokens cannot collide with the host app's variables.

## Porting procedure

Do these in order. Steps 1–2 are the whole visual identity; skip them and
nothing else will look right.

1. **Install the tokens.** Copy `references/tokens.css` into the app (e.g.
   `src/styles/variance-tokens.css`) and import it once at the root. It defines
   the complete light palette on bare `:root`, then redefines *only* the token
   values under `@media (prefers-color-scheme: dark)` guarded as
   `:root:not([data-theme="light"])`, and again under `:root[data-theme="dark"]`.
   Do not edit the structure — see `references/rules.md` § Theming, it is the
   part people break.
2. **Set the type roles.** Two roles, no webfonts: `--vd-sans` (system stack)
   carries everything read as data — headings, body, and every figure including
   the hero number. `--vd-mono` carries eyebrows, rank badges, z-scores,
   timestamps, axis ticks, and SQL. A display or serif face on a hero figure
   reads as off-brand decoration; do not add one.
3. **Build the primitives**, in this order — later ones compose earlier ones:
   `DeltaChip` → `StatusChip` → `Dumbbell` → `MetricCard` → `ReasonPanel` →
   `BriefingBand`. Source for each is in `references/components.md`.
4. **Wire the ranking.** Cards are ordered by how far each metric sits outside
   its own normal movement, not by a fixed list. This is the single change that
   makes the page feel alive. See `references/rules.md` § Ranking.
5. **Check against the anti-patterns** in `references/rules.md` before calling
   it done. Then render it and actually look at it — the rules catch encoding
   mistakes, not layout collisions.

## The rules that carry the design

These are load-bearing. Read `references/rules.md` for the reasoning; this is
the short form.

- **Two observations → a dumbbell, never a sparkline.** A line through two
  points invents a trend that isn't in the data. Gray dot = earlier, accent dot
  = later, connector colored by direction.
- **Absolute change leads; percentage is secondary.** On counts a percentage
  alone lies — 2→4 is "+100%". Render `+145` strong, `(+123%)` muted.
- **Severity is signed.** A large move in the *good* direction is just as far
  outside normal and should rank the same, but it is never a warning. It gets
  its own `improved` state.
- **Status never rides on color alone.** Every severity ships as glyph + word.
- **Lead with the answer.** The page opens with a written summary, not a grid.
- **Counts baseline at zero.** A truncated axis exaggerates the step you are
  trying to report honestly.
- **Explain the top movers before you're asked.** Pre-fetch reasons for the top
  three; the rest explain on click.
- **Hold the frame on refetch.** Previous render at reduced opacity — never a
  skeleton flash, never a layout jump.

## Files

| File | What's in it |
|---|---|
| `references/tokens.css` | The complete token set, drop-in, all three theme states |
| `references/components.md` | React source for every component + styling-layer adapters |
| `references/rules.md` | Why each rule exists, the ranking math, and the anti-pattern list |
