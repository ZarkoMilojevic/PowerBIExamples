# Fleet Fuel & Emissions (Cloud-Connected Power BI)

**Goal:** Track fuel efficiency (L/100km), emissions (kgCO2e), and idling impact by vehicle, depot, and time. Demonstrates **cloud data** and **scheduled refresh**.

## Files
- `data/vehicles.csv`
- `data/fuel_logs.csv`
- `data/telemetry.csv`
- `data/emission_factors.csv`

## Cloud Setup (Azure Blob Storage)
1. Create an Azure Storage Account and **Blob container** (e.g., `fleet-data`).
2. Upload the four CSVs to the container.
3. Generate a **SAS URL** (blob-level, read-only). Copy the full URL for each file.
4. In Power BI Desktop create **Parameters**:
   - `FuelLogsSasUrl` (Text) → paste the SAS URL for `fuel_logs.csv`
   - `VehiclesSasUrl`, `TelemetrySasUrl`, `EmissionsSasUrl` likewise
5. **Get Data → Web** using each parameter. In Power Query, set **Privacy** to Organizational (or as needed).

_Example Power Query (M) for one file:_
```M
let
    Source = Csv.Document(Web.Contents(FuelLogsSasUrl), [Delimiter=",", Encoding=65001, QuoteStyle=QuoteStyle.Csv]),
    Promote = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    ChangeTypes = Table.TransformColumnTypes(Promote,{{"Date", type date},{"VehicleID", type text},{"FuelLiters", type number},{"DistanceKm", type number}})
in
    ChangeTypes
```

## Model
- Relationships:
  - `fuel_logs[VehicleID]` → `vehicles[VehicleID]`
  - `telemetry[VehicleID]` → `vehicles[VehicleID]`
  - `Date` table via `CALENDARAUTO()`; relate `Date[Date]` to `fuel_logs[Date]` and `telemetry[Date]`
- Many-to-one, single-direction filters from `vehicles` and `Date` to facts.

## DAX Measures (table *Fleet Metrics*)
```DAX
Total Distance (km) = SUM(fuel_logs[DistanceKm])

Total Fuel (L) = SUM(fuel_logs[FuelLiters])

Fuel Efficiency (L/100km) = DIVIDE([Total Fuel (L)]*100, [Total Distance (km)])

Total CO2e (kg) =
SUMX(
    fuel_logs,
    fuel_logs[FuelLiters] * RELATED(emission_factors[kgCO2ePerLiter])
)

CO2e Intensity (g/km) = DIVIDE([Total CO2e (kg)] * 1000, [Total Distance (km)])

Idle Minutes = SUM(telemetry[IdleMinutes])

Idle per 100 km = DIVIDE([Idle Minutes]*100, [Total Distance (km)])

Variance to Target (L/100km) =
VAR Target = AVERAGE(vehicles[TargetLper100km])
RETURN [Fuel Efficiency (L/100km)] - Target

Rolling 30d CO2e (kg) =
CALCULATE([Total CO2e (kg)], DATESINPERIOD('Date'[Date], MAX('Date'[Date]), -30, DAY))
```

## Visuals
- **KPI Cards:** `Fuel Efficiency (L/100km)`, `CO2e Intensity (g/km)`, `Idle per 100 km)`.
- **Line chart:** `Rolling 30d CO2e (kg)` by date with slicers for `Depot` and `VehicleType`.
- **Bar chart:** `Variance to Target` by `VehicleID` (conditional color on +/-).
- **Treemap or Matrix:** `Depot` → `VehicleType` by `Total Fuel (L)`.
- **Map (optional):** depot locations (add a small lookup table with lat/lon).

## Publish & Refresh
1. **Publish** the PBIX to Power BI Service.
2. In the dataset **Settings → Data source credentials**, set each Web source to **Anonymous** with **Basic/None** (SAS URL already contains the token).
3. Configure **Scheduled refresh** (e.g., daily).
4. Optionally, connect **Power Automate** or a scheduled job to drop a new `fuel_logs_YYYYMMDD.csv` and use `Folder` connector with a parameterized SAS URL.

## Success Criteria
- Demonstrates cloud connectivity (Azure Blob + SAS), parameters, scheduled refresh, and intermediate DAX complexity.
