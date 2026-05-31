# Monday-Coffee-Expansion-SQL-Project
The goal of this project is to analyze the sales data of Monday Coffee, a company that has been selling its products online since January 2023, and to recommend the top three major cities in India for opening new coffee shop locations based on consumer demand and sales performance.

<img width="1606" height="906" alt="image" src="https://github.com/user-attachments/assets/3ef71c04-a71f-4088-8940-3e8135f418ad" />


## Database Schema

| Table | Description |
|---|---|
| `city` | City name, population, estimated rent, city rank |
| `products` | Coffee product catalog |
| `customers` | Customer records linked to cities |
| `sales` | Transaction records with product, customer, date, and amount |

---

## Analysis Questions & SQL Queries

---

### Q1. Coffee Consumers Count
**How many people in each city are estimated to consume coffee, given that 25% of the population does?**

```sql
SELECT 
   city_name,
   ROUND((population * 0.25) / 1000000, 2) AS coffee_consumers_in_million,
   city_rank 
FROM city
ORDER BY 2 DESC;
```

> **Finding:** Delhi leads with the highest estimated coffee consumer base (~7.7 million), followed by Mumbai and Kolkata.

---

### Q2. Total Revenue from Coffee Sales
**What is the total revenue generated from coffee sales across all cities in the last quarter of 2023?**

```sql
-- Overall Revenue
SELECT 
    SUM(total) AS total_revenue
FROM sales
WHERE 
    EXTRACT(YEAR FROM sale_date) = 2023
    AND EXTRACT(QUARTER FROM sale_date) = 4;

-- Revenue by City
SELECT 
    ci.city_name,
    SUM(s.total) AS total_revenue
FROM sales AS s
JOIN customers AS c ON s.customer_id = c.customer_id
JOIN city AS ci ON ci.city_id = c.city_id
WHERE 
    EXTRACT(YEAR FROM s.sale_date) = 2023
    AND EXTRACT(QUARTER FROM s.sale_date) = 4
GROUP BY 1
ORDER BY 2 DESC;
```

> **Finding:** Pune generated the highest revenue in Q4 2023, making it the most profitable city in the last quarter.

---

### Q3. Sales Count for Each Product
**How many units of each coffee product have been sold?**

```sql
SELECT 
    p.product_name,
    COUNT(s.sale_id) AS total_orders
FROM products AS p
LEFT JOIN sales AS s ON s.product_id = p.product_id
GROUP BY 1
ORDER BY 2 DESC;
```

> **Finding:** A few hero products dominate total orders, while some niche products have near-zero sales — indicating an opportunity to rationalize the menu.

---

### Q4. Average Sales Amount per City
**What is the average sales amount per customer in each city?**

```sql
SELECT
    ci.city_name,
    SUM(s.total) AS total_revenue,
    COUNT(DISTINCT c.customer_id) AS total_customer,
    ROUND(SUM(s.total)::NUMERIC / COUNT(DISTINCT c.customer_id)::NUMERIC, 2) AS avg_sale_pr_cx
FROM customers AS c
JOIN city AS ci USING (city_id)
JOIN sales AS s USING (customer_id)
GROUP BY ci.city_name
ORDER BY SUM(s.total) DESC;
```

> **Finding:** Pune has the highest average sale per customer, showing that customers there spend significantly more per transaction than other cities.

---

### Q5. City Population and Coffee Consumers (25%)
**Provide a list of cities along with their populations and estimated coffee consumers. Return city name, total current customers, and estimated coffee consumers (25%).**

```sql
WITH cte1 AS (
    SELECT
        city_name,
        ROUND((population * 0.25) / 1000000, 2) AS coffee_consumer_million
    FROM city
),
cte2 AS (
    SELECT
        ci.city_name,
        COUNT(DISTINCT c.customer_id) AS unique_customer
    FROM sales AS s
    JOIN customers AS c USING (customer_id)
    JOIN city AS ci USING (city_id)
    GROUP BY 1
)
SELECT
    c1.city_name,
    c1.coffee_consumer_million,
    c2.unique_customer
FROM cte1 AS c1
JOIN cte2 AS c2 USING (city_name);
```

> **Finding:** Most cities have a large untapped gap between estimated coffee consumers and actual customers, highlighting strong room for customer acquisition.

---

### Q6. Top Selling Products by City
**What are the top 3 selling products in each city based on sales volume?**

```sql
WITH cte AS (
    SELECT 
        ci.city_name,
        p.product_name,
        COUNT(s.sale_id) AS total_orders,
        DENSE_RANK() OVER (
            PARTITION BY ci.city_name 
            ORDER BY COUNT(s.sale_id) DESC
        ) AS rnk
    FROM sales AS s
    JOIN products AS p USING (product_id)
    JOIN customers AS c USING (customer_id)
    JOIN city AS ci USING (city_id)
    GROUP BY 1, 2
)
SELECT * FROM cte
WHERE rnk <= 3;
```

> **Finding:** Certain products consistently rank in the top 3 across multiple cities, suggesting they are universal bestsellers suitable for a core menu push nationwide.

---

### Q7. Customer Segmentation by City
**How many unique customers are there in each city who have purchased coffee products?**

```sql
SELECT 
    ci.city_name,
    COUNT(DISTINCT c.customer_id) AS unique_cx
FROM customers AS c
JOIN city AS ci USING (city_id)
JOIN sales AS s ON s.customer_id = c.customer_id
WHERE s.product_id IN (1,2,3,4,5,6,7,8,9,10,11,12,13,14)
GROUP BY 1
ORDER BY 2 DESC;
```

> **Finding:** Jaipur and Delhi have the highest unique customer counts, indicating strong existing demand and a loyal customer base in these cities.

---

### Q8. Average Sale vs Rent
**Find each city and their average sale per customer and average rent per customer.**

```sql
WITH cte1 AS (
    SELECT
        ci.city_name,
        COUNT(DISTINCT c.customer_id) AS total_customer,
        ROUND(SUM(s.total)::NUMERIC / COUNT(DISTINCT c.customer_id)::NUMERIC, 2) AS avg_sale_pr_cx
    FROM customers AS c
    JOIN city AS ci USING (city_id)
    JOIN sales AS s USING (customer_id)
    GROUP BY ci.city_name
    ORDER BY SUM(s.total) DESC
),
cte2 AS (
    SELECT city_name, estimated_rent FROM city
)
SELECT 
    c2.city_name,
    c2.estimated_rent,
    c1.total_customer,
    c1.avg_sale_pr_cx,
    ROUND(c2.estimated_rent::NUMERIC / c1.total_customer::NUMERIC, 2) AS avg_rent_per_cx
FROM cte2 AS c2
JOIN cte1 AS c1 USING (city_name)
ORDER BY 4 DESC;
```

> **Finding:** Pune and Jaipur show the best rent-to-revenue ratio — high average sales per customer paired with low average rent per customer make them the most cost-efficient markets.

---

### Q9. Monthly Sales Growth
**Calculate the percentage growth (or decline) in sales month-over-month by each city.**

```sql
WITH cte1 AS (
    SELECT 
        ci.city_name,
        EXTRACT(MONTH FROM sale_date) AS month,
        EXTRACT(YEAR FROM sale_date) AS year,
        SUM(s.total) AS total_sale
    FROM sales AS s
    JOIN customers AS c USING (customer_id)
    JOIN city AS ci USING (city_id)
    GROUP BY 1, 2, 3
    ORDER BY 1, 3, 2
),
cte2 AS (
    SELECT 
        city_name,
        month,
        year,
        total_sale AS current_month_sale,
        LAG(total_sale, 1) OVER (
            PARTITION BY city_name 
            ORDER BY year, month
        ) AS last_month_sale
    FROM cte1
)
SELECT *,
    ROUND(
        (current_month_sale - last_month_sale)::NUMERIC / last_month_sale::NUMERIC * 100,
    2) AS growth_ratio
FROM cte2
WHERE last_month_sale IS NOT NULL;
```

> **Finding:** Sales growth is inconsistent across cities — some cities show strong month-over-month growth while others experience seasonal dips, suggesting the need for city-specific marketing strategies.

---

### Q10. Market Potential Analysis
**Identify the top 3 cities based on highest sales. Return city name, total sale, total rent, total customers, estimated coffee consumers.**

```sql
WITH cte1 AS (
    SELECT
        ci.city_name,
        SUM(s.total) AS total_revenue,
        COUNT(DISTINCT c.customer_id) AS total_customer,
        ROUND(
            SUM(s.total)::NUMERIC / COUNT(DISTINCT c.customer_id)::NUMERIC
        , 2) AS avg_sale_pr_cx
    FROM customers AS c
    JOIN city AS ci USING (city_id)
    JOIN sales AS s USING (customer_id)
    GROUP BY ci.city_name
    ORDER BY SUM(s.total) DESC
),
cte2 AS (
    SELECT
        city_name,
        estimated_rent,
        ROUND(population * 0.25 / 1000000, 2) AS est_coffee_consumer_million
    FROM city
)
SELECT
    c2.city_name,
    c1.total_revenue,
    c2.estimated_rent AS total_rent,
    c2.est_coffee_consumer_million,
    c1.total_customer,
    c1.avg_sale_pr_cx,
    ROUND(c2.estimated_rent::NUMERIC / c1.total_customer::NUMERIC, 2) AS avg_rent_per_cx
FROM cte2 AS c2
JOIN cte1 AS c1 USING (city_name)
ORDER BY c1.total_revenue DESC;
```

> **Finding:** Pune, Delhi, and Jaipur emerge as the top 3 markets — each excelling in different metrics (revenue, consumer base, and customer count) making them the highest-priority cities for new outlet expansion.

---

## Recommendations

Based on the Q10 Market Potential Analysis, the top 3 recommended cities for expansion are:

### 🥇 Pune
1. Average rent per customer is very low
2. Highest total revenue overall
3. Average sales per customer is also high

### 🥈 Delhi
1. Highest estimated coffee consumers at **7.7 million**
2. Highest total number of customers (68)
3. Average rent per customer is ₹330 (still under ₹500)

### 🥉 Jaipur
1. Highest number of customers (69)
2. Average rent per customer is very low at ₹156
3. Average sales per customer is strong at ₹11.6k

---

## Key SQL Concepts Used

- `JOIN` (INNER, LEFT) across multiple tables
- Aggregate functions: `SUM`, `COUNT`, `ROUND`
- `EXTRACT` for date filtering (year, quarter, month)
- Window functions: `DENSE_RANK`, `LAG`
- Common Table Expressions (`WITH` / CTEs)
- Type casting (`::NUMERIC`) for precise division

---

## Tools & Technologies

- **PostgreSQL** — primary database
- **SQL** — data querying and analysis

---

## Project Structure

```
monday-coffee/
│
├── coffee_project.sql     # All analysis queries
└── README.md              # Project documentation
```
