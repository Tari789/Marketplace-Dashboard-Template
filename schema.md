# Source Schema

Three fact tables and two dimensions. Generic dimensional-model naming, this is a reference
structure, not a specific warehouse.

---

## `marketplace.fact_trip`

One row per trip request. The primary table for volume, cancellation, and reliability metrics.

| Column | Type | Notes |
|---|---|---|
| `trip_id` | STRING | Primary key |
| `city_id` | INTEGER | FK >> `dim_city` |
| `driver_id` | STRING | Null where unfulfilled |
| `rider_id` | STRING | |
| `segment` | STRING | Product segment identifier |
| `requested_at` | TIMESTAMP | |
| `completed_at` | TIMESTAMP | Null unless completed |
| `status` | STRING | `completed` / `cancelled` / `unfulfilled` |
| `cancelled_by` | STRING | `driver` / `rider` / null |
| `cancellation_reason` | STRING | Optional — nullable even when cancelled |
| `eta_minutes` | DECIMAL | Null where unfulfilled |
| `ata_minutes` | DECIMAL | Null where unfulfilled or cancelled pre-arrival |
| `pickup_is_airport` | BOOLEAN | |
| `dropoff_is_airport` | BOOLEAN | |
| `event_date` | DATE | **Partition column** |

**Grain:** one row per request, not per completed trip. A rider who requests, cancels, and
re-requests produces two rows.

**Partitioned on `event_date`.** Always filter on it, and keep it bare, never wrapped in a
function, or partition pruning is lost.

---

## `marketplace.fact_supply_hour`

One row per driver per day per city. Source for all supply-side metrics.

| Column | Type | Notes |
|---|---|---|
| `driver_id` | STRING | |
| `city_id` | INTEGER | FK >> `dim_city` |
| `segment` | STRING | |
| `hours_open` | DECIMAL | Online, no trip assigned |
| `hours_enroute` | DECIMAL | Assigned, driving to pickup |
| `hours_ontrip` | DECIMAL | Rider on board |
| `event_date` | DATE | **Partition column** |

**Grain:** driver-day-city. A driver working two cities in one day produces two rows — so
`COUNT(DISTINCT driver_id)` is required for driver counts, never `COUNT(*)`.

The three hour columns are mutually exclusive and sum to total online time.

---

## `marketplace.fact_rider_session`

One row per rider session. Source for conversion metrics.

| Column | Type | Notes |
|---|---|---|
| `session_id` | STRING | Primary key |
| `rider_id` | STRING | |
| `city_id` | INTEGER | FK >> `dim_city` |
| `segment` | STRING | |
| `requested_trips` | INTEGER | Requests in this session |
| `completed_trips` | INTEGER | Completions in this session |
| `event_date` | DATE | **Partition column** |

**Grain:** one row per session. A session can contain multiple requests, which is why session
conversion and trip completion rate give different answers.

---

## `dim_city`

| Column | Type | Notes |
|---|---|---|
| `city_id` | INTEGER | Primary key |
| `city_name` | STRING | |
| `country_code` | STRING | |
| `country_name` | STRING | |
| `region` | STRING | |
| `timezone` | STRING | |
| `launched_date` | DATE | Segment launch date for this city |

⚠️ **Verify uniqueness on `city_id` before joining.** A dimension with duplicate keys fans out the
fact table — inflating every count silently, with no error. Check with:

```sql
SELECT city_id, COUNT(*) FROM dim_city GROUP BY city_id HAVING COUNT(*) > 1;
```

`launched_date` matters for trend charts: a city contributes zero before launch, and including
pre-launch months drags down any average that spans the launch.

---

## `dim_date`

Standard date dimension — `date`, `week_start`, `month_start`, `quarter`, `year`, `is_weekend`.

Used as the spine for time series so months with no activity still appear as zero rather than
vanishing from the chart.

---

# MODELLING NOTES

**Grain differs across the three fact tables.** Trip level, driver-day-level, and session level
respectively. They cannot be joined directly without aggregating to a common grain first,
attempting it produces a fan-out.

**Distinct counts are not additive.** Active drivers and active riders must be recalculated at
every grain from the base data. Weekly figures cannot be summed into monthly.

**Airport flags are two booleans, not one.** Use `OR` so an airport-to-airport trip counts once.

**Cancellation reason is nullable even when cancelled.** Reason counts will not sum to cancellation
counts. Surface the gap as `not provided` rather than dropping it.
