# Northwind Traders Business Analysis using PostgreSQL

## Project Overview

This project analyzes the **Northwind Traders dataset** using PostgreSQL. It simulates real-world business analysis tasks through SQL queries that uncover insights on customers, products, employees, and suppliers. The project aims to replicate what business analysts do daily—turning raw data into clear insights that support decision-making.

## Objectives

1. **Understand business performance** through customer, product, and sales analysis.
2. **Apply SQL** for data cleaning, aggregation, and analytical reporting.
3. **Identify patterns** that show growth, churn, and efficiency opportunities.
4. **Build a foundation** for visual dashboards and executive reports.

## SQL Analysis & Business Questions

Below are 20 practical SQL questions with their corresponding queries and notes.

---

### 1. Who are our top customers in terms of number of orders (not revenue)?
```sql
SELECT
    company_name,
    COUNT(DISTINCT order_id) AS total_orders
FROM customers AS c
JOIN orders AS o ON c.customer_id = o.customer_id
GROUP BY company_name
ORDER BY total_orders DESC
LIMIT 5;
-- Identifies repeat customers by total order count.
```

### 2. Which customers are spread across multiple regions (ordering from more than one country)?
```sql
SELECT 
    c.customer_id,
    c.company_name,
    COUNT(DISTINCT o.ship_country) AS country_count,
    STRING_AGG(DISTINCT o.ship_country, ', ') AS countries
FROM customers AS c
JOIN orders AS o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.company_name
HAVING COUNT(DISTINCT o.ship_country) > 1
ORDER BY country_count DESC;
-- Detects clients with international reach.
```

### 3. Which customers stopped ordering after their first year?
```sql
SELECT
    c.customer_id,
    c.company_name,
    MIN(EXTRACT(YEAR FROM o.order_date)) AS first_year,
    MAX(EXTRACT(YEAR FROM o.order_date)) AS last_year
FROM customers AS c
JOIN orders AS o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.company_name
HAVING first_year != last_year
ORDER BY first_year;
-- Reveals potential churn customers.
```

### 4. Who are our newest customers, and how much have they spent so far?
```sql
SELECT
    company_name,
    MIN(order_date) AS first_order_date,
    SUM(od.unit_price * od.quantity * (1 - od.discount)) AS revenue
FROM customers AS c
JOIN orders AS o ON c.customer_id = o.customer_id
JOIN order_details AS od ON o.order_id = od.order_id
GROUP BY c.company_name
ORDER BY first_order_date DESC;
-- Tracks new customer acquisition and their early value.
```

### 5. Which customers had the biggest increase in spending from 1997 to 1998?
```sql
WITH customer_year_revenue AS (
    SELECT
        c.customer_id,
        c.company_name,
        EXTRACT(YEAR FROM o.order_date) AS year,
        SUM(od.unit_price * od.quantity * (1 - od.discount)) AS revenue
    FROM customers AS c
    JOIN orders AS o ON c.customer_id = o.customer_id
    JOIN order_details AS od ON o.order_id = od.order_id
    GROUP BY c.customer_id, c.company_name, EXTRACT(YEAR FROM o.order_date)
)
SELECT
    customer_id,
    company_name,
    year,
    revenue,
    LAG(revenue, 1, 0) OVER (PARTITION BY customer_id ORDER BY year) AS prev_revenue,
    revenue - LAG(revenue, 1, 0) OVER (PARTITION BY customer_id ORDER BY year) AS revenue_diff
FROM customer_year_revenue
WHERE year = 1998
ORDER BY revenue_diff DESC;
-- Identifies loyal customers whose purchases grew year-over-year.
```

### 6. Which products have the largest average order size (quantity)?
```sql
SELECT
    product_name,
    AVG(quantity) AS avg_order_size
FROM products AS p
JOIN order_details AS od ON p.product_id = od.product_id
GROUP BY product_name
ORDER BY avg_order_size DESC;
-- Highlights products often bought in bulk.
```

### 7. Which products are ordered by the most unique customers?
```sql
SELECT
    product_name,
    COUNT(DISTINCT o.customer_id) AS unique_customers
FROM products AS p
JOIN order_details AS od ON p.product_id = od.product_id
JOIN orders AS o ON o.order_id = od.order_id
GROUP BY product_name
ORDER BY unique_customers DESC;
-- Measures product popularity across different clients.
```

### 8. Which products are rarely sold (less than 10 total orders)?
```sql
SELECT
    product_name,
    COUNT(DISTINCT order_id) AS orders
FROM products AS p
JOIN order_details AS od ON p.product_id = od.product_id
GROUP BY product_name
HAVING COUNT(DISTINCT order_id) < 10
ORDER BY orders ASC;
-- Identifies low-performing products.
```

### 9. Which products got the deepest average discounts?
```sql
SELECT
    product_name,
    ROUND(AVG(discount) * 100, 2) AS avg_discount_percent
FROM products AS p
JOIN order_details AS od ON p.product_id = od.product_id
GROUP BY product_name
ORDER BY avg_discount_percent DESC;
-- Finds items frequently on heavy discount.
```

### 10. Which products are most often shipped internationally?
```sql
SELECT
    p.product_name,
    SUM(CASE WHEN s.country <> o.ship_country THEN 1 ELSE 0 END) AS international_shipments
FROM products AS p
JOIN suppliers AS s ON p.supplier_id = s.supplier_id
JOIN order_details AS od ON p.product_id = od.product_id
JOIN orders AS o ON o.order_id = od.order_id
GROUP BY p.product_name
ORDER BY international_shipments DESC;
-- Detects export-heavy products.
```

### 11. Which category drives the highest revenue in each country?
```sql
WITH category_revenue AS (
    SELECT
        cs.country,
        c.category_name,
        SUM(od.unit_price * od.quantity * (1 - od.discount)) AS revenue,
        RANK() OVER (PARTITION BY cs.country ORDER BY SUM(od.unit_price * od.quantity * (1 - od.discount)) DESC) AS rank_in_country
    FROM categories AS c
    JOIN products AS p ON c.category_id = p.category_id
    JOIN order_details AS od ON p.product_id = od.product_id
    JOIN orders AS o ON o.order_id = od.order_id
    JOIN customers AS cs ON o.customer_id = cs.customer_id
    GROUP BY cs.country, c.category_name
)
SELECT country, category_name, revenue
FROM category_revenue
WHERE rank_in_country = 1
ORDER BY revenue DESC;
-- Reveals top-selling categories per market.
```

### 12. Which employees handled orders for more than 3 different countries?
```sql
SELECT
    e.employee_id,
    CONCAT(e.first_name, ' ', e.last_name) AS employee_name,
    COUNT(DISTINCT o.ship_country) AS country_count
FROM employees AS e
JOIN orders AS o ON e.employee_id = o.employee_id
GROUP BY e.employee_id, e.first_name, e.last_name
HAVING COUNT(DISTINCT o.ship_country) > 3
ORDER BY country_count DESC;
-- Identifies globally active sales reps.
```

### 13. Which employee generated the highest sales revenue?
```sql
SELECT
    CONCAT(e.first_name, ' ', e.last_name) AS employee_name,
    ROUND(SUM(od.unit_price * od.quantity * (1 - od.discount))) AS sales_revenue
FROM employees AS e
JOIN orders AS o ON e.employee_id = o.employee_id
JOIN order_details AS od ON o.order_id = od.order_id
GROUP BY employee_name
ORDER BY sales_revenue DESC
LIMIT 1;
-- Finds the top-performing employee by revenue.
```

### 14. Which products in each category generated the highest total revenue?
```sql
WITH product_revenue AS (
    SELECT
        c.category_name,
        p.product_name,
        SUM(od.unit_price * od.quantity * (1 - od.discount)) AS revenue
    FROM categories AS c
    JOIN products AS p ON c.category_id = p.category_id
    JOIN order_details AS od ON p.product_id = od.product_id
    GROUP BY c.category_name, p.product_name
),
ranked AS (
    SELECT
        category_name,
        product_name,
        revenue,
        RANK() OVER (PARTITION BY category_name ORDER BY revenue DESC) AS rank_in_category
    FROM product_revenue
)
SELECT category_name, product_name, revenue
FROM ranked
WHERE rank_in_category = 1;
-- Lists the best-selling item in every category.
```

### 15. Which products are below reorder level and need restocking?
```sql
WITH stock_check AS (
    SELECT 
        product_id,
        product_name,
        units_in_stock + units_on_order AS total_available,
        reorder_level
    FROM products
)
SELECT product_name, total_available, reorder_level
FROM stock_check
WHERE total_available < reorder_level;
-- Detects low-inventory products that require replenishment.
```

### 16. How does revenue trend by month and category?
```sql
SELECT 
    c.category_name,
    EXTRACT(YEAR FROM o.order_date) AS year,
    EXTRACT(MONTH FROM o.order_date) AS month,
    ROUND(SUM(od.unit_price * od.quantity * (1 - od.discount))) AS monthly_revenue
FROM categories AS c
JOIN products AS p ON c.category_id = p.category_id
JOIN order_details AS od ON p.product_id = od.product_id
JOIN orders AS o ON o.order_id = od.order_id
GROUP BY c.category_name, year, month
ORDER BY year, month;
-- Evaluates seasonal sales performance per category.
```

### 17. What is the average order value per country?
```sql
SELECT
    o.ship_country AS country,
    ROUND(AVG(od.unit_price * od.quantity * (1 - od.discount))) AS avg_order_value
FROM orders AS o
JOIN order_details AS od ON o.order_id = od.order_id
GROUP BY o.ship_country
ORDER BY avg_order_value DESC;
-- Compares customer value by region.
```

### 18. Which suppliers contribute the most to total sales?
```sql
SELECT
    s.company_name AS supplier_name,
    SUM(od.unit_price * od.quantity * (1 - od.discount)) AS supplier_revenue
FROM suppliers AS s
JOIN products AS p ON s.supplier_id = p.supplier_id
JOIN order_details AS od ON p.product_id = od.product_id
GROUP BY s.company_name
ORDER BY supplier_revenue DESC;
-- Identifies key vendors driving revenue.
```

### 19. Which shipping companies handle the most revenue?
```sql
SELECT
    sh.company_name AS shipper_name,
    SUM(od.unit_price * od.quantity * (1 - od.discount)) AS total_revenue
FROM shippers AS sh
JOIN orders AS o ON sh.shipper_id = o.ship_via
JOIN order_details AS od ON o.order_id = od.order_id
GROUP BY sh.company_name
ORDER BY total_revenue DESC;
-- Evaluates shipper contribution to sales fulfillment.
```

### 20. What is the total freight cost by region?
```sql
SELECT
    ship_country,
    ROUND(SUM(freight)) AS total_freight
FROM orders
GROUP BY ship_country
ORDER BY total_freight DESC;
-- Highlights logistics cost distribution.
```

---

## Summary of Findings

- **Top customers** drive a large share of revenue.
- **Multi-country buyers** reflect strong B2B relationships.
- **Churn detection** helps focus retention strategies.
- **Condiments and Beverages** lead global category sales.
- **Key suppliers** dominate revenue streams.
- **Employees like Nancy Davolio** manage the widest client base.
- **Freight variation** suggests potential savings in logistics.

## Next Steps

- Visualize trends in Power BI or Tableau.
- Automate report generation using Python or SQL views.
- Build executive dashboards showing revenue, churn, and category growth.

---

*Project by Pogi — aspiring Data Analyst using SQL to turn data into business intelligence.*
