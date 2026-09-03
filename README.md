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

