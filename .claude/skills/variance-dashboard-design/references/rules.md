# Rules & anti-patterns

Why each decision exists. The components encode these; if you change a
component, check it still honours the rule underneath.

---

## Form: two observations are not a trend

A dashboard comparing **two dates** has exactly two observations per metric. A
sparkline through them draws a line that implies you measured the days between.
You didn't. The dumbbell states only what was measured, and its connector
length *is* the variance — which makes a grid of them scannable in a single
pass.

| What you have | Form |
|---|---|
| Two observations | **Dumbbell** — two dots + connector |
| Two observations across many sub-categories | **Dumbbell rows on a shared scale** |
| A daily series over a window | Line + area wash, endpoint dot |
| One number that is the whole story | Stat tile / hero figure — no chart |
| Change over time, one metric, many periods | Line; columns if ≤ 14 points |

If you *do* have the days between two dates, plot them as a **secondary**
chart — but only there, and label the causal date with a solid colored rule
(never a dashed one; dashes read as "projection"). A gridline is gray, so a
colored rule cannot be mistaken for one.

---

## Numbers: absolute leads, percentage follows

On counts, a percentage alone lies. 2 → 4 is "+100%" and is noise; 180 → 260 is
"+44%" and is an incident. Render the absolute change in strong type and the
percentage muted beside it.

The inverse holds for rates already expressed as percentages (conversion,
margin): there the **percentage-point** change is the honest figure, not the
relative change. `2.95% → 2.41%` is `−0.54pp`, not `−18.3%` — quote the relative
change only when you say what it's relative to.

---

## Severity is signed

A metric that moved 3 sigma in the *favourable* direction is exactly as far
outside normal as one that moved 3 sigma the wrong way. It should **rank the
same** — you want to know about it — but it must never be labelled a warning.

That's why `severity()` takes `adverse` and has a distinct `improved` state.
Getting this wrong is the most common bug when porting: a naive
`z >= 1 ? "warning" : "steady"` will paint your one piece of good news red.

Also: `--vd-rise` and `--vd-fall` mean **adverse** and **favourable**, not "up"
and "down". For exception counts up is bad; for revenue up is good. Pass
`upIsBad` at the call site — never redefine the tokens per metric, or the same
color will mean opposite things on one page.

---

## Ranking: deviation from normal, not raw magnitude

Ordering by raw variance permanently puts your highest-volume metric on top and
buries the small process that tripled. Rank by how far the change sits outside
each metric's **own** normal movement:

```
sigma_m = STDDEV(daily_count_change) over a trailing window (60–90 days)
z_m     = |count_b − count_a| / sigma_m
```

Compute `sigma` in the warehouse, not the client — it's a window function over
data you already have:

```sql
SELECT
  process_name,
  STDDEV(daily_count - LAG(daily_count) OVER (
    PARTITION BY process_name ORDER BY raised_date
  )) AS sigma
FROM daily_exception_counts
WHERE raised_date >= DATEADD('day', -90, CURRENT_DATE())
GROUP BY 1;
```

Cache it — it changes slowly, so a nightly refresh is plenty.

Two things follow from ranking this way, and both are the point:

- **The order changes with the dates.** The page looks different every time it's
  opened, which is most of what kills "boring".
- **The rank is explainable.** Show `01 · z 4.3` in mono on the card. The
  ordering justifies itself instead of looking arbitrary.

**Sigma is the one piece of real backend work this design needs.** Without it,
fall back to ranking by absolute change and say so in the UI — but plan to add
it, because it's what makes the ranking trustworthy.

---

## The explanation is the product

If the dashboard has an LLM that explains variances, that is the most valuable
thing on the page and must not look like every other button.

- **Pre-fetch the top three.** Their reasons are already on the card as a
  one-line preview. The rest explain on click. Users learn the page is
  *already thinking*.
- **Stream the text.** A multi-second roundtrip behind a spinner feels broken;
  the same wait with words arriving feels like thinking.
- **Cache by `(metric, dateA, dateB, filters)`.** Users toggle back and forth
  constantly and will re-request identical answers.
- **Show the query.** Trust in the explanation is the whole product. Collapsed
  `<details>`, with the real bound parameters substituted in.
- **State the scope.** Grain, dimensions checked, confidence. An explanation
  that doesn't say what it looked at can't be audited.
- **Collect the signal.** A Yes/No on each answer is the cheapest training data
  you will ever get.
- **Design the failure.** "Cortex couldn't answer that" needs a real state, not
  a stack trace. Say what it can answer about.

---

## Theming

Three states, not two: explicit light, explicit dark, and the default "system"
setting which stamps **nothing** on the root element. Most viewers are in the
un-stamped state, where only `prefers-color-scheme` separates the two.

- Bare `:root` defines the **complete** palette.
- The media query redefines **only values**, guarded `:root:not([data-theme="light"])`.
- `:root[data-theme="dark"]` redefines them again.
- Style components through tokens — **never** put a color's only definition
  inside a media or `[data-theme]` block. That token is undefined in the
  un-stamped state, and you get one theme's text on the other theme's ground.
- The dashboard root sets an explicit `background` from a token. A transparent
  root borrows whatever the host app paints behind it.

Before shipping, grep the stylesheet for any color declared only inside a media
or `[data-theme]` block. That is the classic unreadable-dashboard bug.

---

## Interaction

- **One filter row, above everything it scopes.** Never per-card filters. Every
  card re-renders against the same slice so the numbers always agree.
- **Refetch holds the frame.** Previous render at reduced opacity — no skeleton
  flash, no layout jump.
- **Expand in place**, full row, rather than routing to a detail page. Context
  is what makes a variance readable.
- **The collapsed card is a button; the expanded card is a region.** An open
  card contains its own controls and must not keep `role="button"`.
- **Every charted value is reachable without hovering** — a table view twin
  behind a toggle. Tooltips enhance; they never gate.
- **Hit targets exceed the mark.** An 8px dot needs ~24px of transparent hit
  area, or a nearest-point layer.
- **Escape collapses; focus returns** to a sensible card.

---

## Anti-patterns

Check every screen against this list. If it matches an entry, it's wrong.

**❌ A sparkline for a two-date comparison.** Invents a trend you didn't
measure. → Dumbbell.

**❌ A percentage as the headline figure on small counts.** "+100%" on 2→4.
→ Absolute change leads.

**❌ Unsigned severity.** A big improvement painted as a warning.
→ `severity(z, adverse)` with an `improved` state.

**❌ Ranking by raw magnitude.** Buries the small metric that tripled under the
big one that wobbled. → Rank by z.

**❌ A truncated y-axis on counts.** Exaggerates the step you're reporting.
→ Baseline at zero.

**❌ Status carried by color alone.** → Always glyph + word.

**❌ Reusing a status color as a series color** (or vice versa). Status means
good/bad; categorical means identity. Keep them disjoint.

**❌ A dual-axis chart.** Two y-scales invent a correlation that isn't in the
data. → Two charts, small multiples, or index both to a common base.

**❌ Recolor-on-filter.** Color follows the entity, never its current rank — a
reader who learned "Treasury is blue" is misled when filtering repaints it.

**❌ Dashed gridlines.** Reads as "projection" or "threshold" when it's just a
grid. → Solid hairlines, one shade off the surface.

**❌ A border drawn around marks to separate them.** → A 2px gap in the surface
color, and a 2px surface ring on overlapping dots.

**❌ A number on every data point.** → Label the endpoint, the extreme, or the
one series the story is about.

**❌ `tabular-nums` on a large standalone number.** Equal-width digits make
`121` look loose at display sizes. → Proportional for hero and card values;
tabular only in columns.

**❌ A display or serif face on the hero figure.** Reads as off-brand
decoration. → The same sans as everything else.

**❌ A skeleton flash on refetch.** → Hold the previous render at reduced opacity.

**❌ `max-width` in `ch` on a headline whose font-size differs from its
parent's.** `ch` resolves against the element's own font-size — set it on the
heading, or use px. Silently halves your line length otherwise.

**❌ Per-card filters.** → One row above everything it scopes.

**❌ A tooltip as the only way to read a value.** → Table view twin.
