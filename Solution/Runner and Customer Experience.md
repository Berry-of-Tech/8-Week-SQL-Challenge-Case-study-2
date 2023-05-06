# Runner and Customer Experience Questions
1. How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)
2. What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?
3. Is there any relationship between the number of pizzas and how long the order takes to prepare?
4. What was the average distance travelled for each customer?
5. What was the difference between the longest and shortest delivery times for all orders?
6. What was the average speed for each runner for each delivery and do you notice any trend for these values?
7. What is the successful delivery percentage for each runner?

# Solutions
1. How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)
```sql
SELECT DATE_PART('week',registration_date + INTERVAL '6days') AS week, COUNT(*) AS total_signups_per_week
FROM runners
GROUP BY week
ORDER BY week
/*I added the 6days interval to ensure that signups within 7 days would be 
included in the same week, since postgres's week start on Monday.*/
```
 week   | total_signups_per_week
:------------:|:--------------:
 1            | 2
 2            | 1
 3            | 1
 ---

2. What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?
```sql
WITH CTE_time AS (SELECT runner_id, (r.pickup_time - c.order_time) AS time_to_arrival
FROM runner_orders AS r
JOIN customer_orders AS c
ON r.order_id = c.order_id)

SELECT runner_id, AVG(time_to_arrival) AS avg_arrival_time
FROM CTE_time
GROUP BY runner_id
```
 runner_id    | avg_arrival_time
:------------:|:-------------------:
 1            | 00:15:40.666667
 3            | 00:10:28
 2            | 00:23:43.2
 ---
 
 3. Is there any relationship between the number of pizzas and how long the order takes to prepare?

 --viewing prep time per order
 ```sql
SELECT c.order_id, COUNT(*) AS total_pizza, (r.pickup_time - c.order_time) AS time_to_prepare 
FROM runner_orders AS r
JOIN customer_orders AS c
ON r.order_id = c.order_id
WHERE r.pickup_time IS NOT NULL
GROUP BY c.order_id, time_to_prepare
ORDER BY c.order_id;
```
 order_id  | total_pizza  | time_to_prepare
:---------:|:-----------:|:-----------------:
 1         | 1           | 00:10:32
 2         | 1           | 00:10:02
 3         | 2           | 00:21:14
 4         | 3           | 00:29:17
 5         | 1           | 00:10:28
 7         | 1           | 00:10:16
 8         | 1           | 00:20:29
 10        | 2           | 00:15:31
 
 --looking at the average prep time by amount of pizzas in the order
```sql
WITH CTE AS(
	SELECT c.order_id, COUNT(*) AS total_pizza, (r.pickup_time - c.order_time) AS time_to_prepare 
FROM runner_orders AS r
JOIN customer_orders AS c
ON r.order_id = c.order_id
WHERE r.pickup_time IS NOT NULL
GROUP BY c.order_id, time_to_prepare)

SELECT total_pizza, AVG(time_to_prepare) AS avg_time_to_prepare
FROM CTE
GROUP BY total_pizza
ORDER BY avg_time_to_prepare DESC;
~~~
 total_pizzas | avg_time_to_prepare
:----------:|:--------------:
  3         | 00:29:17
  2         | 00:18:22.5
  1         | 00:12:21.4
 
--YES, there is a relationship(correlation) between the number of pizzas and how long the order takes to prepare

---
4. What was the average distance travelled for each customer?
```sql
SELECT c.customer_id, CEIL(AVG(r.distance_km)) AS avg_distance_km
FROM customer_orders AS c
JOIN runner_orders AS r
ON r.order_id = c.order_id
WHERE r.cancellation IS NULL
GROUP BY c.customer_id
ORDER BY c.customer_id;
```
 customer_id  | avg_distance_km
:------------:|:----------------:
  101         | 20
  102         | 17
  103         | 24
  104         | 10
  105         | 25
---

5. What was the difference between the longest and shortest delivery times for all orders?
```sql
SELECT (MAX(duration_mins)) - (MIN(duration_mins)) AS duration_mins_diff
FROM runner_orders;
```
 duration_mins_diff
:--------------:
  30
---

6. What was the average speed for each runner for each delivery and do you notice any trend for these values?

--converting distance and duration to m/s for speed calculation
```sql
ALTER TABLE runner_orders
ADD COLUMN distance_m FLOAT

UPDATE runner_orders
SET distance_m = distance_km * 1000

ALTER TABLE runner_orders
ADD COLUMN duration_s INT

UPDATE runner_orders
SET duration_s = duration_mins * 60
```
  Old table                        | Updated table
 :--------------------------------:|:-------------------------------------:
  ![](https://github.com/imanjokko/PizzaRunner/blob/main/images/runnerordersaltered.png) | ![](https://github.com/imanjokko/PizzaRunner/blob/main/images/runnerordersupdated.png)

--speed for each delivery
~~~sql
SELECT runner_id, order_id, CEIL(distance_m/duration_s) AS speed_per_delivery
FROM runner_orders
WHERE cancellation IS NULL
~~~
 runner_id | order_id | speed_per_delivery
:---------:|:--------:|:---------------:
  1        | 1        | 11
  1        | 2        | 13
  1        | 3        | 12
  2        | 4        | 10
  3        | 5        | 12
  2        | 7        | 17
  2        | 8        | 26
  1        | 10       | 17
  
--calculating avg speed per runner
```sql
SELECT runner_id, CEIL(AVG(distance_m / duration_s)) AS avg_speed_for_each_runner
FROM runner_orders
GROUP BY runner_id
ORDER BY runner_id DESC;
```

 runner_id | avg_speed_for_each_runner
:---------:|:-----------------:
  3        | 12
  2        | 18
  1        | 13
  
-- I didn't noticed any trend

---
7. What is the successful delivery percentage for each runner?
```sql
SELECT runner_id, (100 * SUM(CASE WHEN cancellation IS NULL THEN 1 ELSE 0 END) 
		/ COUNT(*)) AS successful_del_percent
FROM runner_orders
GROUP BY runner_id
ORDER BY runner_id DESC;
```
 runner_id  | successful_del_percent
:----------:|:--------------:
  3         | 50
  2         | 75
  1         | 100
  
---
  # Insights
  
 - Runner 1 has the highest percentage of succesful deliveries
 - Customer 105 is the one riders have travelled the most distance for
 - Week 1 had the most runner signups
 - Runner 2 is the fastest rider
 
