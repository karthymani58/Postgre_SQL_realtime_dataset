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
