
## Amazon Data Engineer SQL Question Bank

A comprehensive collection of SQL interview questions inspired by Amazon’s phone screen rounds and real‑world scenarios. Each entry includes:
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
2. **Aggregate**: `AVG(rating)` calculates the mean rating; `ROUND` formats to two decimals.
3. **Grouping**: Group by `product_id`, `year`, and `month` ensures monthly buckets per product.
4. **Ordering**: Sorts results for readability.

**Edge Cases**:
- **No Reviews**: Product–month combinations with no rows are omitted. Use a calendar table join if zero rows need showing.
- **Null Ratings**: If `rating` can be NULL, exclude via `WHERE rating IS NOT NULL` or use `AVG(COALESCE(rating, 0))`.

**Performance**:
- **Index** on `(product_id, review_date)` speeds grouping and filter by date.
- Avoid functions on columns in `WHERE` clauses for partition pruning.

---

### 2. Top 2 Highest‑Grossing Products per Category
**Scenario**: Table `product_spend(category, product, user_id, spend, transaction_date)`.

**Question**: Identify the top two products by total spend in each category for the year 2022.

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

**Explanation**:
1. **CTE (`totals`)**: Aggregates spend per product and category within the date range.
2. **Window Function**: `ROW_NUMBER() OVER (PARTITION BY category ORDER BY total_spend DESC)` ranks products by spend in each category.
3. **Filter**: `WHERE rn <= 2` retains only the top two entries per category.

**Edge Cases**:
- **Ties**: If two products tie for second, only one is returned. Use `RANK()` or `DENSE_RANK()` to include ties.
- **No Transactions**: Categories without data are excluded automatically.

**Performance**:
- **Index** on `(transaction_date)` and `(category, product)` for fast aggregation.
- Evaluate using `EXPLAIN` to ensure the CTE doesn’t materialize unnecessarily.

---

### 3. Cumulative Shipments by Day
**Scenario**: Table `Shipments(shipment_id, order_id, ship_date)`.

**Question**: Calculate the total shipments per day and the running cumulative count over time.

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

**Explanation**:
1. **Daily Count**: `COUNT(*)` per `ship_date` gives daily shipments.
2. **Running Sum**: `SUM(...) OVER (ORDER BY ship_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` accumulates counts up to each date.

**Edge Cases**:
- **Dates with Zero Shipments**: To display zero‑shipment days, join a calendar table.

**Performance**:
- **Index** on `(ship_date)` enables grouping and ordering efficiency.
- Window functions can be memory‑intensive; ensure sufficient resources.

---

### 4. Second Highest Order Total per Customer
**Scenario**: Table `Orders(order_id, customer_id, total_amount)`.

**Question**: Retrieve each customer’s second largest order amount.

```sql
SELECT
  customer_id,
  total_amount
FROM (
  SELECT
    customer_id,
    total_amount,
    DENSE_RANK() OVER (
      PARTITION BY customer_id
      ORDER BY total_amount DESC
    ) AS rk
  FROM Orders
) t
WHERE rk = 2;
```

**Explanation**:
1. **Partitioned Ranking**: `DENSE_RANK()` assigns ranks to `total_amount` per customer, handling ties.
2. **Filter**: Select only rank = 2 for the second highest values.

**Edge Cases**:
- **Single Order**: Customers with fewer than two orders return no rows. Consider `LEFT JOIN` to include all customers with `NULL` values.

**Performance**:
- **Index** on `(customer_id, total_amount)` to optimize partitioned sort.

---

### 5. Prime Users with No Recent Orders
**Scenario**: Tables `PrimeUsers(user_id, join_date)` and `Orders(order_id, user_id, order_date)`.

**Question**: List Prime users who have not placed any orders in the last six months.

```sql
SELECT p.user_id
FROM PrimeUsers p
LEFT JOIN Orders o
  ON p.user_id = o.user_id
  AND o.order_date >= CURRENT_DATE - INTERVAL '6 MONTH'
WHERE o.user_id IS NULL;
```

**Explanation**:
1. **Anti‑Join Pattern**: `LEFT JOIN` with a date filter brings in recent orders; `WHERE o.user_id IS NULL` finds non‑matches.

**Edge Cases**:
- **New Users**: If `join_date` is within six months, they show as inactive even if they never had a chance. Add `AND p.join_date < CURRENT_DATE - INTERVAL '6 MONTH'` if needed.

**Performance**:
- **Index** on `(user_id, order_date)` accelerates join and filter.

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
