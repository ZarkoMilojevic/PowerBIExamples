# Warehouse Replenishment & Inventory Aging (Power BI)

**Goal:** Monitor days of supply (DoS), fill rate proxy, receipts/shipments flow, and inventory aging across warehouses and SKUs.

## Files
- `data/skus.csv`
- `data/inventory_daily.csv`
- `data/receipts.csv`
- `data/purchase_orders.csv`

## Model Steps
1. Import all CSVs. Create a **Date** table:
   ```DAX
   Date = CALENDARAUTO()
   ```
   Mark as date table.
2. Relationships:
   - `inventory_daily[SKU]` → `skus[SKU]` (M:1)
   - `inventory_daily[Date]` → `Date[Date]` (M:1)
   - `receipts[SKU]` → `skus[SKU]` (M:1) and `receipts[ReceiptDate]` → `Date[Date]`
   - `purchase_orders[SKU]` → `skus[SKU]`; `purchase_orders[ETA]` → `Date[Date]`

## DAX Measures (create table *Inventory Metrics*)
```DAX
Units On Hand = SUM(inventory_daily[OnHand])

Units Shipped = SUM(inventory_daily[Shipments])

Units Received = SUM(inventory_daily[Receipts])

Demand (28d) = 
CALCULATE(SUM(inventory_daily[Demand]), DATESINPERIOD('Date'[Date], MAX('Date'[Date]), -28, DAY))

Avg Daily Demand (28d) = DIVIDE([Demand (28d)], 28)

Days of Supply =
DIVIDE([Units On Hand], [Avg Daily Demand (28d)])

Last Receipt Date =
CALCULATE(MAX(receipts[ReceiptDate]), ALLEXCEPT(inventory_daily, inventory_daily[SKU], inventory_daily[Warehouse]))

Inventory Age (days) =
DATEDIFF([Last Receipt Date], MAX('Date'[Date]), DAY)

Aging Bucket =
VAR age = [Inventory Age (days)]
RETURN
SWITCH(TRUE(),
    age <= 30, "0-30",
    age <= 60, "31-60",
    age <= 90, "61-90",
    "90+"
)

Fill Rate Proxy % =
1 - DIVIDE(
        CALCULATE(SUM(inventory_daily[Demand]) - SUM(inventory_daily[Shipments])),
        SUM(inventory_daily[Demand])
    )
```

> Note: *Fill Rate Proxy* uses shipments vs demand as a rough indicator.

## Visuals to Build
- **KPI Cards:** `Days of Supply`, `Fill Rate Proxy %`, `Units On Hand`.
- **Waterfall:** `Units On Hand` by `Date[Date]` showing Receipts vs Shipments (use measures).
- **Decomposition Tree:** `Days of Supply` → `Warehouse` → `Category` → `SKU`.
- **Matrix:** `Warehouse` x `Aging Bucket` with `[Units On Hand]` and data bars.
- **Line chart:** trend of `[Avg Daily Demand (28d)]` by date.

## Extras
- Tooltip page for SKU with UnitCost and last receipt info.
- Conditional formatting: highlight `Days of Supply` < 10 in red.

## Success Criteria
- Demonstrates time intelligence, buckets via `SWITCH(TRUE())`, and decomposition/tree & waterfall visuals.
