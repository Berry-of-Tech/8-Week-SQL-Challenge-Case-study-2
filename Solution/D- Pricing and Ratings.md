# Questions

1. If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes - how much money has Pizza Runner made so far if there are no delivery fees?
2. What if there was an additional $1 charge for any pizza extras?
   - Add cheese is $1 extra
3. The Pizza Runner team now wants to add an additional ratings system that allows customers to rate their runner, how would you design an additional table for this new dataset - generate a schema for this new table and insert your own data for ratings for each successful customer order between 1 to 5.
4. Using your newly generated table - can you join all of the information together to form a table which has the following information for successful deliveries?
    - customer_id
    - order_id
    - runner_id
    - rating
    - order_time
    - pickup_time
    - Time between order and pickup
    - Delivery duration
    - Average speed
    - Total number of pizzas
5. If a Meat Lovers pizza was $12 and Vegetarian $10 fixed prices with no cost for extras and each runner is paid $0.30 per kilometre traveled - how much money does Pizza Runner have left over after these deliveries?

# Solutions
1. If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes - how much money has Pizza Runner made so far if there are no delivery fees?
```sql
WITH prices AS (
	SELECT c.pizza_id, pn.pizza_name, CASE 
							WHEN c.pizza_id=1 THEN 12 ELSE 10 END AS price 
FROM customer_orders AS c
JOIN pizza_names AS pn
ON c.pizza_id = pn.pizza_id
JOIN runner_orders AS r
ON c.order_id = r.order_id
WHERE r.cancellation IS NULL)

SELECT SUM(price) AS money_made
FROM prices
```
 money_made
:---------:
  138
---   

2. What if there was an additional $1 charge for any pizza extras?
  - Add cheese is $1 extra
```sql
WITH first_price AS (
  SELECT SUM(price) AS money_made
  FROM(
	SELECT c.pizza_id, pn.pizza_name, CASE 
							WHEN c.pizza_id=1 THEN 12 ELSE 10 END AS price 
FROM customer_orders AS c
JOIN pizza_names AS pn
ON c.pizza_id = pn.pizza_id
JOIN runner_orders AS r
ON c.order_id = r.order_id
WHERE r.cancellation IS NULL) AS prices
),

second_price AS (
  SELECT SUM(charges) AS extras_costs
  FROM (
    SELECT tc.extras,
      CASE WHEN tc.extras = 4 THEN 2 ELSE 1 END AS charges
    FROM temp_customer_orders AS tc
    JOIN runner_orders AS r
      ON tc.order_id = r.order_id
    WHERE r.cancellation IS NULL 
	  AND tc.extras IS NOT NULL
  ) AS extras
)

SELECT first_price.money_made + second_price.extras_costs AS new_price
FROM first_price, second_price;

```
 new_price
:---------------:
  143
---  

3. The Pizza Runner team now wants to add an additional ratings system that allows customers to rate their runner, how would you design an additional table for this new dataset - generate a schema for this new table and insert your own data for ratings for each successful customer order between 1 to 5.
```sql
DROP TABLE IF EXISTS runner_ratings;
CREATE TABLE runner_ratings (
  order_id INT,
  rating INT
);
INSERT INTO runner_ratings (order_id, rating)
VALUES 
  (1,4),
  (2,5),
  (3,3),
  (4,1),
  (5,2),
  (7,3),
  (8,5),
  (10,3);

SELECT * 
FROM runner_ratings;

```
 order_id  | ratings
:---------:|:---------:
  1        | 3
  2        | 5
  3        | 3
  4        | 1
  5        | 5
  7        | 3 
  8        | 4 
  10       | 3
---

4. Using your newly generated table - can you join all of the information together to form a table which has the following information for successful deliveries?
  - customer_id
  - order_id
  - runner_id
  - rating
  - order_time
  - pickup_time
  - Time between order and pickup
  - Delivery duration
  - Average speed
  - Total number of pizzas

```sql
SELECT 
  c.customer_id,
  c.order_id,
  r.runner_id,
  rr.rating,
  c.order_time,
  r.pickup_time,
  r.pickup_time - c.order_time AS mins_difference,
  r.duration_mins,
 CEIL(AVG(r.distance_m/r.duration_s * 60)) AS avg_speed,
 COUNT(c.order_id) AS pizza_count
FROM customer_orders AS c
JOIN runner_orders AS r 
   ON r.order_id = c.order_id
JOIN runner_ratings AS rr
   ON c.order_id = rr.order_id
GROUP BY 
  c.customer_id,
  c.order_id,
  r.runner_id,
  rr.rating,
  c.order_time,
  r.pickup_time, 
  r.duration_mins;

```
![](https://github.com/imanjokko/PizzaRunner/blob/main/images/sectiondno4.png)

5. If a Meat Lovers pizza was $12 and Vegetarian $10 fixed prices with no cost for extras and each runner is paid $0.30 per kilometre traveled - how much money does Pizza Runner have left over after these deliveries?

```sql
WITH income AS
(SELECT SUM(price) AS money_made
FROM 
	(SELECT c.pizza_id, pn.pizza_name,
		CASE WHEN c.pizza_id = 1 THEN 12 ELSE 10 END AS price
	 FROM customer_orders AS c
	 JOIN pizza_names AS pn
   	 ON c.pizza_id = pn.pizza_id
	 JOIN runner_orders AS r
   	 ON c.order_id = r.order_id
	 WHERE r.cancellation IS NULL) AS prices
),
expenses AS 
	(SELECT SUM (drivers_payment) AS tot_drivers_payment
	FROM
		(SELECT distance_km * 0.30 AS drivers_payment
		FROM runner_orders
		WHERE distance_km IS NOT NULL) AS drivers_fee
	)
SELECT income.money_made - expenses.tot_drivers_payment AS leftover_cash
FROM income, expenses


```
 leftover_cash
:--------------:
  94.44
 
---
# Insights
- The restaurant has made 138usd base income so far from pizza orders
- When you add the prices of the extras, the total income comes up to 143usd
- After paying the riders for their deliveries from the base income, the leftover cash is 94.44usd
