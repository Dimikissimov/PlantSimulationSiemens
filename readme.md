# Pickstation Order Management System
### Siemens Plant Simulation 2304 — Demo Model
**Author:** M. Fittinghoff  
**Last Updated:** May 2026

---

## Overview

This model simulates a **goods-to-person picking system** with 4 pick stations, autonomous mobile robots (AMRs), and a warehouse management system (WMS) backed by SQLite. Orders are generated dynamically, assigned to pick stations, and fulfilled by workers retrieving SKUs from storage aisles.

---

## System Architecture

```
M_generateOrders (every 3600s)
        │
        ▼
   ORDERS table (SQLite)
        │
        ▼
M_selectOrders (triggered when empty box arrives)
        │
        ├──▶ PS_curOrders_1..4
        ├──▶ DQ_PS_selectedOrders (FIFO per station)
        ├──▶ PS_OrdersInQueue_1..4
        └──▶ @.DT_Box_OrderPos (on the box)
        │
        ▼
M_boxArrivesAtPickStation
        │
        ├──▶ DQ_PS_SKU_OnWay (check if SKU already ordered)
        ├──▶ M_retrieveSKUfromStorage (if SKU not on way)
        └──▶ DQ_PS_KomBox (box joins picking queue)
        │
        ▼
M_retrieveSKUfromStorage
        │
        └──▶ M_createStorageBox (creates physical storage box)
                │
                └──▶ BufferGasse → PickStation conveyor
        │
        ▼
Worker (via Broker — "Choose Nearest Worker")
        │
        ├──▶ Picks SKU from storage box
        ├──▶ Places in KomBox
        └──▶ Marks PickDone = true → returns to WorkerPool
```

---

## Model Parameters (Global Variables)

| Variable | Value | Description |
|----------|-------|-------------|
| `NoSKU` | 1000 | Total number of SKUs in assortment |
| `BoxAisle` | 800 | Max boxes per storage aisle |
| `SizeOrderPool` | 2000 | Target size of order pool |
| `MaxPos_Order` | 10 | Max positions per order |
| `NoPickstation` | 4 | Number of pick stations |

---

## SQLite Table Structure

### Global Tables
| Table | Columns | Description |
|-------|---------|-------------|
| `ORDERS` | ORDER_ID, SKU_ID, NoPICKS, GEN_TIME | Global order pool |
| `WMS` | SKU_ID, GASSE | Warehouse management — which SKU is in which aisle |

### Per Pick Station Tables (suffix _1 to _4)
| Table | Columns | Description |
|-------|---------|-------------|
| `PS_curOrders_n` | ORDER_ID, SKU_ID, NoPICKS, GEN_TIME | Orders assigned to station |
| `PS_OrdersInQueue_n` | ORDER_ID, SKU_ID, NoPICKS, GEN_TIME | Orders currently being processed |
| `PS_OrdersActive_n` | ORDER_ID, SKU_ID, NoPICKS, GEN_TIME | Orders actively being picked |

### Display/Debug Tables
| Table | Description |
|-------|-------------|
| `TBL_ORDERPOOL` | Mirror of ORDERS for UI display |
| `TBL_WMS` | Mirror of WMS for UI display |

---

## Object Structure

### DL_Pickstations (List)
Stores references to all 4 pick station objects.
```
DL_Pickstations[1] → Models.Model.PickStation
DL_Pickstations[2] → Models.Model.PickStation2
DL_Pickstations[3] → Models.Model.PickStation3
DL_Pickstations[4] → Models.Model.PickStation4
```

### DT_PS_SuchIndex (Table)
Defines the aisle search priority per pick station.
```
           Station1  Station2  Station3  Station4
Priority1:    1         2         3         4
Priority2:    2         1         2         3
Priority3:    3         3         4         2
Priority4:    4         4         1         1
```
85% of SKUs are expected to be found in the home aisle (1:1 mapping).

### WorkerPool
```
Travel Mode:  Move freely within area
Broker:       Broker (Choose Nearest Worker)
Parts Buffer: [central — not station-specific]
```

### Broker
```
Choose Nearest Worker: ✅
Importer Request:      M_selectOrders (or M_requestWorker)
Exporter Request:      M_workerDone
```

---

## Per Pick Station Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `Idx` | integer | Station index (1–4) |
| `DQ_PS_selectedOrders` | queue | FIFO queue of assigned order IDs |
| `DQ_PS_SKU_OnWay` | queue | SKUs currently being retrieved from storage |
| `DQ_PS_KomBox` | queue | Boxes waiting to be picked |
| `DL_PS_L` | list | List of boxes currently at station |
| `DT_tempOrders` | table | Temp table for order line processing |
| `BufferGasse` | object | Buffer conveyor from storage aisle |
| `SourceGasse` | object | Source object for storage boxes |
| `Workplace` | object | Worker workplace object |

---

## Methods

### `M_init`
**Called by:** Simulation start (automatic)  
**Purpose:** Initializes all tables, lists, SQLite databases, search index, and triggers first order generation.

**Key actions:**
- Creates WMS with random SKU-to-aisle assignment ("draw without replacement")
- Creates all SQLite tables (ORDERS, PS_curOrders_1..4, PS_OrdersInQueue_1..4, PS_OrdersActive_1..4)
- Sets station index (`Idx`) on each pick station
- Populates `DT_PS_SuchIndex`
- Calls `M_generateOrders`

---

### `M_generateOrders`
**Called by:** `M_init`, then loops every 3600 seconds  
**Purpose:** Continuously fills the ORDERS table up to `SizeOrderPool`.

**Key logic:**
- For each new order: randomly generates positions (1–MaxPos_Order)
- For each position: randomly selects a unique SKU (no duplicates per order)
- Randomly generates number of picks per position (1–8)
- Inserts into ORDERS table
- Refreshes TBL_ORDERPOOL display table

---

### `M_selectOrders`
**Called by:** When an empty box arrives at a pick station  
**Purpose:** Assigns an order from the pool to the arriving box.

**Key logic:**
1. Reads `PickStation_L` attribute from box to identify station
2. Fills `DQ_PS_selectedOrders` up to 7 entries (lookahead buffer)
3. Pops oldest order (FIFO) and assigns to box via `DT_Box_OrderPos`
4. Transfers order lines through: ORDERS → PS_curOrders → PS_OrdersInQueue → DT_Box_OrderPos
5. Cleans up PS_curOrders and PS_OrdersInQueue after assignment

---

### `M_boxArrivesAtPickStation`
**Called by:** Box entry control at pick station  
**Purpose:** Processes box arrival, triggers SKU retrieval from storage.

**Key logic:**
1. Removes box from `DL_PS_L` waiting list
2. Reads order lines from `PS_OrdersInQueue`
3. Checks `DQ_PS_SKU_OnWay` to avoid duplicate storage requests
4. For each new SKU: calls `M_retrieveSKUfromStorage` in new call chain
5. Pushes box to `DQ_PS_KomBox`

---

### `M_createStorageBox`
**Called by:** `M_retrieveSKUfromStorage`  
**Parameters:** `v_sku: integer`, `v_pickstation: integer`  
**Purpose:** Creates a physical storage box in the correct aisle.

**Key logic:**
1. Searches WMS for SKU in home aisle first
2. If not found: searches other aisles using `DT_PS_SuchIndex` priority order
3. `waituntil` buffer has space
4. Creates box at `SourceGasse`, sets SKU/direction/station attributes
5. Moves box to `BufferGasse`

---

## Bug Fixes Applied

| Method | Bug | Fix |
|--------|-----|-----|
| All | `SQlite` typo (7+ places) | Changed to `SQLite` |
| `M_generateOrders` | `SQlite.sql()` used for INSERT | Changed to `SQLite.exec()` |
| `M_generateOrders` | `tbl_tmpOrders` wrong casing | Changed to `TBL_tmpOrders` |
| `M_generateOrders` | `finalize` missing in else branch | Added in both if/else branches |
| `M_generateOrders` | Integer truncation in random calc | Wrapped with `floor()` |
| `M_selectOrders` | `finalize` inside while loop | Moved after `sqlite.step` |
| `M_selectOrders` | `PS_OrdersInQueue` never cleaned | Added DELETE after box assignment |
| `M_selectOrders` | No empty ORDERS guard | Added COUNT check before while loop |
| `M_init` | LIMIT on WMS INSERT commented out | Uncommented `LIMIT BoxAisle` |
| `M_init` | Worker init commented out | Broker handles automatically |
| `M_init` | Missing semicolons in CREATE TABLE | Added throughout |
| `M_createStorageBox` | `SQLite.finalize` commented out | Uncommented, added in both branches |
| `M_createStorageBox` | Alternative aisle search not implemented | Full loop using `DT_PS_SuchIndex` |
| `M_createStorageBox` | `waituntil` only for home aisle | Now uses correct `TargetStation` |
| `M_boxArrivesAtPickStation` | Global `DT_tempOrders` race condition | Changed to station-specific table |
| `M_boxArrivesAtPickStation` | `v_order_id` overwritten in loop | Removed redundant assignment |
| `M_boxArrivesAtPickStation` | Fragile while loop for SKU check | Replaced with for loop + boolean flag |
| `WorkerPool` | Parts Buffer set to PickStation4 only | Should be central/empty |

---

## Known Open Items

- [ ] `M_retrieveSKUfromStorage` — bridge method to `M_createStorageBox`
- [ ] `M_workerDone` — mark PickDone, check if order complete, release worker
- [ ] Box exit control — what happens when completed box leaves pick station
- [ ] Broker Importer/Exporter Request methods need to be assigned
- [ ] WorkerPool Parts Buffer needs to be corrected from PickStation4

---

## Installation / Setup

1. Open `Pickstation.app` in Siemens Plant Simulation 2304 or later
2. Verify global variables are set (NoSKU, BoxAisle, SizeOrderPool, MaxPos_Order)
3. Ensure `Broker` object is connected to `WorkerPool`
4. Reset and start simulation — `M_init` runs automatically
5. Check console for `"End of init"` confirmation

---

## Dependencies

- Siemens Plant Simulation 2304+
- SQLite (built into Plant Simulation)
- SimTalk 2.0

---

## License

Internal demo model — Siemens Plant Simulation training/demonstration purposes.
