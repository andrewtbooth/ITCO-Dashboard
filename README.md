# Trade Compliance Posture Dashboard — proof of concept

A single-page dashboard showing an aerospace & defense contractor's international
trade compliance posture: the health of its ITAR and EAR export authorizations,
the coverage of its jurisdiction and classification determinations, whether the
written procedures are the procedures being followed, and the open items an
Empowered Official is accountable for.

**Every record in this dashboard is synthetic.** Programs, part numbers,
authorization numbers, foreign parties, determinations and audit results are all
generated. It is a proof of concept for layout and metric definitions, not a
compliance record.

**Live: https://andrewtbooth.github.io/ITCO-Dashboard/**

Or open `index.html` in a browser. There is no build step, no package manager,
and no network dependency — the file is entirely self-contained, so it also
works from a local copy, an email attachment or a disconnected machine.

---

## What it covers

Three modules, chosen as the ones that most directly drive an Empowered
Official's risk picture — what is authorized, what the items are, and whether the
procedures are being followed:

| Module | What it answers |
|---|---|
| **Authorizations & agreements** | What is authorized, what is about to lapse, whether anything has been filed to replace it, how much authorized value is left, and which conditions have not been acknowledged |
| **Jurisdiction & classification** | What share of the item master has a determination of record, how old the backlog is, and where the control concentration sits |
| **Process compliance** | Whether the written SOP is the procedure being followed, how strong the evidence behind each control is, and whether audit findings get closed |

The first two are on one page so they can check each other. The parts named on an
authorization join to the item master, which is what lets the dashboard ask the
question neither module can answer alone: **is anything being exported under an
authorization that does not cover it?**

Deliberately out of scope for this POC, and the obvious next modules: restricted
party screening, foreign nationals and deemed exports, AES/EEI filing accuracy,
and voluntary disclosure tracking.

## The four views

**Posture** opens with the count of open compliance actions, six KPI tiles each
carrying a 12-month trend, and a prioritized *Needs attention* queue. Every row
in that queue is a rule evaluated against the record set, with the reason it
matters written out in plain language, the analyst who owns it, and the
regulatory or policy reference it rests on. **Clicking a row opens that record**
in the relevant detail table, highlighted and expanded. Below the queue, an
audit-readiness strip reports whether the supporting paper exists, independent of
whether the underlying decision was right. Between them sits **workload by
owner** — who is carrying the open actions and how much of each analyst's load
is already past its service level.

**Authorizations** covers the expiry pipeline for the next 12 months, value
utilization against the 85% amendment threshold, authorized value by destination
region, licensing turnaround, and a sortable detail table with per-record
provisos and shipment history.

**Classification** covers backlog aging, jurisdiction mix, control
concentration, determination throughput, and a sortable part-level table.

**Process compliance** covers conformance by procedure against the internal
target, the strength of the evidence behind each control, the open corrective
actions, and a sortable control-level table.

### How process compliance works

A **control** is one SOP step under test, and it arrives in one of two shapes:

- An **automated** control names a measure that derives its population and its
  failures from the record set itself. It is therefore tested continuously
  against the whole population rather than a sample, and its evidence *is* the
  data. `SOP-19.1 — items exported temporarily are reconciled as returned on or
  before expiry` is the same computation as the critical action rule of the same
  name; the audit view and the work queue cannot disagree because they are one
  derivation.
- A **manual** control carries the result of a periodic sample: how many
  transactions were examined, how many conformed, by whom, and when.

Both land in the same conformance figure, weighted by what was actually tested,
so a procedure carrying an automated control over thousands of records is not
averaged against a 20-item sample. Eleven of the seventeen controls here are
automated and six are sampled — which is what a compliance program mid-automation
actually looks like.

Two things feed back into the Posture queue, so there is still one place to look
for work: a control that has slipped past **its own testing frequency** (an
untested control is not a control), and a **corrective action past the date it
was committed to** (an audit that produces findings nobody closes reads worse
than no audit at all).

Because automated measures re-derive against the filtered slice, SOP conformance
moves with the program and site filters while a manual sample result — taken once,
across the business — does not. The page says so on the tab.

## Metric definitions

| Metric | Definition |
|---|---|
| Live authorization | Issued, unexpired, and not fully consumed |
| Value utilization | Shipped value ÷ authorized value, over value-bearing authorizations only |
| Expiring ≤ 90 days | Live authorizations within 90 days of their expiration date; the "at risk" figure is their unshipped balance, and the coverage figure is how many have a replacement application in adjudication |
| Open provisos | Provisos on live authorizations with no acknowledgement date recorded |
| Classification coverage | Parts with a determination of record ÷ all parts in the item master |
| Backlog aging | Days since a part without a determination entered the item master |
| Licensing turnaround | Median days from submission to issuance, trailing 90 days |
| Determination throughput | Parts entering the queue vs. determinations completed, by month |
| Control conformance | Items conforming ÷ items tested, weighted by what was actually tested |
| Assurance strength | Continuous whole-population testing > periodic sample in date > sample overdue or never run |
| Action onset | The date the condition actually became true — an expiry date, the shipment that broke a licence, or the date cumulative shipped value crossed the amendment threshold, read off the shipment history |
| Action due date | Onset + a remediation window by severity: 5 days critical, 30 serious, 90 watch |
| Past service level | Open actions whose due date has passed |

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
| Critical | Part determined ITAR named on an EAR authorization | A USML article moves on State Department authorization; shipping one against a BIS licence is an export made without the approval it required |
| Serious | Part with no determination named on a live authorization | The authorization asserts what the item is; nobody has decided |
| Serious | Authorization expiring with no replacement application filed | Renewal lead time is weeks for a licence and months for an agreement — the decision needed making long before the expiry date |
| Serious | Control not tested within its own frequency | An untested control is not a control; the absence of a test is itself the finding |
| Serious | Corrective action past its committed date | The deficiency is now documented and unremediated |
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
queue and bury the other fourteen.

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

### What the rules deliberately do not do

There is no rule for restricted party screening, deemed exports, or filing
accuracy, because those modules are out of scope and a rule that half-checks
them would be worse than none. Every rule here runs on data the two in-scope
modules actually hold.

## Working the queue

Every action carries the analyst who owns it and the date it fell due. The due
date is derived, not assigned: each rule reports the date its condition actually
became true — the expiry date, the shipment that broke the licence, the date
cumulative shipped value crossed 85% — and the remediation window for its
severity is added to that. A finding cannot be made to look fresh by being
discovered late.

Actions can be **reassigned to another analyst** and **cleared from the queue**.
Both are held in the browser session only: there is no system of record behind
this page, and a reload restores everything. The page says so next to the
controls rather than leaving it to be discovered.

### On reading "overdue"

Most of the queue is past its service level, and that is the finding rather than
a display problem. These are standing gaps — a determination held without a
written basis since 2021 has genuinely been deficient since 2021. Because
"overdue" therefore stops discriminating, the hero line reports the median age
of an open action and how many opened in the last 90 days alongside it, so a
chronic backlog cannot be mistaken for a spike.

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

### On using a real company's programs

The programs here are invented. Their *mission types* are drawn from the mission
areas a mid-size aerospace and defense prime actually operates in, because that
is what makes the control categories ring true — but the names are not.

That is deliberate, and it is worth stating plainly: **the rows on this dashboard
are export control violations.** Shipments recorded after expiry, temporary
exports never reconciled, findings routed to the voluntary disclosure decision.
Attaching those to a real company's real programs would produce a document that
reads as a record of that company's compliance failures — and a screenshot of one
view travels without the synthetic-data banner that sits above the tabs.

It is also simply the wrong move for a demo. Nobody pitching a trade compliance
tool to a prime shows that prime committing ITAR violations. Realism comes from
the mission types, the control categories, the authorization mix and the
destination patterns being right — none of which needs a real name.

### Domain modelling decisions

These are the places where the obvious data model would produce a number a
trade compliance professional does not recognise:

- **Agreements do not burn value.** A TAA and an MLA authorize defense services
  and technical data; the hardware supporting them moves on its own licences.
  They carry no value ceiling, so utilization, the amendment threshold and the
  value-by-region roll-up all skip them — showing a TAA at "62% consumed" is
  the fastest way to lose a subject-matter audience. Provisos, records and
  expiry still apply, and that is where agreements actually generate work.
- **The parts named on an authorization follow its regime.** A BIS licence
  lists EAR items, a DSP-5 lists USML items. Drawn at random, roughly half of
  every EAR licence's parts would be USML and the cross-module check would fire
  on almost everything — noise rather than a finding. The mismatches are seeded
  deliberately and sparsely, which is what makes them mean something.
- **Renewals are modelled as what they are:** a fresh application naming the
  authorization it replaces. Without that link, "13 expiring in 90 days" cannot
  be told apart from "13 expiring and nobody has filed anything" — which is the
  only version of that number worth putting on a dashboard.
- **The item-master filter does not reach inside an authorization.** Parts named
  on an authorization are attributes of it, so the cross-module lookup runs over
  the whole item master: a BIS licence listing a USML part is a defect of that
  licence whatever the filter is set to.

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
- **Control codes follow the program's mission.** Each program carries a mission
  — reusable orbital vehicle, high-altitude ISR aircraft, aircraft survivability
  equipment, undergraduate jet trainer, electronic warfare, sustainment — and the
  USML categories and ECCNs its parts fall in are drawn from what that mission
  actually produces. An orbital vehicle program lands on Category XV, an aircraft
  program on VIII, an EW program on XI. Drawn uniformly, a spaceplane is as
  likely to carry Category IV — launch vehicles and missiles — as XV, and that is
  the detail a trade compliance professional spots first. The authorization kinds
  and destination regions follow the mission too: sustainment leans on temporary
  exports, a co-production trainer on a manufacturing licence, and neither ships
  everywhere.
- **Acknowledgement discipline clusters by record**, which is what makes a
  three-or-more proviso finding a different problem from a one-or-two finding.
- **Proviso gaps close over time.** An authorization issued years ago has been
  through internal review and at least one audit cycle, so open provisos
  concentrate on recent issuances. Without that decay the data implied
  conditions left unacknowledged for a decade, and every action in the queue
  read as years overdue — which makes "overdue" mean nothing at all.
- **Agreements name parties in more than one region.** `region` is the region of
  the primary destination, not the only one present, so the value-by-region
  chart is a genuine roll-up rather than a relabelling — which is what its
  footnote claims it is.

Utilization above 100% appears only where value was shipped against an
authorization that had none left — a condition the rules flag.

## Design and accessibility notes

### The shell

A sticky toolbar carries the tabs and the one filter row together, so navigation
and scope stay reachable on a page this tall. Type sizes, radii, elevation and
easing are tokens rather than per-component values, so the four views read as one
system. Section headings carry an eyebrow label, queue rows carry a severity
accent on the leading edge alongside their chip, and cards lift slightly on hover
where they are interactive. Everything transitions on one easing curve, and all of
it is switched off under `prefers-reduced-motion`.

- **No external dependencies.** All CSS and JavaScript is inline and the charts
  are hand-rolled SVG, so the page renders under a strict content security
  policy with no network access.
- **Both themes are designed**, not inverted. Tokens are redefined under
  `prefers-color-scheme: dark` and again under `:root[data-theme="dark"|"light"]`
  so a viewer's explicit theme choice wins over the OS setting in both directions.
- **The chart palette is validated, not eyeballed.** The categorical slots and
  the three ordinal ramps (4-step backlog aging, 5-step value utilization,
  3-step severity) were run through the computable checks — lightness band,
  chroma floor, CVD separation, normal-vision floor and contrast for the
  categorical set; monotone lightness, adjacent ΔL, light-end contrast and
  single hue for the ramps — against every surface this page renders on. When
  the shell was restyled the light surface became pure white and the dark
  surface moved, so the whole set was re-run against the new values rather than
  assumed to carry over. **Sixteen runs, all passing.**
- **Every text token clears 4.5:1 on every surface it lands on.** Restyling
  moved the light surface to pure white, which put muted text — axis ticks,
  footnotes, detail-box labels — at 4.23:1 against white and 3.74:1 against the
  detail-box fill. Muted is therefore `#6b6962` rather than the reference
  palette's `#898781`, which measures 3.59:1 on white. Marks still only owe
  3:1, so the sparkline's de-emphasis hue is unchanged.
- **Severity is a ramp, not the status palette — and that was a measured
  decision.** The workload chart stacks actions by severity, and the obvious
  move is to paint the segments in the status colours they wear everywhere
  else. Measured as a series set, that fails: status-serious `#ec835a` against
  status-warning `#fab219` sits at ΔE 13.6, under the normal-vision floor of
  15, which secondary encoding does not excuse. Those four steps are a fixed
  scale for marking state on a chip or a meter, not a mutually distinguishable
  series set. Severity is an *ordered* scale, so the chart takes a one-hue
  ramp — darker is more severe — while the chips keep the status colours. The
  legend carries the same `◆ ▲ ●` icons as the chips, so identity never rests
  on matching colours across components.
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

## Deployment

`.github/workflows/pages.yml` publishes the site to GitHub Pages on every push
to `main`. The site is the repository root: there is nothing to build, so the
deployed artifact is `index.html` itself.

A workflow rather than a one-off settings change, so the published site cannot
drift from `main` and the deployment is reviewable in the diff.

One step could not be automated. `actions/configure-pages` runs with
`enablement: true`, but the default `GITHUB_TOKEN` cannot *create* a Pages site
even with `pages: write` — it fails with *"Resource not accessible by
integration"*. Pages therefore has to be switched on once by hand, at
**Settings → Pages → Build and deployment → Source → GitHub Actions**. After
that the workflow is self-sustaining.

## Layout

```
index.html                      the entire dashboard — styles, synthetic data,
                                derivations, charts, interactions
README.md                       this file
.github/workflows/pages.yml     publishes the dashboard to GitHub Pages
```
