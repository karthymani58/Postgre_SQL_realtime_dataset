# Odoo ERP Inventory & Sales Intelligence Query (PostgreSQL)

A production-ready PostgreSQL CTE (Common Table Expression) query designed for **Odoo ERP** instances. This script performs comprehensive inventory categorization, cost roll-ups, sales velocity tracking, and demand variability analysis across Product Variants, Point of Sale (POS), Sale Orders, and Bill of Materials (BoM).

---

## Key Features

* **Multi-Layer Cost Fallback Mechanism**: Calculates accurate unit costs by checking direct standard prices, latest stock valuation layers, and Bill of Materials (BoM) roll-ups (at variant and template levels).
* **Dynamic FSN & XYZ Analysis**:
  * **FSN (Fast, Slow, Non-moving)**: Categorizes items based on 6-month sales performance.
  * **XYZ Analysis**: Measures demand predictability using the **Coefficient of Variation ($\text{CV} = \frac{\sigma}{\mu}$)**.
* **Unified Sales Engine**: Aggregates 6 months of historical sales across standard **Sale Orders (`sale.order`)** and **Point of Sale (`pos.order`)**.
* **Enterprise Matrix Mapping**: Blends existing ABC classification with real-time XYZ demand profiles to generate matrix codes (e.g., `AX`, `BY`, `CZ`).
* **Robust Deduplication & Filtering**: Employs PostgreSQL `DISTINCT ON` and safe JSON translation unwrapping (`pt.name ->> 'en_US'`) to ensure exactly **1 row per active SKU**.

---

## 📊 Analytics Breakdown

### 1. FSN Classification (Velocity)

| Classification | Category | Condition (6-Month Sales Volume) |
| :--- | :--- | :--- |
| **F** | Fast-Moving | $\ge 100\text{ units}$ |
| **S** | Slow-Moving | $> 0\text{ and } < 100\text{ units}$ |
| **N** | Non-Moving | $0\text{ units sold}$ |

### 2. XYZ Classification (Predictability)

Evaluated via the Coefficient of Variation ($\text{CV}$) over active selling days:

$$\text{CV} = \frac{\text{Standard Deviation of Daily Sales}}{\text{Mean Daily Sales}}$$

| Classification | Demand Pattern | CV Threshold |
| :--- | :--- | :--- |
| **X** | Constant / High Predictability | $\text{CV} \le 0.5$ |
| **Y** | Fluctuation / Medium Predictability | $0.5 < \text{CV} \le 1.0$ |
| **Z** | Sporadic / Unpredictable | $\text{CV} > 1.0\text{ or } 0\text{ Sales}$ |

---

## Query Architecture & Execution Flow

```mermaid
graph TD
    A[date_config] --> FSN_XYZ[fsn_xyz_calculated]
    B[unique_mktg_map] --> Final[Final Select]
    C[base_costs] --> BOM[bom_calculated]
    C --> Final
    D[stock_valuation_cost] --> Final
    E[sales_so] --> Combined[daily_combined_sales]
    F[sales_pos] --> Combined
    Combined --> Agg[aggregated_sales]
    Agg --> FSN_XYZ
    BOM --> Final
    Agg --> Final
    FSN_XYZ --> Final

## Pipeline Overview

* **`date_config`**: Establishes dynamic 6-month historical windows based on `CURRENT_DATE`.
* **`unique_mktg_map`**: Deduplicates product marketing categories to fetch the latest target month profile.
* **`base_costs`**: Parses standard prices from Odoo's `ir_property` table for both variants and templates.
* **`stock_valuation_cost`**: Obtains the latest unit cost from `stock_valuation_layer`.
* **`bom_calculated`**: Dynamically calculates BoM component costs using active manufacturing structures.
* **`daily_combined_sales`**: Merges POS and B2B/Sales order lines by date and product ID.
* **`aggregated_sales` & `fsn_xyz_calculated`**: Computes standard deviations, active days, daily averages, and FSN/XYZ metrics.

## Schema Requirements

The query assumes a standard Odoo core database structure with the following tables:

* **Products**: `product_product`, `product_template`
* **Configuration & Metadata**: `ir_property`, `mktg_pdt_category_map` *(custom/extension map)*
* **Inventory & Manufacturing**: `stock_valuation_layer`, `mrp_bom`, `mrp_bom_line`
* **Sales**: `sale_order`, `sale_order_line`, `pos_order`, `pos_order_line`
## Output Fields

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| `internal_reference` | `text` | Product Internal Reference (`default_code`) |
| `barcode` | `text` | Product EAN/UPC barcode |
| `product_name` | `text` | Clean translated product name (JSON/Text handled) |
| `description_full` | `text` | Display format: `[SKU] Product Name` |
| `mtkg_category_l1` | `text` | Top-level marketing category |
| `abc_classification` | `text` | Priority class (`A`, `B`, or `C`) |
| `fsn_classification` | `text` | Velocity class (`F`, `S`, or `N`) |
| `xyz_classification` | `text` | Predictability class (`X`, `Y`, or `Z`) |
| `abc_xyz_matrix` | `text` | Matrix code (e.g., `AX`, `CZ`) |
| `cost_price` | `numeric` | Evaluated unit cost (multi-tier fallback) |
| `mrp` | `numeric` | Base list price (`list_price`) |
| `shelf_life_months` | `numeric` | Computed shelf life derived from expiration/use time |
| `total_6m_sold_qty` | `numeric` | Total units sold over the past 6 months |
| `daily_avg_sales_qty` | `numeric` | Average sales volume per calendar day |
| `max_daily_sales_qty` | `numeric` | Peak daily sales volume achieved |
| `6M Avg Sales (Units/ Month)` | `numeric` | Monthly run-rate over the 6-month period |

