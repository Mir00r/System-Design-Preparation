# Top Frequently Asked SQL Interview Questions and Answers

Here's a comprehensive list of common SQL interview questions with detailed answers, comments, and example outputs.

## Basic SQL Questions

### 1. What is the difference between WHERE and HAVING clauses?

```sql
-- WHERE filters rows before grouping (GROUP BY)
-- HAVING filters groups after grouping

-- Example:
SELECT department, AVG(salary) as avg_salary
FROM employees
WHERE hire_date > '2020-01-01'  -- Filters individual rows
GROUP BY department
HAVING AVG(salary) > 50000;     -- Filters groups

-- Output might look like:
-- department | avg_salary
-- -----------+-----------
-- Engineering | 75000
-- Sales      | 60000
```

### 2. Explain different types of JOINs

```sql
-- INNER JOIN: Returns rows when there's a match in both tables
SELECT e.name, d.department_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.id;

-- LEFT JOIN: Returns all rows from left table, matched rows from right (or NULL)
SELECT e.name, d.department_name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.id;

-- RIGHT JOIN: Returns all rows from right table, matched rows from left (or NULL)
SELECT e.name, d.department_name
FROM employees e
RIGHT JOIN departments d ON e.dept_id = d.id;

-- FULL OUTER JOIN: Returns all rows when there's a match in either table
SELECT e.name, d.department_name
FROM employees e
FULL OUTER JOIN departments d ON e.dept_id = d.id;

-- CROSS JOIN: Cartesian product of all rows
SELECT e.name, d.department_name
FROM employees e
CROSS JOIN departments d;
```

### 3. What is the difference between UNION and UNION ALL?

```sql
-- UNION combines results and removes duplicates
SELECT product_id FROM current_products
UNION
SELECT product_id FROM discontinued_products;

-- UNION ALL combines results keeping all duplicates
SELECT product_id FROM current_products
UNION ALL
SELECT product_id FROM discontinued_products;

-- Example output for UNION:
-- product_id
-- ----------
-- 101
-- 102
-- 103

-- Example output for UNION ALL might have duplicates:
-- product_id
-- ----------
-- 101
-- 101
-- 102
-- 103
```

## Intermediate SQL Questions

### 4. How would you find the second highest salary?

```sql
-- Method 1: Using subquery
SELECT MAX(salary) 
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);

-- Method 2: Using window functions (more modern approach)
SELECT salary
FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) as rank
    FROM employees
) ranked
WHERE rank = 2;

-- Output:
-- salary
-- ------
-- 85000
```

### 5. Write a query to find duplicate emails in a table

```sql
SELECT email, COUNT(*) as count
FROM users
GROUP BY email
HAVING COUNT(*) > 1;

-- Output:
-- email            | count
-- -----------------+------
-- john@example.com | 2
-- jane@example.com | 3
```

### 6. How would you delete duplicate rows?

```sql
-- Using Common Table Expression (CTE) with ROW_NUMBER()
WITH duplicates AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) as row_num
    FROM users
)
DELETE FROM duplicates
WHERE row_num > 1;

-- This keeps one copy of each duplicate
```

## Advanced SQL Questions

### 7. Explain the difference between RANK(), DENSE_RANK(), and ROW_NUMBER()

```sql
-- Example showing all three functions:
SELECT 
    employee_id,
    salary,
    RANK() OVER (ORDER BY salary DESC) as rank,
    DENSE_RANK() OVER (ORDER BY salary DESC) as dense_rank,
    ROW_NUMBER() OVER (ORDER BY salary DESC) as row_num
FROM employees;

-- Output:
-- employee_id | salary | rank | dense_rank | row_num
-- ------------+--------+------+------------+--------
-- 101         | 100000 | 1    | 1          | 1
-- 102         | 90000  | 2    | 2          | 2
-- 103         | 90000  | 2    | 2          | 3
-- 104         | 80000  | 4    | 3          | 4
```

### 8. Write a recursive CTE to find all subordinates under a manager

```sql
WITH RECURSIVE org_hierarchy AS (
    -- Base case: select top manager
    SELECT id, name, manager_id, 1 as level
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive case: join employees with their managers
    SELECT e.id, e.name, e.manager_id, oh.level + 1
    FROM employees e
    JOIN org_hierarchy oh ON e.manager_id = oh.id
)
SELECT * FROM org_hierarchy
ORDER BY level, id;

-- Output:
-- id | name      | manager_id | level
-- ---+-----------+------------+------
-- 1  | CEO      | NULL       | 1
-- 2  | VP Sales | 1          | 2
-- 3  | VP Eng   | 1          | 2
-- 4  | Sales Mgr| 2          | 3
-- 5  | Eng Mgr  | 3          | 3
```

### 9. How would you calculate a running total?

```sql
SELECT 
    date,
    sales,
    SUM(sales) OVER (ORDER BY date) as running_total
FROM daily_sales;

-- Output:
-- date       | sales | running_total
-- -----------+-------+--------------
-- 2023-01-01 | 100   | 100
-- 2023-01-02 | 150   | 250
-- 2023-01-03 | 200   | 450
-- 2023-01-04 | 50    | 500
```

## Database Design Questions

### 10. Explain normalization and its forms

```
Normalization is the process of organizing data to minimize redundancy. The main forms are:

1. First Normal Form (1NF):
   - Each column contains atomic values
   - No repeating groups

2. Second Normal Form (2NF):
   - Meets 1NF requirements
   - All non-key attributes fully dependent on the primary key

3. Third Normal Form (3NF):
   - Meets 2NF requirements
   - No transitive dependencies (non-key attributes don't depend on other non-key attributes)

Example of normalization:
- Unnormalized: Orders(order_id, customer_name, product1, product2, product3)
- 1NF: Orders(order_id, customer_name, product), Order_Items(order_id, product_id, quantity)
- 2NF: Ensure all attributes depend on the whole primary key
- 3NF: Remove transitive dependencies (e.g., customer_address should be in a Customers table)
```

### 11. What are indexes and when would you use them?

```sql
-- Indexes are special lookup tables that speed up data retrieval
-- Create an index:
CREATE INDEX idx_employee_name ON employees(last_name, first_name);

-- When to use indexes:
-- 1. Columns frequently used in WHERE clauses
-- 2. Columns used in JOIN conditions
-- 3. Columns used in ORDER BY clauses

-- When NOT to use indexes:
-- 1. On small tables
-- 2. On tables with frequent write operations (they slow down INSERT/UPDATE/DELETE)
-- 3. On columns with low cardinality (few unique values)
```

## Performance Tuning Questions

### 12. How would you optimize a slow-running query?

```
1. Check the execution plan (EXPLAIN ANALYZE in PostgreSQL)
2. Look for full table scans - add appropriate indexes
3. Avoid SELECT * - only select needed columns
4. Check for inefficient JOINs - ensure joined columns are indexed
5. Look for subqueries that could be rewritten as JOINs
6. Consider partitioning large tables
7. Check for OR conditions that could be rewritten as UNION
8. Verify statistics are up to date (ANALYZE in PostgreSQL)
```

### 13. What is query execution plan and how do you read it?

```sql
-- In PostgreSQL:
EXPLAIN ANALYZE SELECT * FROM employees WHERE department = 'Sales';

-- Key things to look for:
-- 1. Seq Scan (full table scan) vs Index Scan
-- 2. Cost estimates (higher is worse)
-- 3. Actual time (execution time)
-- 4. Join types (Nested Loop, Hash Join, Merge Join)
-- 5. Sort operations (could be expensive)
-- 6. Filter conditions being applied

-- Example output:
-- QUERY PLAN
-- -------------------------------------------------------------------------
-- Index Scan using idx_dept on employees  (cost=0.29..8.30 rows=1 width=36)
--   Index Cond: (department = 'Sales'::text)
-- Planning Time: 0.113 ms
-- Execution Time: 0.029 ms
```

## Window Functions

### 14. How would you compare each employee's salary to their department average?

```sql
SELECT 
    employee_id,
    name,
    department,
    salary,
    AVG(salary) OVER (PARTITION BY department) as dept_avg,
    salary - AVG(salary) OVER (PARTITION BY department) as diff_from_avg
FROM employees;

-- Output:
-- employee_id | name  | department | salary | dept_avg | diff_from_avg
-- ------------+-------+------------+--------+----------+--------------
-- 101         | John  | Sales      | 60000  | 55000    | 5000
-- 102         | Jane  | Sales      | 50000  | 55000    | -5000
-- 103         | Bob   | Engineering| 80000  | 75000    | 5000
-- 104         | Alice | Engineering| 70000  | 75000    | -5000
```

### 15. How would you find the top 3 earners in each department?

```sql
WITH ranked_employees AS (
    SELECT 
        employee_id,
        name,
        department,
        salary,
        DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) as rank
    FROM employees
)
SELECT * FROM ranked_employees
WHERE rank <= 3;

-- Output:
-- employee_id | name  | department | salary | rank
-- ------------+-------+------------+--------+------
-- 101         | John  | Sales      | 90000  | 1
-- 102         | Jane  | Sales      | 85000  | 2
-- 103         | Bob   | Sales      | 80000  | 3
-- 104         | Alice | Engineering| 95000  | 1
-- 105         | Dave  | Engineering| 92000  | 2
-- 106         | Eve   | Engineering| 91000  | 3
```
---

# Additional SQL Query Interview Questions and Answers

## Advanced Query Techniques

### 1. How would you pivot rows to columns?

```sql
-- Using CASE statements for pivoting
SELECT 
    product_id,
    SUM(CASE WHEN month = 'Jan' THEN sales ELSE 0 END) AS jan_sales,
    SUM(CASE WHEN month = 'Feb' THEN sales ELSE 0 END) AS feb_sales,
    SUM(CASE WHEN month = 'Mar' THEN sales ELSE 0 END) AS mar_sales
FROM monthly_sales
GROUP BY product_id;

-- In databases that support PIVOT (like SQL Server):
SELECT *
FROM monthly_sales
PIVOT (
    SUM(sales) FOR month IN ('Jan' AS jan_sales, 'Feb' AS feb_sales, 'Mar' AS mar_sales)
) AS pivoted_sales;

-- Output:
-- product_id | jan_sales | feb_sales | mar_sales
-- -----------+-----------+-----------+----------
-- 101        | 1000      | 1500      | 2000
-- 102        | 500       | 750       | 1000
```

### 2. How would you find gaps in sequential data?

```sql
-- Find missing IDs in a sequence
WITH sequence AS (
    SELECT MIN(id) as min_id, MAX(id) as max_id FROM orders
),
all_ids AS (
    SELECT generate_series(min_id, max_id) as id FROM sequence
)
SELECT a.id
FROM all_ids a
LEFT JOIN orders o ON a.id = o.id
WHERE o.id IS NULL;

-- Output:
-- id
-- ---
-- 5
-- 8
-- 12
```

### 3. How would you calculate the percentage of total for each row?

```sql
SELECT 
    product_id,
    sales,
    sales / SUM(sales) OVER () * 100 AS percent_of_total
FROM product_sales;

-- For percentage by category:
SELECT 
    product_id,
    category,
    sales,
    sales / SUM(sales) OVER (PARTITION BY category) * 100 AS percent_in_category
FROM product_sales;

-- Output:
-- product_id | category | sales | percent_of_total
-- -----------+----------+-------+-----------------
-- 101        | Electronics | 1000 | 25.00
-- 102        | Electronics | 1500 | 37.50
-- 201        | Clothing    | 500  | 12.50
-- 202        | Clothing    | 1000 | 25.00
```

## Date and Time Queries

### 4. How would you calculate month-over-month growth?

```sql
WITH monthly_totals AS (
    SELECT 
        DATE_TRUNC('month', order_date) AS month,
        SUM(amount) AS total_sales
    FROM orders
    GROUP BY DATE_TRUNC('month', order_date)
)
SELECT 
    month,
    total_sales,
    LAG(total_sales) OVER (ORDER BY month) AS prev_month_sales,
    (total_sales - LAG(total_sales) OVER (ORDER BY month)) / 
    LAG(total_sales) OVER (ORDER BY month) * 100 AS growth_percentage
FROM monthly_totals
ORDER BY month;

-- Output:
-- month      | total_sales | prev_month_sales | growth_percentage
-- -----------+-------------+------------------+------------------
-- 2023-01-01 | 10000       | NULL             | NULL
-- 2023-02-01 | 12000       | 10000            | 20.00
-- 2023-03-01 | 15000       | 12000            | 25.00
```

### 5. How would you find active users who logged in consecutively for at least 3 days?

```sql
WITH consecutive_logins AS (
    SELECT 
        user_id,
        login_date,
        LAG(login_date, 1) OVER (PARTITION BY user_id ORDER BY login_date) AS prev1,
        LAG(login_date, 2) OVER (PARTITION BY user_id ORDER BY login_date) AS prev2
    FROM user_logins
)
SELECT DISTINCT user_id
FROM consecutive_logins
WHERE login_date = prev1 + INTERVAL '1 day'
  AND login_date = prev2 + INTERVAL '2 days';

-- Output:
-- user_id
-- -------
-- 101
-- 205
```

## Complex Joins and Subqueries

### 6. How would you find employees who earn more than their managers?

```sql
SELECT e.employee_id, e.name, e.salary, m.salary as manager_salary
FROM employees e
JOIN employees m ON e.manager_id = m.employee_id
WHERE e.salary > m.salary;

-- Output:
-- employee_id | name  | salary | manager_salary
-- ------------+-------+--------+---------------
-- 105         | Alice | 90000  | 85000
-- 108         | Bob   | 95000  | 90000
```

### 7. How would you find customers who purchased all products in a category?

```sql
SELECT c.customer_id, c.customer_name
FROM customers c
WHERE NOT EXISTS (
    SELECT p.product_id
    FROM products p
    WHERE p.category = 'Electronics'
    AND NOT EXISTS (
        SELECT 1
        FROM orders o
        WHERE o.customer_id = c.customer_id
        AND o.product_id = p.product_id
    )
);

-- Output:
-- customer_id | customer_name
-- ------------+--------------
-- 1001        | John Smith
-- 1005        | Jane Doe
```

## Advanced Aggregation

### 8. How would you calculate a cumulative distribution of salaries?

```sql
SELECT 
    employee_id,
    salary,
    CUME_DIST() OVER (ORDER BY salary) AS cumulative_distribution,
    PERCENT_RANK() OVER (ORDER BY salary) AS percentile_rank
FROM employees;

-- Output:
-- employee_id | salary | cumulative_distribution | percentile_rank
-- ------------+--------+-------------------------+----------------
-- 103         | 50000  | 0.10                    | 0.00
-- 102         | 60000  | 0.30                    | 0.22
-- 101         | 75000  | 0.60                    | 0.44
-- 105         | 80000  | 0.80                    | 0.67
-- 104         | 95000  | 1.00                    | 0.89
```

### 9. How would you calculate a moving average?

```sql
SELECT 
    date,
    sales,
    AVG(sales) OVER (ORDER BY date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg_3day
FROM daily_sales;

-- Output:
-- date       | sales | moving_avg_3day
-- -----------+-------+----------------
-- 2023-01-01 | 100   | 100.00
-- 2023-01-02 | 150   | 125.00
-- 2023-01-03 | 200   | 150.00
-- 2023-01-04 | 50    | 133.33
-- 2023-01-05 | 100   | 116.67
```

## Data Quality and Validation

### 10. How would you find inconsistent or invalid data?

```sql
-- Example: Find orders with invalid dates (future dates or before company founding)
SELECT order_id, order_date
FROM orders
WHERE order_date > CURRENT_DATE
   OR order_date < '2010-01-01';

-- Example: Find products with negative prices
SELECT product_id, price
FROM products
WHERE price < 0;

-- Example: Find customers with invalid email formats
SELECT customer_id, email
FROM customers
WHERE email NOT LIKE '%@%.%';

-- Output for invalid dates:
-- order_id | order_date
-- ---------+------------
-- 1005     | 2025-01-01
-- 1020     | 2009-12-15
```

### 11. How would you implement a soft delete pattern?

```sql
-- Table design with soft delete:
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2),
    is_deleted BOOLEAN DEFAULT FALSE,
    deleted_at TIMESTAMP NULL
);

-- Soft delete operation:
UPDATE products
SET is_deleted = TRUE,
    deleted_at = CURRENT_TIMESTAMP
WHERE product_id = 101;

-- Query active products:
SELECT * FROM products WHERE is_deleted = FALSE;

-- To restore:
UPDATE products
SET is_deleted = FALSE,
    deleted_at = NULL
WHERE product_id = 101;
```

## Performance-Oriented Queries

### 12. How would you optimize a query that joins multiple large tables?

```sql
-- Optimization techniques:
-- 1. Ensure join columns are indexed
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_product ON orders(product_id);

-- 2. Use appropriate join types
SELECT c.customer_name, p.product_name, o.quantity
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
INNER JOIN products p ON o.product_id = p.product_id
WHERE o.order_date BETWEEN '2023-01-01' AND '2023-03-31';

-- 3. Consider materialized views for frequently accessed aggregations
CREATE MATERIALIZED VIEW monthly_sales_summary AS
SELECT 
    DATE_TRUNC('month', order_date) AS month,
    product_id,
    SUM(quantity) AS total_quantity,
    SUM(quantity * price) AS total_sales
FROM orders
GROUP BY DATE_TRUNC('month', order_date), product_id;

-- 4. Use query hints if necessary (database-specific)
```

### 13. How would you handle hierarchical data (e.g., organizational chart)?

```sql
-- Using recursive CTE for hierarchical queries
WITH RECURSIVE org_chart AS (
    -- Base case: top-level employees (CEO)
    SELECT employee_id, name, title, manager_id, 1 AS level
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive case: join employees with their managers
    SELECT e.employee_id, e.name, e.title, e.manager_id, oc.level + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.employee_id
)
SELECT 
    employee_id,
    LPAD(' ', (level-1)*4, ' ') || name AS name_hierarchy,
    title,
    level
FROM org_chart
ORDER BY level, name;

-- Output:
-- employee_id | name_hierarchy    | title              | level
-- ------------+-------------------+--------------------+------
-- 1           | CEO               | Chief Executive    | 1
-- 2           |     John Smith    | VP Engineering     | 2
-- 3           |     Jane Doe      | VP Marketing       | 2
-- 4           |         Bob Brown | Engineering Manager| 3
-- 5           |         Alice Lee | Marketing Manager  | 3
```

## Window Functions Deep Dive

### 14. How would you find the first and last order for each customer?

```sql
SELECT 
    customer_id,
    order_id,
    order_date,
    FIRST_VALUE(order_id) OVER (PARTITION BY customer_id ORDER BY order_date) AS first_order,
    FIRST_VALUE(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS first_order_date,
    LAST_VALUE(order_id) OVER (PARTITION BY customer_id ORDER BY order_date 
                              ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_order,
    LAST_VALUE(order_date) OVER (PARTITION BY customer_id ORDER BY order_date 
                                ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_order_date
FROM orders;

-- Alternative with simpler approach:
WITH order_ranks AS (
    SELECT 
        customer_id,
        order_id,
        order_date,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) AS order_seq_asc,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS order_seq_desc
    FROM orders
)
SELECT 
    customer_id,
    MAX(CASE WHEN order_seq_asc = 1 THEN order_id END) AS first_order,
    MAX(CASE WHEN order_seq_asc = 1 THEN order_date END) AS first_order_date,
    MAX(CASE WHEN order_seq_desc = 1 THEN order_id END) AS last_order,
    MAX(CASE WHEN order_seq_desc = 1 THEN order_date END) AS last_order_date
FROM order_ranks
GROUP BY customer_id;

-- Output:
-- customer_id | first_order | first_order_date | last_order | last_order_date
-- ------------+-------------+------------------+------------+----------------
-- 101         | 1001        | 2023-01-05       | 1005       | 2023-03-15
-- 102         | 1002        | 2023-01-10       | 1008       | 2023-04-20
```

### 15. How would you calculate the difference between current row and previous row?

```sql
SELECT 
    date,
    sales,
    sales - LAG(sales, 1) OVER (ORDER BY date) AS daily_change,
    (sales - LAG(sales, 1) OVER (ORDER BY date)) / 
    LAG(sales, 1) OVER (ORDER BY date) * 100 AS daily_change_percent
FROM daily_sales;

-- Output:
-- date       | sales | daily_change | daily_change_percent
-- -----------+-------+--------------+---------------------
-- 2023-01-01 | 100   | NULL         | NULL
-- 2023-01-02 | 150   | 50           | 50.00
-- 2023-01-03 | 200   | 50           | 33.33
-- 2023-01-04 | 50    | -150         | -75.00
-- 2023-01-05 | 100   | 50           | 100.00
```


