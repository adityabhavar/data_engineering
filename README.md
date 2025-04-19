
# Amazon Data Engineer SQL Preparation Guide

This document consolidates a comprehensive SQL question bank (10 problems + extra examples) and a detailed coaching guide for your Amazon Data Engineer phone screen prep.

---

## Part 1: SQL Question Bank (10 Core Problems + Extra Examples)

A comprehensive collection of SQL interview questions inspired by Amazon’s phone screens and real‑world scenarios. Each entry includes:
- **Scenario & Question**
- **Solution SQL**
- **Step‑by‑Step Explanation**
- **Edge Cases & Handling**
- **Performance Considerations**

---

### 1. Average Monthly Review Rating
**Scenario**: Table `Reviews(review_id, product_id, user_id, rating, review_date)`.

**Question**: Compute the average rating for each product for every month.

```sql
SELECT
  product_id,
  EXTRACT(YEAR FROM review_date) AS year,
  EXTRACT(MONTH FROM review_date) AS month,
  ROUND(AVG(rating), 2) AS avg_rating
FROM Reviews
GROUP BY product_id,
         EXTRACT(YEAR FROM review_date),
         EXTRACT(MONTH FROM review_date)
ORDER BY product_id, year, month;
```

**Explanation**:
1. **Extract Year & Month**: Use `EXTRACT` to break `review_date` into year and month for grouping.
2. **Aggregate**: `AVG(rating)` calculates the mean; `ROUND` formats to two decimals.
3. **Grouping**: Ensures monthly buckets per product.
4. **Ordering**: Sorts results by product and date.

**Edge Cases**:
- **No Reviews**: Omitted; use calendar table join to include zero counts.
- **Null Ratings**: Exclude via `WHERE rating IS NOT NULL`.

**Performance**:
- Index on `(product_id, review_date)` for grouping.
- Avoid functions in `WHERE` for partition pruning.

---

### 2. Top 2 Highest‑Grossing Products per Category
**Scenario**: Table `product_spend(category, product, user_id, spend, transaction_date)`.

**Question**: Identify the top two products by total spend in each category for 2022.

```sql
WITH totals AS (
  SELECT
    category,
    product,
    SUM(spend) AS total_spend
  FROM product_spend
  WHERE transaction_date BETWEEN '2022-01-01' AND '2022-12-31'
  GROUP BY category, product
)
SELECT
  category,
  product,
  total_spend
FROM (
  SELECT
    category,
    product,
    total_spend,
    ROW_NUMBER() OVER (PARTITION BY category ORDER BY total_spend DESC) AS rn
  FROM totals
) t
WHERE rn <= 2
ORDER BY category, total_spend DESC;
```

**Edge Cases & Performance**: See original Q2 in question bank.

---

### 3. Cumulative Shipments by Day
**Scenario**: Table `Shipments(shipment_id, order_id, ship_date)`.

**Question**: Calculate total and running cumulative shipments per day.

```sql
SELECT
  ship_date,
  COUNT(*) AS daily_shipments,
  SUM(COUNT(*)) OVER (
    ORDER BY ship_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS cumulative_shipments
FROM Shipments
GROUP BY ship_date
ORDER BY ship_date;
```

---

### 4. Second Highest Order Total per Customer
**Scenario**: Table `Orders(order_id, customer_id, total_amount)`.

```sql
SELECT
  customer_id,
  total_amount
FROM (
  SELECT
    customer_id,
    total_amount,
    DENSE_RANK() OVER (PARTITION BY customer_id ORDER BY total_amount DESC) AS rk
  FROM Orders
) t
WHERE rk = 2;
```

---

### 5. Prime Users with No Recent Orders
**Scenario**: `PrimeUsers(user_id, join_date)` + `Orders(order_id, user_id, order_date)`.

```sql
SELECT p.user_id
FROM PrimeUsers p
LEFT JOIN Orders o
  ON p.user_id = o.user_id
  AND o.order_date >= CURRENT_DATE - INTERVAL '6 MONTH'
WHERE o.user_id IS NULL;
```

---

### 6. Delayed Shipment Rate
**Scenario**: `Orders(order_id, order_date, ship_date)`, `Deliveries(order_id, delivery_date)`.

```sql
WITH shipped AS (
  SELECT order_id, ship_date FROM Orders WHERE ship_date IS NOT NULL
), delivered AS (
  SELECT order_id, delivery_date FROM Deliveries
)
SELECT
  s.ship_date,
  ROUND(
    SUM(CASE WHEN d.delivery_date > s.ship_date + INTERVAL '5 DAY' THEN 1 ELSE 0 END)::NUMERIC
    / COUNT(*) * 100, 2
  ) AS pct_delayed
FROM shipped s
JOIN delivered d ON s.order_id = d.order_id
GROUP BY s.ship_date
ORDER BY s.ship_date;
```

---

### 7. Monthly Active Prime Users
**Scenario**: `PrimeUsers(user_id, join_date, cancel_date)`.

```sql
WITH months AS (
  SELECT generate_series(
    date_trunc('month', MIN(join_date)),
    date_trunc('month', CURRENT_DATE),
    INTERVAL '1 MONTH'
  ) AS month_start
  FROM PrimeUsers
), active AS (
  SELECT
    m.month_start,
    COUNT(p.user_id) AS active_members
  FROM months m
  LEFT JOIN PrimeUsers p
    ON p.join_date <= m.month_start + INTERVAL '1 MONTH' - INTERVAL '1 DAY'
    AND (p.cancel_date IS NULL OR p.cancel_date >= m.month_start)
  GROUP BY m.month_start
)
SELECT month_start, active_members
FROM active
ORDER BY month_start;
```

---

### 8. Inventory Replenishment Recommendation
**Scenario**: `Inventory(item_id, warehouse_id, stock, reorder_threshold)`, `Sales(sale_id, item_id, warehouse_id, sale_date, quantity)`.

```sql
WITH avg_sales AS (
  SELECT
    item_id, warehouse_id,
    AVG(quantity) AS avg_daily_sales
  FROM Sales
  WHERE sale_date >= CURRENT_DATE - INTERVAL '30 DAY'
  GROUP BY item_id, warehouse_id
)
SELECT
  i.warehouse_id, i.item_id, i.stock, i.reorder_threshold,
  CEIL((i.reorder_threshold * 2 - i.stock) / NULLIF(a.avg_daily_sales,0)) AS reorder_qty
FROM Inventory i
LEFT JOIN avg_sales a ON i.item_id = a.item_id AND i.warehouse_id = a.warehouse_id
WHERE i.stock < i.reorder_threshold;
```

---

### 9. Hierarchical Category Traversal
**Scenario**: `Categories(cat_id, parent_id, name)`.

```sql
WITH RECURSIVE cat_path AS (
  SELECT cat_id, CAST(name AS TEXT) AS path
  FROM Categories
  WHERE parent_id IS NULL
  UNION ALL
  SELECT c.cat_id, cp.path || ' > ' || c.name
  FROM Categories c
  JOIN cat_path cp ON c.parent_id = cp.cat_id
)
SELECT cat_id, path FROM cat_path;
```

---

### 10. Top 3 Products by Daily Revenue (Live Walkthrough)
**Scenario**: `Sales(sale_id, product_id, revenue, sale_date)`.

```sql
WITH daily_totals AS (
  SELECT sale_date, product_id, SUM(revenue) AS total_rev
  FROM Sales
  WHERE sale_date BETWEEN CURRENT_DATE - INTERVAL '6 DAY' AND CURRENT_DATE
  GROUP BY sale_date, product_id
)
SELECT sale_date, product_id, total_rev
FROM (
  SELECT
    sale_date, product_id, total_rev,
    ROW_NUMBER() OVER (PARTITION BY sale_date ORDER BY total_rev DESC) AS rn
  FROM daily_totals
) t
WHERE rn <= 3
ORDER BY sale_date, total_rev DESC;
```

---

## Extra Practice Examples

### 11. Detect Duplicate Orders
**Scenario**: Table `Orders(order_id, customer_id, order_date, amount)`.

```sql
SELECT customer_id, order_date, COUNT(*) AS dup_count
FROM Orders
GROUP BY customer_id, order_date, amount
HAVING COUNT(*) > 1;
```

---

### 12. Customer Churn Rate
**Scenario**: `Users(user_id, signup_date, cancel_date)`.

```sql
WITH monthly AS (
  SELECT
    date_trunc('month', signup_date) AS month,
    COUNT(*) AS signups
  FROM Users GROUP BY 1
), churn AS (
  SELECT
    date_trunc('month', cancel_date) AS month,
    COUNT(*) AS cancellations
  FROM Users WHERE cancel_date IS NOT NULL GROUP BY 1
)
SELECT
  m.month,
  m.signups,
  COALESCE(c.cancellations,0) AS cancellations,
  ROUND(COALESCE(c.cancellations,0)::NUMERIC / m.signups * 100, 2) AS churn_rate_pct
FROM monthly m
LEFT JOIN churn c USING (month)
ORDER BY m.month;
```

---

## Part 2: Coaching Guide

*(Refer to the previous sections for detailed coaching content on study plans, answer structures, and Amazon-specific tips.)*

---


# Amazon Data Engineer Phone Screen Prep & Coaching Guide

A comprehensive preparation document to get you Amazon-ready for your Data Engineer SQL phone screen. Includes a structured study plan, detailed answer guides for common SQL patterns, and Amazon-specific tips and tricks.

---

## 1. Understanding the Phone Screen
- **Duration**: 45–60 minutes
- **Format**: 2 SQL problems (medium/hard) + 1 behavioral/SQL explanation
- **Focus**: Correctness, clarity, performance optimization, thought process
- **Evaluation**:
  - Communication & structure
  - Handling edge cases
  - Performance considerations (indexes, explain)
  - Alignment with Leadership Principles (e.g., Dive Deep, Customer Obsession)

---

## 2. Structured Preparation Plan (1 Week)

### Week at a Glance
| Day | Focus                              | Output                                             |
|-----|------------------------------------|----------------------------------------------------|
| 1   | SQL Fundamentals & CRUD + Joins    | 15 practice queries (JOIN variations)              |
| 2   | Aggregations & GROUP BY + HAVING   | 10 aggregation queries + edge case notes           |
| 3   | Window Functions & Ranking         | 8 window-query solutions + performance notes       |
| 4   | CTEs & Subqueries vs. Joins        | 6 complex queries refactored into CTEs             |
| 5   | Optimization & Explain Plans       | 5 queries with EXPLAIN ANALYZE insights            |
| 6   | Real-World Scenarios Practice      | 4 Amazon-inspired problems                        |
| 7   | Mock Interviews & Behavioral Prep  | 2 full mock screens + LP-driven answers           |

### Daily Breakdown
**Morning (2 hrs)**: Concept deep-dive & example walkthroughs.  
**Afternoon (2 hrs)**: Hands-on practice—write, test, and explain queries aloud.  
**Evening (1 hr)**: Reflect, review mistakes, re-solve tough problems.

---

## 3. Core SQL Answer Guide

### 3.1 Joins & CRUD Patterns
- **INNER vs LEFT vs RIGHT**: Always restate how many rows expected and why.  
- **Answer Tip**: Sketch Venn diagrams; explicitly handle NULLs on outer joins.

**Example Response Structure**:
1. Restate: “We need all orders with shipments, so INNER JOIN…”  
2. Code: Write `SELECT … FROM Orders o JOIN Shipments s ON …`  
3. Edge Case: “If an order has no shipment, it’s excluded; use LEFT JOIN if needed.”

### 3.2 Aggregations & GROUP BY
- **Pattern**: Aggregate → GROUP BY non-aggregates → HAVING for post-filter  
- **Tip**: Name aggregations clearly, e.g., `COUNT(*) AS total_orders`.

**Sample Q**: Monthly active users.  
**Answer Steps**:
1. Filter date range in WHERE.  
2. Use `EXTRACT(YEAR… ), EXTRACT(MONTH…)` for grouping.  
3. `ROUND(AVG(… ),2)` for readability.

### 3.3 Window Functions
- **ROW_NUMBER vs RANK vs DENSE_RANK**: Clarify tie behavior.  
- **Frame Clauses**: Explain default (`RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`).

**Answer Blueprint**:
1. “Partition by X because we group within each….”  
2. “Order by Y DESC to get highest….”  
3. “Filter on ROW_NUMBER() <= k for top-k selection.”

### 3.4 CTEs & Subqueries
- **When to use CTE**: Improves readability for multi-step logic.  
- **Trade-off**: May or may not materialize; mention if performance critical.

**Explain**: “I’m defining `base` CTE to aggregate spend, then ranking in outer query.”

### 3.5 Optimization & Explain Plans
- **Use EXPLAIN**: Highlight index scans vs seq scans.  
- **Indexing**: Suggest indexes on filter/join columns.  
- **Predicate Pushdown**: Apply filters early.

**Answer Tip**: “With `EXPLAIN ANALYZE`, I see a hash join; I’d add index on… to reduce build time.”

---

## 4. Real-World Amazon Scenarios & Solutions

| Scenario                            | Key Patterns                                  |
|-------------------------------------|-----------------------------------------------|
| Delayed Shipment Rate               | CTEs, CASE WHEN, DATE arithmetic              |
| Monthly Active Prime Users          | Calendar table, JOIN on date intervals        |
| Inventory Replenishment             | Rolling avg (window or CTE), NULLIF handling  |
| Hierarchical Category Traversal     | Recursive CTE                                 |

> Refer to the Question Bank for full SQL answers and performance notes.

---

## 5. Behavioral-Style SQL Explanations
1. **Tell me about a time you optimized a query.**
   - **STAR**: Situation (huge table), Task (reduce runtime), Action (added composite index & refactored subquery), Result (500× faster).

2. **Explain how you would detect data quality issues in ETL.**
   - Describe key SQL checks: `COUNT(*) vs COUNT(DISTINCT)`, null ratio, referential integrity tests.

3. **How do you ensure your queries scale?**
   - Discuss partitioning/sharding strategy, incremental processing, and use of EXPLAIN.

---

## 6. Amazon-Specific Tips & Tricks
- **Communicate Clearly**: Verbalize assumptions and test examples.  
- **Edge Cases First**: State how you’d handle NULLs, empty sets, ties.  
- **Performance Mindset**: Always mention index suggestions & explain plans.  
- **Leadership Principles**:
  - _Dive Deep_: Ask about table size & distribution.  
  - _Customer Obsession_: Focus on data accuracy and timeliness.  
  - _Ownership_: Propose monitoring and alerting for query performance.
