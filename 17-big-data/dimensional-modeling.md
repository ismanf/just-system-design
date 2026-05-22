# Data Modeling for Analytics (Star, Snowflake Schemas)

> **TL;DR** — **Dimensional modeling** is the technique of organizing analytical data into **fact tables** (events, transactions, measurements) joined to **dimension tables** (the descriptive context — customer, product, store, time). The two canonical shapes are the **star schema** (a central fact surrounded by denormalized dimensions) and the **snowflake schema** (the same idea with dimensions further normalized into sub-dimensions). Designed by **Ralph Kimball** in the 1990s, dimensional modeling beat fully normalized OLTP designs for analytical workloads because it minimizes joins on the hot path, optimizes for human-comprehensible BI queries, and handles *history* via slowly-changing dimensions. Modern columnar warehouses (Snowflake, BigQuery, Databricks SQL) make the modeling matter less than it used to — wide denormalized tables ("one big table") are often fine — but **dimensional modeling remains the clearest way to reason about analytics**. The honest take: **use star schemas as the default mental model; reach for wide tables when query patterns are well-known; avoid snowflakes unless you have a specific reason**.

---

## 1. The big picture

```
                       ┌──────────────────┐
                       │   dim_customer   │
                       └────────┬─────────┘
                                │
                                │
┌──────────────┐          ┌─────┴──────┐         ┌──────────────┐
│   dim_date   │──────────│ fact_orders│─────────│   dim_store  │
└──────────────┘          └─────┬──────┘         └──────────────┘
                                │
                                │
                       ┌────────┴─────────┐
                       │    dim_product   │
                       └──────────────────┘
```

A **star schema**: one fact at the center, dimensions hanging off. Queries look like:

```sql
SELECT
  d.year,
  c.country,
  p.category,
  SUM(f.amount) AS revenue
FROM fact_orders f
JOIN dim_date     d ON f.date_key     = d.date_key
JOIN dim_customer c ON f.customer_key = c.customer_key
JOIN dim_product  p ON f.product_key  = p.product_key
GROUP BY 1, 2, 3
```

Every analytical question is the same shape: filter / group by dimension attributes, aggregate fact measures. BI tools (Looker, Tableau, Power BI, Hex) were designed around this shape. So were warehouses.

---

## 2. Why dimensional modeling exists

OLTP databases are normalized to **3rd Normal Form** for good reason: each fact stored exactly once, mutations small, write integrity guaranteed. But 3NF is *terrible* for analytics:

- Reporting queries join 8–15 tables.
- Each join blows up the query plan.
- Even simple "revenue by country" requires understanding the entire schema.
- History is implicit — a customer changes address; the old address is gone.
- BI tools choke on the complexity.

Dimensional modeling traded write efficiency for read efficiency. The trade-off was tolerable in the 90s warehouse era (loads happened overnight; no live writes) and is overwhelmingly favorable now (columnar storage, separated compute, ELT pipelines that rebuild downstream).

The Kimball approach asks one organizing question: **what business processes do we measure, and what is the grain (atomic unit) of each measurement?** A row in `fact_orders` is one order. A row in `fact_page_views` is one view. A row in `fact_inventory_snapshot` is one product-at-one-time. **Grain first**; everything else follows.

---

## 3. Fact tables — the measurements

A fact table records *events* or *states*. Each row is one occurrence, at the declared grain, with:

- **Foreign keys** to dimension tables (one per dimension that gives context).
- **Measures** — the numeric quantities you'll sum, average, count, or aggregate.
- **Optional degenerate dimensions** — IDs that don't deserve their own table (order_id, invoice_number).

### Three fact-table styles

| Style | What it stores | Example |
|---|---|---|
| **Transactional fact** | One row per event; immutable | `fact_orders`: one row per order placed |
| **Periodic snapshot** | One row per entity per time period | `fact_inventory_daily`: row per (product, store, day) |
| **Accumulating snapshot** | One row per business process instance, updated as it progresses | `fact_order_fulfillment`: order placed → packed → shipped → delivered, one row updated over time |

The transactional fact is by far the most common. Snapshots show up for inventory, balances, headcount, slowly-changing states. Accumulating snapshots are useful but more complex; many teams skip them and reconstruct as needed from event facts.

### Measures — the numbers you aggregate

- **Additive**: sum across any dimension. Revenue, units sold, page views.
- **Semi-additive**: sum across some dimensions, not all. Inventory levels sum across products but not across dates.
- **Non-additive**: can only be averaged or used with care. Ratios, percentages, prices.

Choosing additive measures wherever possible is what makes BI tools fast and correct. A ratio like "average margin" stored in a fact is dangerous — averaging averages is mathematically wrong.

### The fact-table grain rule

**Every fact table declares one grain in plain English at the top.** "One row per order line item, by date, by customer." Then every row obeys it. Every dimension joins at that grain. Every measure makes sense at that grain.

Violating grain is the #1 source of "the numbers don't tie out." A `fact_orders` table where some rows are one order and others are aggregated to one customer-day is unusable.

---

## 4. Dimension tables — the context

A dimension table holds descriptive attributes. Wide (lots of columns), short (relative to fact), and queried mostly via foreign-key joins from facts.

`dim_customer` typically looks like:

| customer_key | customer_id | first_name | last_name | email | country | city | tier | signup_date | ... |
|---|---|---|---|---|---|---|---|---|---|
| 1 | C-001 | Alice | Smith | alice@ex.com | US | Boston | enterprise | 2023-01-15 | ... |

Conventions:

- **Surrogate key** (`customer_key`) — integer, generated by the ETL/ELT process, used in joins. Stable even if natural keys change.
- **Natural key** (`customer_id`) — the source-system identifier. Kept for traceability.
- **Wide and denormalized** — lots of attributes in one table. Joining is rare; columnar warehouses prune unused columns at read time.
- **Conformed dimensions** — `dim_customer` is shared across `fact_orders`, `fact_support_tickets`, `fact_marketing_emails`. Same meaning, same keys, every place.
- **No measures** — dimensions describe; facts measure.

The denormalization is deliberate. A `dim_customer` row repeats "US" in the country field across millions of customers. So what — disk is cheap, joins are not (relatively), and humans understand wide flat tables.

### Conformed dimensions — the spine of the warehouse

A **conformed dimension** is one that means the same thing across every fact table that uses it. `dim_date` is the canonical example: every fact joins to it; every "yesterday" means the same yesterday.

Conformed dimensions are how you compare across business processes. "How does customer support volume correlate with order volume?" → both join to `dim_customer` and `dim_date` with the same keys. If they don't, the question can't be answered without painful reconciliation.

Building conformed dimensions is one of the most important investments a data team makes. It pays back forever.

---

## 5. Star vs snowflake — the two shapes

### Star schema

Dimensions are flat. Each row is fully self-contained.

```
fact_orders
   ├── dim_customer (country, city, tier, ...)
   ├── dim_product (category, subcategory, brand, ...)
   ├── dim_store (region, store_type, ...)
   └── dim_date (year, quarter, month, day_of_week, ...)
```

One join per dimension. Easy to query, easy for BI tools.

### Snowflake schema

Dimensions normalized into sub-dimensions.

```
fact_orders
   ├── dim_customer
   │     └── dim_country (country_name, region, continent)
   ├── dim_product
   │     ├── dim_subcategory
   │     │     └── dim_category
   │     └── dim_brand
   ├── dim_store
   │     └── dim_region
   └── dim_date
```

More joins; more tables; smaller dimension footprints. Useful when dimensions are huge and repeat a lot of data, or when sub-dimensions are reused across many parent dimensions.

### Which to use

**Default to star.** Snowflake adds joins on every query and obscures the model. Modern warehouses can hide a few extra joins, but the cognitive load doesn't go away.

Snowflake **only** when:
- A sub-dimension is genuinely shared by many parent dimensions and you want one source of truth (rare).
- A dimension is so big that denormalization is prohibitive (rare in modern warehouses).
- A regulatory or governance reason forces normalization.

If you find yourself snowflaking "for elegance," stop. Elegance for the modeler ≠ usability for the analyst.

---

## 6. Slowly Changing Dimensions (SCDs) — handling history

Customer Alice was in Boston; now she's in Seattle. Did she order from Boston or Seattle? Both, at different times. How does the warehouse remember?

Kimball's **Type 0–7 SCD typology** is the canonical answer. The two that matter most:

### Type 1 — overwrite

The dimension keeps only the current value. History is lost.

```sql
UPDATE dim_customer SET city = 'Seattle' WHERE customer_id = 'C-001';
```

Use when history of that attribute doesn't matter (e.g., a typo correction, name normalization).

### Type 2 — versioned rows

A new row is inserted; the old row is "expired." Each row has effective dates.

| customer_key | customer_id | city | valid_from | valid_to | is_current |
|---|---|---|---|---|---|
| 17 | C-001 | Boston | 2023-01-15 | 2026-02-09 | false |
| 4099 | C-001 | Seattle | 2026-02-10 | 9999-12-31 | true |

Facts reference whichever surrogate key was current when the event happened. Reports honor the "as-was" history.

Type 2 is the workhorse for "as-of-event-time" analytics. It is also the source of much complexity — every change creates a new row; every join must resolve to the right version.

### Type 3 — add a column

Keep one extra column for the previous value. Limited (only one previous value), simple. Useful when only the last change matters.

### Other types

- **Type 4** — split current vs history into two tables.
- **Type 6** — combination of Type 1, 2, and 3.

In practice, most modern warehouses use Type 1 for cosmetic changes and Type 2 for anything analytically meaningful. dbt has `snapshot` materializations that automate Type 2 SCDs from a source table.

---

## 7. Bridge tables and many-to-many

Sometimes a fact maps to a *set* of dimension values, not a single one. A patient has multiple diagnoses; a song has multiple genres; an order has multiple promotions applied.

The Kimball pattern: a **bridge table** between fact and dimension.

```
fact_orders ─── bridge_order_promotions ─── dim_promotion
                (order_id, promotion_id,
                 allocation_factor)
```

The bridge can carry weighting (e.g., split revenue across promotions). Without it, you double-count rows.

In practice, many teams cheat with arrays / JSON columns in modern warehouses (`promotions ARRAY<STRING>`) and use `UNNEST` queries. Works fine for read-only analytics; less BI-tool-friendly.

---

## 8. Fact-to-fact joins — the trap

Joining two fact tables directly is almost always wrong:

```sql
-- WRONG
SELECT o.customer_id, o.amount, t.ticket_count
FROM fact_orders o
JOIN fact_support_tickets t ON o.customer_id = t.customer_id;
```

This produces a Cartesian explosion: every order × every ticket for the same customer.

The right shape: aggregate one side first, then join.

```sql
WITH ticket_counts AS (
  SELECT customer_id, COUNT(*) AS ticket_count
  FROM fact_support_tickets
  GROUP BY 1
)
SELECT o.customer_id, SUM(o.amount) AS revenue, t.ticket_count
FROM fact_orders o
LEFT JOIN ticket_counts t ON o.customer_id = t.customer_id
GROUP BY o.customer_id, t.ticket_count;
```

Or, more idiomatically in dimensional modeling: each fact joins to dimensions; you combine measures across facts via a wider join through a shared dimension or via separate queries.

**Rule of thumb**: in a SQL query, you should never see two `fact_` tables joined directly.

---

## 9. One Big Table (OBT) — the modern alternative

Modern columnar warehouses (Snowflake, BigQuery, Databricks SQL) are so fast at scanning wide tables that some teams skip dimensional modeling entirely and build **one giant denormalized table per business process**.

```sql
-- One big fact_orders_obt
SELECT
  order_id,
  order_date,
  customer_id, customer_country, customer_tier,
  product_id, product_category, product_brand,
  store_id, store_region,
  amount, units
FROM fact_orders_obt
WHERE customer_country = 'US'
GROUP BY ...
```

Everything an analyst needs in one table. No joins. Easy for BI tools. Easy to grok.

Trade-offs:

- **Pro**: simplest mental model. Fastest queries (no joins). Easy denormalization patterns in dbt.
- **Con**: storage cost (every order row repeats customer attributes). History is harder (Type 2 SCDs require careful logic). Conforming dimensions is harder (you can't fix customer attributes in one place).
- **Con**: changes propagate weirdly. Renaming a country requires updating millions of rows. With star + conformed dimensions, you update one row.

**When OBT is right**: highly columnar warehouses, well-understood query patterns, small-to-mid scale. **When star is right**: many fact tables sharing dimensions, frequent changes to dimensional attributes, BI tools that expect star.

Many modern teams use both — star schemas as the "modeled" layer, OBT as the "consumption" layer materialized for performance.

---

## 10. Date / time dimensions

`dim_date` is the most important dimension you build. One row per day from a long history to a long future. Attributes:

| date_key | full_date | day | month | year | quarter | week_of_year | day_of_week | is_weekend | is_holiday | fiscal_year | ... |
|---|---|---|---|---|---|---|---|---|---|---|---|

Two reasons it's special:
- It's used by *every* fact. Conform it once, use everywhere.
- Calendar logic is messy (fiscal years, week numbering, holidays per country) — get it right once in a dimension and never write that logic in a query again.

For sub-day precision, add `dim_time` (one row per minute or hour, often combined with `dim_date` via two foreign keys: `date_key + time_key`).

---

## 11. The Kimball process — a checklist

The four-step design process from *The Data Warehouse Toolkit*:

1. **Select the business process** — what activity do we measure? (Orders, page views, support tickets, inventory movements.)
2. **Declare the grain** — what does one row in the fact represent? Be specific.
3. **Identify the dimensions** — what context describes each row? (Time, customer, product, ...)
4. **Identify the facts (measures)** — what numbers do we record? Make them additive when possible.

Do these four in order. Skip step 2 and you'll build something that doesn't tie out. Skip step 1 and you'll build a model that doesn't reflect any business reality.

---

## 12. Common Mistakes / Anti-Patterns

- **No declared grain.** Fact rows mix granularities. Reports give different totals depending on which dimension you slice by.
- **Mixing facts at different grains in one table.** Order-level rows and line-item-level rows interleaved. Use separate facts.
- **Non-additive measures stored in facts.** Ratios that get averaged again at query time → mathematically wrong.
- **Direct fact-to-fact joins.** Cartesian product blow-ups.
- **Snowflaking for elegance.** Adds joins, hides the model. Don't.
- **Junk dimensions for everything.** A "misc flags" dimension piled with unrelated attributes — hard to query, hard to maintain.
- **No surrogate keys.** Using natural keys in joins. When the source renumbers (or you re-source), everything breaks.
- **Conformed dimensions not actually conformed.** `dim_customer` v1 in `fact_orders` and v2 in `fact_tickets` — analyses won't combine.
- **Type 1 everywhere.** History is lost; "as-of" reports impossible.
- **Type 2 everywhere.** Every dimension query is harder, queries pay extra cost for history you don't need.
- **Bridge tables forgotten** for true many-to-many relationships. Double-counting.
- **OBT for everything without thinking about SCDs.** Customer attributes overwritten in millions of rows; history corrupted.
- **`dim_date` rebuilt by hand each year.** Pre-generate decades; it costs nothing.
- **Models that don't match how analysts ask questions.** "Sales by region by quarter" is awkward → the model is wrong.
- **No semantic layer.** Same metric defined differently across BI tools. dbt + a semantic layer (Cube, Looker LookML, MetricFlow) fixes this.
- **Letting BI calculations diverge from warehouse models.** "Marketing's revenue" ≠ "Finance's revenue." Put the metric definition in dbt, not in Tableau formulas.

---

## 13. Cheat Card

```
PURPOSE   Organize analytical data so business questions are easy
          to ask and fast to answer.

CORE OBJECTS
  Fact table       events / measurements; FK to dimensions; numeric measures
  Dimension table  descriptive context (customer, product, date, ...)
  Surrogate key    integer, ETL-managed, used in joins
  Conformed dim    same meaning everywhere it's used

SHAPES
  Star      flat dimensions; one join per dim  (default)
  Snowflake dims normalized into sub-dims     (rare)
  OBT       one big denormalized table         (modern fast warehouses)

FACT TYPES
  Transactional   one row per event
  Periodic snap   one row per entity per period
  Accumulating    row per process instance, updated as it progresses

MEASURES
  Additive        sum across any dimension (best)
  Semi-additive   sum across some dimensions only
  Non-additive    ratios, percentages — store components instead

SLOWLY-CHANGING DIMENSIONS (SCD)
  Type 1   overwrite — no history
  Type 2   versioned rows with valid_from/valid_to — history kept
  Type 3   add an "old value" column — one previous value
  dbt snapshots automate Type 2

KIMBALL 4-STEP DESIGN
  1. Pick the business process
  2. Declare the grain (one sentence)
  3. Identify the dimensions
  4. Identify the facts (measures)

WHEN STAR vs OBT
  Star    many shared dimensions, evolving attributes, BI tools want it
  OBT     well-known queries, mid-scale, fastest read path
  Both    star = source of truth, OBT = consumption layer

PITFALLS
  No declared grain
  Direct fact-to-fact joins
  Non-additive measures stored in facts
  Snowflaking by reflex
  Type 1 everywhere → no history
  Conformed dims that aren't actually conformed
  Metric definitions in BI tools, not in dbt

RULE   Declare the grain. Conform the dimensions. Make measures
       additive. Star schema first; OBT when reads demand it.
       Put metric definitions where pipelines and BI both read them.
```

---

## 14. Resources

### Books
- *The Data Warehouse Toolkit* (3rd ed.) — Ralph Kimball & Margy Ross. **The** book. Read it.
- *The Data Warehouse Lifecycle Toolkit* — Kimball, Ross, Thornthwaite, Becker, Mundy.
- *Star Schema: The Complete Reference* — Christopher Adamson.
- *Agile Data Warehouse Design* — Lawrence Corr & Jim Stagnitto.

### Documentation
- **dbt docs on modeling** — <https://docs.getdbt.com/docs/build/models>
- **dbt snapshots** (Type 2 SCDs) — <https://docs.getdbt.com/docs/build/snapshots>
- **Snowflake / BigQuery / Databricks** — best-practices guides on modeling.

### Articles
- "Functional data engineering" — Maxime Beauchemin.
- "One big table" debates — many takes; Tristan Handy (dbt) writes well on this.
- "Modeling for analytics in the modern data stack" — Locally Optimistic.
- Kimball Group archive — <https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/>

### Videos
- *Kimball: Data Warehousing in the Modern Era* — older talks, still relevant.
- *Coalesce* (dbt) sessions on modeling.
- ByteByteGo — "Star Schema vs Snowflake."

### Tools
- **dbt** — model and document dimensional data.
- **MetricFlow / Cube / LookML** — semantic layers on top of warehouses.
- **AtScale / Apache Druid / ClickHouse** — for OLAP cube-like layers.
- **DataHub / Amundsen** — data catalogs to track conformed dimensions.

### Adjacent reading
- [MapReduce →](./mapreduce.md)
- [Hadoop Ecosystem →](./hadoop.md)
- [Apache Spark →](./spark.md)
- [Apache Flink →](./flink.md)
- [ETL vs ELT →](./etl-vs-elt.md)
- [Data Pipelines & Orchestration →](./data-pipelines.md)
- [OLTP vs OLAP →](../04-databases/oltp-vs-olap.md)
- [Data Warehouses & Data Lakes →](../04-databases/warehouses-lakes.md)
- [Lakehouse Architecture →](../04-databases/lakehouse.md)
- [Database Normalization & Denormalization →](../04-databases/normalization.md)
- [Database Indexing →](../04-databases/indexing.md)

---

*Previous:* [← Data Pipelines & Orchestration](./data-pipelines.md)  |  *Up:* [README ↑](../README.md)
