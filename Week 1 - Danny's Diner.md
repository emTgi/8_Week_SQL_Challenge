### 1. What is the total amount each customer spent at the restaurant?
```sql
SELECT
	customer_id,
  	SUM(price)
FROM dannys_diner.sales a
JOIN dannys_diner.menu b
	ON a.product_id = b.product_id
GROUP BY customer_id
ORDER BY customer_id;
```

| customer_id | sum |
| ----------- | --- |
| A           | 76  |
| B           | 74  |
| C           | 36  |

### 2. How many days has each customer visited the restaurant?
```sql
SELECT
  	customer_id,
  	COUNT(DISTINCT order_date)
FROM dannys_diner.sales
GROUP BY customer_id;
```

| customer_id | count |
| ----------- | ----- |
| A           | 4     |
| B           | 6     |
| C           | 2     |

### 3. What was the first item from the menu purchased by each customer?
##### _If each row is a separate order:_
```sql
WITH order_no AS (
	SELECT 	*,
		ROW_NUMBER() OVER(PARTITION BY customer_id ORDER BY order_date) AS cust_order_no
  	FROM dannys_diner.sales
)

SELECT
  	customer_id,
  	product_name
FROM order_no a
JOIN dannys_diner.menu b
	ON a.product_id = b.product_id
WHERE cust_order_no = 1;
```

| customer_id | product_name |
| ----------- | ------------ |
| A           | sushi        |
| B           | curry        |
| C           | ramen        |

##### _If multiple products bought in one day are one order:_
```sql
WITH order_no AS (
  	SELECT 	*,
		DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY order_date) AS cust_order_no
 	FROM dannys_diner.sales
)

SELECT
  	customer_id,
  	product_name
FROM order_no a
JOIN dannys_diner.menu b
	ON a.product_id = b.product_id
WHERE cust_order_no = 1
GROUP BY customer_id, product_name
ORDER BY customer_id;
```

| customer_id | product_name |
| ----------- | ------------ |
| A           | curry        |
| A           | sushi        |
| B           | curry        |
| C           | ramen        |

### 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
```sql
SELECT
	product_name,
    	COUNT(sales.product_id) AS times_purchased
FROM dannys_diner.sales a
JOIN dannys_diner.menu b
	ON a.product_id = b.product_id
GROUP BY product_name
ORDER BY times_purchased DESC;
```

| product_name | times_purchased |
| ------------ | --------------- |
| ramen        | 8               |
| curry        | 4               |
| sushi        | 3               |

### 5. Which item was the most popular for each customer?
```sql
WITH items_rank AS (
	SELECT
  		customer_id,
  		product_name,
  		COUNT(a.product_id) AS times_purchased,
  		DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY COUNT(a.product_id) DESC) AS rank
  	FROM dannys_diner.sales a
  	JOIN dannys_diner.menu b
	  	ON a.product_id = b.product_id
  	GROUP BY customer_id, product_name
)

SELECT
  	customer_id,
	product_name,
	times_purchased
FROM items_rank
WHERE rank = 1;
```

| customer_id | product_name | times_purchased |
| ----------- | ------------ | --------------- |
| A           | ramen        | 3               |
| B           | ramen        | 2               |
| B           | curry        | 2               |
| B           | sushi        | 2               |
| C           | ramen        | 3               |

### 6. Which item was purchased first by the customer after they became a member?
##### _Orders made on the day of purchasing membership are included._
```sql
WITH members_orders AS (
	SELECT
		a.customer_id,
    		product_id,
    		order_date,
    		join_date,
    		DENSE_RANK() OVER(PARTITION BY a.customer_id ORDER BY order_date) AS rank
	FROM dannys_diner.sales a
	JOIN dannys_diner.members b
  		ON a.customer_id = b.customer_id 
		AND a.order_date >= b.join_date
)

SELECT
	customer_id,
  	join_date,
  	product_name,
  	order_date
FROM members_orders a
JOIN dannys_diner.menu b
  	ON a.product_id = b.product_id
WHERE rank = 1
ORDER BY customer_id;
```

| customer_id | join_date                | product_name | order_date               |
| ----------- | ------------------------ | ------------ | ------------------------ |
| A           | 2021-01-07T00:00:00.000Z | curry        | 2021-01-07T00:00:00.000Z |
| B           | 2021-01-09T00:00:00.000Z | sushi        | 2021-01-11T00:00:00.000Z |

### 7. Which item was purchased just before the customer became a member?
##### _Orders made on the day of purchasing membership are not included._
```sql
WITH members_orders AS (
	SELECT
		a.customer_id,
    		product_id,
    		order_date,
    		join_date,
    		DENSE_RANK() OVER(PARTITION BY a.customer_id ORDER BY order_date DESC) AS rank
	FROM dannys_diner.sales a
	JOIN dannys_diner.members b
  		ON a.customer_id = b.customer_id 
		AND a.order_date < b.join_date
)

SELECT
	customer_id,
  	join_date,
	product_name,
  	order_date
FROM members_orders a
JOIN dannys_diner.menu b
  	ON a.product_id = b.product_id
WHERE rank = 1
ORDER BY customer_id;
```

| customer_id | join_date                | product_name | order_date               |
| ----------- | ------------------------ | ------------ | ------------------------ |
| A           | 2021-01-07T00:00:00.000Z | sushi        | 2021-01-01T00:00:00.000Z |
| A           | 2021-01-07T00:00:00.000Z | curry        | 2021-01-01T00:00:00.000Z |
| B           | 2021-01-09T00:00:00.000Z | sushi        | 2021-01-04T00:00:00.000Z |

### 8. What is the total items and amount spent for each member before they became a member?
```sql
WITH members_orders AS (
  	SELECT
	  	a.customer_id,
    		product_id,
    		order_date,
    		join_date,
    		ROW_NUMBER() OVER(PARTITION BY a.customer_id ORDER BY order_date DESC) AS rank
	FROM dannys_diner.sales a
	JOIN dannys_diner.members b
  		ON a.customer_id = b.customer_id 
		AND a.order_date < b.join_date
)

SELECT
	customer_id,
  	COUNT(b.product_id) AS nr_of_orders,
  	SUM(price) AS total_spent
FROM members_orders a
JOIN dannys_diner.menu b
  	ON a.product_id = b.product_id
GROUP BY customer_id
ORDER BY customer_id;
```

| customer_id | nr_of_orders | total_spent |
| ----------- | ------------ | ----------- |
| A           | 2            | 25          |
| B           | 3            | 40          |

### 9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
```sql
WITH menu_points AS (  
	SELECT 	*,
    		CASE
  			WHEN product_name = 'sushi' THEN price*10*2
  			ELSE price*10
  		END AS points
  	FROM dannys_diner.menu
)

SELECT
	customer_id,
  	SUM(points)
FROM menu_points a
JOIN dannys_diner.sales b
  	ON a.product_id = b.product_id
GROUP BY customer_id
ORDER BY customer_id;
```

| customer_id | sum |
| ----------- | --- |
| A           | 860 |
| B           | 940 |
| C           | 360 |

### 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
```sql
WITH customer_points AS (
  	SELECT
	  	customer_id,
    		order_date,
    		product_name,
    		points
  	FROM dannys_diner.sales a
  	JOIN (SELECT 	*,
			CASE
				WHEN product_name = 'sushi' THEN price*10*2
  				ELSE price*10
  			END AS points
    		FROM dannys_diner.menu
	) b ON a.product_id = b.product_id
)

SELECT
	b.customer_id,
  	SUM(CASE
		WHEN order_date >= join_date
		AND order_date < join_date + integer '7'
		AND product_name != 'sushi'
		THEN points*2
		ELSE points
		END
	) 
FROM customer_points a
JOIN dannys_diner.members b
	ON a.customer_id = b.customer_id
WHERE EXTRACT(MONTH FROM order_date) < 2
GROUP BY b.customer_id;
```

| customer_id | sum  |
| ----------- | ---- |
| A           | 1370 |
| B           | 820  |

### 11. Bonus
##### Join All The Things
```sql
SELECT
	a.customer_id,
	order_date,
  	product_name,
  	price,
  	CASE
    		WHEN a.order_date >= join_date THEN 'Y'
    		ELSE 'N'
  	END AS member
FROM dannys_diner.sales a
JOIN dannys_diner.menu b
	ON a.product_id = b.product_id
LEFT JOIN dannys_diner.members c
	ON a.customer_id = c.customer_id
ORDER BY a.customer_id, order_date;
```

| customer_id | order_date               | product_name | price | member |
| ----------- | ------------------------ | ------------ | ----- | ------ |
| A           | 2021-01-01T00:00:00.000Z | sushi        | 10    | N      |
| A           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |
| A           | 2021-01-07T00:00:00.000Z | curry        | 15    | Y      |
| A           | 2021-01-10T00:00:00.000Z | ramen        | 12    | Y      |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      |
| B           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |
| B           | 2021-01-02T00:00:00.000Z | curry        | 15    | N      |
| B           | 2021-01-04T00:00:00.000Z | sushi        | 10    | N      |
| B           | 2021-01-11T00:00:00.000Z | sushi        | 10    | Y      |
| B           | 2021-01-16T00:00:00.000Z | ramen        | 12    | Y      |
| B           | 2021-02-01T00:00:00.000Z | ramen        | 12    | Y      |
| C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      |
| C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      |
| C           | 2021-01-07T00:00:00.000Z | ramen        | 12    | N      |

##### Rank All The Things
```sql
WITH tb1 AS (
	SELECT
	  	a.customer_id,
    		order_date,
    		product_name,
    		price,
		CASE
    			WHEN a.order_date >= join_date THEN 'Y'
      			ELSE 'N'
    		END AS member
  FROM dannys_diner.sales a
  JOIN dannys_diner.menu b
	  ON a.product_id = b.product_id
  LEFT JOIN dannys_diner.members c
	  ON a.customer_id = c.customer_id
  ORDER BY a.customer_id, order_date
  )

SELECT 	*,
	CASE
		WHEN member = 'N' THEN NULL
      		ELSE DENSE_RANK() OVER(PARTITION BY customer_id, member ORDER BY order_date)
    	END AS ranking
FROM tb1
```

| customer_id | order_date               | product_name | price | member | ranking |
| ----------- | ------------------------ | ------------ | ----- | ------ | ------- |
| A           | 2021-01-01T00:00:00.000Z | sushi        | 10    | N      |         |
| A           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |         |
| A           | 2021-01-07T00:00:00.000Z | curry        | 15    | Y      | 1       |
| A           | 2021-01-10T00:00:00.000Z | ramen        | 12    | Y      | 2       |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      | 3       |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      | 3       |
| B           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |         |
| B           | 2021-01-02T00:00:00.000Z | curry        | 15    | N      |         |
| B           | 2021-01-04T00:00:00.000Z | sushi        | 10    | N      |         |
| B           | 2021-01-11T00:00:00.000Z | sushi        | 10    | Y      | 1       |
| B           | 2021-01-16T00:00:00.000Z | ramen        | 12    | Y      | 2       |
| B           | 2021-02-01T00:00:00.000Z | ramen        | 12    | Y      | 3       |
| C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      |         |
| C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      |         |
| C           | 2021-01-07T00:00:00.000Z | ramen        | 12    | N      |         |
