# 🏝️ IslandLedger — Animal Crossing Economy Tracker

### A complete database project explained for absolute beginners

**Built for:** IS480 Advanced Database Management (CSULB, Spring 2026)

---

## What is this project?

This is a **full data pipeline** — the same architecture used by Netflix, Uber, and Airbnb — built around the economy of *Animal Crossing: New Horizons*.

If you've never touched a database, don't worry. This README explains everything from scratch. By the end, you'll understand how data flows through a real company's tech stack.

---

## Table of Contents

- [The Big Picture](#the-big-picture)
- [Why Animal Crossing?](#why-animal-crossing)
- [Phase 1: The Database & Business Logic](#phase-1-the-database--business-logic)
  - [What is a Schema?](#what-is-a-schema)
  - [What is PL/SQL?](#what-is-plsql)
  - [What is a Package?](#what-is-a-package)
  - [Our Tables Explained](#our-tables-explained)
  - [Our Procedures Explained](#our-procedures-explained)
  - [Our Functions Explained](#our-functions-explained)
  - [Key Concepts Demonstrated](#key-concepts-demonstrated)
- [Phase 2: The Data Warehouse](#phase-2-the-data-warehouse)
  - [Why Can't We Just Use Phase 1 Tables?](#why-cant-we-just-use-phase-1-tables)
  - [What is a Star Schema?](#what-is-a-star-schema)
  - [What is ETL?](#what-is-etl)
  - [Our Analytical Queries Explained](#our-analytical-queries-explained)
- [Phase 3: Big Data with Spark](#phase-3-big-data-with-spark)
  - [The One-Sentence Explanation](#the-one-sentence-explanation)
  - [What is Spark?](#what-is-spark)
  - [What is Databricks?](#what-is-databricks)
  - [What is Hive?](#what-is-hive)
  - [What is PySpark?](#what-is-pyspark)
  - [Why is the SQL Almost Identical?](#why-is-the-sql-almost-identical)
  - [The Scale Discussion](#the-scale-discussion)
- [How to Run This Project](#how-to-run-this-project)
- [Project Structure](#project-structure)

---

## The Big Picture

Every tech company has the same three-layer data system. Here's what each layer does:

```
Layer 1: OLTP (Online Transaction Processing)
├── Handles day-to-day operations
├── "Someone just bought something — record it"
├── "Someone just signed up — save their info"
├── Optimized for fast reads/writes of individual rows
└── This is Phase 1

Layer 2: OLAP (Online Analytical Processing)
├── Reorganizes data for answering big-picture questions
├── "What was our best-selling product last month?"
├── "Who are our top 10 customers?"
├── Optimized for aggregating millions of rows
└── This is Phase 2

Layer 3: Distributed Computing
├── When data gets too big for one computer
├── "Analyze 1 billion transactions across 50 machines"
├── Same SQL, different engine
└── This is Phase 3
```

Our project builds all three layers. The data flows from Phase 1 → Phase 2 → Phase 3, just like it does at Netflix or Amazon.

---

## Why Animal Crossing?

This might seem like a random choice, but Animal Crossing's island economy is actually a perfect miniature version of a real economy:

| Animal Crossing | Real World |
|---|---|
| Villagers | Customers / Users |
| Bells | Currency (Dollars) |
| Nook's Cranny, Able Sisters | Retail stores |
| Buying furniture, clothing | E-commerce transactions |
| Tom Nook's home loans | Mortgages / Debt |
| Savings goals (island projects) | Savings accounts |
| Turnip stalk market | Stock market / Speculative trading |
| Budget categories with limits | Personal finance apps (Mint, YNAB) |

We're tracking the same things any fintech app would track — just with cute animals instead of real money.

---

## Phase 1: The Database & Business Logic

> **Worth:** 40 / 100 points

### What is a Schema?

A schema is just a collection of tables. A table is like a spreadsheet — it has rows (records) and columns (fields). When we say "design a schema," we mean "decide what spreadsheets we need and what columns each one should have."

### What is PL/SQL?

PL/SQL stands for "Procedural Language / SQL." Regular SQL can only ask questions (`SELECT * FROM users`) or make changes (`INSERT INTO users ...`). PL/SQL lets you write actual programs — with variables, if/else logic, loops, and error handling — that live inside the database.

Think of it this way: SQL is like asking a librarian a question. PL/SQL is like giving the librarian a detailed procedure to follow: "Look up this book. If it exists, check if it's available. If it is, check it out to this person. If not, tell them it's unavailable."

### What is a Package?

A package is Oracle's way of bundling related code together. It has two parts:

- **Specification (the menu):** Lists what's available — "we have these 5 procedures and these 5 functions." Doesn't show any code.
- **Body (the kitchen):** Contains the actual code that does the work.

Why separate them? Because other code only needs to know *what* your package can do (the spec), not *how* it does it (the body). This is called **encapsulation** — the same concept as classes in Java or Python.

### Our Tables Explained

We designed 6 tables. Here's each one and why it exists:

**`ac_villagers`** — The people on our island.

```sql
-- Each row is one villager
-- Real-world equivalent: a "users" or "customers" table
villager_id    -- unique ID (like a customer number)
villager_name  -- their name (Marshal, Isabelle, etc.)
species        -- squirrel, dog, cat, etc.
personality    -- Smug, Normal, Snooty, etc. (game mechanic)
birthday       -- date of birth
move_in_date   -- when they joined the island
is_resident    -- Y/N — are they still here?
```

**`ac_pouches`** — Wallets / bank accounts.

Each villager can have multiple pouches — a main checking account, a savings jar, a debt account. This mirrors how you might have a checking account, savings account, and credit card in real life.

```sql
pouch_id       -- unique ID for this wallet
villager_id    -- who owns it (foreign key → ac_villagers)
pouch_name     -- "Nook ABD Main", "Rainy Day Jar", etc.
pouch_type     -- bells, savings, debt, turnips, nook_miles
balance        -- current balance (negative for debt)
is_active      -- Y/N
```

**`ac_activity_types`** — Budget categories.

Like the categories in a budgeting app: Groceries, Dining Out, Clothing, etc. Each category has a monthly spending limit.

```sql
activity_id       -- unique ID
villager_id       -- who this category belongs to
activity_name     -- "Groceries", "Clothing", "Mortgage", etc.
monthly_bell_limit -- budget cap for this category
is_income         -- Y = income source, N = expense category
```

**`ac_trades`** — Every transaction.

Every time someone buys or sells anything, a row goes here. This is the heart of the database.

```sql
trade_id    -- unique transaction ID (auto-generated)
pouch_id    -- which wallet was used (foreign key → ac_pouches)
activity_id -- which category (foreign key → ac_activity_types)
trade_date  -- when it happened
shop        -- where ("Nook's Cranny", "Able Sisters", etc.)
item_name   -- what was bought/sold
bells       -- how much (positive = spent, negative = earned)
notes       -- optional memo
```

**`ac_island_projects`** — Savings goals.

Like saving up for a vacation or an emergency fund.

```sql
project_id    -- unique ID
villager_id   -- whose goal
project_name  -- "Emergency Bell Fund", "Trip to Mystery Isle"
project_type  -- infrastructure, event, upgrade, personal
bell_target   -- how much we need
bells_saved   -- how much we have so far
deadline      -- target date
is_complete   -- Y/N (auto-set when target reached)
```

**`ac_turnip_ledger`** — Stalk market investments.

Turnips in Animal Crossing work like stocks — you buy on Sunday, prices fluctuate during the week, and you try to sell for a profit. If you hold too long, they spoil (expire worthless).

```sql
entry_id     -- unique ID
villager_id  -- who bought the turnips
week_start   -- the Sunday they were purchased
buy_price    -- price per turnip from Daisy Mae
qty_bought   -- how many turnips
sell_price   -- NULL until sold
sold_on      -- NULL until sold
spoiled      -- Y/N (did they rot?)
```

### Our Procedures Explained

Procedures **do things** — they perform operations on the database.

**`add_villager`** — Registers a new resident on the island.

*Why not just use a plain INSERT?* Because we need to check if that villager ID already exists first. Without this check, you'd get a cryptic Oracle error. Our procedure gives a friendly, specific error message instead.

```
Thought process:
1. Check if villager_id already exists → COUNT(*) check
2. If it exists → throw custom error with RAISE_APPLICATION_ERROR
3. If not → INSERT the new villager
4. Print confirmation with DBMS_OUTPUT
```

**`log_trade`** — Records a purchase or sale and updates the wallet balance.

This is what happens behind the scenes when you swipe a debit card. Two things have to happen together: the transaction gets recorded AND your balance gets updated. If one fails, both should fail (that's why we use COMMIT/ROLLBACK).

```
Thought process:
1. Reject zero-amount trades (pointless transaction)
2. Verify the wallet (pouch) exists and is active → COUNT(*) check
3. Verify the spending category exists → COUNT(*) check
4. Generate a new trade ID from the sequence
5. INSERT the trade
6. UPDATE the wallet balance
7. COMMIT both together (or ROLLBACK both on error)
```

**`contribute_to_project`** — Adds bells toward a savings goal.

```
Thought process:
1. Reject negative amounts
2. Look up the project → if not found, NO_DATA_FOUND exception
3. Check if already complete → throw error if yes
4. UPDATE the saved amount
5. Auto-mark complete if target reached (CASE in UPDATE)
6. Print progress message
```

**`transfer_funds`** — Moves bells between two wallets owned by the same villager.

Think of this as transferring money from checking to savings. You need to validate both accounts exist, verify the same person owns both (you can't transfer from someone else's account), and check you have enough money.

```
Thought process:
1. Reject negative amounts
2. Look up source wallet → NO_DATA_FOUND if invalid
3. Look up destination wallet → NO_DATA_FOUND if invalid
4. Verify same villager owns both → throw error if different
5. Check sufficient balance → throw error if not enough
6. UPDATE source (subtract) and destination (add)
7. COMMIT together
```

**`monthly_budget_report`** — Prints a formatted income vs. expense report.

This is the most complex procedure. It's what you see when you open your credit card statement or look at your spending dashboard in Mint.

```
Thought process:
1. Verify villager exists → COUNT(*) check
2. Print header
3. FOR loop: calculate total income from all income trades
4. Explicit cursor (OPEN/FETCH/EXIT WHEN/CLOSE):
   loop through each expense category
5. For each category: compare spending to budget limit
   - Under 50% of limit → "OK"
   - Over 90% of limit → "NEAR LIMIT"
   - Over limit → "OVER BUDGET"
6. Print totals and net (income - expenses)
```

### Our Functions Explained

Functions **calculate and return a value** — you call them and get an answer back.

**`get_balance`** — Returns the current balance of a wallet. Simple, but demonstrates NO_DATA_FOUND exception handling.

**`get_monthly_spending`** — Returns total bells spent in one category for a given month. Uses COUNT(*) to verify the category belongs to the right villager.

**`get_turnip_profit`** — The computational function the professor specifically asked for. Calculates `(sell_price - buy_price) × quantity`. Handles edge cases: what if the turnips spoiled? What if they haven't been sold yet?

**`budget_health_score`** — Grades the villager A through F based on how many budget categories are under their spending limit. Uses a cursor FOR loop and a CASE expression for the letter grade. This directly mirrors the `Grading()` function from the class labs.

**`get_net_worth`** — Sums up all wallet balances (including debt) for a villager. Marshal has a positive checking balance but a huge negative mortgage balance, so his net worth is negative — just like real life.

### Key Concepts Demonstrated

The professor grades Phase 1 on a checklist. Here's every concept and where we used it:

| Concept | What it is | Where we used it |
|---|---|---|
| **Procedures** | Code that performs an action | add_villager, log_trade, contribute_to_project, transfer_funds, monthly_budget_report |
| **Functions** | Code that returns a value | get_balance, get_monthly_spending, get_turnip_profit, budget_health_score, get_net_worth |
| **Explicit Cursors** | Manually stepping through query results one row at a time (OPEN/FETCH/EXIT WHEN/CLOSE) | monthly_budget_report uses this for the expense category loop |
| **FOR Loops** | Automatically looping through query results | monthly_budget_report (income), budget_health_score, get_net_worth |
| **Exception Handling** | Catching and handling errors gracefully | NO_DATA_FOUND in transfer_funds, contribute_to_project, get_balance, get_turnip_profit |
| **RAISE_APPLICATION_ERROR** | Throwing custom error messages with specific error codes | Custom errors -20001 through -20030 throughout |
| **%TYPE** | Declaring a variable with the same data type as a table column (so if the column changes, your code doesn't break) | Every parameter and local variable |
| **COUNT(*)** | Checking if a row exists before doing something | add_villager, log_trade, monthly_budget_report, get_monthly_spending, budget_health_score, get_net_worth |
| **IF/ELSIF/ELSE** | Conditional branching | Budget status labels, project progress messages, validation checks |
| **CASE** | Multi-way branching (like a switch statement) | budget_health_score letter grading, contribute_to_project completion |
| **DBMS_OUTPUT** | Printing diagnostic messages (like console.log) | Every procedure and function |
| **WHEN OTHERS** | Catch-all exception handler with ROLLBACK | Every procedure to prevent partial data corruption |

---

## Phase 2: The Data Warehouse

> **Worth:** 30 / 100 points

### Why Can't We Just Use Phase 1 Tables?

Phase 1 tables are **normalized** — data is spread across multiple tables to avoid redundancy. For example, to find out "how much did Marshal spend on groceries in February," you need to JOIN four tables together:

```sql
-- This is complicated and slow on large datasets:
SELECT SUM(t.bells)
FROM ac_trades t
JOIN ac_pouches p ON t.pouch_id = p.pouch_id
JOIN ac_activity_types a ON t.activity_id = a.activity_id
JOIN ac_villagers v ON p.villager_id = v.villager_id
WHERE v.villager_name = 'Marshal'
  AND a.activity_name = 'Groceries'
  AND t.trade_date BETWEEN ...
```

Normalization is great for transactions (no duplicate data, easy updates) but terrible for analytics (too many JOINs, slow on large datasets).

### What is a Star Schema?

A star schema solves this by reorganizing the data into one **fact table** surrounded by **dimension tables**:

```
                    dim_date
                       │
dim_villager ──── fact_trades ──── dim_activity
                       │
                    dim_pouch
```

**Fact table** = the measurements/events (every trade, with the bell amount)

**Dimension tables** = the context (who, what, when, where)

It's called a "star" because the diagram looks like a star — fact table in the center, dimensions radiating outward.

Now that same query becomes simpler:

```sql
-- Star schema version — cleaner and faster:
SELECT SUM(f.bells_amount)
FROM fact_trades f
JOIN dim_villager v ON f.villager_key = v.villager_key
JOIN dim_activity a ON f.activity_key = a.activity_key
WHERE v.villager_name = 'Marshal'
  AND a.activity_name = 'Groceries'
```

### What is ETL?

ETL stands for **Extract, Transform, Load**:

1. **Extract** — Pull data from the Phase 1 tables
2. **Transform** — Clean it up, derive new fields (like converting month number to season name), map keys
3. **Load** — Insert it into the star schema tables

We wrote 5 PL/SQL procedures for this — one per table, plus a wrapper that runs them all in order.

The most interesting transformation is deriving the **season** from the month number:

```sql
CASE
    WHEN month_num IN (3, 4, 5)  THEN 'Spring'
    WHEN month_num IN (6, 7, 8)  THEN 'Summer'
    WHEN month_num IN (9, 10, 11) THEN 'Fall'
    ELSE 'Winter'
END
```

This doesn't exist in the original data — we created it during the Transform step so analysts can ask questions like "do villagers spend more in Winter or Summer?"

### Our Analytical Queries Explained

We wrote 6 queries demonstrating different analytical SQL techniques:

**Query 1 — GROUP BY:** "How much did each villager spend total?" The most basic aggregation. GROUP BY collects all rows for each villager and SUMs their spending.

**Query 2 — ROLLUP:** "Show me monthly spending with subtotals and a grand total." ROLLUP is like GROUP BY but it automatically adds summary rows. Instead of just seeing "February: 99,600 / March: 4,800," you also get "2026 total: 104,400" and a grand total row.

**Query 3 — RANK():** "Rank villagers from biggest to smallest spender." RANK is a window function — it assigns a ranking number to each row based on a sort order. Marshal gets rank 1 (75,200 bells), Isabelle gets rank 2 (29,200 bells).

**Query 4 — DENSE_RANK() with PARTITION BY:** "For each villager, rank their spending categories." PARTITION BY means "do the ranking separately within each group." So Marshal's categories get ranked 1-5, and Isabelle's get ranked 1-2, independently.

**Query 5 — Subquery:** "Show me transactions that were above the average amount." The inner query calculates the average, and the outer query finds all transactions above that average. Only the Mortgage Payment (60,000) and Cottage Rent (20,000) qualify.

**Query 6 — CUBE + CASE:** "Show me spending by every combination of villager and season, with spending tiers." CUBE is like ROLLUP but generates *all possible combinations*. CASE classifies each combination as "Heavy Spender," "Moderate," or "Light."

---

## Phase 3: Big Data with Spark

> **Worth:** 30 / 100 points

### The One-Sentence Explanation

**"What happens when your data gets too big for one computer?"**

That's the entire point of Phase 3. Everything else is just demonstrating the answer.

### What is Spark?

Think of moving furniture. In Phase 1 and 2, you have **one person** (Oracle) carrying every piece one at a time. Works fine for 11 pieces. But what about a billion?

**Spark is like hiring 50 people** and saying "everyone grab some furniture and carry it at the same time." The instructions are the same. The work gets split up.

Technically: Apache Spark is a distributed computing engine that splits data across many machines (a "cluster") and processes them in parallel. A query that takes 60 minutes on one machine might take 1-2 minutes on 50 machines.

### What is Databricks?

Databricks is just **a website that runs Spark**. Instead of installing Spark on your own computer (which is a nightmare), Databricks gives you a free cloud environment with a notebook interface. You write code, click Run, and it executes on a Spark cluster behind the scenes.

Think of it like Google Docs vs. Microsoft Word. Word runs on your computer. Google Docs runs in the cloud. Same concept — Databricks is "Spark in the cloud."

### What is Hive?

Hive is a layer that lets you **run SQL queries on distributed data**. When we called `saveAsTable()` in our notebook, we registered our DataFrames as Hive tables. After that, we could query them with regular SQL instead of only Python.

Without Hive: You'd need to write Python code for every query.
With Hive: You can write the same SQL you already know.

### What is PySpark?

PySpark is **Python talking to Spark**. Instead of writing SQL:

```sql
SELECT villager_name, SUM(bells_amount) AS total
FROM fact_trades f
JOIN dim_villager v ON f.villager_key = v.villager_key
WHERE is_income_flag = 0
GROUP BY villager_name
ORDER BY total DESC
```

You write Python:

```python
(fact_trades_df
    .filter(F.col("is_income_flag") == 0)
    .join(dim_villager_df, "villager_key")
    .groupBy("villager_name")
    .agg(F.sum("bells_amount").alias("total"))
    .orderBy(F.desc("total"))
    .show()
)
```

Same result. Different syntax. The reason PySpark matters is that **you can't feed SQL output into a machine learning model**, but you can feed a PySpark DataFrame directly into TensorFlow or scikit-learn. That's why data scientists prefer it.

### Why is the SQL Almost Identical?

This is the key insight of Phase 3. Look at these side by side:

**Oracle:**
```sql
SELECT v.villager_name, SUM(f.bells_amount) AS total_spent
FROM fact_trades f
JOIN dim_villager v ON f.villager_key = v.villager_key
WHERE f.is_income_flag = 0
GROUP BY v.villager_name
ORDER BY total_spent DESC
```

**SparkSQL:**
```sql
SELECT v.villager_name, SUM(f.bells_amount) AS total_spent
FROM fact_trades f
JOIN dim_villager v ON f.villager_key = v.villager_key
WHERE f.is_income_flag = 0
GROUP BY v.villager_name
ORDER BY total_spent DESC
```

They're **identical**. The professor designed the project this way intentionally. The syntax doesn't change — the **engine** changes. Oracle runs that query on one CPU. Spark splits the data across 50 machines, each one computes a partial SUM, and then they merge the results.

### The Scale Discussion

| | Oracle (1 server) | Spark (50-node cluster) |
|---|---|---|
| **Our data (11 rows)** | Instant | Instant (actually slower due to cluster overhead) |
| **1 million rows** | A few seconds | About the same |
| **1 billion rows** | 30-60+ minutes, might crash | 1-2 minutes |
| **Need more power?** | Buy a bigger server (vertical scaling, $$$) | Add more machines (horizontal scaling, cheaper) |

**Why not just use Spark for everything?** Spark has overhead for coordinating the cluster. For small data and transactional work (insert one row, update one row), Oracle is faster. The sweet spot for Spark is analytical queries on large datasets.

**Real companies use both:** Oracle for the transactional layer (Phase 1), ETL into a warehouse (Phase 2), then Spark for large-scale analytics (Phase 3). Netflix, Uber, and Airbnb all follow this pattern — which is exactly the pipeline we built.

---

## How to Run This Project

### Prerequisites

- Oracle SQL Developer (or VS Code with Oracle extension)
- Access to an Oracle database (Oracle RDS, Oracle XE, or LiveSQL)
- Databricks Community Edition account (free, no credit card)

### Phase 1

```bash
# Run in SQL Developer / VS Code against your Oracle connection
# Execute in this order:
1. phase1/schema.sql        # Creates tables + seed data
2. phase1/package_spec.sql   # Compiles package specification
3. phase1/package_body.sql   # Compiles package body
4. phase1/test_script.sql    # Runs all 27 tests
```

Make sure `SET SERVEROUTPUT ON` is set before running the test script.

### Phase 2

```bash
# Run in SQL Developer / VS Code (Phase 1 tables must exist first)
1. phase2/warehouse_schema.sql   # Creates star schema tables
2. phase2/etl_procedures.sql     # Compiles ETL procedures
3. Execute: BEGIN load_all_warehouse; END; /
4. phase2/analytical_queries.sql # Runs 6 analytical queries
```

### Phase 3

```bash
# In Databricks Community Edition:
1. Import phase3/notebook.html (or the .py source)
2. Attach to a cluster
3. Run All
# Data is embedded in the notebook — no CSV upload required
```

---

## Project Structure

```
islandledger/
├── README.md                          ← You are here
├── phase1/
│   ├── schema.sql                     ← Tables + seed data
│   ├── package_spec.sql               ← Package specification (the menu)
│   ├── package_body.sql               ← Package body (the code)
│   ├── test_script.sql                ← 27 tests proving it all works
│   └── phase1_report.md               ← Writeup + test output
├── phase2/
│   ├── warehouse_schema.sql           ← Star schema (1 fact + 4 dims)
│   ├── etl_procedures.sql             ← ETL: OLTP → OLAP
│   ├── analytical_queries.sql         ← 6 queries: ROLLUP, RANK, CUBE
│   └── phase2_report.md               ← Star schema diagram + explanations
└── phase3/
    ├── notebook.html                  ← Databricks notebook (exported)
    ├── data/                          ← CSV exports from star schema
    │   ├── dim_date.csv
    │   ├── dim_villager.csv
    │   ├── dim_activity.csv
    │   ├── dim_pouch.csv
    │   └── fact_trades.csv
    └── phase3_report.md               ← Comparison + scale discussion
```

---

## What I Learned

1. **SQL is SQL.** The syntax barely changes between Oracle, SparkSQL, and HiveQL. Learn SQL once, use it everywhere.
2. **The engine matters more than the syntax.** The same query runs on one machine (Oracle) or thousands (Spark). The difference shows up at scale.
3. **Normalization vs. denormalization is a tradeoff.** Normalized tables (Phase 1) prevent data duplication but make analytics slow. Denormalized star schemas (Phase 2) duplicate data but make analytics fast. Real systems use both.
4. **Error handling isn't optional.** Every procedure needs to handle bad input gracefully. Users will always find ways to break your code.
5. **The full pipeline is what matters.** OLTP → ETL → OLAP → Spark is how the world's most valuable companies handle data. This project is a miniature version of that same architecture.

---

*Built with Oracle PL/SQL, Apache Spark, Apache Hive, and Databricks.*
