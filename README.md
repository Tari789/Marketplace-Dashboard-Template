# Regional Marketplace Performance Dashboard Template

A reference implementation of a performance measurement framework for a two-sided mobility
marketplace, covering a single product segment across multiple countries and cities.

This repository contains the **measurement design**, not a production system: the metric
dictionary, the calculated-field definitions, and the dashboard layout specification.

All data referenced here is synthetic. Table and field names follow generic dimensional-modelling
conventions.

---

## What this is for

Most dashboard portfolios show a screenshot. The interesting part of building a reporting layer
isn't the chart, it's the decisions underneath it:

- What exactly counts as an "active driver"?
- Is utilization measured against online time or against total time?
- Does a cancelled trip count in the completion-rate denominator?
- What granularity does each audience actually need?

This repo documents those decisions for a marketplace product rolled out across ~8 countries.

---

## Structure

```
metrics/
  metric_dictionary.md      — every metric: definition, formula, source, edge cases
  calculated_fields.md      — implementation syntax (Looker Studio, Power BI, Sheets)
dashboard/
  layout_spec.md            — page structure, filters, chart types, audience
data/
  schema.md                 — the tables the metrics read from
```

---

## The design problem

Two audiences wanted different things from the same data:

| Audience | Granularity | Cadence | Purpose |
|---|---|---|---|
| Local market teams | City | Weekly | Operational troubleshooting |
| Central operations | Country | Monthly | Performance review |

The previous state was two separately-maintained reports that disagreed with each other, because
the same metric name meant different things to different teams.

**The approach taken here:** one metric definition layer underneath, two views on top. The city
weekly view and the country monthly view aggregate from the same base — so they can never
disagree, only differ in grain.

That's the core design decision in this repo and the reason the metric dictionary exists at all.

---

## Metric groups

| Group | Metrics |
|---|---|
| **Volume** | Completed Trips, Requested Trips, Unfulfilled Trips, Airport Trips |
| **Marketplace health** | Completion Rate, Driver Cancellations, Rider Cancellations, cancellation reasons |
| **Supply** | Supply Hours, Supply Hours per Active Driver, Utilization, Efficiency |
| **Reliability** | ETA, ATA, ETA-vs-ATA variance |
| **Demand** | Active Riders, Requesting Sessions, Completed Sessions |

See `metrics/metric_dictionary.md` for full definitions.

---

## Notes on scope

- Synthetic data only — no real volumes, markets, or identifiers
- Metric formulas are standard two-sided-marketplace definitions
- Table names are generic dimensional-model conventions, not from any specific warehouse
