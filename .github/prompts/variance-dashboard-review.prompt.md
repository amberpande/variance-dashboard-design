---
mode: agent
description: Review dashboard code against the variance design system's anti-pattern checklist.
---

# Review this dashboard against the design system

Check the dashboard code in scope against the anti-pattern checklist in
[rules.md](../../.claude/skills/variance-dashboard-design/references/rules.md).
Report only real violations you can point at in the code — not stylistic
preferences, and not things the checklist doesn't cover.

For each finding give: the file and line, which rule it breaks, what goes wrong
for a reader, and the fix.

## Checklist

**Encoding**

- A sparkline (or any line) drawn through exactly two observations
- A percentage as the headline figure on small counts
- A relative change quoted for a metric that is already a rate, where
  percentage points are the honest figure
- Unsigned severity — a favourable move labelled a warning
- Ranking by raw magnitude rather than deviation from normal
- A truncated y-axis on counts
- A dual-axis chart (two y-scales on one plot)
- Color assigned by current rank, so filtering repaints the survivors
- A status color used as a series color, or the reverse
- More than 8 categorical hues, or generated/cycled hues past the palette

**Marks and chrome**

- Dashed gridlines or axis rules
- A border drawn around marks to separate them, instead of a surface gap or ring
- A number on every data point
- A label clipped by, or overflowing, its own mark
- `tabular-nums` on a large standalone number
- A display or serif face on the hero figure
- Text colored with the series color instead of a text token
- A chart container whose fixed height cuts off the x-axis band

**Theming**

- A hard-coded hex in a component instead of a `--vd-` token
- A color whose only definition sits inside a media query or `[data-theme]`
  block — undefined in the default "system" state
- A dashboard root with no explicit background token
- `--vd-rise` / `--vd-fall` redefined per metric rather than `upIsBad` passed
  at the call site

**Interaction and accessibility**

- Per-card filters, or a filter inside a chart card
- A skeleton flash on refetch instead of holding the previous render
- An expanded card still carrying `role="button"` while containing controls
- A tooltip as the only way to read a value — no table view twin
- Status carried by color alone, with no glyph and word
- Hit targets no bigger than the painted mark
- Motion that ignores `prefers-reduced-motion`
- No visible keyboard focus state

**Layout**

- `max-width` in `ch` on a heading whose font-size differs from its parent's —
  `ch` resolves against the element's own font-size and silently halves the
  measure
- Wide content (tables, code, charts) that scrolls the page body sideways
  instead of scrolling inside its own container

## Output

Group findings by severity: things that mislead a reader first, then things
that break in one theme or at one viewport, then polish. If nothing violates
the checklist, say so plainly rather than inventing findings.
