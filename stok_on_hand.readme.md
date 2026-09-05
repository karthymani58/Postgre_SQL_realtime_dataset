# Odoo Stock on Hand (SOH) Analysis Query

A production-ready SQL CTE query designed for **PostgreSQL / Odoo ERP** and optimized for BI tools like **Metabase**. This query extracts, cleans, and aggregates stock-on-hand quantities across internal warehouse locations while handling multi-language JSONB translations and location hierarchies.

---

## 📑 Table of Contents
- [Overview](#overview)
- [Key Features](#key-features)
- [Database Dependencies](#database-dependencies)
- [Logic Breakdown](#logic-breakdown)
- [Metabase Setup](#metabase-setup)
- [Output Schema](#output-schema)

---

## 🔍 Overview

In Odoo, inventory counts (`stock_quant`) are tied to granular locations and stored as raw data. This query:
1. Filters exclusively for **Internal Locations** (ignoring supplier, customer, and virtual locations).
2. Parses Odoo's location path string (`complete_name`) into **Primary** and **Sub-Location** groupings.
3. Decodes Odoo's dynamic `JSONB` product name translations (`pt.name`).
4. Aggregates stock by location and SKU into **Physical**, **Reserved**, and **Available** Stock on Hand (SOH).

---

## ✨ Key Features

- **Location Hierarchy Extraction**: Splitting Odoo paths (e.g., `WH/Stock/Shelf 1`) into `WH` (Primary) and `Stock` (Sub-Location).
- **JSONB Translation Handling**: Safely extracts English product names (`en_US` or `en_GB`), falling back to raw text if localized keys are missing.
- **Dynamic Filtering**: Includes optional Metabase Field Filter syntax (`[[ AND primary_location_group IN ({{primary_group}}) ]]`).
- **Cleaned Data**: Excludes zero-balance records using `HAVING SUM(physical_soh) <> 0`.

---

## 🗄️ Database Dependencies

This query targets a standard **Odoo PostgreSQL database** (v12+) and relies on the following tables:

| Table | Alias | Purpose |
| :--- | :--- | :--- |
| `stock_quant` | `sq` | Stores inventory quantities per product location |
| `stock_location` | `sl` | Stores warehouse, rack, shelf, and bin locations |
| `product_product` | `pp` | Stores product variant data (e.g., SKU / `default_code`) |
| `product_template` | `pt` | Stores main product metadata and localized names |

---

## ⚙️ Logic Breakdown

### 1. Location Parsing
* `SPLIT_PART(sl.complete_name, '/', 1)`: Extracts the top-level parent location (e.g., `WH`).
* `SPLIT_PART(sl.complete_name, '/', 2)`: Extracts the second level (e.g., `Stock`). If empty, it defaults to `'Main Level'`.

### 2. Localized Product Names
In newer Odoo versions, `product_template.name` is stored as a `jsonb` field (e.g., `{"en_US": "Widget A", "fr_FR": "Gadget A"}`). The `->>` operator extracts JSON values as text.

### 3. Inventory Calculation
* **Physical SOH**: Total physical units located at the site.
* **Reserved SOH**: Units allocated to outbound delivery orders or manufacturing orders.
* **Available SOH**: `Physical SOH - Reserved SOH` (units free to sell/allocate).

---

## 📊 Metabase Setup

To use this query inside **Metabase**:

1. Paste the query into the SQL Editor.
2. In the right panel, configure the parameter **`primary_group`**:
   - **Variable Type**: `Field Filter` or `Text`
   - **Field to mapped to**: `primary_location_group` (or `stock_location -> Complete Name`)
   - **Filter Widget Type**: Dropdown / Category

> **Note:** If running outside of Metabase (e.g., in DBeaver, PgAdmin), remove or comment out the line containing:
> `[[ AND primary_location_group IN ({{primary_group}}) ]]`

---

## 📋 Output Schema

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| `primary_location_group` | `Text` | Top-level location path segment (e.g., `WH`) |
| `sub_location_group` | `Text` | Secondary location path segment or default |
| `full_location_path` | `Text` | Full location hierarchy path in Odoo |
| `sku` | `Text` | Product internal reference code (`default_code`) |
| `product_name` | `Text` | English/Primary product template name |
| `total_physical_soh` | `Numeric` | Total physical stock on hand |
| `total_reserved_soh` | `Numeric` | Total stock locked in pending transfers |
| `total_available_soh` | `Numeric` | Unreserved stock available for usage |

## 🏗️ Advanced Architecture Diagram

```mermaid
flowchart TB
    %% --- SOURCE DATA LAYER ---
    subgraph SOURCES["🗄️ DATA SOURCE LAYER (PostgreSQL / Odoo ERP)"]
        direction LR
        
        subgraph Quant_Source["Inventory Core"]
            sq[("<b>stock_quant (sq)</b><br/>• location_id<br/>• product_id<br/>• quantity<br/>• reserved_quantity")]
        end

        subgraph Dimension_Sources["Dimension Entities"]
            sl[("<b>stock_location (sl)</b><br/>• id<br/>• usage<br/>• complete_name")]
            pp[("<b>product_product (pp)</b><br/>• id<br/>• product_tmpl_id<br/>• default_code (SKU)")]
            pt[("<b>product_template (pt)</b><br/>• id<br/>• name (JSONB)")]
        end
    end

    %% --- CTE TRANSFORMATION LAYER ---
    subgraph CTE_LAYER["⚡ ETL CTE TRANSFORMATION LAYER (soh_data)"]
        direction TB

        subgraph JOIN_OPS["1. Entity Resolution & Joining"]
            j1["LEFT JOIN stock_location ON sq.location_id = sl.id"]
            j2["LEFT JOIN product_product ON sq.product_id = pp.id"]
            j3["LEFT JOIN product_template ON pp.product_tmpl_id = pt.id"]
        end

        subgraph FILTER_OPS["2. Location Scope Control"]
            f1["<b>WHERE Clause Filter</b><br/>sl.usage = 'internal'"]
        end

        subgraph PARSE_OPS["3. Field Expressions & Data Parsing"]
            p1["<b>Location Parsing</b><br/>• primary: SPLIT_PART(complete_name, '/', 1)<br/>• sub_group: COALESCE(NULLIF(SPLIT_PART(..., '/', 2)), 'Main Level')"]
            p2["<b>JSONB Translation Unnesting</b><br/>COALESCE(pt.name->>'en_US', pt.name->>'en_GB', pt.name::text)"]
            p3["<b>Row-Level Calculations</b><br/>available_soh = (sq.quantity - sq.reserved_quantity)"]
        end

        JOIN_OPS --> FILTER_OPS --> PARSE_OPS
    end

    %% --- BI INJECTION LAYER ---
    subgraph BI_FILTER["🎯 DYNAMIC QUERY CONTROLLER (Metabase)"]
        m1["<b>Optional Field Filter Directive</b><br/>[[ AND primary_location_group IN ({{primary_group}}) ]]"]
    end

    %% --- AGGREGATION LAYER ---
    subgraph AGG_LAYER["📊 AGGREGATION & CONSOLIDATION LAYER"]
        direction TB

        g1["<b>Multi-Dimension Grouping</b><br/>GROUP BY 1, 2, 3, 4, 5"]
        a1["<b>Quantity Summarization</b><br/>• SUM(physical_soh)<br/>• SUM(reserved_soh)<br/>• SUM(available_soh)"]
        h1["<b>Zero-Balance Suppression</b><br/>HAVING SUM(physical_soh) <> 0"]
        s1["<b>Sorting & Ranking</b><br/>ORDER BY primary ASC, sub_group ASC, total_physical DESC"]

        g1 --> a1 --> h1 --> s1
    end

    %% --- CONSUMPTION LAYER ---
    subgraph OUTPUT_LAYER["📈 CONSUMPTION LAYER"]
        out[("<b>Processed SOH Dataset</b><br/>• Structured Inventory Matrix<br/>• Real-time Availability Analysis")]
    end

    %% --- RELATIONSHIPS & DATAFLOW ---
    sq ==> j1
    sl ==> j1
    sq ==> j2
    pp ==> j2
    pp ==> j3
    pt ==> j3

    j3 ==> CTE_LAYER
    PARSE_OPS ==> BI_FILTER
    BI_FILTER ==> AGG_LAYER
    s1 ==> OUTPUT_LAYER

    %% --- STYLING ---
    classDef source fill:#1e293b,stroke:#475569,stroke-width:2px,color:#f8fafc;
    classDef cte fill:#0f172a,stroke:#3b82f6,stroke-width:2px,color:#f8fafc;
    classDef bi fill:#1e1b4b,stroke:#8b5cf6,stroke-width:2px,color:#f8fafc;
    classDef agg fill:#064e3b,stroke:#10b981,stroke-width:2px,color:#f8fafc;
    classDef output fill:#78350f,stroke:#f59e0b,stroke-width:2px,color:#f8fafc;

    class sq,sl,pp,pt source;
    class j1,j2,j3,f1,p1,p2,p3 cte;
    class m1 bi;
    class g1,a1,h1,s1 agg;
    class out output;
