# Case Study #2 - Pizza Runner
<img src="https://8weeksqlchallenge.com/images/case-study-designs/2.png" width=50% height=50%>

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

