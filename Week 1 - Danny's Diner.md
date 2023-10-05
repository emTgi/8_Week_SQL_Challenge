### 1. What is the total amount each customer spent at the restaurant?
```sql
SELECT
  customer_id,
  SUM(price)
FROM dannys_diner.sales
JOIN dannys_diner.menu
	ON sales.product_id = menu.product_id
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
### 3. What was the first item from the menu purchased by each customer?
##### _If each row is a separate order:_
```sql
WITH order_no AS (
  SELECT *,
    ROW_NUMBER() OVER(PARTITION BY customer_id ORDER BY order_date) AS cust_order_no
  FROM dannys_diner.sales AS s
  )

SELECT
  customer_id,
  product_name
FROM order_no
JOIN dannys_diner.menu AS m
	ON order_no.product_id = m.product_id
WHERE cust_order_no = 1;
```
##### _If multiple products bought in one day are one order:_
```sql
WITH order_no AS (
  SELECT *,
	DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY order_date) AS cust_order_no
  FROM dannys_diner.sales AS s
  )

SELECT
  customer_id,
  product_name
FROM order_no
JOIN dannys_diner.menu AS m
	ON order_no.product_id = m.product_id
WHERE cust_order_no = 1
GROUP BY customer_id, product_name
ORDER BY customer_id;
```
### 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
```sql
SELECT
    product_name,
    COUNT(sales.product_id) AS times_purchased
FROM dannys_diner.sales
JOIN dannys_diner.menu
	ON sales.product_id = menu.product_id
GROUP BY product_name
ORDER BY times_purchased DESC;
```
### 5. Which item was the most popular for each customer?
```sql
WITH items_rank AS (
  SELECT
  	customer_id,
  	product_name,
  	COUNT(sales.product_id) AS times_purchased,
  	DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY COUNT(sales.product_id) DESC) AS rank
  FROM dannys_diner.sales
  JOIN dannys_diner.menu
	  ON sales.product_id = menu.product_id
  GROUP BY customer_id, product_name
  )

SELECT
  customer_id,
  product_name,
  times_purchased
FROM items_rank
WHERE rank = 1;
```
### 6. Which item was purchased first by the customer after they became a member?
##### _Orders made on the day of purchasing membership are included._
```sql
WITH members_orders AS (
  SELECT
	  s.customer_id,
    product_id,
    order_date,
    join_date,
    DENSE_RANK() OVER(PARTITION BY s.customer_id ORDER BY order_date) AS rank
FROM dannys_diner.sales AS s
JOIN dannys_diner.members AS mbr
  ON s.customer_id = mbr.customer_id 
	AND s.order_date >= mbr.join_date
  )

SELECT
	customer_id,
  join_date,
  product_name,
  order_date
FROM members_orders
JOIN dannys_diner.menu
  ON members_orders.product_id = menu.product_id
WHERE rank = 1
ORDER BY customer_id;
```
### 7. Which item was purchased just before the customer became a member?
##### _Orders made on the day of purchasing membership are not included._
```sql
WITH members_orders AS (
  SELECT
	  s.customer_id,
    product_id,
    order_date,
    join_date,
    DENSE_RANK() OVER(PARTITION BY s.customer_id ORDER BY order_date DESC) AS rank
FROM dannys_diner.sales AS s
JOIN dannys_diner.members AS mbr
  ON s.customer_id = mbr.customer_id 
	AND s.order_date < mbr.join_date
  )

SELECT
	customer_id,
  join_date,
  product_name,
  order_date
FROM members_orders
JOIN dannys_diner.menu
  ON members_orders.product_id = menu.product_id
WHERE rank = 1
ORDER BY customer_id;
```
### 8. What is the total items and amount spent for each member before they became a member?
```sql
WITH members_orders AS (
  SELECT
	  s.customer_id,
    product_id,
    order_date,
    join_date,
    ROW_NUMBER() OVER(PARTITION BY s.customer_id ORDER BY order_date DESC) AS rank
FROM dannys_diner.sales AS s
JOIN dannys_diner.members AS mbr
  ON s.customer_id = mbr.customer_id 
	AND s.order_date < mbr.join_date
  )

SELECT
	customer_id,
  COUNT(m.product_id) AS nr_of_orders,
  SUM(price) AS total_spent
FROM members_orders AS mbro
JOIN dannys_diner.menu AS m
  ON mbro.product_id = m.product_id
GROUP BY customer_id
ORDER BY customer_id;
```
### 9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
```sql
WITH menu_points AS (  
  SELECT *,
    CASE
  		WHEN product_name = 'sushi' THEN price*10*2
  		ELSE price*10
  	END
  	AS points
  FROM dannys_diner.menu
  )

SELECT
	customer_id,
  SUM(points)
FROM menu_points
JOIN dannys_diner.sales
  ON menu_points.product_id = sales.product_id
GROUP BY customer_id
ORDER BY customer_id;
```
### 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
```sql
WITH customer_points AS (
  SELECT
	  customer_id,
    order_date,
    product_name,
    points
  FROM dannys_diner.sales AS s
  JOIN (
    SELECT *,
    CASE
  		WHEN product_name = 'sushi' THEN price*10*2
  		ELSE price*10
  	END
  	AS points
    FROM dannys_diner.menu
  ) AS p ON s.product_id = p.product_id
)

SELECT
	mbr.customer_id,
  SUM(
    CASE
      WHEN order_date >= join_date
      AND order_date < join_date + integer '7'
      AND product_name != 'sushi'
      THEN points*2
      ELSE points
    END) 
FROM customer_points
JOIN dannys_diner.members AS mbr
	ON customer_points.customer_id = mbr.customer_id
WHERE EXTRACT(MONTH FROM order_date) < 2
GROUP BY mbr.customer_id;
```
### 11. Bonus
##### Join All The Things
```sql
SELECT
	s.customer_id,
  order_date,
  product_name,
  price,
  CASE
    WHEN s.order_date >= join_date THEN 'Y'
    ELSE 'N'
  END AS member
FROM dannys_diner.sales AS s
JOIN dannys_diner.menu AS m
	ON s.product_id = m.product_id
LEFT JOIN dannys_diner.members AS mbr
	ON s.customer_id = mbr.customer_id
ORDER BY s.customer_id, order_date;
```
##### Rank All The Things
```sql
WITH tb1 AS (
  SELECT
	  s.customer_id,
    order_date,
    product_name,
    price,
    CASE
    	WHEN s.order_date >= join_date THEN 'Y'
      ELSE 'N'
    END AS member
  FROM dannys_diner.sales AS s
  JOIN dannys_diner.menu AS m
	  ON s.product_id = m.product_id
  LEFT JOIN dannys_diner.members AS mbr
	  ON s.customer_id = mbr.customer_id
  ORDER BY s.customer_id, order_date
  )

SELECT *,
	CASE
    	WHEN member = 'N' THEN NULL
      ELSE DENSE_RANK() OVER(PARTITION BY customer_id, member ORDER BY order_date)
    END AS ranking
FROM tb1
```
