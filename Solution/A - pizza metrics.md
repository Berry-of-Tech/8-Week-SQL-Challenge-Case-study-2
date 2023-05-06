# Pizza Metrics Questions
1. How many pizzas were ordered?
2. How many unique customer orders were made?
3. How many successful orders were delivered by each runner?
4. How many of each type of pizza was delivered?
5. How many Vegetarian and Meatlovers were ordered by each customer?
6. What was the maximum number of pizzas delivered in a single order?
7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?
8. How many pizzas were delivered that had both exclusions and extras?
9. What was the total volume of pizzas ordered for each hour of the day?
10. What was the volume of orders for each day of the week?
---

# Solutions
1. How many pizzas were ordered?
```sql
SELECT COUNT(*) AS total_pizzas
FROM customer_orders; 
```
  total_pizzas
:--------------:
 14               
 ---

2. How many unique customer orders were made?
```sql
SELECT COUNT(DISTINCT order_id) AS total_unique_orders
FROM customer_orders;
```
  total_unique_orders
:--------------------:
 10
 ---
 
 3. How many successful orders were delivered by each runner?
```sql
SELECT runner_id, COUNT(*) AS orders_delivered
FROM runner_orders
WHERE cancellation IS NULL
GROUP BY runner_id;
```
  runner_id          |  orders_delivered
:-------------------:|:---------------------:
  1                  | 4
  3                  | 1
  2                  | 3
  ---

4. How many of each type of pizza was delivered?
~~~sql
SELECT DISTINCT pizza_id, COUNT(*) AS num_delivered
FROM customer_orders c
JOIN runner_orders r
ON c.order_id = r.order_id
WHERE cancellation IS NULL
GROUP BY 1
ORDER BY 2 DESC;
~~~
 pizza_id       | num_delivered
:--------------:|:----------------:
 1              | 9
 2              | 3
 ---

5. How many Vegetarian and Meatlovers were ordered by each customer?
```sql
SELECT c.customer_id, COUNT(CASE WHEN p.pizza_name = 'Vegetarian' 
						THEN 1 ELSE NULL END) AS vegetarian_count,
					COUNT(CASE WHEN p.pizza_name = 'Meatlovers'
					THEN 1 ELSE NULL END) AS meatlovers_count
FROM customer_orders AS c
JOIN pizza_names AS p
ON c.pizza_id = p.pizza_id
GROUP BY c.customer_id;
```
 customer_id     | vegetarian_count   |  meatlovers_count
:---------------:|:------------------:|:--------------------:
 103             | 1                  | 3 
 105             | 1                  | 0
 101             | 1                  | 2
 104             | 0                  | 3
 102             | 1                  | 2
 ----

6. What was the maximum number of pizzas delivered in a single order?
```sql
SELECT COUNT(*) AS max_pizzas_delivered_per_order
FROM customer_orders c
JOIN runner_orders r
ON c.order_id = r.order_id
WHERE cancellation IS NULL
GROUP BY r.order_id
ORDER BY 1 DESC
LIMIT 1;
```
 max_pizzas_delivered_per_order
:-----------------------------:
 3
 ---

7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?
```sql
SELECT customer_id, 
        COUNT(CASE WHEN c.exclusions IS NOT NULL OR c.extras IS NOT NULL THEN 1 END) AS changed_count,
					COUNT(CASE WHEN c.exclusions IS NULL AND c.extras IS NULL THEN 1 END) AS Unchanged_count
FROM customer_orders c
JOIN runner_orders r
ON c.order_id = r.order_id
WHERE cancellation IS NULL
GROUP BY customer_id
ORDER BY customer_id
  ```
  customer_id   |  changed_count   | unchanged_count
 :-------------:|:-----------------:|:------------------:
  101           | 0                 | 2
  102           | 0                 | 3
  103           | 3                 | 0
  104           | 2                 | 1
  105           | 1                 | 0
  ---

8. How many pizzas were delivered that had both exclusions and extras?
```sql
SELECT COUNT(*) AS pizzas_del_with_both_exclusions_and_extras
FROM customer_orders c
JOIN runner_orders r
ON c.order_id = r.order_id
WHERE cancellation IS NULL
	AND exclusions IS NOT NULL
	AND extras IS NOT NULL;
```
 pizzas_del_with_both_exclusions_and_extras
:-------------------------------------:
 1
 ---

9. What was the total volume of pizzas ordered for each hour of the day?
```sql
SELECT DATE_PART('hour', order_time) AS hour, 
  COUNT(*) AS vol_per_hour
FROM customer_orders
GROUP BY hour
ORDER BY hour;
```
 hour     | vol_per_hour
:--------:|:-----------------:
 11       | 1
 13       | 3
 18       | 3
 19       | 1
 21       | 3
 23       | 3
 ---
 
 10. What was the volume of orders for each day of the week?
```sql
SELECT TO_CHAR(order_time, 'Day') AS day_of_week, 
  COUNT(*) AS vol_per_dow
FROM customer_orders
GROUP BY day_of_week;
```
 day_of_week    | vol_per_dow
:--------------:|:------------------:
 Friday          | 1
 Wednesday       | 5
 Thursday        | 3
 Saturday        | 5
---

# Insights

- The Pizza Runner HQ has a total number of 14 pizzas and had a total number of 10 orders
- **Runner_id 1** has the highest number of deliveries
- The most delivered pizza is pizza_id 1 which is the **Meatlovers** pizza
- Their biggest delivery was an order of 3 pizzas
- Their busiest days of the week so far, are Wednesdays and Saturdays.
