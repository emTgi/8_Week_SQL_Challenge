# Case Study #2 - Pizza Runner
<img src="https://8weeksqlchallenge.com/images/case-study-designs/2.png" width=50% height=50%>

## Table of Contents
1. [Task](#task)
2. [Data Model](#data-model)
3. [Data Cleaning](#data-cleaning)
4. [Data Analysis](#data-analysis)
    - [Pizza Metrics](#pizza-metric)
    - [Runner and Customer Experience](#runner_and_customer_experience)

## Task
Danny has opened a new business called "Pizza Runners", which specialises in delivering fresh pizza through "runners". Danny understands that data is crucial for business growth so he already set up a database, but it needs some cleaning. Also, he wants to optimise his business operations by analysing current data.

## Data Model

![Week_2_Diagram](https://github.com/emTgi/8_Week_SQL_Challenge/assets/114177110/bb161e2d-9e16-4fb2-9c69-273cb3858f28)

## Data Cleaning

There are two tables that need cleaning. The first one is `customers_orders`, which currently looks like this
```sql
SET search_path = pizza_runner;

SELECT column_name, data_type
FROM information_schema.columns
WHERE table_name = 'customer_orders';

SELECT * FROM customer_orders;
```

| column_name | data_type                   |
| ----------- | --------------------------- |
| order_id    | integer                     |
| customer_id | integer                     |
| pizza_id    | integer                     |
| order_time  | timestamp without time zone |
| exclusions  | character varying           |
| extras      | character varying           |

| order_id | customer_id | pizza_id | exclusions | extras | order_time               |
| -------- | ----------- | -------- | ---------- | ------ | ------------------------ |
| 1        | 101         | 1        |            |        | 2020-01-01T18:05:02.000Z |
| 2        | 101         | 1        |            |        | 2020-01-01T19:00:52.000Z |
| 3        | 102         | 1        |            |        | 2020-01-02T23:51:23.000Z |
| 3        | 102         | 2        |            |        | 2020-01-02T23:51:23.000Z |
| 4        | 103         | 1        | 4          |        | 2020-01-04T13:23:46.000Z |
| 4        | 103         | 1        | 4          |        | 2020-01-04T13:23:46.000Z |
| 4        | 103         | 2        | 4          |        | 2020-01-04T13:23:46.000Z |
| 5        | 104         | 1        | null       | 1      | 2020-01-08T21:00:29.000Z |
| 6        | 101         | 2        | null       | null   | 2020-01-08T21:03:13.000Z |
| 7        | 105         | 2        | null       | 1      | 2020-01-08T21:20:29.000Z |
| 8        | 102         | 1        | null       | null   | 2020-01-09T23:54:33.000Z |
| 9        | 103         | 1        | 4          | 1, 5   | 2020-01-10T11:22:59.000Z |
| 10       | 104         | 1        | null       | null   | 2020-01-11T18:34:49.000Z |
| 10       | 104         | 1        | 2, 6       | 1, 4   | 2020-01-11T18:34:49.000Z |

As the table above shows, `exclusions` and `extras` columns have some null values or blank spaces. We need to make them consistent, so for this project, we will stick with blank spaces. Additionally, there is no need to change the data types of the columns.
We will create a temporary table for the purpose of cleaning data as we do not want to corrupt the original data if something goes wrong.
```sql
CREATE TEMPORARY TABLE clean_customer_orders AS
  SELECT
    order_id,
    customer_id,
    pizza_id,
    CASE
      WHEN exclusions IS NULL OR exclusions LIKE '%null%' THEN '' -- Some rows had no value at all, while others had a 'null' value, so we need to change both.
      ELSE exclusions
    END AS exclusions,
    CASE
      WHEN extras IS NULL or extras LIKE '%null%' THEN ''
      ELSE extras
    END AS extras,
    order_time
FROM customer_orders;
```
The `customer_orders` table has been successfully cleaned as you can see below:

| order_id | customer_id | pizza_id | exclusions | extras | order_time               |
| -------- | ----------- | -------- | ---------- | ------ | ------------------------ |
| 1        | 101         | 1        |            |        | 2020-01-01T18:05:02.000Z |
| 2        | 101         | 1        |            |        | 2020-01-01T19:00:52.000Z |
| 3        | 102         | 1        |            |        | 2020-01-02T23:51:23.000Z |
| 3        | 102         | 2        |            |        | 2020-01-02T23:51:23.000Z |
| 4        | 103         | 1        | 4          |        | 2020-01-04T13:23:46.000Z |
| 4        | 103         | 1        | 4          |        | 2020-01-04T13:23:46.000Z |
| 4        | 103         | 2        | 4          |        | 2020-01-04T13:23:46.000Z |
| 5        | 104         | 1        |            | 1      | 2020-01-08T21:00:29.000Z |
| 6        | 101         | 2        |            |        | 2020-01-08T21:03:13.000Z |
| 7        | 105         | 2        |            | 1      | 2020-01-08T21:20:29.000Z |
| 8        | 102         | 1        |            |        | 2020-01-09T23:54:33.000Z |
| 9        | 103         | 1        | 4          | 1, 5   | 2020-01-10T11:22:59.000Z |
| 10       | 104         | 1        |            |        | 2020-01-11T18:34:49.000Z |
| 10       | 104         | 1        | 2, 6       | 1, 4   | 2020-01-11T18:34:49.000Z |

The second table `runner_orders` currently looks like this:
```sql
SELECT column_name, data_type
FROM information_schema.columns
WHERE table_name = 'runner_orders';

SELECT * FROM runner_orders;
```

| column_name  | data_type         |
| ------------ | ----------------- |
| order_id     | integer           |
| runner_id    | integer           |
| pickup_time  | character varying |
| distance     | character varying |
| duration     | character varying |
| cancellation | character varying |

| order_id | runner_id | pickup_time         | distance | duration   | cancellation            |
| -------- | --------- | ------------------- | -------- | ---------- | ----------------------- |
| 1        | 1         | 2020-01-01 18:15:34 | 20km     | 32 minutes |                         |
| 2        | 1         | 2020-01-01 19:10:54 | 20km     | 27 minutes |                         |
| 3        | 1         | 2020-01-03 00:12:37 | 13.4km   | 20 mins    |                         |
| 4        | 2         | 2020-01-04 13:53:03 | 23.4     | 40         |                         |
| 5        | 3         | 2020-01-08 21:10:57 | 10       | 15         |                         |
| 6        | 3         | null                | null     | null       | Restaurant Cancellation |
| 7        | 2         | 2020-01-08 21:30:45 | 25km     | 25mins     | null                    |
| 8        | 2         | 2020-01-10 00:15:02 | 23.4 km  | 15 minute  | null                    |
| 9        | 2         | null                | null     | null       | Customer Cancellation   |
| 10       | 1         | 2020-01-11 18:50:20 | 10km     | 10minutes  | null                    |

As seen above, both `distance` and `duration` columns have inconsistent value formats, so we will be removing everything except for numbers and changing the data type to a numeric one. Also, from `pickup_time` and `cancellation` columns we will remove 'null' values and change the data type of `pickup_time` to a date/time type.
```sql
CREATE TEMPORARY TABLE clean_runner_orders AS
  SELECT
    order_id,
    runner_id,
    CAST(CASE
      WHEN pickup_time LIKE '%null%' THEN NULL --TIMESTAMP cannot store blank spaces
      ELSE pickup_time
    END AS TIMESTAMP) pickup_time,
    CAST(CASE
      WHEN distance LIKE '%null%' THEN NULL --FLOAT cannot store blank spaces
      ELSE TRIM(TRIM('km' FROM distance))
    END AS FLOAT) distance,
    CAST(CASE
      WHEN duration LIKE '%null%' THEN NULL --INT cannot store blank spaces
      ELSE TRIM(TRIM('minutes' FROM duration))
    END AS INTEGER) duration,
    CASE
      WHEN cancellation IS NULL OR cancellation LIKE '%null%' THEN ''
      ELSE cancellation
    END AS cancellation
  FROM runner_orders;
```
The `runner_orders` table has been successfully cleaned and data types have been changed as you can see below:

| column_name  | data_type                   |
| ------------ | --------------------------- |
| order_id     | integer                     |
| runner_id    | integer                     |
| pickup_time  | timestamp without time zone |
| distance     | double precision            |
| duration     | integer                     |
| cancellation | character varying           |

| order_id | runner_id | pickup_time              | distance | duration | cancellation            |
| -------- | --------- | ------------------------ | -------- | -------- | ----------------------- |
| 1        | 1         | 2020-01-01T18:15:34.000Z | 20       | 32       |                         |
| 2        | 1         | 2020-01-01T19:10:54.000Z | 20       | 27       |                         |
| 3        | 1         | 2020-01-03T00:12:37.000Z | 13.4     | 20       |                         |
| 4        | 2         | 2020-01-04T13:53:03.000Z | 23.4     | 40       |                         |
| 5        | 3         | 2020-01-08T21:10:57.000Z | 10       | 15       |                         |
| 6        | 3         |                          |          |          | Restaurant Cancellation |
| 7        | 2         | 2020-01-08T21:30:45.000Z | 25       | 25       |                         |
| 8        | 2         | 2020-01-10T00:15:02.000Z | 23.4     | 15       |                         |
| 9        | 2         |                          |          |          | Customer Cancellation   |
| 10       | 1         | 2020-01-11T18:50:20.000Z | 10       | 10       |                         |

## Data Analysis

### Pizza Metrics

**How many pizzas were ordered?**
```sql
SELECT COUNT(*)
FROM clean_customer_orders;
```
| count |
| ----- |
| 14    |

**How many unique customer orders were made?**
```sql
SELECT COUNT(DISTINCT order_id)
FROM clean_customer_orders;
```
| count |
| ----- |
| 10    |

**How many successful orders were delivered by each runner?**
```sql
SELECT 
  runner_id,
  COUNT(*) AS orders_delivered
FROM clean_runner_orders
WHERE pickup_time IS NOT NULL
GROUP BY runner_id;
```
| runner_id | orders_delivered |
| --------- | ---------------- |
| 3         | 1                |
| 2         | 3                |
| 1         | 4                |

**How many of each type of pizza was delivered?**
```sql
SELECT
  pizza_name,
  COUNT(*) AS orders_delivered
FROM clean_customer_orders a
JOIN clean_runner_orders b
  ON a.order_id = b.order_id
JOIN pizza_names c
  ON a.pizza_id = c.pizza_id
WHERE pickup_time IS NOT NULL
GROUP BY pizza_name;
```
| pizza_name | orders_delivered |
| ---------- | ---------------- |
| Meatlovers | 9                |
| Vegetarian | 3                |

**How many Vegetarian and Meatlovers were ordered by each customer?**
```sql
SELECT
  customer_id,
  pizza_name,
  COUNT(*)
FROM clean_customer_orders a
JOIN pizza_names b
  ON a.pizza_id = b.pizza_id
GROUP BY 
  customer_id, 
  pizza_name
ORDER BY customer_id;
```
| customer_id | pizza_name | count |
| ----------- | ---------- | ----- |
| 101         | Meatlovers | 2     |
| 101         | Vegetarian | 1     |
| 102         | Meatlovers | 2     |
| 102         | Vegetarian | 1     |
| 103         | Meatlovers | 3     |
| 103         | Vegetarian | 1     |
| 104         | Meatlovers | 3     |
| 105         | Vegetarian | 1     |

**What was the maximum number of pizzas delivered in a single order?**
```sql
SELECT
  a.order_id,
  COUNT(pizza_id) AS orders_delivered
FROM clean_customer_orders a
JOIN clean_runner_orders b
  ON a.order_id = b.order_id
WHERE pickup_time IS NOT NULL
GROUP BY a.order_id
ORDER BY orders_delivered DESC
LIMIT 1;
```
| order_id | orders_delivered |
| -------- | ---------------- |
| 4        | 3                |

**For each customer, how many delivered pizzas had at least 1 change and how many had no changes?**
```sql
SELECT
  customer_id,
  SUM(
  CASE
    WHEN exclusions <> '' OR extras <> '' THEN 1
    ELSE 0
  END) customised_orders,
  SUM(
  CASE
    WHEN exclusions = '' AND extras = '' THEN 1
    ELSE 0
  END) uncustomised_orders
FROM clean_customer_orders a
JOIN clean_runner_orders b
  ON a.order_id = b.order_id
WHERE pickup_time IS NOT NULL
GROUP BY customer_id
ORDER BY customer_id;
```
| customer_id | customised_orders | uncustomised_orders |
| ----------- | ----------------- | ------------------- |
| 101         | 0                 | 2                   |
| 102         | 0                 | 3                   |
| 103         | 3                 | 0                   |
| 104         | 2                 | 1                   |
| 105         | 1                 | 0                   |

**How many pizzas were delivered that had both exclusions and extras?**
```sql
SELECT
  SUM(
  CASE
    WHEN exclusions <> '' AND extras <> '' THEN 1
    ELSE 0
  END)
FROM clean_customer_orders a
JOIN clean_runner_orders b
  ON a.order_id = b.order_id
WHERE pickup_time IS NOT NULL;
```
| sum |
| --- |
| 1   |

**What was the total volume of pizzas ordered for each hour of the day?**
```sql
SELECT
  EXTRACT(HOUR FROM order_time) AS hour,
  COUNT(pizza_id)    
FROM clean_customer_orders
GROUP BY hour
ORDER BY hour;
```
| hour | count |
| ---- | ----- |
| 11   | 1     |
| 13   | 3     |
| 18   | 3     |
| 19   | 1     |
| 21   | 3     |
| 23   | 3     |

**What was the volume of orders for each day of the week?**
```sql
SELECT
  TO_CHAR(order_time, 'Day') AS day,
  COUNT(DISTINCT order_id)
FROM clean_customer_orders
GROUP BY day;
```
| day       | count |
| --------- | ----- |
| Friday    | 1     |
| Saturday  | 2     |
| Thursday  | 2     |
| Wednesday | 5     |

### Runner and Customer Experience

**How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)**
```sql
SELECT
  DATE_PART('week', registration_date + interval '3 day') AS week, --3 days have been added for the week to start on the date specified
  COUNT(*)
FROM runners
GROUP BY week
ORDER BY week;
```
| week | count |
| ---- | ----- |
| 1    | 2     |
| 2    | 1     |
| 3    | 1     |

**What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?**
```sql
SELECT
  runner_id,
  AVG(time)
FROM (
  SELECT
    a.order_id,
    runner_id,
    EXTRACT(MINUTE FROM pickup_time - order_time) AS time
  FROM clean_runner_orders a
  JOIN clean_customer_orders b
    ON a.order_id = b.order_id
  WHERE pickup_time IS NOT NULL
  GROUP BY a.order_id, runner_id, time) sub
GROUP BY runner_id
ORDER BY runner_id;
```
| runner_id | avg                |
| --------- | ------------------ |
| 1         | 14                 |
| 2         | 19.666666666666668 |
| 3         | 10                 |

**Is there any relationship between the number of pizzas and how long the order takes to prepare?**
```sql
SELECT corr(pizza_volume, time) AS corr_coef
FROM (
  SELECT
    a.order_id,
    COUNT(b.order_id) pizza_volume,
    EXTRACT(MINUTE FROM pickup_time - order_time) AS time
  FROM clean_runner_orders a
  JOIN clean_customer_orders b
    ON a.order_id = b.order_id
  WHERE pickup_time IS NOT NULL
  GROUP BY a.order_id, time) sub;
```
| corr_coef          |
| ------------------ |
| 0.8372508882083807 |
The result shows a strong positive correlation, so when the number of pizza increases the time to prepare the orders increases as well.

**What was the average distance travelled for each customer?**
```sql
SELECT
  customer_id,
  AVG(distance)
FROM clean_runner_orders a
JOIN clean_customer_orders b
  ON a.order_id = b.order_id
WHERE pickup_time IS NOT NULL
GROUP BY customer_id
ORDER BY customer_id;
```
| customer_id | avg                |
| ----------- | ------------------ |
| 101         | 20                 |
| 102         | 16.733333333333334 |
| 103         | 23.399999999999995 |
| 104         | 10                 |
| 105         | 25                 |

**What was the difference between the longest and shortest delivery times for all orders?**
```sql
SELECT MAX(duration) - MIN(duration) AS delivery_diff
FROM clean_runner_orders;
```
| delivery_diff |
| ------------- |
| 30            |

**What was the average speed for each runner for each delivery and do you notice any trend for these values?**
```sql
SELECT
  a.order_id,
  runner_id,
  (distance*60/duration) AS avg_speed
FROM clean_runner_orders a
JOIN clean_customer_orders b
  ON a.order_id = b.order_id
WHERE pickup_time IS NOT NULL
GROUP BY a.order_id, runner_id, avg_speed
ORDER BY a.order_id;
```
| order_id | runner_id | avg_speed         |
| -------- | --------- | ----------------- |
| 1        | 1         | 37.5              |
| 2        | 1         | 44.44444444444444 |
| 3        | 1         | 40.2              |
| 4        | 2         | 35.1              |
| 5        | 3         | 40                |
| 7        | 2         | 60                |
| 8        | 2         | 93.6              |
| 10       | 1         | 60                |

**What is the successful delivery percentage for each runner?**
```sql
SELECT 
  runner_id,
  ROUND(100*COUNT(pickup_time)/COUNT(*)) AS successful_orders_perc
FROM clean_runner_orders
GROUP BY runner_id
ORDER BY runner_id;
```
| runner_id | successful_orders_perc |
| --------- | ---------------------- |
| 1         | 100                    |
| 2         | 75                     |
| 3         | 50                     |
