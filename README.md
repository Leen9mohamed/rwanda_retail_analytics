# SmartMart Rwanda – Retail Sales Analytics with SQL JOINs and Window Functions

**Course:** SQL Database – Data Science Year 2, Kigali Independent University (ULK)
**Instructor:** Eric Maniraguha  |  **Assignment:** Group Assignment VI

## 1. Group members
| # | Name | Student ID | GitHub username |
|---|------|-----------|-----------------|
| 1 | Leen Mohamed | 202510947 | |
| 2 | Hadia Aldirdiri | 202512812 | |
| 3 | Eilaf Gamaledin | 202510531 | |
| 4 | Lina Nagi | 202512809 | |
| 5 | Award Mohamed | 202512287 | |

## 2. Business background
SmartMart Rwanda is a retail chain with five active branches in four regions (Kigali, Northern, Southern, Eastern) and a sixth branch (Nyagatare) opened in May 2026. It sells groceries, electronics, clothing, accessories and household goods. Large purchases (≥ 100,000 RWF) can be paid in three monthly instalments.

## 3. Problem statement
Management has transactional data but cannot answer, from one place: which branches, employees, products and customers drive revenue; how revenue evolves month by month; which customers are most valuable; and which payments are coming due. Without this, stock, staffing and marketing decisions are made by intuition.

## 4. Project objectives (five measurable success criteria)
| # | Objective | Function | Contribution to the business problem |
|---|-----------|----------|--------------------------------------|
| 1 | Top 5 products in every region | `RANK() OVER (PARTITION BY region ...)` | Adapts stock and promotions to regional demand |
| 2 | Cumulative monthly sales | `SUM() OVER (ORDER BY month ...)` | Tracks progress toward half-year revenue targets |
| 3 | Month-over-month comparison | `LAG()` / `LEAD()` | Detects growth, drops and seasonality early |
| 4 | Customer segmentation into 4 groups | `NTILE(4)` | Enables loyalty programmes for Platinum/Gold and win-back for Bronze |
| 5 | 3-month moving average | `AVG() OVER (ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)` | Smooths volatile revenue to show the real trend |

## 5. Database schema (7 tables)
| Table | Purpose | Keys |
|-------|---------|------|
| `regions` | Regions of Rwanda | PK `region_id` |
| `branches` | Shops | PK `branch_id`; FK `region_id` |
| `employees` | Staff and hierarchy | PK `employee_id`; FK `branch_id`; self-FK `manager_id` |
| `customers` | Registered customers | PK `customer_id`; FK `region_id` |
| `products` | Product catalogue | PK `product_id` |
| `sales` | One row per transaction (78 rows, Jan–Jun 2026) | PK `sale_id`; FKs `customer_id` (nullable), `product_id`, `branch_id`, `employee_id` |
| `payments` | Instalment schedule (33 rows) | PK `payment_id`; FK `sale_id` |

Constraints: NOT NULL, UNIQUE (`region_name`, `email`), CHECK (`unit_price > 0`, `quantity > 0`, `total_amount > 0`, `status IN ('PAID','PENDING')`), foreign keys and indexes (see `database/`).

## 6. ER diagram


![ERD](documentation/ERD.png)



## 7. How to run the scripts
Tested on **PostgreSQL 13+** (also on SQLite 3.39+ for the queries).
```bash
createdb smartmart
psql -d smartmart -f database/01_create_tables.sql
psql -d smartmart -f database/02_insert_data.sql
psql -d smartmart -f database/03_constraints.sql
psql -d smartmart -f queries/01_inner_join.sql      # ... then 02 to 09
```
MySQL 8 notes: no `FULL OUTER JOIN` (use the `UNION` version in `04_full_outer_join.sql`); replace `SUBSTR(CAST(sale_date AS VARCHAR(10)),1,7)` with `DATE_FORMAT(sale_date,'%Y-%m')` and `CAST(... AS NUMERIC)` with `CAST(... AS DECIMAL(10,3))`.

## 8. JOIN implementations (`queries/01–05`)
| JOIN | File | What it shows | Business interpretation |
|------|------|---------------|-------------------------|
| INNER | `01_inner_join.sql` | Complete sales report; revenue by region | Only identified-customer sales appear; Kigali earns the most revenue (2,523,500 RWF). |
| LEFT | `02_left_join.sql` | Products, branches and employees with no sales | *Wall Clock* never sold; *Nyagatare* (new) has no sales; 5 managers record no sales. |
| RIGHT | `03_right_join.sql` | Customers who never bought | Two recently registered customers (Kayitesi Solange, Mugabo Vincent) never purchased → onboarding follow-up. |
| FULL OUTER | `04_full_outer_join.sql` | Customers without sales **and** sales without customer | 2 inactive customers and 3 walk-in sales (158,500 RWF) that are not linked to any customer profile. |
| SELF | `05_self_join.sql` | Employee–manager hierarchy, customers in same region, colleagues in same branch | Reveals reporting lines and potential peer groups for regional campaigns. |

## 9. Window function implementations (`queries/06–09`)
| Category | Functions | File | Frames / clauses demonstrated |
|----------|-----------|------|-------------------------------|
| Ranking | `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `PERCENT_RANK` | `06_ranking_functions.sql` | `PARTITION BY region/branch`, `ORDER BY` |
| Aggregate | `SUM`, `AVG`, `MIN`, `MAX`, `COUNT` `OVER()` | `07_aggregate_window_functions.sql` | `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW`, `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` |
| Navigation | `LAG`, `LEAD` | `08_navigation_functions.sql` | `PARTITION BY customer`, default values |
| Distribution | `NTILE(4)`, `CUME_DIST` | `09_distribution_functions.sql` | `ORDER BY` revenue / frequency |

**ROW_NUMBER vs RANK vs DENSE_RANK.** With transaction counts 14, 13, 13, 10: `ROW_NUMBER` gives 1,2,3,4 (always unique); `RANK` gives 1,2,2,4 (tie, then a gap); `DENSE_RANK` gives 1,2,2,3 (tie, no gap). See Q2 in `06_ranking_functions.sql`.

**ROWS vs RANGE.** `ROWS` counts physical rows; `RANGE` includes all rows with the same `ORDER BY` value. In Q1 of `07_...` sales 19 and 20 share 2026-02-09: `ROWS` gives 461,000 then 509,000, while `RANGE` gives 509,000 for both.

Business problems solved with window functions (minimum required: 3): (1) regional product ranking for stock planning, (2) revenue trend and growth detection (cumulative, moving average, LAG), (3) customer/employee segmentation and payment follow-up (NTILE, LEAD).

## 10. Key analytical findings
**Descriptive – what happened?**
- Total revenue Jan–Jun 2026: **6,006,000 RWF**; best month May (1,803,000), lowest June (389,500); April fell 54.1% from March, May rose 185.7%.
- Kigali region produced 42% of revenue; **Kigali City Centre** is the top branch (2,222,000 RWF, 37%). Remera is last among active branches (301,500).
- **Samsung Phone A15** alone generated 2,750,000 RWF (about 46% of revenue) and is #1 in three of four regions (Northern: Rice).
- Top customers: Iradukunda Chantal (1,160,000) and Mukeshimana Alice (1,147,500). Top employee: Ingabire Claire (1,426,000).
- 3-month moving average peaked in May (1,269,667) and dropped to 941,167 in June.

**Diagnostic – why?**
- Revenue is driven by a few high-value items: phones brought 1,000,000 RWF (55% of May revenue) in the best month, versus only 250,000 in April and none in June, which explains the swings in the monthly trend.
- Kigali City Centre has two sales representatives and the largest urban customer base; Remera has one representative.
- Nyagatare opened on 2026-05-20, so no sales yet; walk-in sales are not captured as customers, hiding repeat-buying behaviour.

**Prescriptive – what next?**
1. Secure phone stock and run promotions in Northern/Southern regions where phone sales are low; keep a steady promotion of staples (rice, oil, LED bulbs) to reduce monthly volatility.
2. Launch loyalty rewards for Platinum/Gold customers and re-engagement offers for Bronze customers and the two inactive customers.
3. Require customer registration at checkout (reduce walk-in sales) and set up Nyagatare with a sales representative and local promotions.
4. Use the `LEAD` payment query weekly to call customers whose next instalment is due (e.g., 10–28 July 2026).
5. Review Remera's staffing and targets; reward top performers (performance group 1) and coach group 4.

## 11. Screenshots
Execution screenshots are stored in `screenshots/joins/` and `screenshots/window_functions/`.

## 12. References
- PostgreSQL Global Development Group. *PostgreSQL Documentation – Window Functions*. https://www.postgresql.org/docs/current/tutorial-window.html
- SQLite. *Window Functions*. https://www.sqlite.org/windowfunctions.html
- Oracle. *MySQL 8.0 Reference Manual – Window Functions*. https://dev.mysql.com/doc/refman/8.0/en/window-functions.html
- Course notes: SQL Database, ULK Data Science Programme (Maniraguha, E.).

## 13. Academic integrity statement
All sources were properly cited. The SQL implementations and analysis represent the group's original work. No AI-generated content was copied and presented as original work.
