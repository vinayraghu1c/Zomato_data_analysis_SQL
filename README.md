# SQL Project: Data Analysis for Zomato - A Food Delivery Company

## Overview

This project demonstrates my SQL problem-solving skills through the analysis of data for Zomato, a popular food delivery company in India. The project involves setting up the database, importing data, handling null values, and solving a variety of business problems using complex SQL queries.

## Project Structure

- **Database Setup:** Creation of the `zomato_db` database and the required tables.
- **Data Import:** Inserting sample data into the tables.
- **Data Cleaning:** Handling null values and ensuring data integrity.
- **Business Problems:** Solving 20 specific business problems using SQL queries.

![ERD](https://github.com/ShivakrishnaMacha/Zomato_SQL_Data_Analysis/blob/main/erd.png)

## Database Setup
```sql
create database Zomato;
```

### 1. Dropping Existing Tables
```sql
DROP TABLE IF EXISTS deliveries;
DROP TABLE IF EXISTS Orders;
DROP TABLE IF EXISTS customers;
DROP TABLE IF EXISTS restaurants;
DROP TABLE IF EXISTS riders;

-- 2. Creating Tables

CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    customer_name VARCHAR(100) NOT NULL,
    reg_date DATE
);

CREATE TABLE restaurants (
    restaurant_id SERIAL PRIMARY KEY,
    restaurant_name VARCHAR(100) NOT NULL,
    city VARCHAR(50),
    opening_hours VARCHAR(100)
);

CREATE TABLE orders (
    order_id SERIAL,
    customer_id BIGINT UNSIGNED,
    restaurant_id BIGINT UNSIGNED,
    order_item VARCHAR(260),
    order_date DATE NOT NULL,
    order_time TIME NOT NULL,
    order_status VARCHAR(30) DEFAULT 'Pending',
    total_amount DECIMAL(10 , 2 ) NOT NULL,
    FOREIGN KEY (customer_id)
        REFERENCES customers (customer_id),
    FOREIGN KEY (restaurant_id)
        REFERENCES restaurant (restaurant_id)
);
alter table orders add primary key(order_id);

CREATE TABLE riders (
    rider_id SERIAL PRIMARY KEY,
    rider_name VARCHAR(100) NOT NULL,
    sign_up DATE
);

CREATE TABLE deliveries (
    delivery_id SERIAL PRIMARY KEY,
    order_id INT,
    delivery_status VARCHAR(80),
    delivery_time TIME,
    rider_id BIGINT UNSIGNED,
    FOREIGN KEY (rider_id)
        REFERENCES riders (rider_id)
);
SHOW CREATE TABLE orders;
alter table deliveries modify column order_id bigint unsigned;
ALTER TABLE deliveries 
ADD FOREIGN KEY (order_id) REFERENCES orders(order_id);
```

## Data Import
'''
SHOW VARIABLES LIKE 'local_infile';
SET GLOBAL local_infile = 1;

LOAD DATA LOCAL INFILE 'D:\\Projects\\Zomato\\customers.csv'
INTO TABLE customers
FIELDS TERMINATED BY ',' 
ENCLOSED BY '"'
LINES TERMINATED BY '\r\n'
IGNORE 1 ROWS;

LOAD DATA LOCAL INFILE 'D:\\Projects\\Zomato\\restaurants.csv'
INTO TABLE restaurants
FIELDS TERMINATED BY ',' 
ENCLOSED BY '"'
LINES TERMINATED BY '\r\n'
IGNORE 1 ROWS;

LOAD DATA LOCAL INFILE 'D:\\Projects\\Zomato\\orders.csv'
INTO TABLE orders
FIELDS TERMINATED BY ',' 
ENCLOSED BY '"'
LINES TERMINATED BY '\r\n'
IGNORE 1 ROWS;

LOAD DATA LOCAL INFILE 'D:\\Projects\\Zomato\\riders.csv'
INTO TABLE riders
FIELDS TERMINATED BY ',' 
ENCLOSED BY '"'
LINES TERMINATED BY '\r\n'
IGNORE 1 ROWS;

LOAD DATA LOCAL INFILE 'D:\\Projects\\Zomato\\deliveries.csv'
INTO TABLE deliveries
FIELDS TERMINATED BY ',' 
ENCLOSED BY '"'
LINES TERMINATED BY '\r\n'
IGNORE 1 ROWS;
'''


## Checking Null Values

Before performing analysis, I ensured that the data was clean and free from null values where necessary. For instance:

```sql
SELECT 
    *
FROM
    customers
WHERE
    reg_date IS NULL;
SELECT 
    *
FROM
    restaurants
WHERE
    city IS NULL OR opening_hours IS NULL;
SELECT 
    *
FROM
    orders
WHERE
    order_item IS NULL;
SELECT 
    *
FROM
    riders
WHERE
    sign_up IS NULL;
SELECT 
    *
FROM
    deliveries
WHERE
    delivery_time IS NULL;


```

## Business Problems Solved

### 1. Write a query to find the top 5 most frequently ordered dishes by customer called "Vikram Singh" in the last 1 year(2023)

```sql
SELECT
    customer_name,
    order_item AS dish,
    total_orders
FROM (
    SELECT 
        c.customer_name,
        o.order_item,
        COUNT(*) AS total_orders,
        DENSE_RANK() OVER (ORDER BY COUNT(*) DESC) AS ranks
    FROM orders o
    JOIN customers c ON c.customer_id = o.customer_id
    WHERE 
	 order_date < '2024-01-01'
        AND c.customer_name = 'Vikram Singh'
    GROUP BY c.customer_name, o.order_item
) ranked_dishes
WHERE ranks <= 5
ORDER BY total_orders DESC;
```

### 2. Popular Time Slots
-- Question: Identify the time slots during which the most orders are placed. based on 2-hour intervals

```sql
SELECT 
    CONCAT(start_time, ' - ', end_time) AS time_range,
    order_count
FROM
    (SELECT 
        FLOOR(EXTRACT(HOUR FROM order_time) / 2) * 2 AS start_time,
            FLOOR(EXTRACT(HOUR FROM order_time) / 2) * 2 + 2 AS end_time,
            COUNT(*) AS order_count
    FROM
        orders
    GROUP BY 1 , 2
    ORDER BY 1 ASC) AS t1;
```

### 3. Order Value Analysis
-- Question: Find the average order value per customer who has placed more than 750 orders.
-- Return customer_name, and aov(average order value)

```sql
SELECT 
    c.customer_name, AVG(o.total_amount) AS aov
FROM
    orders AS o
        JOIN
    customers AS c ON c.customer_id = o.customer_id
GROUP BY customer_name
HAVING COUNT(order_id) > 750;

```

### 4. High-Value Customers
-- Question: List the customers who have spent more than 100K in total on food orders.
-- return customer_name

```sql
SELECT 
    c.customer_name, SUM(o.total_amount) AS spent
FROM
    orders AS o
        JOIN
    customers AS c ON c.customer_id = o.customer_id
GROUP BY 1
HAVING SUM(o.total_amount) > 100000;

```

### 5. Orders Without Delivery
-- Question: Write a query to find orders that were placed but not delivered. 
-- Return each restuarant name, city and number of not delivered orders 
```sql

SELECT 
    restaurant_name,
    city,
    COUNT(delivery_status) AS not_delivered
FROM
    restaurants r
        JOIN
    orders o ON r.restaurant_id = o.restaurant_id
        JOIN
    deliveries d ON o.order_id = d.order_id
WHERE
    delivery_status = 'Not delivered'
GROUP BY 1 , 2
ORDER BY COUNT(delivery_status) DESC;

```


### 6. Restaurant Revenue Ranking: 
-- Rank restaurants by their total revenue from the last year, including their name, 
-- total revenue, and rank within their city.

```sql
select restaurant_name, city,
rank() over( partition by city order by sum(total_amount)) as city_rnk,
sum(total_amount) as revenue 
from restaurants r join orders o on r.restaurant_id=o.restaurant_id where order_date < '2024-01-01'
group by restaurant_name,city order by city,city_rnk ;

```

### 7. Most Popular Dish by City: 
-- Identify the most popular dish in each city based on the number of orders.
```sql
with rnk_tbl as(
select city,order_item as dish,count(order_id) as ordr_qnt, 
rank () over( partition by city order by count(order_id) desc) as rnk from orders o 
join restaurants r on r.restaurant_id=o.restaurant_id
group by city,order_item )
select city,dish, ordr_qnt from rnk_tbl where rnk=1;

```

### 8. Customer Churn: 
-- Find customers who haven’t placed an order in 2024 but did in 2023.

```sql
SELECT DISTINCT
    customer_name
FROM
    customers c
        JOIN
    orders o ON c.customer_id = o.customer_id
WHERE
    order_date < '2024-01-01'
        AND EXTRACT(YEAR FROM order_date) != '2024';

```

###  9. Cancellation Rate Comparison: 
-- Calculate and compare the order cancellation rate for each restaurant between the  
-- current year and the previous year.

```sql
SELECT 
    cancel_23.restaurant_id,
    cancel_23.total_orders_2023,
    cancel_23.not_delivered_2023,
    ROUND(CASE
                WHEN cancel_23.total_orders_2023 > 0 THEN cancel_23.not_delivered_2023 * 100.0 / cancel_23.total_orders_2023
                ELSE NULL
            END,
            2) AS cancellation_rate_2023,
    cancel_24.total_orders_2024,
    cancel_24.not_delivered_2024,
    ROUND(CASE
                WHEN cancel_24.total_orders_2024 > 0 THEN cancel_24.not_delivered_2024 * 100.0 / cancel_24.total_orders_2024
                ELSE NULL
            END,
            2) AS cancellation_rate_2024
FROM
    (SELECT 
        r.restaurant_id,
            COUNT(o.order_id) AS total_orders_2023,
            COUNT(CASE
                WHEN o.order_status != 'Completed' THEN 1
            END) AS not_delivered_2023
    FROM
        restaurants r
    JOIN orders o ON r.restaurant_id = o.restaurant_id
    WHERE
        EXTRACT(YEAR FROM o.order_date) = 2023
    GROUP BY r.restaurant_id) AS cancel_23
        LEFT JOIN
    (SELECT 
        r.restaurant_id,
            COUNT(o.order_id) AS total_orders_2024,
            COUNT(CASE
                WHEN o.order_status != 'Completed' THEN 1
            END) AS not_delivered_2024
    FROM
        restaurants r
    JOIN orders o ON r.restaurant_id = o.restaurant_id
    WHERE
        EXTRACT(YEAR FROM o.order_date) = 2024
    GROUP BY r.restaurant_id) AS cancel_24 ON cancel_23.restaurant_id = cancel_24.restaurant_id 
UNION SELECT 
    cancel_24.restaurant_id,
    cancel_23.total_orders_2023,
    cancel_23.not_delivered_2023,
    ROUND(CASE
                WHEN cancel_23.total_orders_2023 > 0 THEN cancel_23.not_delivered_2023 * 100.0 / cancel_23.total_orders_2023
                ELSE NULL
            END,
            2) AS cancellation_rate_2023,
    cancel_24.total_orders_2024,
    cancel_24.not_delivered_2024,
    ROUND(CASE
                WHEN cancel_24.total_orders_2024 > 0 THEN cancel_24.not_delivered_2024 * 100.0 / cancel_24.total_orders_2024
                ELSE NULL
            END,
            2) AS cancellation_rate_2024
FROM
    (SELECT 
        r.restaurant_id,
            COUNT(o.order_id) AS total_orders_2023,
            COUNT(CASE
                WHEN o.order_status != 'Completed' THEN 1
            END) AS not_delivered_2023
    FROM
        restaurants r
    JOIN orders o ON r.restaurant_id = o.restaurant_id
    WHERE
        EXTRACT(YEAR FROM o.order_date) = 2023
    GROUP BY r.restaurant_id) AS cancel_23
        RIGHT JOIN
    (SELECT 
        r.restaurant_id,
            COUNT(o.order_id) AS total_orders_2024,
            COUNT(CASE
                WHEN o.order_status != 'Completed' THEN 1
            END) AS not_delivered_2024
    FROM
        restaurants r
    JOIN orders o ON r.restaurant_id = o.restaurant_id
    WHERE
        EXTRACT(YEAR FROM o.order_date) = 2024
    GROUP BY r.restaurant_id) AS cancel_24 ON cancel_23.restaurant_id = cancel_24.restaurant_id
WHERE
    cancel_23.restaurant_id IS NULL
ORDER BY restaurant_id;
```

###  10. Rider Average Delivery Time: 
-- Determine each rider's average delivery time.

```sql
SELECT 
    r.rider_name,
    ROUND(AVG(CASE
                WHEN
                    d.delivery_time < o.order_time
                THEN
                    TIMESTAMPDIFF(SECOND,
                        o.order_time,
                        d.delivery_time + INTERVAL 1 DAY)
                ELSE TIMESTAMPDIFF(SECOND,
                    o.order_time,
                    d.delivery_time)
            END) / 60,
            2) AS avg_delivery_time_minutes
FROM
    orders o
        JOIN
    deliveries d ON o.order_id = d.order_id
        JOIN
    riders r ON d.rider_id = r.rider_id
GROUP BY r.rider_name
ORDER BY avg_delivery_time_minutes;

```

### 11. Monthly Restaurant Growth Ratio: 
-- Calculate each restaurant's growth ratio based on the total number of delivered orders since its joining

```sql
SELECT 
    restaurant_name,cr_month_order,pre_month_order,month,year,
    ROUND(((cr_month_order - pre_month_order) / pre_month_order) * 100, 2) AS growth_ratio
FROM (
    SELECT 
        restaurant_id,
        restaurant_name,
        year,
        month,
        cr_month_order,
        LAG(cr_month_order, 1) OVER (
            PARTITION BY restaurant_id 
            ORDER BY year, month
        ) AS pre_month_order
    FROM (
        SELECT 
            r.restaurant_id,
            r.restaurant_name,
            YEAR(o.order_date) AS year,
            MONTH(o.order_date) AS month,
            COUNT(o.order_id) AS cr_month_order
        FROM restaurants r
        JOIN orders o ON r.restaurant_id = o.restaurant_id
        JOIN deliveries d ON o.order_id = d.order_id 
        WHERE d.delivery_status = 'Delivered'
        GROUP BY r.restaurant_id, r.restaurant_name, YEAR(o.order_date), MONTH(o.order_date)
    ) AS monthly_orders
) AS t1
WHERE pre_month_order IS NOT NULL;

```

### 12. Customer Segmentation: 
-- Customer Segmentation: Segment customers into 'Gold' or 'Silver' groups based on their total spending 
-- compared to the average order value (AOV). If a customer's total spending exceeds the AOV, 
-- label them as 'Gold'; otherwise, label them as 'Silver'. Write an SQL query to determine each segment's 
-- total number of orders and total revenue

```sql

with aov_cte as
 (select avg(total_amount) as
 aov from orders) select segment,
 count(*) as customer_count,sum(order_count) 
 as total_orders ,sum(spent) as total_revenue 
 from (select customer_name,sum(o.total_amount)
 as spent, count(o.order_id) as order_count,
 case when sum(o.total_amount)> (select aov from aov_cte)then 
 'Gold' else 'Silver' 
end as segment from orders o join customers c
 on c.customer_id=o.customer_id group by customer_name)
 as customer_segments group by segment;

```

### 13. Rider Monthly Earnings: 
-- Calculate each rider's total monthly earnings, assuming they earn 8% of the order amount.

```sql
SELECT 
	d.rider_id,
	TO_CHAR(o.order_date, 'mm-yy') as month,
	SUM(total_amount) as revenue,
	SUM(total_amount)* 0.08 as riders_earning
FROM orders as o
JOIN deliveries as d
ON o.order_id = d.order_id
GROUP BY 1, 2
ORDER BY 1, 2;SELECT 
    rider_name,
    YEAR(order_date) AS year,
    MONTH(order_date) AS month,
    SUM(total_amount) * 0.08 AS riders_earning,
    SUM(total_amount) AS total_revenue,
    COUNT(d.order_id) AS order_count
FROM
    orders o
        JOIN
    deliveries d ON o.order_id = d.order_id
        JOIN
    riders r ON d.rider_id = r.rider_id
GROUP BY rider_name , year , month
ORDER BY rider_name , year , month;

```

### Q.14 Rider Ratings Analysis: 
-- Find the number of 5-star, 4-star, and 3-star ratings each rider has.
-- riders receive this rating based on delivery time.
-- If orders are delivered less than 15 minutes of order received time the rider get 5 star rating,
-- if they deliver 15 and 20 minute they get 4 star rating 
-- if they deliver after 20 minute they get 3 star rating.

```sql
SELECT 
    r.rider_name,
    SUM(CASE
        WHEN
            TIMESTAMPDIFF(MINUTE,
                o.order_time,
                d.delivery_time) < 15
        THEN
            1
        ELSE 0
    END) AS five_star_count,
    SUM(CASE
        WHEN
            TIMESTAMPDIFF(MINUTE,
                o.order_time,
                d.delivery_time) BETWEEN 15 AND 20
        THEN
            1
        ELSE 0
    END) AS four_star_count,
    SUM(CASE
        WHEN
            TIMESTAMPDIFF(MINUTE,
                o.order_time,
                d.delivery_time) > 20
        THEN
            1
        ELSE 0
    END) AS three_star_count
FROM
    orders o
        JOIN
    deliveries d ON o.order_id = d.order_id
        JOIN
    riders r ON d.rider_id = r.rider_id
WHERE
    d.delivery_status = 'delivered'
GROUP BY r.rider_name
ORDER BY r.rider_name;


```

### 15. Q.15 Order Frequency by Day: 
-- Analyze order frequency per day of the week and identify the peak day for each restaurant.

```sql
WITH t1 AS (
    SELECT 
        r.restaurant_name, 
        DAYNAME(o.order_date) AS day_of_week,
        COUNT(o.order_id) AS order_count
    FROM 
        orders o 
    JOIN restaurants r ON o.restaurant_id = r.restaurant_id 
    GROUP BY 
        r.restaurant_name, DAYNAME(o.order_date)
)
SELECT 
    restaurant_name,
    day_of_week,
    order_count
FROM (
    SELECT 
        restaurant_name,
        day_of_week,
        order_count,
        RANK() OVER (PARTITION BY restaurant_name ORDER BY order_count DESC) AS day_rank
    FROM t1
) ranked_days
WHERE day_rank = 1
ORDER BY restaurant_name;

```

### 16. Customer Lifetime Value (CLV): 
-- Calculate the total revenue generated by each customer over all their orders.

```sql
SELECT 
    customer_name,
    SUM(total_amount) AS total_revenue,
    COUNT(order_id) AS order_count
FROM
    orders o
        RIGHT JOIN
    customers c ON o.customer_id = c.customer_id
GROUP BY customer_name
ORDER BY total_revenue DESC;
```

### 17. Monthly Sales Trends: 
-- Identify sales trends by comparing each month's total sales to the previous month.

```sql
with monthly_sale as 
(SELECT 
	YEAR(order_date) as year,
	MONTH (order_date)as month,
	SUM(total_amount) as total_sale,
	LAG(SUM(total_amount)) OVER(ORDER BY YEAR(order_date),
    MONTH(order_date)) as prev_month_sale
FROM orders
GROUP BY year,month) 
select year,
month, 
total_sale,
prev_month_sale,
(prev_month_sale-total_sale) as sale_diff 
from monthly_sale;

```

### 18. Rider Efficiency: 
-- Evaluate rider efficiency by determining average delivery times and identifying those with the lowest and highest averages.

```sql
WITH new_table AS (
    SELECT 
        o.order_id,
        o.order_date,
        o.order_time,
        d.delivery_time,
        d.delivery_status,
        d.rider_id AS riders_id,
        ROUND(
            CASE 
                WHEN delivery_time < order_time THEN 
                    TIMESTAMPDIFF(MINUTE, order_time, delivery_time) + 1440
                ELSE 
                    TIMESTAMPDIFF(MINUTE, order_time, delivery_time)
            END
        ) AS time_deliver
    FROM orders AS o
    JOIN deliveries AS d ON o.order_id = d.order_id
    WHERE d.delivery_status = 'Delivered'
),

riders_time AS (
    SELECT 
        riders_id,
        AVG(time_deliver) AS avg_time
    FROM new_table
    GROUP BY riders_id
)
SELECT 
    MIN(avg_time) AS min_avg_delivery_time,
    MAX(avg_time) AS max_avg_delivery_time
FROM riders_time;

```

### 19. Order Item Popularity: 
-- Track the popularity of specific order items over time and identify seasonal demand spikes.

```sql
SELECT 
    (CASE
        WHEN MONTH(order_date) BETWEEN 3 AND 5 THEN 'Summer'
        WHEN MONTH(order_date) BETWEEN 6 AND 9 THEN 'Rainy'
        ELSE 'Winter'
    END) AS season,
    order_item,
    COUNT(order_id) AS order_count
FROM
    orders
GROUP BY season , order_item
ORDER BY season , order_count DESC;
```

### 20. Rank each city based on the total revenue for last year 2023
```sql
WITH city_revenue AS (
    SELECT 
        r.city,
        SUM(o.total_amount) AS total_city_revenue
    FROM restaurants r
    JOIN orders o ON r.restaurant_id = o.restaurant_id 
    where year(order_date) ='2023'
    GROUP BY r.city 
)
SELECT 
    city,
    total_city_revenue,
    RANK() OVER (ORDER BY total_city_revenue DESC) AS revenue_rank
FROM city_revenue;

```

## Conclusion

This project highlights my ability to handle complex SQL queries and provides solutions to real-world business problems in the context of a food delivery service like Zomato. The approach taken here demonstrates a structured problem-solving methodology, data manipulation skills, and the ability to derive actionable insights from data.This project demonstrates my ability to analyze structured business data using SQL, focusing on a pizza delivery service. Through a series of practical queries, I’ve explored key performance metrics such as total revenue, best-selling items, sales trends by time and category, and customer ordering behavior. The project reflects a strong foundation in data modeling, querying normalized databases, and delivering insights that could help drive real-world decisions in the food and beverage industry.
