---
applyTo: "**/*.{tsx,jsx,css,scss}"
description: Always-on design rules for dashboard and metric-display code.
---

# Variance dashboard design rules

> **Narrow the `applyTo` glob above** to the paths where your dashboard code
> actually lives (e.g. `src/dashboard/**/*.{tsx,css}`). Copilot loads this file
> in full for every matching request — unlike a Claude skill it has no
> progressive disclosure, so scope it tightly and keep it short.

When writing or editing dashboard, metric-card, or chart code in this app,
follow these. Full reasoning and the anti-pattern checklist:
[rules.md](../../.claude/skills/variance-dashboard-design/references/rules.md).
Component source:
[components.md](../../.claude/skills/variance-dashboard-design/references/components.md).

## Color and theming

- Use the `--vd-` tokens from `tokens.css`. Never hard-code a hex value in a
  component, and never rename the prefix.
- A color's definition must exist on bare `:root`. Media queries and
  `[data-theme]` blocks redefine **values only**. A color defined solely inside
  one of those blocks is undefined in the default "system" theme state — that
  is the classic unreadable-dashboard bug.
- `--vd-rise` / `--vd-fall` mean **adverse** / **favourable**, not "up" /
  "down". Pass `upIsBad` at the call site; never redefine the tokens per
  metric, or the same color will mean opposite things on one page.
- Status colors (`good`/`warning`/`serious`/`critical`) are reserved. Never
  reuse one as a series color, or a series color as a status.

## Encoding

- **Exactly two observations → a `Dumbbell`, never a sparkline.** A line
  through two points implies you measured the days between. You didn't.
- **Absolute change leads, percentage follows.** Render `+145` in strong type
  and `(+123%)` muted. On counts a percentage alone lies.
- For metrics that are already rates (conversion, margin), quote the
  **percentage-point** change, not the relative change.
- **Severity is signed** — `severity(z, adverse)`. A big favourable move ranks
  the same but returns `improved`, never `warning`.
- **Rank by z** (deviation from the metric's own normal movement), not raw
  magnitude — otherwise the highest-volume metric sits on top forever.
- **Counts baseline at zero.** A truncated axis exaggerates the change.
- Never a dual-axis chart. Two measures of different scale → two charts, small
  multiples, or index both to a common base.

## Marks and chrome

- Bars ≤ 24px thick, 4px rounded data-end, square at the baseline.
- Lines 2px; markers ≥ 8px; area fills ~10% opacity, never a saturated block.
- Gridlines and axes: solid hairlines one shade off the surface. **Never
  dashed** — dashes read as "projection".
- Separate touching marks with a 2px gap in the surface color and overlapping
  dots with a 2px surface ring. Never draw a border around a mark.
- Label selectively — the endpoint, the extreme, or the one series the story is
  about. Never a number on every point.
- Text wears text tokens, never the series color. Identity comes from the
  colored mark beside the text.
- `tabular-nums` only where digits align in columns. Never on a large
  standalone number — it makes `121` look loose.
- The hero figure uses the same sans as everything else. No display or serif
  face.

## Language and workflow

- **No statistical notation on the face of a card.** Show "4× normal", not
  `z 4.3`; keep the precise figure in a tooltip. This is read by ops and
  finance people, not analysts — a ranking they can't explain at a glance
  reads as arbitrary.
- **A card preview comes from a `headline` field the model returns**, never
  from truncating the explanation. The first sentence usually just restates the
  number already on the card.
- **The reason panel ends in an action** — assign, create ticket, link ticket.
  An explanation the reader can't act on is a dead end.
- **Recurring expected variances can be marked as such.** They still appear and
  still rank, but render as normal with a note saying why. Never hide them —
  hiding makes the dashboard lie.
- **Take a follow-up question** under the explanation, and let every number
  drill through to the underlying rows.

## Interaction and accessibility

- One filter row above everything it scopes. Never per-card filters.
- Encode the date pair, sort, and expanded card in the **URL** so findings are
  shareable. Without it, readers share by screenshot.
- Default to the reader's own team's metrics, with "show all" one click away.
- Keyboard navigation for daily users: `J`/`K` between cards, `Enter` to
  expand, `Escape` to collapse.
- Show data freshness, and mark a ranking provisional when its sigma rests on
  fewer than ~30 days of history.
- Refetch holds the previous render at reduced opacity — no skeleton flash, no
  layout jump.
- A collapsed card is `role="button"`; an expanded one is `role="region"`. An
  open card contains its own controls and must not keep the button role.
- Every charted value is reachable without hovering — ship a table view twin.
  Tooltips enhance, never gate.
- Status always ships as glyph + word, never color alone.
- Hit targets exceed the mark: an 8px dot needs ~24px of hit area.
- Honour `prefers-reduced-motion` — show streamed text immediately, drop
  transforms.
