# Case Study #3 - Foodie-Fi
<img src="https://8weeksqlchallenge.com/images/case-study-designs/3.png" width=50% height=50%>

## Table of Contents
1. [Task](#task)
2. [Data Model](#data-model)
3. [Data Analysis](#data-analysis)
    - [Basic Questions](#basic-questions)
    - [Challenge Payment Question](#challenge-payment-question)
    - [Outside The Box Questions](#outside-the-box-questions)

## Task
The task is to analyze subscription-based data for Foodie-Fi, a food-related streaming service, to inform business decisions and enhance the platform.

## Data Model

![image](https://github.com/emTgi/8_Week_SQL_Challenge/assets/114177110/bf6875db-cc24-4b53-9424-9e2ae2650994)

## Data Analysis

### Basic Questions

**How many customers has Foodie-Fi ever had?**
```sql
SELECT COUNT(DISTINCT customer_id)
FROM subscriptions;
```
| count |
| ----- |
| 1000  |

**What is the monthly distribution of trial plan start_date values for our dataset - use the start of the month as the group by value**
```sql
SELECT
  DATE_PART('month', start_date) AS month,
  COUNT(DISTINCT customer_id)
FROM subscriptions
WHERE plan_id = 0
GROUP BY month;
```
| month | count |
| ----- | ----- |
| 1     | 88    |
| 2     | 68    |
| 3     | 94    |
| 4     | 81    |
| 5     | 88    |
| 6     | 79    |
| 7     | 89    |
| 8     | 88    |
| 9     | 87    |
| 10    | 79    |
| 11    | 75    |
| 12    | 84    |

**What plan start_date values occur after the year 2020 for our dataset? Show the breakdown by count of events for each plan_name**
```sql
SELECT
  plan_name,
  COUNT(DISTINCT customer_id)
FROM subscriptions a
JOIN plans b
  ON a.plan_id = b.plan_id
WHERE EXTRACT(YEAR FROM start_date) > 2020
GROUP BY plan_name;
```
| plan_name     | count |
| ------------- | ----- |
| basic monthly | 8     |
| churn         | 71    |
| pro annual    | 63    |
| pro monthly   | 60    |

**What is the customer count and percentage of customers who have churned rounded to 1 decimal place?**
```sql
WITH churn_cust AS (
  SELECT
    COUNT(DISTINCT customer_id) AS churn_cust
  FROM subscriptions
  WHERE plan_id = 4
)
    
SELECT
  COUNT(DISTINCT customer_id) AS churn_cust,
  ROUND(100.0 * COUNT(customer_id)/
  (SELECT COUNT(DISTINCT customer_id)
  FROM subscriptions), 1) AS percentage
FROM subscriptions
WHERE plan_id = 4;
```
| churn_cust | percentage |
| ---------- | ---------- |
| 307        | 30.7       |

**How many customers have churned straight after their initial free trial - what percentage is this rounded to the nearest whole number?**
```sql
WITH churn_cte AS (
  SELECT *,
    CASE
      WHEN rnk = 2 AND plan_id = 4 THEN 1
      ELSE 0
    END AS churned
  FROM(
    SELECT *,
      ROW_NUMBER() OVER(PARTITION BY customer_id) AS rnk
    FROM subscriptions) sub
)
    
SELECT
  SUM(churned),
  ROUND(100.0 * SUM(churned)/
  (SELECT COUNT(DISTINCT customer_id) 
  FROM subscriptions))
FROM churn_cte;
```
| sum | round |
| --- | ----- |
| 92  | 9     |

**What is the number and percentage of customer plans after their initial free trial?**
```sql
WITH retain_cust AS (
  SELECT *,
    CASE
      WHEN rnk = 2 THEN 1
      ELSE 0
    END AS count
  FROM(
    SELECT *,
      ROW_NUMBER() OVER(PARTITION BY customer_id) AS rnk
    FROM subscriptions) sub
)
    
SELECT
  plan_name,
  SUM(count) AS retained_cust,
  ROUND(100.0 * SUM(count)/
  (SELECT COUNT(DISTINCT customer_id) 
  FROM subscriptions), 1) AS retain_percentage
FROM retain_cust a
JOIN plans b
  ON a.plan_id = b.plan_id
WHERE a.plan_id != 0
GROUP BY plan_name
ORDER BY plan_name;
```
| plan_name     | retained_cust | retain_percentage |
| ------------- | ------------- | ----------------- |
| basic monthly | 546           | 54.6              |
| churn         | 92            | 9.2               |
| pro annual    | 37            | 3.7               |
| pro monthly   | 325           | 32.5              |

**What is the customer count and percentage breakdown of all 5 plan_name values at 2020-12-31?**
```sql
WITH year_cust AS (
  SELECT *,
    ROW_NUMBER() OVER(PARTITION BY customer_id ORDER BY start_date DESC) AS rnk
  FROM subscriptions
  WHERE start_date <= '2020-12-31'
)
    
SELECT
  plan_name,
  COUNT(customer_id),
  ROUND(100.0 * COUNT(customer_id)/
  (SELECT COUNT(DISTINCT customer_id) 
  FROM subscriptions), 1) AS retain_percentage
FROM year_cust a
JOIN plans b
  ON a.plan_id = b.plan_id
WHERE rnk = 1
GROUP BY plan_name;
```
| plan_name     | count | retain_percentage |
| ------------- | ----- | ----------------- |
| basic monthly | 224   | 22.4              |
| churn         | 236   | 23.6              |
| pro annual    | 195   | 19.5              |
| pro monthly   | 326   | 32.6              |
| trial         | 19    | 1.9               |

**How many customers have upgraded to an annual plan in 2020?**
```sql
SELECT COUNT(DISTINCT customer_id)
FROM subscriptions
WHERE plan_id = 3
AND EXTRACT(YEAR FROM start_date) = 2020;
```
| count |
| ----- |
| 195   |

**How many days on average does it take for a customer to an annual plan from the day they join Foodie-Fi?**
```sql
WITH trial AS (
  SELECT *
  FROM subscriptions
  WHERE plan_id = 0
),
    
annual AS (
  SELECT *
  FROM subscriptions
  WHERE plan_id = 3
)
    
SELECT ROUND(AVG(a.start_date - b.start_date))
FROM annual a
JOIN trial b
  ON a.customer_id = b.customer_id;
```
| round |
| ----- |
| 105   |

### Challenge Payment Question

### Outside The Box Questions
