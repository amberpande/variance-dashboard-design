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
- **Return a headline field; never truncate prose.** The card preview needs one
  line. Taking the first sentence of the full explanation gives you "Invoice
  matching exceptions more than doubled" — which just restates the number
  already on the card. Have the model return a separate short `headline`
  alongside the full `explanation`, and put the headline on the card.
- **Take the follow-up.** The panel answers one question and then stops, but
  the reader's next question is already forming: *"which two suppliers?"*,
  *"show me the 121 invoices"*. A follow-up input under the explanation costs
  little — the semantic model is already there — and it is the difference
  between a report and a conversation.
- **Drill to the records.** Every number should reach the underlying rows. An
  explanation the reader cannot verify against the actual exceptions is asking
  for trust it hasn't earned.

---

## Speak the reader's language

This class of dashboard is read by ops, finance and compliance people, not by
analysts. Statistical notation on the card is honest and unhelpful.

Show **"4× normal"**, not `z 4.3`. Keep the precise figure in a tooltip for
whoever wants it. The same applies to sigma, p-values, confidence intervals and
percentile ranks — the ranking must be *explainable at a glance* by someone who
has never taken a statistics course, or it reads as arbitrary and the ordering
loses its authority.

Name things as the reader recognises them. A person manages *exceptions* and
*processes*, not `fct_exception_records` and `process_dim_key`.

---

## Close the loop

An explanation the reader cannot act on is a dead end. They read "the vendor
migration broke invoice matching", and then open a ticketing tool and retype it
by hand.

The reason panel should terminate in an action row:

- **Assign** to the owning team
- **Create a ticket**, pre-filled with the explanation, the affected count, and
  the query
- **Link an existing ticket** — and once linked, the card shows that ticket's
  status instead of re-asking tomorrow

This is usually the highest-value addition to a working explanation dashboard,
and it is almost always the last thing anyone builds.

---

## Earn continued trust

Recurring, expected variances are the fastest way to train users to ignore your
severity colors. A quarterly access recertification spikes every quarter; by
the third quarter a red CRITICAL card for a known cycle is actively harmful.

Give the reader **"Mark as expected — recurring"**. The variance still appears
and still ranks, but it renders as normal with a note saying why it was
suppressed and who suppressed it. Feed those marks back so the system learns
which spikes are business-as-usual.

Two more things that keep trust:

- **Show freshness.** When the data last loaded, always visible.
- **Flag thin evidence.** If a metric's sigma is computed from fewer than ~30
  days of history, mark its ranking provisional. A confident number built on
  thin evidence is worse than an honest hedge.

---

## The best session is often the one that didn't happen

For a dashboard people open daily, the strongest UX outcome most mornings is
that they don't need to. Push the briefing headline and top three movers as a
morning digest — email, Teams, Slack — each deep-linking into the expanded
card. On quiet days they read four lines and move on; the dashboard is for the
days something is actually wrong.

This is a product decision, not a styling one, but it follows directly from
"lead with the answer" and belongs in the same conversation.

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
- **Put the state in the URL** — the date pair, the sort, the expanded card.
  "Look at this" is the single most common thing a reader does with a finding,
  and without shareable URLs their only option is a screenshot. Cheap to build,
  disproportionate payoff.
- **Default to the reader's own scope.** Someone in Treasury does not care
  about HR Ops timesheets. Default to their team's metrics with "show all" one
  click away, and persist the choice.
- **Keyboard navigation for daily users.** `J`/`K` between cards, `Enter` to
  expand, `Escape` to collapse. For a tool someone opens every morning this
  compounds fast.
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

**❌ Statistical notation on the face of a card** — `z 4.3`, `σ`, `p < 0.05`.
→ "4× normal", with the precise figure on hover.

**❌ A card preview built by truncating the explanation.** The first sentence
usually restates the number already on the card. → A separate `headline` field
from the model.

**❌ An explanation with no action.** The reader has to retype the finding into
a ticket by hand. → Assign / create ticket / link ticket in the reason panel.

**❌ A known recurring spike flagged CRITICAL every cycle.** Trains readers to
ignore the color. → "Mark as expected — recurring".

**❌ State that lives only in component memory.** No shareable URL, so readers
screenshot findings. → Encode the date pair, sort, and expanded card in the URL.

**❌ A ranking presented with equal confidence regardless of history depth.**
→ Mark it provisional when sigma rests on a thin window.
