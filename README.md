# Build prompt — FX seasonality dashboard (Dash / Python)

> Copy everything below the line into a fresh conversation with an AI coding assistant.

---

## Context

I am building an internal research tool for an FX trading desk. It tracks **documented
seasonality patterns in foreign exchange** — recurring windows where a currency tends to
strengthen or weaken because of an identifiable, mechanical flow (dividend repatriation,
tax payment deadlines, remittance peaks, harvest export cycles, fiscal year-ends,
tourism receipts).

Coverage is roughly 35 currency pairs, weighted toward emerging markets (Asia, EMEA,
LATAM, Africa) with some G10.

You are building **Phase 1** of the dashboard. A statistical validation engine (backtest)
exists as a separate future component and is explicitly **out of scope**.

## The single most important rule

**The dashboard must never display a number it cannot justify.**

Phase 1 has no backtest, therefore it must show **no inferential statistics**: no hit
rate, no p-value, no Sharpe ratio, no composite score, no confidence interval, no
"validated/rejected" verdict badge.

Descriptive statistics are allowed and encouraged: individual yearly outcomes, medians,
ratios of current-year to prior-year values, counts.

Do not add greyed-out placeholder columns "for later". An absent column is honest; a
placeholder acquires false credibility within weeks. If a view feels empty without a
score, that is the correct and intended state.

**Corollary — prefer showing the distribution over the summary statistic.** Where you
would be tempted to display "median return +1.7%", instead plot all fifteen individual
yearly outcomes as points. The reader extracts more, and it doubles as a data-quality
check: a missing year exposes a calendar bug, an extreme outlier exposes an unflagged
regime break.

## Tech constraints

- Python 3.11+, **Dash 2.x** with `plotly.graph_objects` (not Dash Bootstrap Components
  unless you need a grid; keep dependencies minimal)
- Data is read from **local parquet files** at startup and on an explicit refresh button.
  No database, no live API calls, no Bloomberg dependency in this deliverable.
- Must run end-to-end with `python app.py` on a machine with **no market data access**,
  using a synthetic fixture generator (see below). This is a hard requirement, not a
  nice-to-have — the real data lives on a separate terminal machine.
- Single-user tool. No auth, no multi-tenancy, no deployment config.
- Use a consistent, restrained visual style. Do not rely on red/green alone to encode
  direction; pair every colour with a shape, label, or position cue.

## Data model

Load these from `data/`. Assume they exist; the fixture generator creates them.

### `registry.yml` — the seasonality definitions (hand-authored, YAML)

```yaml
seasonals:
  - id: KRW_DIV_APR
    pair: USDKRW
    region: asia                # asia | emea | latam | africa | g10
    direction: long_usd         # long_usd | short_usd | long_base | short_base
    mechanism: "Korean dividend season, high foreign ownership of KOSPI"
    anchor:
      type: dividend_season     # fixed_month_day | fiscal_year_end | lunisolar |
                                # tax_calendar | dividend_season | nth_business_day
      market: KOSPI
    window: {entry_offset_bd: -15, exit_offset_bd: 12}
    corroborating_flows: [foreign_equity_ownership_kr]
    status: candidate
```

### `anchors.parquet` — resolved anchor dates, one row per (seasonal, year)

| column | type | notes |
|---|---|---|
| `seasonal_id` | str | FK to registry |
| `pair` | str | e.g. `USDKRW` |
| `year` | int | |
| `anchor_date` | datetime | the resolved event date, t=0 |
| `level` | int | **1** = rule-based fallback, **2** = derived/computed, **3** = observed from real data |
| `source` | str | how it was resolved, free text |
| `dispersion_days` | float | nullable; interquartile spread of the underlying flow |
| `entry_date` | datetime | anchor_date shifted by entry_offset_bd business days |
| `exit_date` | datetime | |
| `direction` | str | |

`level` is important and must be surfaced in the UI — see the pair detail view.

### `fx_panel.parquet` — daily prices

`date`, `pair`, `spot`, `fwd_points`, `fwd_outright`

### `window_results.parquet` — realised outcome of each historical window

`seasonal_id`, `pair`, `year`, `entry_date`, `exit_date`, `n_days`, `ret_net`,
`contrib_spot`, `contrib_carry`, `regime_flag` (bool — year overlaps a known regime
break such as a devaluation or capital controls)

These are **realised historical returns**, computed arithmetically. They are descriptive
facts, not backtest output, and are therefore in scope.

### `flows.parquet` — corroborating macro/flow series

`series_id`, `date`, `value`, `unit`, `freq`

### `events.parquet` — economic event calendar

`date`, `country`, `kind` (`CB_MEETING` | `CPI` | `GDP` | `EMPLOYMENT` | `INDEX_REBAL` |
`ELECTION` | `BUDGET`), `impact` (1 = major, 2 = secondary, 3 = minor),
`scheduled_ahead_days` (int)

## Fixture generator — build this first

Write `fixtures/generate.py` producing all parquet files above plus a `registry.yml`
containing **10 seasonalities spanning all five regions**.

Requirements:

- 15 years of daily prices per pair, geometric random walk, plausible vol per pair
  (G10 ~7% annualised, EM Asia ~8%, LATAM ~14%, Africa ~12%)
- Realistic forward points implying plausible carry (e.g. USDTRY strongly positive
  forward premium, USDJPY negative, USDKRW near zero)
- **Inject a known seasonal effect** into each pair inside its declared window, with a
  configurable effect size and noise level. This makes the dashboard verifiable: if a
  view does not show the injected pattern, the view is wrong.
- Deliberately include edge cases so the UI is tested against them:
  - one seasonality with a window crossing 31 December
  - one pair with three missing years of price data
  - one year flagged `regime_flag = True` with an extreme return
  - one seasonality whose anchors are `level=1` before 2016 and `level=3` after
  - one pair with only 6 years of history
- Deterministic: seed the RNG and expose the seed.

## Views

Build four tabs plus one modal/drawer.

### Tab 1 — Upcoming windows (landing view)

A table of seasonal windows whose entry date falls within the next 60 calendar days or
the last 5 days.

**Sort by entry date ascending. Nothing else.** There is no ranking in Phase 1.

Columns: pair · direction (as a labelled arrow, not colour alone) · seasonality name ·
mechanism (truncated, full text on hover) · entry date · days to entry · exit date ·
anchor level (icon: filled = observed, half = derived, hollow = rule) · flow status
(see Tab 4) · count of major events falling inside the window.

Clicking a row opens the window detail drawer.

### Tab 2 — Seasonality calendar (Gantt)

A `px.timeline`-style horizontal Gantt.

- x-axis: rolling 12 months from today, with a vertical "today" line and a shaded band
  covering the next 45 days
- y-axis: one row per seasonality, grouped and separated by region
- bar per window, coloured by direction (local currency under pressure vs supported)
- bar opacity or hatching by anchor level — do not encode anything statistical
- a thin density strip pinned below the plot showing the count of `impact = 1` events
  per week, aggregated across all countries. Background information only, no detail.
- filters: region, direction, anchor level, pair
- clicking a bar navigates to Tab 3 for that seasonality

Windows crossing year-end must render correctly — do not split them into two bars
without visually connecting them.

### Tab 3 — Pair detail

Selected via dropdown or by clicking through from Tab 2.

Three stacked panels:

**a) Yearly outcome strip.** A dot plot, one row per year, x = realised net return of
that window, y = year. A solid vertical line at zero, a dashed vertical line at the
median. Colour points by sign; render `regime_flag` years in a third distinct colour and
**exclude them from the median calculation**, stating so in a caption.

This is the most important chart in Phase 1. Make it clean and prominent.

**b) Anchor provenance timeline.** One marker per year showing the resolved anchor date
(as day-of-year or calendar position) with the marker style encoding `level`. This
reveals whether the anchor is drifting over time and whether early years rely on the
rule-based fallback.

**c) Return decomposition.** Stacked bar per year splitting `ret_net` into
`contrib_spot` and `contrib_carry`. If carry dominates, the user needs to see that
immediately — a seasonality whose return is mostly carry is a different trade.

Header block: pair, region, direction, mechanism text, window offsets in business days,
number of years available, count of years by anchor level.

### Tab 4 — Flow monitor

For each seasonality with `corroborating_flows`, show whether the underlying mechanism
is materialising **this year**.

Per flow series: a small multiple line chart with the current year in bold, the prior
five years in light grey, and the 5-year median as a dashed line. Above it, a single
ratio — current-year cumulative value as a percentage of the 5-year median at the same
point in the calendar.

Traffic-light style status, but with text labels alongside colour:
`≥90%` normal · `70–90%` below normal · `<70%` flow not materialising.

This ratio is descriptive arithmetic, not a statistic, and is in scope.

### Drawer — Window detail with event overlay

Opens from Tab 1 or Tab 2. For a single upcoming window:

- a horizontal timeline spanning entry date to exit date, with a bar for the window
- below it, one row per event type (FOMC, US CPI, local central bank, local CPI, index
  rebalance, election), each with markers at the actual dates
- the anchor date itself marked on its own row
- vertical shaded columns highlighting dates where an `impact = 1` event falls inside
  the window
- an explicit note if a major event falls within the first 3 business days of the
  window, since that is a reason to delay entry

Event rows must not overlap. Do not rotate text.

## File structure

```
fxdash/
├── app.py                  Dash app, layout, callbacks
├── config/registry.yml
├── data/                   parquet files
├── fixtures/generate.py
├── fxdash/
│   ├── loaders.py          parquet -> dataframes, schema validation on load
│   ├── transforms.py       pure functions: window filtering, flow ratios, event overlap
│   └── views/
│       ├── upcoming.py
│       ├── calendar.py
│       ├── pair_detail.py
│       ├── flow_monitor.py
│       └── window_drawer.py
└── tests/
```

Keep every figure-building function **pure**: dataframe in, `go.Figure` out, no globals,
no I/O. Callbacks only wire inputs to those functions. This matters because these figure
functions will later be reused in a static PDF report generator.

## Data validation on load

`loaders.py` must raise on load, not warn, if:

- a `seasonal_id` in `anchors.parquet` is absent from `registry.yml`
- a pair referenced in the registry has no rows in `fx_panel.parquet`
- any `anchor_date` falls outside its stated year
- `entry_date >= exit_date` on any row
- more than 30% of business days are missing from a window's date range

Rationale: the failure mode that matters here is not a crash, it is a plausible-looking
chart built on broken data that someone then acts on. Fail loudly at load time.

## Acceptance criteria

1. `python fixtures/generate.py && python app.py` works on a clean machine with no
   market data access.
2. Every injected synthetic seasonal effect is visible in Tab 3 for its pair.
3. No inferential statistic appears anywhere in the UI.
4. The year-end-crossing window renders correctly in Tab 2.
5. The pair with missing years shows gaps rather than interpolated points.
6. The `regime_flag` year is visually distinct and excluded from the median.
7. Deliberately corrupting one parquet file causes a clear, named error at startup.
8. All figure functions are unit-testable without Dash running.

## Out of scope

Backtesting, statistical tests, composite scoring, alert ranking, notifications
(email/Slack), Bloomberg integration, authentication, deployment, and any form of trade
execution or order management.

## What I want from you

Start by restating the data contracts you will implement and asking me about anything
ambiguous. Then build the fixture generator first and show me a sample of each parquet
before writing any UI code.
