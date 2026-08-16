# Components

React + TypeScript source for the system. No external dependencies — charts are
inline SVG, motion is CSS. Every color comes from a `--vd-` token, so a
component is correct in both themes by construction.

Build them in this order; later ones compose earlier ones.

---

## Adapting to your styling layer

The tokens are plain CSS custom properties and work unchanged in all four
setups. Only the class syntax below changes.

**Plain CSS / CSS Modules** — what the source below is written for. Put each
component's CSS in a sibling `.module.css` and swap `className="vd-card"` for
`className={s.card}`.

**Tailwind** — map the tokens once in `tailwind.config.js`, then use utilities:

```js
// tailwind.config.js
theme: {
  extend: {
    colors: {
      plane:   "var(--vd-plane)",
      surface: "var(--vd-surface)",
      sunk:    "var(--vd-surface-sunk)",
      ink:     { DEFAULT: "var(--vd-ink)", 2: "var(--vd-ink-2)", 3: "var(--vd-ink-3)" },
      hairline:{ DEFAULT: "var(--vd-hairline)", 2: "var(--vd-hairline-2)" },
      accent:  "var(--vd-accent)",
      now:     "var(--vd-now)",
      then:    "var(--vd-then)",
      rise:    "var(--vd-rise)",
      fall:    "var(--vd-fall)",
    },
    fontFamily: { sans: "var(--vd-sans)", mono: "var(--vd-mono)" },
  },
}
```

Keep `tokens.css` imported even with Tailwind — it is what makes the three
theme states resolve. Do NOT put the dark values in Tailwind's `dark:` variant
instead; that only handles the class strategy and breaks the un-stamped
"system" state.

**styled-components / emotion** — reference tokens directly in template
literals: ``background: var(--vd-surface);``. Do not lift them into a JS theme
object; you lose the automatic OS-preference response.

---

## `variance.ts` — the logic the design depends on

```ts
export type Severity = "critical" | "serious" | "warning" | "improved" | "steady";

/**
 * Severity is SIGNED. A large move in the favourable direction is just as far
 * outside normal movement and should rank the same — but it is good news and
 * must never be labelled a warning.
 *
 * @param z        magnitude of the change in units of the metric's own normal
 *                 day-to-day movement (see `zScore`)
 * @param adverse  whether the change went in the bad direction
 */
export function severity(z: number, adverse: boolean): Severity {
  if (!adverse) return z >= 1.0 ? "improved" : "steady";
  if (z >= 2.5) return "critical";
  if (z >= 1.7) return "serious";
  if (z >= 1.0) return "warning";
  return "steady";
}

export const SEVERITY_GLYPH: Record<Severity, string> = {
  critical: "◆", serious: "▲", warning: "●", improved: "▼", steady: "—",
};

export const SEVERITY_WORD: Record<Severity, string> = {
  critical: "Critical", serious: "Serious", warning: "Warning",
  improved: "Improved", steady: "Normal",
};

export const SEVERITY_STRIPE: Record<Severity, string> = {
  critical: "var(--vd-critical)",
  serious:  "var(--vd-serious)",
  warning:  "var(--vd-warning)",
  improved: "var(--vd-good)",
  steady:   "var(--vd-hairline-2)",
};

/**
 * How far this change sits outside the metric's OWN normal movement.
 * `sigma` is the standard deviation of that metric's period-over-period change,
 * computed over a trailing window — see rules.md § Ranking. Without it you can
 * only sort by raw magnitude, which permanently buries small-volume metrics.
 */
export function zScore(a: number, b: number, sigma: number): number {
  return sigma > 0 ? Math.abs(b - a) / sigma : 0;
}

/** Did this metric move in the bad direction? */
export function isAdverse(a: number, b: number, upIsBad: boolean): boolean {
  return upIsBad ? b > a : b < a;
}

/** Rank by deviation from normal — the ordering that makes the page feel alive. */
export function rankByVariance<T extends { z: number }>(rows: T[]): T[] {
  return [...rows].sort((x, y) => y.z - x.z);
}

export const formatCount = (v: number) => Math.round(v).toLocaleString();
```

---

## `hooks.ts`

```ts
import { useEffect, useState } from "react";

export function usePrefersReducedMotion(): boolean {
  const [reduce, setReduce] = useState(
    () => typeof window !== "undefined" &&
          window.matchMedia("(prefers-reduced-motion: reduce)").matches
  );
  useEffect(() => {
    const mq = window.matchMedia("(prefers-reduced-motion: reduce)");
    const onChange = () => setReduce(mq.matches);
    mq.addEventListener("change", onChange);
    return () => mq.removeEventListener("change", onChange);
  }, []);
  return reduce;
}

/**
 * Reveals text word by word. A multi-second LLM roundtrip behind a spinner
 * feels broken; the same wait with text arriving feels like thinking.
 * Honours reduced-motion by showing the full text immediately.
 */
export function useStreamedText(text: string, intervalMs = 24) {
  const reduce = usePrefersReducedMotion();
  const [shown, setShown] = useState("");

  useEffect(() => {
    if (reduce) { setShown(text); return; }
    setShown("");
    const words = text.split(" ");
    let i = 0;
    const id = setInterval(() => {
      i += 1;
      setShown(words.slice(0, i).join(" "));
      if (i >= words.length) clearInterval(id);
    }, intervalMs);
    return () => clearInterval(id);
  }, [text, reduce, intervalMs]);

  return { shown, done: shown === text };
}
```

If your reason text arrives as a real token stream from the backend, drop
`useStreamedText` and append chunks to state as they land — the visual result is
the same and the timing is honest.

---

## `DeltaChip`

Absolute change leads, percentage follows. On counts a percentage alone lies:
2→4 is "+100%" and means nothing.

```tsx
type DeltaChipProps = { a: number; b: number; upIsBad?: boolean; suffix?: string };

export function DeltaChip({ a, b, upIsBad = true, suffix }: DeltaChipProps) {
  const d = b - a;
  const flat = d === 0;
  const adverse = upIsBad ? d > 0 : d < 0;
  const tone = flat ? "flat" : adverse ? "bad" : "good";

  return (
    <span className={`vd-delta vd-delta--${tone}`}>
      <span className="vd-delta__arrow" aria-hidden="true">
        {flat ? "→" : d > 0 ? "▲" : "▼"}
      </span>
      <span>{d > 0 ? "+" : d < 0 ? "−" : "±"}{Math.abs(d).toLocaleString()}</span>
      {a > 0 && !flat && (
        <span className="vd-delta__pct">
          ({d > 0 ? "+" : "−"}{Math.abs((d / a) * 100).toFixed(0)}%)
        </span>
      )}
      {suffix && <span className="vd-delta__pct">{suffix}</span>}
    </span>
  );
}
```

```css
.vd-delta {
  display: inline-flex; align-items: center; gap: 5px;
  font-size: 12.5px; font-weight: 600; font-variant-numeric: tabular-nums;
  padding: 3px 8px; border-radius: var(--vd-radius-chip); white-space: nowrap;
}
.vd-delta--bad  { color: var(--vd-bad-text);  background: rgba(208,59,59,0.10); }
.vd-delta--good { color: var(--vd-good-text); background: rgba(12,163,12,0.10); }
.vd-delta--flat { color: var(--vd-ink-2);     background: var(--vd-surface-sunk); }
.vd-delta__arrow { font-size: 10px; line-height: 1; }
.vd-delta__pct   { font-weight: 450; color: var(--vd-ink-3); font-size: 11.5px; }
```

---

## `StatusChip`

Glyph **and** word, always. A status color never carries meaning alone.

```tsx
import { Severity, SEVERITY_GLYPH, SEVERITY_WORD } from "./variance";

export function StatusChip({ severity }: { severity: Severity }) {
  return (
    <span className="vd-status" data-sev={severity}>
      <span className="vd-status__glyph" aria-hidden="true">{SEVERITY_GLYPH[severity]}</span>
      <span>{SEVERITY_WORD[severity]}</span>
    </span>
  );
}
```

```css
.vd-status {
  display: inline-flex; align-items: center; gap: 5px;
  font-family: var(--vd-mono); font-size: 9.5px; letter-spacing: 0.08em;
  text-transform: uppercase; font-weight: 600; color: var(--vd-ink-2);
}
.vd-status__glyph { font-size: 10px; line-height: 1; }
.vd-status[data-sev="critical"] .vd-status__glyph { color: var(--vd-critical); }
.vd-status[data-sev="serious"]  .vd-status__glyph { color: var(--vd-serious); }
.vd-status[data-sev="warning"]  .vd-status__glyph { color: var(--vd-warning); }
.vd-status[data-sev="improved"] .vd-status__glyph { color: var(--vd-good); }
.vd-status[data-sev="steady"]   .vd-status__glyph { color: var(--vd-ink-3); }
```

---

## `Dumbbell` — the signature mark

**Use this whenever you have exactly two observations.** A line through two
points invents a trend that isn't in the data; a dumbbell states only what you
measured. Connector length *is* the variance, so a grid of these is scannable
in one pass.

```tsx
type DumbbellProps = {
  a: number; b: number;
  labelA?: string; labelB?: string;   // e.g. "10 Aug" / "17 Aug"
  upIsBad?: boolean;
  showEnds?: boolean;
};

export function Dumbbell({
  a, b, labelA, labelB, upIsBad = true, showEnds = true,
}: DumbbellProps) {
  const max = Math.max(a, b, 1);
  const PAD = 5;                                   // % inset so end dots never clip
  const pos = (v: number) => PAD + (v / max) * (100 - PAD * 2);
  const xa = pos(a), xb = pos(b);
  const adverse = upIsBad ? b > a : b < a;
  const connector =
    a === b ? "var(--vd-hairline-2)" : adverse ? "var(--vd-rise)" : "var(--vd-fall)";

  return (
    <div className="vd-db">
      <div className="vd-db__track">
        <div
          className="vd-db__line"
          style={{
            left: `${Math.min(xa, xb)}%`,
            width: `${Math.abs(xb - xa)}%`,
            background: connector,
          }}
        />
        <div className="vd-db__dot" style={{ left: `${xa}%`, background: "var(--vd-then)" }} />
        <div className="vd-db__dot" style={{ left: `${xb}%`, background: "var(--vd-now)" }} />
      </div>
      {showEnds && (
        <div className="vd-db__ends">
          <span>{labelA} <b>{a.toLocaleString()}</b></span>
          <span>{labelB} <b>{b.toLocaleString()}</b></span>
        </div>
      )}
    </div>
  );
}
```

```css
.vd-db__track { position: relative; height: 16px; }
.vd-db__line  { position: absolute; top: 7px; height: 2px; border-radius: 1px; }
.vd-db__dot   {
  position: absolute; top: 4px; width: 8px; height: 8px; border-radius: 50%;
  transform: translateX(-50%);
  /* the 2px surface ring keeps dots legible where they nearly touch —
     never draw a border to separate marks */
  box-shadow: 0 0 0 2px var(--vd-surface);
}
.vd-db__ends {
  display: flex; justify-content: space-between; margin-top: 3px;
  font-family: var(--vd-mono); font-size: 10px; color: var(--vd-ink-3);
}
.vd-db__ends b { color: var(--vd-ink-2); font-weight: 600; }
```

**Scaling.** Each card scales `0 … max(a,b)` so one dot always pins right and
the gap is proportional to the relative change. For a **breakdown table** of
several rows (by reason code, by team) use a **shared** scale across all rows
instead — that is the point of the breakdown, and it makes the outlier row
obvious.

---

## `MetricCard`

```tsx
import { DeltaChip } from "./DeltaChip";
import { StatusChip } from "./StatusChip";
import { Dumbbell } from "./Dumbbell";
import { Severity, SEVERITY_STRIPE } from "./variance";

type MetricCardProps = {
  label: string; owner?: string;
  a: number; b: number; labelA: string; labelB: string;
  z: number; severity: Severity; rank?: number;
  /* A short headline RETURNED BY THE MODEL, not the explanation truncated —
     the first sentence of an explanation usually just restates the number
     already on the card. Pre-fetched for the top movers only. */
  reasonHeadline?: string;
  expanded?: boolean;
  onExpand: () => void;
  children?: React.ReactNode;      // the detail panel, when expanded
};

export function MetricCard(p: MetricCardProps) {
  const interactive = !p.expanded;
  return (
    <div
      className="vd-card"
      style={{ ["--vd-stripe" as string]: SEVERITY_STRIPE[p.severity] }}
      aria-expanded={p.expanded}
      /* Only the COLLAPSED card is a button. Once open it contains its own
         controls, so it must not keep claiming that role. */
      role={interactive ? "button" : "region"}
      tabIndex={interactive ? 0 : -1}
      aria-label={interactive ? `Explain the variance in ${p.label}` : `${p.label} detail`}
      onClick={interactive ? p.onExpand : undefined}
      onKeyDown={(e) => {
        if (interactive && (e.key === "Enter" || e.key === " ")) {
          e.preventDefault(); p.onExpand();
        }
      }}
    >
      <div className="vd-card__top">
        <div>
          <div className="vd-card__name">{p.label}</div>
          {p.owner && <div className="vd-card__owner">{p.owner}</div>}
        </div>
        {/* Plain language, not statistical notation — this is read by ops and
            finance people. Precise figure goes in the tooltip. */}
        <span className="vd-card__rank" title={`z-score ${p.z.toFixed(2)} — deviations from this metric's normal movement`}>
          {p.rank != null && `${String(p.rank).padStart(2, "0")} · `}
          {p.z >= 1 ? `${p.z.toFixed(1)}× normal` : "normal"}
        </span>
      </div>

      <div className="vd-card__value">{p.b.toLocaleString()}</div>

      <div className="vd-card__meta">
        <DeltaChip a={p.a} b={p.b} />
        <StatusChip severity={p.severity} />
      </div>

      {!p.expanded && (
        <>
          <Dumbbell a={p.a} b={p.b} labelA={p.labelA} labelB={p.labelB} />
          {p.reasonHeadline ? (
            <div className="vd-card__preview">
              <span className="vd-card__tag">Why</span>
              <span>{p.reasonHeadline}</span>
            </div>
          ) : (
            <div className="vd-card__cta">Explain the variance →</div>
          )}
        </>
      )}

      {p.expanded && p.children}
    </div>
  );
}
```

```css
.vd-card {
  position: relative;
  background: var(--vd-surface);
  border: 1px solid var(--vd-hairline);
  border-radius: var(--vd-radius-card);
  padding: 15px 16px 13px 19px;
  display: flex; flex-direction: column; gap: 9px;
  cursor: pointer; overflow: hidden; color: var(--vd-ink);
  transition: border-color 140ms ease, box-shadow 140ms ease, transform 140ms ease;
}
/* severity stripe — state encoded in form, not only in a number */
.vd-card::before {
  content: ""; position: absolute; left: 0; top: 0; bottom: 0;
  width: 3px; background: var(--vd-stripe, var(--vd-hairline-2));
}
.vd-card:hover { border-color: var(--vd-hairline-2); box-shadow: var(--vd-shadow); }
@media (prefers-reduced-motion: no-preference) {
  .vd-card:hover { transform: translateY(-1px); }
}
.vd-card[aria-expanded="true"] {
  grid-column: 1 / -1;            /* expand in place, full row */
  cursor: default; transform: none;
  border-color: var(--vd-accent-ring); box-shadow: var(--vd-shadow);
}

.vd-card__top { display: flex; align-items: flex-start; justify-content: space-between; gap: 12px; }
/* reserve two lines so values and dumbbells share a baseline across a row
   whether or not the metric name wraps */
.vd-card__name  { font-size: 12.5px; font-weight: 560; color: var(--vd-ink-2); min-height: 38px; }
.vd-card[aria-expanded="true"] .vd-card__name { min-height: 0; }
.vd-card__owner {
  font-family: var(--vd-mono); font-size: 9.5px; letter-spacing: 0.07em;
  text-transform: uppercase; color: var(--vd-ink-3); margin-top: 3px;
}
.vd-card__rank {
  font-family: var(--vd-mono); font-size: 10.5px; color: var(--vd-ink-3);
  letter-spacing: 0.04em; flex: none; padding-top: 1px;
}
/* proportional figures — tabular-nums makes a large standalone number look loose */
.vd-card__value { font-size: 30px; font-weight: 600; letter-spacing: -0.028em; line-height: 1.06; }
.vd-card__meta  { display: flex; align-items: center; gap: 8px; flex-wrap: wrap; }
.vd-card__preview {
  display: flex; gap: 8px; font-size: 12.5px; line-height: 1.45; color: var(--vd-ink-2);
  border-top: 1px solid var(--vd-hairline); padding-top: 10px;
}
.vd-card__tag {
  font-family: var(--vd-mono); font-size: 9.5px; letter-spacing: 0.07em;
  text-transform: uppercase; color: var(--vd-accent); font-weight: 600;
  flex: none; padding-top: 2px;
}
.vd-card__cta {
  display: flex; align-items: center; gap: 6px; font-size: 12px;
  font-weight: 560; color: var(--vd-accent); margin-top: auto; padding-top: 2px;
}

/* the grid the cards live in */
.vd-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(268px, 1fr));
  gap: 12px;
  transition: opacity 180ms ease;
}
/* refetch holds the previous render — no skeleton flash, no layout jump */
.vd-grid[data-reloading="true"] { opacity: 0.42; }
```

---

## `BreakdownRows` — dumbbells on a shared scale

The detail view's primary chart: every sub-category's A and B on one scale, so
the row carrying the variance is unmistakable.

```tsx
type Row = { name: string; a: number; b: number };

export function BreakdownRows({ rows, labelA, labelB }: {
  rows: Row[]; labelA: string; labelB: string;
}) {
  const ordered = [...rows].sort((x, y) => (y.b - y.a) - (x.b - x.a));
  const max = Math.max(1, ...ordered.flatMap((r) => [r.a, r.b]));
  const PAD = 3;
  const pos = (v: number) => PAD + (v / max) * (100 - PAD * 2);

  return (
    <div>
      <div className="vd-panel-head">
        <div className="vd-panel-title">By reason code</div>
        <div className="vd-legend">
          <span className="vd-legend__item">
            <span className="vd-legend__dot" style={{ background: "var(--vd-then)" }} />{labelA}
          </span>
          <span className="vd-legend__item">
            <span className="vd-legend__dot" style={{ background: "var(--vd-now)" }} />{labelB}
          </span>
        </div>
      </div>

      <div className="vd-rows">
        {ordered.map((r) => {
          const xa = pos(r.a), xb = pos(r.b);
          const connector =
            r.a === r.b ? "var(--vd-hairline-2)"
            : r.b > r.a ? "var(--vd-rise)" : "var(--vd-fall)";
          return (
            <div className="vd-row" key={r.name}>
              <span className="vd-row__name">{r.name}</span>
              <div className="vd-db__track">
                <div className="vd-db__line" style={{
                  left: `${Math.min(xa, xb)}%`,
                  width: `${Math.abs(xb - xa)}%`,
                  background: connector,
                }} />
                <div className="vd-db__dot" style={{ left: `${xa}%`, background: "var(--vd-then)" }} />
                <div className="vd-db__dot" style={{ left: `${xb}%`, background: "var(--vd-now)" }} />
              </div>
              <span className="vd-row__val">
                {r.a.toLocaleString()} → <b>{r.b.toLocaleString()}</b>
              </span>
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

```css
.vd-panel-head  { display: flex; align-items: center; justify-content: space-between;
                  gap: 12px; margin-bottom: 12px; flex-wrap: wrap; }
.vd-panel-title { font-size: 12.5px; font-weight: 620; }
.vd-legend      { display: flex; gap: 14px; align-items: center; flex-wrap: wrap; }
.vd-legend__item{ display: inline-flex; align-items: center; gap: 6px;
                  font-size: 11.5px; color: var(--vd-ink-2); }
.vd-legend__dot { width: 9px; height: 9px; border-radius: 50%; flex: none; }

.vd-rows { display: flex; flex-direction: column; gap: 3px; }
.vd-row  { display: grid; grid-template-columns: 152px minmax(0,1fr) 74px;
           align-items: center; gap: 12px; padding: 4px 0; border-radius: 5px; }
.vd-row:hover { background: var(--vd-surface-sunk); }
.vd-row__name { font-size: 12px; color: var(--vd-ink-2);
                overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.vd-row:hover .vd-row__name { color: var(--vd-ink); }
.vd-row__val  { font-family: var(--vd-mono); font-size: 11px; text-align: right;
                color: var(--vd-ink-2); font-variant-numeric: tabular-nums; }
.vd-row__val b { color: var(--vd-ink); font-weight: 600; }
```

A **table view** is not optional — render the same rows as a `<table>` behind a
"Show table" toggle. Every value a chart shows must be reachable without
hovering.

---

## `ReasonPanel`

The explanation is the product. It gets a recessed surface and an accent rule,
not another card.

```tsx
import { useStreamedText } from "./hooks";

type ReasonPanelProps = {
  text: string;
  scope?: string;        // grain, dimensions checked, confidence
  sql?: string;
  latencyLabel?: string; // e.g. "2.1s"
  onFeedback?: (useful: boolean) => void;
  /* Closing the loop. An explanation the reader can't act on is a dead end —
     without these they retype the finding into a ticket by hand. */
  ticket?: { id: string; url: string; status: string };
  onCreateTicket?: () => void;
  onMarkExpected?: () => void;
  /* The reader's next question is already forming. The semantic model is
     already here; taking the follow-up costs little. */
  onFollowUp?: (question: string) => void;
};

export function ReasonPanel({
  text, scope, sql, latencyLabel, onFeedback,
  ticket, onCreateTicket, onMarkExpected, onFollowUp,
}: ReasonPanelProps) {
  const { shown, done } = useStreamedText(text);
  const [vote, setVote] = useState<boolean | null>(null);

  return (
    <div className="vd-reason">
      <div className="vd-reason__head">
        <span className="vd-eyebrow" style={{ color: "var(--vd-accent)", fontWeight: 600 }}>
          Cortex Analyst
        </span>
        {latencyLabel && <span className="vd-eyebrow">· {latencyLabel}</span>}
      </div>

      <div className="vd-reason__body">
        {shown}
        {!done && <span className="vd-reason__caret" />}
      </div>

      {scope && <div className="vd-reason__scope">{scope}</div>}

      {sql && (
        <details className="vd-sql" onClick={(e) => e.stopPropagation()}>
          <summary>Query behind this answer</summary>
          <pre>{sql}</pre>
        </details>
      )}

      {onFollowUp && (
        <form
          className="vd-followup"
          onSubmit={(e) => {
            e.preventDefault();
            const input = e.currentTarget.elements.namedItem("q") as HTMLInputElement;
            if (input.value.trim()) { onFollowUp(input.value.trim()); input.value = ""; }
          }}
          onClick={(e) => e.stopPropagation()}
        >
          <input name="q" type="text" placeholder="Ask a follow-up…" aria-label="Ask a follow-up question" />
          <button type="submit">Ask</button>
        </form>
      )}

      {/* The action row — where the explanation stops being a report */}
      <div className="vd-actions">
        {ticket ? (
          <a className="vd-actions__ticket" href={ticket.url} target="_blank" rel="noreferrer"
             onClick={(e) => e.stopPropagation()}>
            {ticket.id} · {ticket.status}
          </a>
        ) : (
          onCreateTicket && (
            <button type="button" onClick={(e) => { e.stopPropagation(); onCreateTicket(); }}>
              Create ticket
            </button>
          )
        )}
        {onMarkExpected && (
          <button type="button" onClick={(e) => { e.stopPropagation(); onMarkExpected(); }}>
            Mark as expected
          </button>
        )}
        <span className="vd-actions__spacer" />
        <span className="vd-eyebrow">Useful?</span>
        {[true, false].map((v) => (
          <button key={String(v)} type="button"
            aria-pressed={vote === v}
            onClick={(e) => { e.stopPropagation(); setVote(vote === v ? null : v); onFeedback?.(v); }}>
            {v ? "Yes" : "No"}
          </button>
        ))}
      </div>
    </div>
  );
}
```

`Mark as expected` should not hide the variance — it still appears and still
ranks, but renders as `steady` with a note saying why it was suppressed and by
whom. Hiding it makes the dashboard lie; downgrading it makes the dashboard
learn. See `rules.md` § Earn continued trust.

```css
.vd-reason {
  background: var(--vd-surface-sunk); border-radius: var(--vd-radius-card);
  padding: 16px 18px; border-left: 2px solid var(--vd-accent);
  display: flex; flex-direction: column; gap: 12px;
}
.vd-reason__head { display: flex; align-items: center; gap: 8px; }
.vd-reason__body { font-size: 14px; line-height: 1.62; color: var(--vd-ink);
                   max-width: 62ch; min-height: 3em; }
.vd-reason__caret {
  display: inline-block; width: 7px; height: 1.05em; margin-left: 2px;
  background: var(--vd-accent); vertical-align: text-bottom;
}
@media (prefers-reduced-motion: no-preference) {
  .vd-reason__caret { animation: vd-blink 1s steps(2, start) infinite; }
}
@keyframes vd-blink { 50% { opacity: 0; } }

.vd-reason__scope {
  font-family: var(--vd-mono); font-size: 10.5px; line-height: 1.5;
  color: var(--vd-ink-3); border-top: 1px solid var(--vd-hairline-2); padding-top: 11px;
}
.vd-sql summary {
  font-family: var(--vd-mono); font-size: 10.5px; letter-spacing: 0.06em;
  text-transform: uppercase; font-weight: 600; color: var(--vd-ink-2);
  cursor: pointer; list-style: none; display: flex; align-items: center; gap: 6px;
}
.vd-sql summary::-webkit-details-marker { display: none; }
.vd-sql summary::before { content: "▸"; font-size: 9px; color: var(--vd-ink-3);
                          transition: transform 120ms ease; }
.vd-sql[open] summary::before { transform: rotate(90deg); }
.vd-sql pre {
  margin: 10px 0 0; padding: 12px 14px; background: var(--vd-surface);
  border: 1px solid var(--vd-hairline); border-radius: var(--vd-radius-ctl);
  overflow-x: auto; font-family: var(--vd-mono); font-size: 11px;
  line-height: 1.62; color: var(--vd-ink-2);
}
.vd-actions { display: flex; gap: 8px; align-items: center; flex-wrap: wrap; }
.vd-actions__spacer { flex: 1 1 auto; }
.vd-actions button {
  background: var(--vd-surface); border: 1px solid var(--vd-hairline);
  border-radius: 6px; font-family: var(--vd-sans); font-size: 11.5px;
  font-weight: 520; color: var(--vd-ink-2); padding: 4px 10px; cursor: pointer;
}
.vd-actions button:hover { border-color: var(--vd-hairline-2); color: var(--vd-ink); }
.vd-actions button[aria-pressed="true"] {
  border-color: var(--vd-accent-ring); color: var(--vd-accent); background: var(--vd-accent-wash);
}
.vd-actions__ticket {
  font-family: var(--vd-mono); font-size: 11px; color: var(--vd-accent);
  text-decoration: none; border: 1px solid var(--vd-accent-ring);
  background: var(--vd-accent-wash); border-radius: 6px; padding: 4px 10px;
}

.vd-followup { display: flex; gap: 8px; }
.vd-followup input {
  flex: 1 1 auto; min-width: 0;
  background: var(--vd-surface); border: 1px solid var(--vd-hairline);
  border-radius: 6px; padding: 6px 10px;
  font-family: var(--vd-sans); font-size: 12.5px; color: var(--vd-ink);
}
.vd-followup input::placeholder { color: var(--vd-ink-3); }
.vd-followup input:focus { border-color: var(--vd-accent-ring); }
.vd-followup button {
  background: var(--vd-accent); border: 0; border-radius: 6px; color: #fff;
  font-family: var(--vd-sans); font-size: 11.5px; font-weight: 600;
  padding: 4px 12px; cursor: pointer;
}
```

---

## `BriefingBand` — lead with the answer

The page opens with a written summary and one hero figure. Exactly one hero per
view, ≥48px, in the same sans as everything else.

```tsx
export function BriefingBand({ headline, emphasis, tail, meta, heroLabel, hero, heroDelta }: {
  headline: string; emphasis?: string; tail?: string;
  meta: string[]; heroLabel: string; hero: string; heroDelta?: React.ReactNode;
}) {
  return (
    <section className="vd-briefing">
      <div className="vd-briefing__body">
        <div className="vd-briefing__head">
          <span className="vd-pulse" aria-hidden="true" />
          <span className="vd-eyebrow">Cortex Analyst · variance read</span>
        </div>
        <h1 className="vd-briefing__h1">
          {headline}{emphasis && <b>{emphasis}</b>}{tail}
        </h1>
        <div className="vd-briefing__foot">
          {meta.map((m, i) => (
            <React.Fragment key={i}>
              {i > 0 && <span className="vd-briefing__dot">·</span>}
              <span>{m}</span>
            </React.Fragment>
          ))}
        </div>
      </div>
      <div className="vd-hero">
        <div className="vd-hero__label">{heroLabel}</div>
        <div className="vd-hero__value vd-tnum">{hero}</div>
        {heroDelta}
      </div>
    </section>
  );
}
```

```css
.vd-briefing {
  display: grid; grid-template-columns: minmax(0, 1fr) auto; gap: 40px;
  align-items: start; padding: 34px 0 30px; border-bottom: 1px solid var(--vd-hairline);
}
/* measure in px, NOT ch — `ch` resolves against the 14px parent, not the 27px
   headline, and silently collapses the line length to about half */
.vd-briefing__body { max-width: 780px; }
.vd-briefing__h1 {
  font-size: 27px; line-height: 1.32; font-weight: 500;
  letter-spacing: -0.017em; margin: 0; text-wrap: balance;
}
.vd-briefing__h1 b { font-weight: 680; }
.vd-briefing__head { display: flex; align-items: center; gap: 9px; margin-bottom: 14px; }
.vd-briefing__foot {
  margin-top: 16px; display: flex; gap: 8px; align-items: center; flex-wrap: wrap;
  font-family: var(--vd-mono); font-size: 11px; color: var(--vd-ink-3);
}
.vd-briefing__dot { color: var(--vd-hairline-2); }

.vd-pulse { width: 7px; height: 7px; border-radius: 50%;
            background: var(--vd-accent); flex: none; }
@media (prefers-reduced-motion: no-preference) {
  .vd-pulse { animation: vd-pulse 2.6s ease-in-out infinite; }
}
@keyframes vd-pulse {
  0%, 100% { opacity: 1; transform: scale(1); }
  50%      { opacity: 0.45; transform: scale(0.8); }
}

.vd-hero { text-align: right; border-left: 1px solid var(--vd-hairline); padding-left: 40px; }
.vd-hero__label {
  font-family: var(--vd-mono); font-size: 10.5px; letter-spacing: 0.10em;
  text-transform: uppercase; color: var(--vd-ink-3); margin-bottom: 8px;
}
.vd-hero__value { font-size: 58px; font-weight: 600; letter-spacing: -0.035em;
                  line-height: 1; margin-bottom: 10px; }

@media (max-width: 880px) {
  .vd-briefing { grid-template-columns: minmax(0, 1fr); gap: 26px; }
  .vd-hero { text-align: left; border-left: 0; padding-left: 0;
             border-top: 1px solid var(--vd-hairline); padding-top: 22px; }
}
```

---

## `DatePair` — the two-date control

One filter row above everything it scopes; never per-card. The colored dots tie
each input to its dot in every dumbbell on the page.

```tsx
export function DatePair({ a, b, onChange }: {
  a: string; b: string; onChange: (a: string, b: string) => void;
}) {
  const commit = (next: { a?: string; b?: string }) => {
    let na = next.a ?? a, nb = next.b ?? b;
    if (new Date(na) > new Date(nb)) [na, nb] = [nb, na];  // keep earlier → later
    onChange(na, nb);
  };
  return (
    <div className="vd-datepair">
      <span className="vd-datepair__leg">
        <span className="vd-datepair__key" style={{ background: "var(--vd-then)" }} />
        <input type="date" value={a} aria-label="Earlier date"
               onChange={(e) => commit({ a: e.target.value })} />
      </span>
      <span aria-hidden="true" style={{ color: "var(--vd-ink-3)", fontSize: 12 }}>→</span>
      <span className="vd-datepair__leg">
        <span className="vd-datepair__key" style={{ background: "var(--vd-now)" }} />
        <input type="date" value={b} aria-label="Later date"
               onChange={(e) => commit({ b: e.target.value })} />
      </span>
    </div>
  );
}
```

```css
.vd-datepair {
  display: inline-flex; align-items: center; gap: 8px;
  background: var(--vd-surface); border: 1px solid var(--vd-hairline);
  border-radius: 8px; padding: 5px 10px;
}
.vd-datepair__leg { display: inline-flex; align-items: center; gap: 7px; }
.vd-datepair__key { width: 9px; height: 9px; border-radius: 50%; flex: none; }
.vd-datepair input[type="date"] {
  appearance: none; background: transparent; border: 0; color: var(--vd-ink);
  font-family: var(--vd-sans); font-size: 12.5px; font-weight: 560;
  font-variant-numeric: tabular-nums; padding: 2px; cursor: pointer;
  color-scheme: inherit;   /* makes the native picker follow the theme */
}
```

Pair it with presets — "Day on day / Week on week / Month on month" — that set
both dates at once. Nobody wants to fight two calendar pickers for "yesterday".
