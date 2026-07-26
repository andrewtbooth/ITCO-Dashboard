# Trade Compliance Posture Dashboard — proof of concept

A single-page dashboard showing an aerospace & defense contractor's international
trade compliance posture: the health of its ITAR and EAR export authorizations,
the coverage of its jurisdiction and classification determinations, and the open
items an Empowered Official is accountable for.

**Every record in this dashboard is synthetic.** Programs, part numbers,
authorization numbers, foreign parties, and determinations are all generated. It
is a proof of concept for layout and metric definitions, not a compliance record.

Open `index.html` in a browser. There is no build step, no package manager, and
no network dependency.

---

## What it covers

Two modules, chosen as the two that most directly drive an Empowered Official's
risk picture:

| Module | What it answers |
|---|---|
| **Authorizations & agreements** | What is authorized, what is about to lapse, how much authorized value is left, and which conditions have not been acknowledged |
| **Jurisdiction & classification** | What share of the item master has a determination of record, how old the backlog is, and where the control concentration sits |

Deliberately out of scope for this POC, and the obvious next modules: restricted
party screening, foreign nationals and deemed exports, AES/EEI filing accuracy,
voluntary disclosure tracking, and training completion.

## The three views

**Posture** opens with the count of open compliance actions, six KPI tiles each
carrying a 12-month trend, and a prioritized *Needs attention* queue. Every row
in that queue is a rule evaluated against the record set, with the reason it
matters written out in plain language, the analyst who owns it, and the
regulatory or policy reference it rests on. **Clicking a row opens that record**
in the relevant detail table, highlighted and expanded. Below the queue, an
audit-readiness strip reports whether the supporting paper exists, independent of
whether the underlying decision was right.

**Authorizations** covers the expiry pipeline for the next 12 months, value
utilization against the 85% amendment threshold, authorized value by destination
region, licensing turnaround, and a sortable detail table with per-record
provisos and shipment history.

**Classification** covers backlog aging, jurisdiction mix, control
concentration, determination throughput, and a sortable part-level table.

## Metric definitions

| Metric | Definition |
|---|---|
| Live authorization | Issued, unexpired, and not fully consumed |
| Value utilization | Shipped value ÷ authorized value |
| Expiring ≤ 90 days | Live authorizations within 90 days of their expiration date; the "at risk" figure is their unshipped balance |
| Open provisos | Provisos on live authorizations with no acknowledgement date recorded |
| Classification coverage | Parts with a determination of record ÷ all parts in the item master |
| Backlog aging | Days since a part without a determination entered the item master |
| Licensing turnaround | Median days from submission to issuance, trailing 90 days |
| Determination throughput | Parts entering the queue vs. determinations completed, by month |

Nothing on the page is a hardcoded number. Every tile, chart, and queue row is
computed from the record set at load time, so filtering recomputes all of them
together. Each measure takes an "as of" date, which is what lets the same code
produce both the current figure and its 12-month trend: an authorization counts
only once it was issued, a shipment only once it happened, a part only once it
entered the item master.

## Action queue rules

| Severity | Rule | Why it fires |
|---|---|---|
| Critical | Temporary authorization expired with items not reconciled as returned | Items abroad past expiry are outside an authorization until reconciled or re-authorized |
| Critical | Shipment recorded after authorization expiry | Assess against the voluntary disclosure decision, 22 CFR 127.12 |
| Serious | Authorized value >85% consumed with >6 months of term remaining | The value will run out long before the authorization does, and an amendment takes longer to obtain than the balance will cover |
| Serious | 3+ provisos not acknowledged on a live authorization | The most common finding in a licensing audit |
| Serious | Supporting records not on file | Five-year retention applies, 22 CFR 122.5 and 15 CFR 762 |
| Serious | Part in the item master 90+ days without a determination | Blocks shipment, quoting, and foreign-national disclosure decisions |
| Serious | ITAR determination with no written basis retained | An unsupported determination does not survive an audit |
| Watch | Determination past its re-review date | Control lists and configurations move; a stale determination becomes the wrong one |
| Watch | Authorization in adjudication beyond the normal window | May be on hold pending an unanswered request for information |
| Watch | 1–2 provisos not acknowledged | Same exposure as above, lower volume |

Severity orders the queue, but within a tier the rule kinds are dealt out
round-robin — one rule matching forty records would otherwise fill the visible
queue and bury the other nine.

The queue is truncated by default, but **never across the critical tier**: the
default view always runs to the end of critical and a few rows into the next
tier, so no critical action is ever parked behind a "show more" button. The line
above that button says what the fold is hiding, by severity, rather than only
how much.

### Regulatory references

Each rule carries either a regulatory citation or an explicitly labelled policy
setting — never a bare assertion. The citations in use:

| Reference | Used for |
|---|---|
| 22 CFR 123.5 | Temporary export licences (DSP-73) and the obligation to return the items |
| 22 CFR 127.1 | Violations, including failing to comply with the terms or conditions of a licence — the basis for the proviso rules |
| 22 CFR 127.12 | Voluntary disclosure |
| 22 CFR 122.5 | ITAR recordkeeping, five years — supporting records and the retained basis for a determination |
| 15 CFR 762 | EAR recordkeeping, five years |

Everything else the queue raises is a `Policy:` setting, shown as such on the
row. Citations locate the obligation; **they are not legal advice**, and the
thresholds used here (85% utilization, a 90-day expiry window, a 90-day
classification service level, a 3-year re-review cycle) are illustrative
internal policy rather than regulatory deadlines. Every threshold the page
applies is named in one `POLICY` object at the top of the script, with the
re-review cycle as its own constant, `REVIEW_CYCLE_YEARS`.

## Replacing the synthetic data

The mock data layer is fenced inside `index.html` by a single comment block:

```js
/* ===== MOCK DATA LAYER — replace this entire section with a real feed ===== */
```

It exposes exactly one object:

```js
DATA = { asOf: Date, authorizations: Authorization[], classifications: Part[] }
```

Both record schemas are documented in full in the comment block directly above
the generators — that comment is the integration contract. Point `DATA` at a
real feed (SAP GTS, Descartes, OCR, or an internal export management system) and
every derivation, chart, and action rule below it works unchanged.

The generator is seeded, so the dataset is identical on every load and for every
viewer. That keeps a demo reproducible and keeps diffs meaningful. Dates are
handled in UTC throughout for the same reason.

### Notes on the synthetic data itself

The generator models a few things deliberately rather than drawing uniformly,
because uniform draws produce charts that quietly contradict what they claim:

- **Issuance is a steady state.** Issue dates are spread across a full term plus
  a tail, so the live population stays flat over the trailing year and the expiry
  pipeline falls out of the term. Reverse-engineering the data from a desired
  expiry chart instead makes the portfolio appear to double in twelve months.
- **Shipments spread across each authorization's term**, front-loaded, and only
  those already in the past are records. Packing them into the past instead makes
  every authorization look fully drawn down the moment it is issued, and the
  utilization trend then rises for no reason but the shape of the generator.
- **Determination dates are drawn first and parts placed into the item master a
  working lag earlier**, not the other way round, so recent months aren't forced
  to show either zero completions or a spike.
- **Control codes cluster by program.** A radar program is mostly one USML
  category. Drawn uniformly, the concentration chart flattens into ten equal bars
  and contradicts its own title.
- **Acknowledgement discipline clusters by record**, which is what makes a
  three-or-more proviso finding a different problem from a one-or-two finding.
- **Agreements name parties in more than one region.** `region` is the region of
  the primary destination, not the only one present, so the value-by-region
  chart is a genuine roll-up rather than a relabelling — which is what its
  footnote claims it is.

Utilization above 100% appears only where value was shipped against an
authorization that had none left — a condition the rules flag.

## Design and accessibility notes

- **No external dependencies.** All CSS and JavaScript is inline and the charts
  are hand-rolled SVG, so the page renders under a strict content security
  policy with no network access.
- **Both themes are designed**, not inverted. Tokens are redefined under
  `prefers-color-scheme: dark` and again under `:root[data-theme="dark"|"light"]`
  so a viewer's explicit theme choice wins over the OS setting in both directions.
- **The chart palette is validated, not eyeballed.** The categorical slots and
  both ordinal ramps (4-step backlog aging, 5-step value utilization) were run
  through the computable checks — lightness band, chroma floor, CVD separation,
  normal-vision floor and contrast for the categorical set; monotone lightness,
  adjacent ΔL, light-end contrast and single hue for the ramps — against this
  page's actual light and dark surfaces. All six runs pass. Status colours are
  reserved, never reused as a series colour, and never share a chart with a
  categorical or ordinal scale.
- **Every chart has a table-view twin**, so no value is reachable only by hover.
  Charts with two or more series always carry a legend; direct labels are used
  selectively rather than on every point.
- Charts are keyboard-reachable, hit targets span the full category band, and
  `prefers-reduced-motion` is respected. Series and category names reach the DOM
  through `textContent`, never through `innerHTML`.
- **The tooltip is deliberately not a live region.** It updates on every
  pointer move, so announcing it would flood a screen reader. Assistive
  technology gets the same values from the `aria-label` on the focused chart
  band and from each chart's table view.
- Tabs follow the ARIA tabs pattern — one tab stop for the set, arrow keys and
  Home/End to move between them. Every table carries a caption as its accessible
  name, every chart carries its title, and opening a detail row keeps focus on
  the row you opened.
- Only the visible view holds DOM. Panes that leave the screen are emptied
  rather than parked, since the active pane is rebuilt from scratch anyway —
  which keeps the posture view at ~690 nodes and 9 tab stops instead of ~14,700
  and 490.

## Printing

Compliance work ends up in audit files and board packs, so the page has a print
stylesheet. Interactive chrome is dropped, the light tokens are forced regardless
of the theme on screen, scroll boxes open out to full length, and nothing breaks
across a page mid-record. A print-only line carries the view, the active filters,
the extract date and the synthetic-data warning, so a printed page is never
adrift from the selection that produced it.

Each view prints separately — the tab you are on is the one that comes out.

## Layout

```
index.html    the entire dashboard — styles, synthetic data, derivations, charts, interactions
README.md     this file
```
