<div align="center">

# ZuuBites · Excel + SQL Capstone

**Turning three messy system exports from a Sri Lankan cloud-kitchen brand into answers managers can act on.**

![Excel](https://img.shields.io/badge/Excel-Power%20Query-217346?logo=microsoftexcel&logoColor=white)
![SQL Server](https://img.shields.io/badge/SQL%20Server-T--SQL-CC2927?logo=microsoftsqlserver&logoColor=white)
![Status](https://img.shields.io/badge/status-in%20progress-orange)
![Deadline](https://img.shields.io/badge/deadline-06%20Dec%202026-blue)
![Marks](https://img.shields.io/badge/marks-100-purple)

</div>

---

## The story

ZuuBites cooks kottu, rice & curry, fried rice, burgers, short eats, desserts and drinks in **8 delivery-only kitchens**. It sells through **4 channels**: App, Web, Phone, and two aggregator platforms (TukEats and DashLK).

Until now, every report has been built by hand from system exports, and nobody trusts the numbers. As ZuuBites' first data analyst, I take every order from **1 Jan 2024 to 18 Dec 2025** and:

1. **Clean** the raw exports in Excel and Power Query
2. **Load** them into a SQL Server database, then validate them
3. **Answer** nine managers' questions with SQL and Excel
4. **Recommend** a decision in a short memo

> *Part of the Zuu Crew "Zero to Data Analyst: Data Foundations" programme.*

---

## How the data flows

```mermaid
flowchart LR
    A[("Raw CSV exports<br/>orders · order_items · customers")] --> B["Excel + Power Query<br/>clean, dedupe, standardise"]
    R[("Reference files<br/>kitchens · menu_items")] --> C
    B --> C[("SQL Server<br/>ZuuBites database")]
    C --> D["Validate<br/>row counts + checkpoints"]
    D --> E["SQL analysis<br/>Q2 – Q7"]
    D --> F["Excel pivots & dashboard<br/>Q8 – Q9"]
    E --> G["Memo to Head of Growth<br/>Q10"]
    F --> G

    classDef raw fill:#fde2e2,stroke:#c0392b,color:#000
    classDef work fill:#fff4d6,stroke:#d68910,color:#000
    classDef db fill:#dbeafe,stroke:#2563eb,color:#000
    classDef out fill:#dcfce7,stroke:#16a34a,color:#000
    class A,R raw
    class B,D work
    class C db
    class E,F,G out
```

---

## The data model

Five tables go into the `dbo` schema of the **ZuuBites** database.

```mermaid
erDiagram
    CUSTOMERS ||--o{ ORDERS : places
    KITCHENS  ||--o{ ORDERS : cooks
    ORDERS    ||--|{ ORDER_ITEMS : contains
    MENU_ITEMS ||--o{ ORDER_ITEMS : "sold as"

    CUSTOMERS {
        string customer_id PK
        string first_name
        string last_name
        string email
        string mobile
        string city
        date   signup_date
        string nic
    }
    KITCHENS {
        string kitchen_id PK
        string kitchen_name
        string district
        string province
        date   opened_date
    }
    ORDERS {
        string   order_id PK
        string   customer_id FK
        string   kitchen FK
        string   channel
        string   aggregator
        datetime order_placed
        int      promised_mins
        datetime delivered_at
        string   status
        string   payment_method
        string   promo_code
        decimal  discount
        decimal  delivery_fee
    }
    ORDER_ITEMS {
        string  order_id FK
        int     line_no
        string  item_code FK
        string  item_name
        int     qty
        decimal unit_price
    }
    MENU_ITEMS {
        string  item_code PK
        string  item_name
        string  category
        decimal current_price
        decimal unit_cost
    }
```

> Aggregator orders (TukEats, DashLK) have **no Customer ID** and **no delivery fee**, because the platforms don't share customer details.

---

## How messy was the raw data?

I profiled the three raw files with **24 checks across 21 columns and 5 quality dimensions**. These are the biggest problems, as **% of rows affected**:

```mermaid
---
config:
  themeVariables:
    xyChart:
      plotColorPalette: "#e67e22"
---
xychart-beta horizontal
    title "Raw data issues (% of rows affected)"
    x-axis ["Mobile format", "Order status", "Signup date", "Channel", "Order date", "Delivery fee", "Kitchen name", "Unit price", "Promo code", "Discount"]
    y-axis "% of rows" 0 --> 70
    bar [60.78, 51.76, 50.65, 45.88, 45.43, 44.70, 43.06, 42.61, 38.41, 13.01]
```

<details>
<summary><b>See all 24 checks</b></summary>

| # | File | Column | Check | Dimension | Rows | % rows |
|--:|------|--------|-------|-----------|-----:|------:|
| 1 | orders | (whole row) | Exact duplicate rows | Uniqueness | 320 | 0.98% |
| 2 | orders | Kitchen | Not one of the 8 official kitchen names | Consistency | 14,095 | 43.06% |
| 3 | orders | Kitchen | "Test Kitchen" (not a real kitchen) | Validity | 6 | 0.02% |
| 4 | orders | Channel | Not exactly App / Web / Phone / Aggregator | Consistency | 15,021 | 45.88% |
| 5 | orders | Status | Not exactly Delivered / Cancelled | Consistency | 16,946 | 51.76% |
| 6 | orders | Order Placed | Not in ISO yyyy-mm-dd hh:mm format | Validity | 14,872 | 45.43% |
| 7 | orders | Discount | Text with % or a whole number | Validity | 4,260 | 13.01% |
| 8 | orders | Delivery Fee | Text with an "Rs" prefix | Validity | 14,633 | 44.70% |
| 9 | orders | Promo Code | Placeholder for "no code" (none, N/A, NULL, -) | Consistency | 12,573 | 38.41% |
| 10 | orders | Customer ID | Blank on a non-aggregator order | Completeness | 0 | 0.00% |
| 11 | orders | Delivered At | Delivered order with no delivery time | Completeness | 0 | 0.00% |
| 12 | order_items | (whole row) | Exact duplicate rows | Uniqueness | 714 | 0.98% |
| 13 | order_items | Qty | Blank (rule: blank means 1) | Completeness | 447 | 0.61% |
| 14 | order_items | Qty | Negative quantity | Validity | 293 | 0.40% |
| 15 | order_items | Unit Price | Text ("Rs." or thousands comma) | Validity | 31,117 | 42.61% |
| 16 | order_items | Item Code | Code not in menu_items (M090, M091) | Validity | 565 | 0.77% |
| 17 | order_items | Item Name | Name differs from the menu name for its code | Consistency | 3,685 | 5.05% |
| 18 | customers | (whole row) | Exact duplicate rows | Uniqueness | 35 | 0.61% |
| 19 | customers | NIC | Same person under 2 Customer IDs | Uniqueness | 110 | 1.91% |
| 20 | customers | Email | Blank | Completeness | 418 | 7.28% |
| 21 | customers | Email | Contains upper-case letters | Consistency | 848 | 14.76% |
| 22 | customers | Mobile | Not in 07XXXXXXXX format | Validity | 3,492 | 60.78% |
| 23 | customers | City | Blank | Completeness | 183 | 3.19% |
| 24 | customers | Signup Date | Not in ISO yyyy-mm-dd format | Validity | 2,910 | 50.65% |

Rows used for %: orders 32,737 · order_items 73,033 · customers 5,745 (raw, before cleaning).

</details>

### From raw to clean: the checkpoints

| Table | Raw rows | Clean rows (checkpoint) | Removed | Status |
|-------|---------:|------------------------:|--------:|:------:|
| customers | 5,745 | **5,600** unique people | 145 | ✅ cleaned |
| orders | 32,737 | **32,411** | 326 | 🔄 in progress |
| order_items | 73,033 | **72,313** | 720 | 🔄 in progress |

**2025 checkpoint:** 18,305 delivered orders worth **Rs 59,928,441** in net revenue.

> 💡 **Key finding so far:** net revenue only matches the checkpoint when negative quantities are treated as data-entry sign errors and turned positive. Keeping them negative gives Rs 59,435,682, which is Rs 492,759 short.

---

## 🍽️ The business at a glance

<table>
<tr>
<td width="50%" valign="top">

```mermaid
pie showData
    title Menu items by category (42)
    "Beverages" : 8
    "Kottu" : 7
    "Rice & Curry" : 6
    "Short Eats" : 6
    "Fried Rice" : 5
    "Burgers & Wraps" : 5
    "Desserts" : 5
```

</td>
<td width="50%" valign="top">

```mermaid
timeline
    title Kitchen openings
    2023 : Colombo 03 (Jun)
         : Rajagiriya (Aug)
         : Nugegoda (Sep)
         : Kandy (Oct)
         : Dehiwala (Nov)
         : Negombo (Dec)
         : Kurunegala (Dec)
    2025 : Galle (Jun) · newest
```

</td>
</tr>
</table>

---

## The questions I'm answering

```mermaid
pie showData
    title Marks by part (100 total)
    "Q1 · Data readiness" : 20
    "Q2–Q7 · SQL" : 40
    "Q8–Q9 · Excel analysis" : 25
    "Q10 · Judgement memo" : 15
```

| Q | Asked by | Question | Tool | Marks | Status |
|:-:|----------|----------|:----:|:-----:|:------:|
| 1 | Data lead | Make the data trustworthy: scorecard, cleaning log, load, validate | Excel / PQ + SQL | 20 | 🔄 |
| 2 | Head of Operations | Which kitchen is really performing? | SQL | 6 | ⬜ |
| 3 | Operations manager | Are we keeping our delivery promise? | SQL | 6 | ⬜ |
| 4 | Head of Menu | What should stay on the menu? | SQL | 7 | ⬜ |
| 5 | CRM manager | Who should we win back? | SQL | 7 | ⬜ |
| 6 | CEO | Are we growing? | SQL | 7 | ⬜ |
| 7 | Head of Loyalty | Does a bad first order cost us the customer? | SQL | 7 | ⬜ |
| 8 | Marketing manager | Did Kottu Tuesday (buy one, get one free) pay off? | Excel | 12 | ⬜ |
| 9 | Operations director | A one-page operations dashboard | Excel | 13 | ⬜ |
| 10 | Head of Growth | Should we push customers to the aggregators? | Memo + SQL | 15 | ⬜ |

✅ done · 🔄 in progress · ⬜ to do

---

## Business rules used everywhere

| Term | Rule |
|------|------|
| **Net revenue** | Qty × Unit Price (on the order line) × (1 − Discount) |
| **Gross profit** | Net revenue − Qty × unit cost (from menu_items) |
| **AOV** | Net revenue ÷ number of orders |
| **What counts** | Only *Delivered* orders count towards revenue, profit and AOV |
| **Delivery fee** | Never revenue |
| **On time** | Delivered within Promised Mins of Order Placed |
| **Blank Qty** | Means 1 |
| **Free (BOGO) items** | Their own order line with Unit Price 0 |

---

## Roadmap

```mermaid
gantt
    title Road to submission
    dateFormat YYYY-MM-DD
    axisFormat %d %b
    todayMarker stroke-width:3px,stroke:#e11d48

    section Q1 · Data
    Setup & profiling            :done,    s1, 2026-11-18, 3d
    Customers cleaning           :done,    s2, 2026-11-21, 4d
    Orders & order_items cleaning:active,  s3, 2026-11-24, 3d
    Scorecard & Cleaning_Log     :         s4, after s3, 1d
    SQL load & validate          :         s5, after s4, 2d

    section Analysis
    Q2 – Q7 SQL                  :         a1, 2026-11-29, 3d
    Q8 – Q9 Excel & dashboard    :         a2, 2026-12-02, 2d
    Q10 memo & Answers           :         a3, 2026-12-04, 1d

    section Submit
    QA, zip & submit             :crit,    q1, 2026-12-05, 2d
```

---

## Repository structure

```text
zuubites-excel-sql-capstone/
├── Data/
│   ├── Original Dataset/   ← raw CSVs, never edited
│   ├── Original Excels/    ← .xlsx working copies of the raw files
│   ├── Cleaning Dataset/   ← cleaning workbooks (customers, orders, order_items)
│   └── Data_Cleaned/       ← final cleaned outputs
├── docs/                   ← project plan & progress report
├── ZuuBites Take-Home Assessment.pdf
└── README.md
```

## Branches

```mermaid
gitGraph
    commit id: "start"
    branch develop
    branch feature/data-cleaning
    commit id: "clean data"
    checkout develop
    merge feature/data-cleaning
    branch feature/sql-analysis
    commit id: "SQL Q1–Q7"
    checkout develop
    merge feature/sql-analysis
    branch feature/dashboard
    commit id: "Q8–Q9 Excel"
    checkout develop
    merge feature/dashboard
    branch docs/report
    commit id: "memo & report"
    checkout develop
    merge docs/report
    checkout main
    merge develop id: "submit"
```

`feature/*` and `docs/report` → `develop` → `main` (final, submission-ready).

---

<div align="center">

**Arundethie Wijerathne** · Zero to Data Analyst capstone · 2026

</div>
