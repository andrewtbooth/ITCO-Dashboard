# Practitioner panel review

Six reviewers evaluated this dashboard independently, then saw each other's findings
and responded. This document records where they converged, where they disagreed, and
what I verified in code rather than taking on report.

The panel:

| Reviewer | Seat |
| --- | --- |
| EO / VP Trade Compliance | Empowered official, signs the authorizations |
| GC / risk executive | Counsel, owns discovery exposure |
| Export licensing manager | Runs the licensing desk |
| Classification lead | Owns jurisdiction and classification determinations |
| Internal audit / ECP lead | Tests the programme, owns the audit file |
| Trade compliance technology lead | Owns the system and its integrations |

Every claim below marked **verified** was checked against the code or the generated
dataset. Claims the panel made that I could not confirm, or that the data contradicts,
are marked as such — including one the panel was near-unanimous about.

## What the panel agreed on

### Unanimous, all six

**WDA is not value-bearing.** `index.html:1010` sets `valueBearing: true` on the
warehouse and distribution agreement. A WDA is a Part 124 agreement: it authorizes the
distribution arrangement, and hardware moves on DSP-5s that cite it. It has no value
ceiling to consume. The README already applies exactly this reasoning to TAAs and MLAs
one line up, so this is an internal inconsistency rather than a judgement call. Today
it double-counts value in the region roll-up and lets an agreement silently exit
`liveAuths` once notional shipments pass a ceiling that does not exist.

**Stop pooling conformance into one percentage.** Every reviewer rejected both the
current item-weighted figure *and* the obvious alternative of weighting each control
equally. Item weighting pools shipments, parts and agreements into a denominator nobody
can name; control-count weighting lets a 20-item sample outvote a 4,000-record
population test. The consensus replacement is a small vector, not a scalar:

- **controls effective / controls in scope** — each control judged against its own
  tolerable deviation rate, on its own unit
- **testing coverage per control** — items examined ÷ items in scope, never blended
- **controls with no valid current result** — stale, never tested, or unresolvable
- **zero-tolerance exceptions** as a raw count, outside both

The audit lead added a statistical guard the others did not: a 20-item sample cannot
support a 5% tolerable deviation rate at 95% confidence — that needs roughly 59 items
with zero expected deviations. Where sample size does not support the threshold, the
conclusion is **insufficient evidence**, not **effective**. The generator currently
draws sample sizes at `int(12, 40)` with no such test (`index.html:1400`).

**Drop legal conclusions from rule copy — but keep the rationale.** All six sided with
counsel on the characterisation and all six refused the bare-facts version. The queue's
plain-English "why" is the best thing on the page and every reviewer said so. The
resolution is to state the facts and the required next step, and delete the adjective:
keep "refer to the empowered official," lose "an export made without the approval it
required." Counsel's own replacement wording for the three critical rules keeps the
routing instruction and the citation; it removes only the words *violation*, *unlawful*
and the assertion that a disclosure decision is owed.

### Five of six

**Split `conditionAroseOn` from `firstDetectedOn`.** Counsel's argument is that
`onsetOn: a.issuedOn` on a proviso rule asserts the company knew in 2021 — a
knowledge date the tool fabricates, in the company's own disfavour, in a discoverable
record. Everyone accepted the operational objection that a detection-start clock lets a
chronic gap reset by being found late, and everyone proposed the same guard set:
`firstDetectedOn` system-stamped and immutable, detection lag reported as its own aged
metric, and both clocks shown on every row so neither party picks its number.

## The disagreement worth reading

The systems lead broke ranks on the two-clock model, and I think correctly.

> In a page that recomputes from scratch with no persistence, `firstDetectedOn` cannot
> be derived at all. The only defensible representation is *earliest snapshot in which
> the rule matched* — which makes it a property of the snapshot series, not of the
> record. With one snapshot it is `asOf` for everything.

So the honest move for a proof of concept is not to synthesise a detection date. Keep
`conditionAroseOn` as the fact it is, render detection as *no detection history*, and
remove the part counsel actually objects to — the tool characterising a long-standing
condition as **4 yr 2 mo overdue**, which is the sentence that reads as an admission.
Detection dating is a snapshot-store feature, and it arrives free the day monthly
snapshots exist.

The classification lead drew a second boundary: the split means nothing for
`part-overdue`, because a part sitting in our own item master *is* self-generated
notice. There the real fairness fix is to pause the clock while a determination is
genuinely blocked — CJ pending, awaiting engineering data — not to move its start.

## Corrections

Defects, not preferences. Ordered by how badly each misleads a reader.

### C1 — The regime filter inverts classification coverage · verified

`keepPart` (`index.html:2617`) tests `p.jurisdiction === STATE.regime`. An undetermined
part has no jurisdiction, so filtering to ITAR or EAR removes precisely the population
the coverage metric exists to count.

Measured against the current dataset:

| Filter | Parts in scope | Determined | Coverage |
| --- | --- | --- | --- |
| All | 780 | 665 | 85.3% |
| ITAR | 310 | 310 | **100.0%** |
| EAR | 307 | 307 | **100.0%** |

One click produces a screenshot reading 100.0% classification coverage and a backlog of
zero, printable with the scope line the page already renders. The comment above the
filter states the intent was to avoid quietly hiding the backlog; the code does exactly
that.

The fix is to stop using a determination *outcome* as a scope dimension. Filter on
presumptive regime — the part's determination where it has one, otherwise the regime of
the authorizations naming its program — so undetermined parts land in the bucket
matching their actual exposure and can never vanish. Coverage under a filter is then
strictly below 100% whenever a backlog exists, and that becomes a regression assertion.

### C2 — WDA marked value-bearing · verified

`index.html:1010`. Unanimous. One flag, plus excluding agreement value from the region
roll-up so it stops double-counting.

### C3 — Five of fifteen citations wrong or wrong-as-applied · verified

- `22 CFR 127.12` (voluntary disclosure) is hardcoded on `ship-after-expiry`, which has
  no regime test and runs over BIS licences — so EAR rows carry an ITAR disclosure cite
- `127.1` sits on proviso rules that also run over EAR authorizations
- `127.1` sits on `usml-on-ear-auth`, which never tests whether anything shipped
- `122.5` is cited as the source of an obligation to *reason* about jurisdiction; it is
  a records-retention rule. This is policy, and should say so

All three critical-tier rules are affected. Citations must be gated on `a.regime`.

### C4 — `isLive` drops over-consumed authorizations, and no rule catches them · verified, with a correction to the panel

`index.html:1478` excludes any authorization where `shippedBy >= authorizedValueUsd`.
Such a record exits `liveAuths` and therefore exits all fifteen rules and every measure,
surfacing only as the label **Fully consumed**. There is no over-consumption rule among
the fifteen — I listed all fifteen keys to confirm. The README asserts the opposite:
*"Utilization above 100% appears only where value was shipped against an authorization
that had none left — a condition the rules flag."* A documented control that does not
exist is itself the finding.

**Where the panel overstated it.** Five of six ranked this their single highest-priority
change, on the reasoning that it is concealing a live violation right now. Against this
dataset it is not. Measured:

- exactly **one** authorization in 194 value-bearing records exceeds its ceiling —
  `DSP5-2420584`, at 102.5%
- it expired 2025-12-04, six months before the as-of date, so it fails `isLive` on the
  expiry test regardless of the consumption clause
- **zero** authorizations inside their live window are at or over 100%

So the consumption clause is currently hiding nothing, and utilization above 100% never
renders for a live authorization. The blind spot is real and worth closing — a feed that
did produce an over-shipped live licence would lose it silently — but it is an
uncovered condition plus a false documentation claim, not an active concealment. On the
evidence, C1 is the more serious defect: it manufactures a reassuring wrong number on
screen today.

### C5 — A broken automated control fails silently · verified

A mistyped `AUTOMATED_MEASURES` key resolves to `undefined`, yielding population 0,
`null` conformance, zero weight in the roll-up — and exemption from
`control-test-overdue`, which filters `method !== 'Automated'` (`index.html:1892`). The
control renders as **Continuous** with an em dash and disappears from every place it
should appear. Needs a referential-integrity assertion at load.

### C6 — Process compliance findings are not as-of filtered · verified

`index.html:1675`, `1690` and `3500` all use `findings.filter(f => !f.closedOn)` with no
`raisedOn <= at` / `closedOn > at` test, so every historical point on the 12-month
corrective-action trend shows today's open findings. The trend is retroactively
falsified. Same root cause as C7 in the systems review: current-state attributes
back-projected onto a time series.

### C7 — `expiring-no-renewal` is born due · verified

`onsetOn` is `expiresOn − 120 days`; the serious tier SLA is 30 days; the rule fires
when expiry is within 90 days. So `dueOn` equals `expiresOn − 90` — the exact day the
rule first becomes visible. It is due on sight and overdue from day two, which is what
makes "overdue" stop meaning anything. Either the rule fires at 120 days or the onset is
the day it fires.

### C8 — `usml-on-ear-auth` fires critical on a paperwork mismatch

It never tests whether anything shipped. The classification lead's split is the right
shape: a **serious** rule for the listing disagreeing with the determination of record,
and a **critical** rule only where shipments post-date the determination — because
before that date we were not on notice.

### C9 — CAPAs are generated independently of failures · verified

`index.html:1405` raises corrective actions on automated controls at `chance(0.3)`
regardless of whether the control actually failed. A generator artefact, but it breaks
the causal link the page implies.

### C10 — Clearing an action leaves no trace

Unlogged, unreasoned deletion, raised independently by counsel, audit and the empowered
official. Clearing needs a reason code and a record.

## Enhancements

Capability the panel wants that the dashboard does not have. These are scope decisions
rather than fixes.

| | Change | Argued by | Notes |
| --- | --- | --- | --- |
| E1 | **Line items** — promote `partNumbers: string[]` to `lines[]` with quantity and value per line | Licensing, Systems | The minimum schema change unlocking the most: quantity-based expiry under 22 CFR 123.21, over-consumption on both axes, per-line destination tiering |
| E2 | DS-2032 registration status and empowered-official register | EO | No ITAR authorization survives a lapsed registration; nothing on the page shows it |
| E3 | Voluntary disclosure pipeline with the 60-day 127.12 clock | EO, GC | Currently the queue points at a decision it cannot track |
| E4 | Restricted-party screening | Systems, GC | `foreignParties` is stored and used by **zero** rules — capability evidenced without use |
| E5 | Destination risk tiering | Systems | `destinations` and `region` feed exactly one chart |
| E6 | Queue trend, aging and closure rate | Systems | The queue has no time dimension at all |
| E7 | Control-list versioning and trigger-driven re-review | Classification | A calendar cannot see the Federal Register amendment that actually invalidates a determination |
| E8 | Conformance scoreboard rebuilt as the vector above | Audit | Follows from the unanimous finding |
| E9 | Proviso typing, and `acknowledgedOn` → `implementedOn` | EO | A live retransfer-restriction breach currently sits at Watch while three boilerplate clauses sit at Serious |
| E10 | Status-pausing for blocked classifications | Classification | CJ pending and awaiting-engineering-data should stop the clock |
| E11 | Scope-headroom panel for Part 124 agreements | Licensing | Once WDA is not value-bearing, its real limits are article list, territory, sublicensees, term and the 124.14(e) annual report |

## The standing rule the panel converged on

Three separate mechanisms in this build convert *we do not know* into *nothing to see*:
the consumption clause in `isLive`, the regime filter in `keepPart`, and an unresolvable
measure key in `controlResult`. All three produce a clean-looking page from an
unexamined population.

The generalisable fix, and the best single sentence to come out of the review:

> No record may exit scope silently. Every scope-narrowing expression must route the
> records it excludes somewhere visible, rather than dropping them.

That is a test, not a preference, and it is worth asserting in code.
