# Last-Mile Delivery Performance Dashboard (Power BI)

**Goal:** Analyze on-time delivery, delays by reason, route productivity, and geographic dispersion for a last‑mile operation.

## Files
- `data/drivers.csv`
- `data/routes.csv`
- `data/deliveries.csv`

## Model Steps
1. **Get Data → Text/CSV** and import all three CSVs.
2. Create relationships:
   - `deliveries[RouteID]` → `routes[RouteID]` (Many-to-One, Single)
   - `deliveries[DriverID]` → `drivers[DriverID]` (Many-to-One, Single)
3. **Date table (DAX):**
```DAX
Date =
ADDCOLUMNS (
    CALENDARAUTO(),
    "Year", YEAR ( [Date] ),
    "Month Number", MONTH ( [Date] ),
    "Month", FORMAT ( [Date], "MMM" ),
    "Year-Month", FORMAT ( [Date], "yyyy-MM" ),
    "Quarter", "Q" & FORMAT ( [Date], "Q" )
)
```
   Mark it as Date table with `Date[Date]`.
   
4. Create relationships `routes[RouteDate]` → `Date[Date]` (if needed, set to Many-to-One).

## DAX Measures
Create a measure table `Delivery Metrics`.
```DAX
'Delivery Metrics' =
SELECTCOLUMNS(
    { (0) },
    "Placeholder", 0
)
```
Then add:

```DAX
Total Deliveries =
COUNTROWS ( deliveries )

Delivered =
CALCULATE (
    [Total Deliveries],
    deliveries[Status] = "Delivered"
)

On-Time Deliveries =
CALCULATE (
    [Total Deliveries],
    deliveries[Status] = "Delivered",
    deliveries[LateFlag] = 0
)

On-Time Rate % =
DIVIDE ( [On-Time Deliveries], [Delivered] )

Avg Delay (mins) =
CALCULATE (
    AVERAGEX (
        deliveries,
        DATEDIFF ( deliveries[PlannedDateTime], deliveries[ActualDateTime], MINUTE )
    ),
    deliveries[Status] = "Delivered",
    deliveries[LateFlag] = 1
)

Stops per Route =
AVERAGEX (
    VALUES ( routes[RouteID] ),
    CALCULATE ( DISTINCTCOUNT ( deliveries[DeliveryID] ) )
)

Km per Stop =
DIVIDE (
    SUM ( deliveries[DistanceKm] ),
    [Delivered]
)

SLA Breach % by Reason =
DIVIDE (
    CALCULATE (
        [Delivered],
        deliveries[LateFlag] = 1
    ),
    [Delivered]
)

```

_Optional calculated column (for convenience):_
```DAX
Delay Minutes =
VAR d = DATEDIFF(deliveries[PlannedDateTime], deliveries[ActualDateTime], MINUTE)
RETURN IF(deliveries[Status] = "Delivered" && d > 0, d, 0)
```

### Field Parameter (metric switcher)
Use **Modeling → New parameter → Fields** and select `[On-Time Rate %]`, `[Avg Delay (mins)]`, `[Km per Stop]`.
Place the parameter in a slicer to change the main KPI card dynamically.

## Visuals to Build
- **KPI Card**: `On-Time Rate %` with target (e.g., 95%). Apply conditional formatting.
- **Decomposition Tree**: Analyze `On-Time Rate %` by `Depot → DriverName → DelayReason`.
- **Map**: Use `CustomerLat/CustomerLon`. Size by `Delay Minutes` or color by `LateFlag`.
- **Ribbon Chart**: `Depot` by `On-Time Rate %` over `Date[Date]` (daily trend).
- **Matrix**: `DriverName` vs `DelayReason` with `Avg Delay (mins)`.

## Extras
- Tooltip page showing “Route summary” (deliveries, average delay, distance).
- Drillthrough to a *Route detail* page filtered by `RouteID`.
- Bookmarks to toggle between *Performance* and *Geography* views.

## Success Criteria
- Demonstrates DAX filters (CALCULATE/FILTER), time diffs (DATEDIFF), and field parameters.
- Clear visuals with interactive drillthrough.
