# Design: Why they died — shutdown reasons

**Date:** 2026-07-27
**Status:** Approved, ready for implementation planning

## Context

Signal Lost currently records **what** shut down and **when**: 109 projects, each with a
name, date, source link, announcement channel, and category. It is a solid archive, but
it is purely descriptive. It cannot answer the question people actually care about —
*why* did these projects die?

The raw material is already partly in hand. When the 46 RootData additions were
researched, the agents surfaced the reason in nearly every case (ZeroLend: unsustainable
economics plus hacks; HYTOPIA: waiting on an IRS refund with no cash to cover costs;
EXMO: UK sanctions; Poolin: Chapter 11 with $173.1M in liabilities). That information was
never captured — it lives only in the research notes.

This design adds the reason as a first-class field, making the registry analytical rather
than archival. It is also the dimension no other 2026 shutdown list covers.

## Goals

- Every project carries a categorized reason, a one-line explanation, and key figures.
- Readers can filter and chart by reason, and cross-reference reason against category.
- Unverifiable reasons are marked as such, never guessed.

## Non-goals

- Withdrawal-deadline tracking ("do you still have funds stuck?"). Real and valuable,
  but a separate feature. Deferred to a later phase.
- Funding-raised and project-lifespan fields. Deferred.
- Contagion/dependency graph visualization. The `dependency` reason tag captures the
  data; the visualization is deferred.

## Deferred: capital raised

Worth building later, with two caveats worked out first.

**Feasibility, measured 2026-07-27.** 14 of 107 entries already carry a raised amount,
picked up incidentally during the reason research — those alone total ~$339M, so the full
set plausibly lands between $1B and $2B in disclosed funding. RootData was tested as a
cheap source and is not sufficient: its `Total Raised` field is frequently `--`. It does
reliably expose **founding year** and **investor names**. So amounts need web research
(roughly one parallel-agent round, comparable in cost to the reason research), while
founding year — which would enable a "died at age X" stat — is close to free to scrape.

**Framing caveat, and the reason this is deferred.** "Money VCs lost" is not the same as
"total raised", and publishing it as such would be wrong:

- Some capital was explicitly returned. Entropy returned its $27M; OpenRank returned its
  remaining capital; Ranger liquidated and returned its treasury. Counting those as lost
  is false, and they are among the best-documented entries in the registry.
- Not all raised capital is VC capital — token sales, public rounds, grants and
  self-funding are mixed in (Powerloom's founders funded operations personally).
- Capital raised and then spent over several years on salaries and product is not
  "lost" in the same sense as capital destroyed in an exploit.

If built, label it **"disclosed capital raised by projects that shut down"** and track
returned-to-investors cases as a separate state. That distinction is more interesting than
a single total anyway: it separates an orderly wind-down from a wipeout.

## Data model

Three new optional fields per entry in the `DATA` array in `index.html`:

```js
{ n:"Stream Finance", iso:"2026-05-11", day:true, url:"…", c:"x", cat:"DeFi",
  why:  "exploit",
  note: "Lost $93M and xUSD depegged; wound down via a Delaware holding company.",
  figs: [{k:"Loss", v:"$93M"}]
}
```

- `why` — one tag from the taxonomy below. Required. Use `"unknown"` when not found.
- `note` — one sentence, plain English, describing what actually happened. Omitted when
  `why` is `"unknown"`.
- `figs` — 0 to 3 key figures, each `{k, v}`. Optional. Only include figures stated in a
  source.

All existing fields keep their current meaning. Every stat, chart, and filter continues
to be computed from `DATA` at runtime.

## Reason taxonomy

Eight reason tags plus an explicit `unknown`, derived from the observed data rather than
invented up front:

| Tag | Label | Meaning | Observed example |
|---|---|---|---|
| `funding` | Ran out of funding | No runway; failed or absent raise | HYTOPIA, Strobe Finance |
| `exploit` | Exploit / hack | Lost funds to an attack | Summer.fi, Pyra, Stream Finance |
| `traction` | No traction | Product never found demand; revenue/TVL collapse | Oxium, 0xPPL, Pingu |
| `market` | Market contraction | Sector-wide decline, not project-specific failure | NFTfi, Exchange Art, DL News |
| `regulatory` | Regulatory / sanctions | Forced or constrained by authorities | EXMO.com |
| `insolvency` | Insolvency / bankruptcy | Formal insolvency proceedings | Poolin |
| `dependency` | Dependency shut down | Died because something upstream died | Felix (USDH), Valhalla (MegaETH) |
| `voluntary` | Wound down by choice | Deliberate close: returning capital, consolidating into a parent product, or the team moving on | OpenRank, StableLab, Slingshot |
| `unknown` | Not reported | No reason found in any source | — |

`traction` and `market` are deliberately separate: the first is the project failing, the
second is the sector failing around it.

## Editorial standard

A shutdown reason is an interpretation, not a hard fact. A project may say "strategic
decision" when it in fact ran out of money.

The rule: **the tag reflects what the source says.** The `note` carries the nuance. Where
sources conflict or the stated reason looks incomplete, the note says so plainly. This
mirrors how `news` versus `x` source channels are already handled — the reader is always
told what kind of claim they are reading.

This standard is documented on the Data & Contribute tab so it is visible to readers, not
just to contributors.

## Research

All 109 projects need research. The 63 original entries have no reason data at all (they
came from the source spreadsheet with only name, date, and link). The 46 RootData
additions have partial reason data in the research notes but no consistently captured
figures.

Method — the same one that worked for the RootData merge:

- Parallel research agents, batched across the 109 projects.
- Hard anti-fabrication rules: only report a reason or figure literally stated in a
  fetched source; return `unknown` rather than infer.
- Prefer the project's own announcement (already stored in `url`) as the primary source,
  since it usually states the reason directly.
- Figures must be verifiable amounts (exploit size, liabilities, TVL at death), not
  estimates.

Expected outcome: high coverage, with a residual set marked `unknown`. That residual is
reported honestly rather than filled in.

## Interface

**Registry table** gains a `Why` column showing the tag as a compact colored chip.
Column layout becomes: `#`, `Project`, `Category`, `Why`, `Date`, `Announcement`. Widths
must be verified against the announcement-button overflow fixed previously — the button
needs its 210px column. If six columns prove too tight at desktop width, the fallback is
to render the `Why` chip beneath the project name inside the existing `Project` cell
rather than shrink the announcement column.

**Row expansion.** Clicking a row expands a panel beneath it containing the `note` and
any `figs`. Only one row expands at a time. Rows without a note are not expandable and do
not present themselves as clickable. Keyboard accessible; expansion state resets on
filter change.

**Filter.** A reason dropdown alongside the existing category and channel filters,
combining with them the same way.

**Charts.** A new "Why they died" horizontal bar chart, click-to-filter, matching the
existing category chart. Plus a category × reason cross-reference — the analytically
interesting view, showing e.g. that DeFi dies to exploits while Gaming dies to funding.

**Mobile** keeps the existing card layout; the reason chip and note fold into the card.

## Verification

- All 109 entries render; every chart sums to 109, including the new reason chart.
- Reason counts across the taxonomy sum to 109 with `unknown` included.
- Row expansion opens and closes, is keyboard operable, and resets on filter change.
- Reason filter combines correctly with search, category, channel, and month.
- No horizontal overflow at desktop width; announcement buttons stay inside the table.
- No console errors; light and dark themes both correct.
- Spot-check that every figure and note traces to the entry's cited source.
