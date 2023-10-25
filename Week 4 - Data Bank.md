# Case Study #4 - Data Bank
<img src="https://8weeksqlchallenge.com/images/case-study-designs/4.png" width=50% height=50%>

## Table of Contents
1. [Task](#task)
2. [Data Model](#data-model)
3. [Data Analysis](#data-analysis)
    - [Customer Nodes Exploration](#customer-nodes-exploration)
    - [Customer Transactions](#customer-transactions)

## Task
The task is to launch Data Bank, a digital bank with secure data storage, seeking to grow its customer base and analyze storage needs for future development.

## Data Model

<img src="https://8weeksqlchallenge.com/images/case-study-4-erd.png">

## Data Analysis

### Customer Nodes Exploration

**How many unique nodes are there on the Data Bank system?**
```sql
SELECT COUNT(DISTINCT node_id)
FROM customer_nodes;
```
| count |
| ----- |
| 5     |

**What is the number of nodes per region?**
```sql
SELECT 
  region_name, 
  COUNT(DISTINCT node_id)
FROM customer_nodes a
JOIN regions b
  ON a.region_id = b.region_id
GROUP BY region_name;
```
| region_name | count |
| ----------- | ----- |
| Africa      | 5     |
| America     | 5     |
| Asia        | 5     |
| Australia   | 5     |
| Europe      | 5     |

**How many customers are allocated to each region?**
```sql
SELECT 
  region_name, 
  COUNT(DISTINCT customer_id)
FROM customer_nodes a
JOIN regions b
  ON a.region_id = b.region_id
GROUP BY region_name;
```
| region_name | count |
| ----------- | ----- |
| Africa      | 102   |
| America     | 105   |
| Asia        | 95    |
| Australia   | 110   |
| Europe      | 88    |

**How many days on average are customers reallocated to a different node?**
```sql
WITH node_days AS (
  SELECT *,
    end_date - start_date AS days_in_node	
  FROM customer_nodes
  WHERE EXTRACT(YEAR FROM end_date) != 9999
)
    
SELECT ROUND(AVG(days_in_node), 2)
FROM node_days;
```
| round |
| ----- |
| 14.63 |

**What is the median, 80th and 95th percentile for this same reallocation days metric for each region?**
```sql
WITH node_days AS (
  SELECT *,
    end_date - start_date AS days_in_node	
  FROM customer_nodes
  WHERE EXTRACT(YEAR FROM end_date) != 9999
)
    
SELECT
  region_name,
  PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY days_in_node),
  PERCENTILE_DISC(0.8) WITHIN GROUP (ORDER BY days_in_node),
  PERCENTILE_DISC(0.95) WITHIN GROUP (ORDER BY days_in_node)
FROM node_days a
JOIN regions b
  ON a.region_id = b.region_id
GROUP BY region_name;
```
| region_name | percentile_disc | percentile_disc | percentile_disc |
| ----------- | --------------- | --------------- | --------------- |
| Africa      | 15              | 24              | 28              |
| America     | 15              | 23              | 28              |
| Asia        | 15              | 23              | 28              |
| Australia   | 15              | 23              | 28              |
| Europe      | 15              | 24              | 28              |

### Customer Transactions

**What is the unique count and total amount for each transaction type?**
```sql
SELECT
  txn_type,
  COUNT(customer_id),
  SUM(txn_amount)
FROM customer_transactions
GROUP BY txn_type;
```
| txn_type   | count | sum     |
| ---------- | ----- | ------- |
| purchase   | 1617  | 806537  |
| deposit    | 2671  | 1359168 |
| withdrawal | 1580  | 793003  |

**What is the average total historical deposit counts and amounts for all customers?**
```sql
WITH customer_deposits AS (
  SELECT
    customer_id,
    COUNT(customer_id) AS transactions,
    AVG(txn_amount) AS average_amount
  FROM customer_transactions
  WHERE txn_type = 'deposit'
  GROUP BY customer_id
)
    
SELECT
  ROUND(AVG(transactions), 2),
  ROUND(AVG(average_amount), 2)
FROM customer_deposits;
```
| round | round  |
| ----- | ------ |
| 5.34  | 508.61 |

**For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?**
```sql
WITH customers_cte AS (
  SELECT 
    EXTRACT(MONTH FROM txn_date) AS month,
    customer_id,
    SUM(CASE WHEN txn_type = 'deposit' THEN 1 ELSE 0 END) AS deposits,
    SUM(CASE WHEN txn_type = 'purchase' THEN 1 ELSE 0 END) AS purchases,
    SUM(CASE WHEN txn_type = 'withdrawal' THEN 1 ELSE 0 END) AS withdrawals
  FROM customer_transactions
  GROUP BY month, customer_id
)
    
SELECT
  month,
  COUNT(customer_id)
FROM customers_cte
WHERE deposits > 1
AND (purchases = 1 OR withdrawals = 1)
GROUP BY month;
```
| month | count |
| ----- | ----- |
| 1     | 115   |
| 2     | 108   |
| 3     | 113   |
| 4     | 50    |

**What is the closing balance for each customer at the end of the month?**
```sql
WITH monthly_flow AS (
  SELECT
    customer_id,
    DATE_TRUNC('month', txn_date) + interval '1 month' - interval '1 day' AS months,
    SUM(
      CASE WHEN txn_type = 'deposit' then txn_amount
      ELSE -txn_amount
      END) as amount
  FROM customer_transactions
  GROUP BY customer_id, months
),
    
month_last_day AS (
  SELECT
    DISTINCT customer_id,
    DATE '2020-01-31' + (interval '1 month' * generate_series(0,3)) AS end_of_month
  FROM customer_transactions
)
    
SELECT
  a.customer_id,
  end_of_month,
  SUM(amount) OVER(PARTITION BY a.customer_id ORDER BY end_of_month) AS balance
FROM month_last_day a
LEFT JOIN monthly_flow b
  ON a.customer_id = b.customer_id
  AND a.end_of_month = b.months
ORDER BY a.customer_id, end_of_month
LIMIT 10; -- Limited to 10 as there are over 1000 rows
```
| customer_id | end_of_month             | balance |
| ----------- | ------------------------ | ------- |
| 1           | 2020-01-31T00:00:00.000Z | 312     |
| 1           | 2020-02-29T00:00:00.000Z | 312     |
| 1           | 2020-03-31T00:00:00.000Z | -640    |
| 1           | 2020-04-30T00:00:00.000Z | -640    |
| 2           | 2020-01-31T00:00:00.000Z | 549     |
| 2           | 2020-02-29T00:00:00.000Z | 549     |
| 2           | 2020-03-31T00:00:00.000Z | 610     |
| 2           | 2020-04-30T00:00:00.000Z | 610     |
| 3           | 2020-01-31T00:00:00.000Z | 144     |
| 3           | 2020-02-29T00:00:00.000Z | -821    |
