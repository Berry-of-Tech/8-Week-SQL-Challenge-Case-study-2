# Questions

1. What are the standard ingredients for each pizza?
2. What was the most commonly added extra?
3. What was the most common exclusion?
4. Generate an order item for each record in the customers_orders table in the format of one of the following:
  - Meat Lovers
  - Meat Lovers - Exclude Beef
  - Meat Lovers - Extra Bacon
  - Meat Lovers - Exclude Cheese, Bacon - Extra Mushroom, Peppers
5. Generate an alphabetically ordered comma separated ingredient list for each pizza order from the customer_orders table and add a 2x in front of any relevant ingredients
  - For example: "Meat Lovers: 2xBacon, Beef, ... , Salami"
6. What is the total quantity of each ingredient used in all delivered pizzas sorted by most frequent first?

# Solution
## Creating Temp Tables

-- Create a temp table clean_extras to separate extras with delimiter into multiple rows
```sql
DROP TABLE IF EXISTS clean_extras;

CREATE TEMP TABLE clean_extras AS
SELECT c.record_id, 
		CAST(TRIM(e.extras) AS INTEGER) AS extra_id
FROM customer_orders AS c
LEFT JOIN LATERAL UNNEST(STRING_TO_ARRAY(c.extras,',')) AS e(extras) ON TRUE;


SELECT *
FROM clean_extras
```
![](https://github.com/Berry-of-Tech/8-Week-SQL-Challenge-Case-study-2/blob/main/Images/temp%20extras%20table.png)

-- Create a temp table clean_exclusions to separate exclusions with delimiter into multiple rows
```sql
DROP TABLE IF EXISTS clean_exclusions;

CREATE TEMP TABLE clean_exclusions AS
SELECT c.record_id, CAST(TRIM(e.exclusions) AS INTEGER) AS exclusions_id
FROM customer_orders AS c
LEFT JOIN LATERAL UNNEST(STRING_TO_ARRAY(c.exclusions,',')) AS e(exclusions) ON TRUE;

SELECT *
FROM clean_exclusions
```
![](https://github.com/Berry-of-Tech/8-Week-SQL-Challenge-Case-study-2/blob/main/Images/temp%20extrusions%20table.png)

--Create a temp table clean_toppings to separate toppings into multiple rows
```sql
DROP TABLE IF EXISTS clean_toppings;

CREATE TEMP TABLE clean_toppings AS
SELECT
  pr.pizza_id,
  TRIM(t.value) :: INTEGER AS topping_id,
  pt.topping_name
FROM
  pizza_recipes AS pr
CROSS JOIN 
	LATERAL UNNEST(STRING_TO_ARRAY(pr.toppings, ',')) AS t(value)
JOIN
  (SELECT DISTINCT topping_id, topping_name 
   FROM pizza_toppings) pt
  ON CAST(TRIM(t.value) AS INTEGER) = pt.topping_id
GROUP BY pr.pizza_id, t.value, pt.topping_name
ORDER BY pr.pizza_id;


SELECT *
FROM clean_toppings;

```
![](https://github.com/Berry-of-Tech/8-Week-SQL-Challenge-Case-study-2/blob/main/Images/clean%20toppings.png)

--Create a temp table new_recipes to separate toppings into multiple rows
```sql
DROP TABLE IF EXISTS new_recipes;

CREATE TEMPORARY TABLE new_recipes AS
SELECT 
  pr.pizza_id,
  CAST(TRIM(nr.toppings) AS INTEGER) AS topping_id
FROM pizza_recipes AS pr
LEFT JOIN LATERAL UNNEST(STRING_TO_ARRAY(pr.toppings, ',')) AS nr(toppings) ON true;


SELECT *
FROM new_recipes
```

1. What are the standard ingredients for each pizza?
```sql
WITH Ingredients_CTE AS (
    SELECT nr.pizza_id, pn.pizza_name, pt.topping_name
    FROM new_recipes AS nr
    JOIN pizza_toppings AS pt
        ON nr.topping_id = pt.topping_id
    JOIN pizza_names AS pn
        ON pn.pizza_id = nr.pizza_id
    ORDER BY pn.pizza_name
)
SELECT pizza_name, STRING_AGG(topping_name, ', ') as str_topping_name
FROM Ingredients_CTE
GROUP BY pizza_name;
```
 pizza_name | std_topping_names
:----------:|:-------------:
 Meatlovers |BBQ Sauce, Pepperoni, Cheese, Salami, Chicken, Bacon, Mushrooms, Beef
 Vegetarian |Tomato Sauce, Cheese, Mushrooms, Onions, Peppers, Tomatoes
 
 ---
2. What was the most commonly added extra?
```sql
SELECT  COUNT (tc.extras) extras_count, pt.topping_name		
FROM pizza_toppings AS pt
JOIN temp_customer_orders AS tc  
  ON tc.extras = pt.topping_id
GROUP BY extras, topping_name
ORDER BY extras_count DESC
LIMIT 1;
 
```
  extras_count | topping_name
:------:|:-------------:
  4     | Bacon
  
---
3. What was the most common exclusion?
```sql
SELECT  COUNT (tc.exclusions) exclusion_count,  pt.topping_name		
FROM pizza_toppings AS pt
JOIN temp_customer_orders AS tc  
  ON tc.exclusions = pt.topping_id
GROUP BY exclusions, topping_name
ORDER BY exclusion_count DESC
LIMIT 1;
```
  exclusion_count	 | topping_name
:------:|:--------------:
 4 	| Cheese
 
---
4. Generate an order item for each record in the customers_orders table in the format of one of the following:
   - Meat Lovers
   - Meat Lovers - Exclude Beef
   - Meat Lovers - Extra Bacon
   - Meat Lovers - Exclude Cheese, Bacon - Extra Mushroom, Peppers

**To solve this question:** 
- I Created 3 CTEs: CTE_Extras, CTE_Exclusions, and CTE_Union combining two tables
- I used the CTE_Union to LEFT JOIN with the customer_orders table and JOIN with the pizza_name table
- I used the CONCAT_WS with STRING_AGG to get the result

--adding record id column
```sql
ALTER TABLE customer_orders
ADD COLUMN record_id SERIAL PRIMARY KEY

SELECT *
FROM customer_orders;
```
 Old Table                                    | Altered Table
:--------------------------------------------:|:--------------:
  ![](https://github.com/Berry-of-Tech/8-Week-SQL-Challenge-Case-study-2/blob/main/Images/clean%20customer%20orders%20table.png)|![](https://github.com/Berry-of-Tech/8-Week-SQL-Challenge-Case-study-2/blob/main/Images/customer%20orders%20table%20with%20record_id%20.png)


```SQL
WITH CTE_Extras AS
(SELECT
ce.record_id,
'Extra ' || STRING_AGG(t.topping_name, ', ') AS record_options
FROM clean_extras AS ce
JOIN pizza_toppings AS t
  ON ce.extra_id = t.topping_id
GROUP BY ce.record_id),

CTE_Exclusions AS 
(SELECT
ce.record_id,
'Exclude ' || STRING_AGG(t.topping_name, ', ') AS record_options
FROM clean_exclusions AS ce
JOIN pizza_toppings AS t
  ON ce.exclusions_id = t.topping_id
GROUP BY ce.record_id),

CTE_Union AS 
(SELECT * FROM CTE_Extras
UNION
SELECT * FROM CTE_Exclusions)

SELECT
c.record_id,
c.order_id,
c.customer_id,
c.pizza_id,
c.order_time,
CONCAT_WS(' - ', p.pizza_name, STRING_AGG(u.record_options, ' - ')) AS pizza_info
FROM customer_orders AS c
LEFT JOIN CTE_Union AS u
  ON c.record_id = u.record_id
JOIN pizza_names p
  ON c.pizza_id = p.pizza_id
GROUP BY
c.record_id,
c.order_id,
c.customer_id,
c.pizza_id,
c.order_time,
p.pizza_name
ORDER BY record_id;
```
![](https://github.com/Berry-of-Tech/8-Week-SQL-Challenge-Case-study-2/blob/main/Images/ingredient%20optimisation%204.png)

5. Generate an alphabetically ordered comma separated ingredient list for each pizza order from the customer_orders table and add a 2x in front of any relevant ingredients
  - For example: "Meat Lovers: 2xBacon, Beef, ... , Salami"

```sql
WITH CTE_ingredients AS (
  SELECT 
    c.record_id,
    c.order_id,
    c.customer_id,
    c.pizza_id,
    c.order_time,
    pn.pizza_name,
    CASE 
      WHEN t.topping_id::int IN 
	( SELECT ce.extra_id 
        FROM clean_extras ce 
        WHERE ce.record_id = c.record_id) THEN '2x ' || pt.topping_name
      ELSE pt.topping_name
    END AS topping
  FROM 
    customer_orders c
    LEFT JOIN pizza_names pn 
      ON c.pizza_id = pn.pizza_id
    LEFT JOIN pizza_recipes pr 
      ON c.pizza_id = pr.pizza_id
    LEFT JOIN UNNEST (STRING_TO_ARRAY(pr.toppings, ',')) AS t(topping_id) 
      ON TRUE
    LEFT JOIN pizza_toppings pt 
      ON t.topping_id::int = pt.topping_id
  WHERE 
    NOT EXISTS 
	(SELECT 1 
      FROM clean_exclusions cex 
      WHERE c.record_id = cex.record_id 
      AND pt.topping_id = cex.exclusions_id))
SELECT 
  record_id,
  order_id,
  customer_id,
  pizza_id,
  order_time,
  CONCAT(pizza_name, ': ', STRING_AGG(topping, ', ')) AS ingredients_list
FROM CTE_ingredients
GROUP BY 
  record_id, 
  order_id,
  customer_id,
  pizza_id,
  order_time,
  pizza_name
ORDER BY 
  record_id;
```
![](https://github.com/Berry-of-Tech/8-Week-SQL-Challenge-Case-study-2/blob/main/Images/ingredient%20optimisation%205.png)

6. What is the total quantity of each ingredient used in all delivered pizzas sorted by most frequent first?
**To solve this question:**
- I Created a CTE to record the number of times each ingredient was used
  - if extra ingredient, add 2
  - if excluded ingredient, add 0
  - no extras or no exclusions, add 1

```sql
WITH frequent_Ingredients_CTE AS (
  SELECT 
    c.record_id,
    ct.topping_name,
    CASE
      -- if extra ingredient, add 2
      WHEN ct.topping_id IN (
          SELECT extra_id 
          FROM clean_extras ce
          WHERE ce.record_id = c.record_id) 
      THEN 2
      -- if excluded ingredient, add 0
      WHEN ct.topping_id IN (
          SELECT exclusions_id 
          FROM clean_exclusions cex 
          WHERE c.record_id = cex.record_id)
      THEN 0
      -- no extras, no exclusions, add 1
      ELSE 1
    END AS number_of_times_used
  FROM customer_orders c
  JOIN clean_toppings ct
    ON ct.pizza_id = c.pizza_id
  JOIN pizza_names p
    ON p.pizza_id = c.pizza_id
)

SELECT 
  topping_name,
  SUM(number_of_times_used) AS number_of_times_used 
FROM frequent_Ingredients_CTE
GROUP BY topping_name
ORDER BY number_of_times_used DESC;
```


  topping_name  | number_of_times_used
:--------------:|:--------------:
  Mushrooms     | 13
  Bacon         | 13
  Chicken       | 11
  Cheese        | 11
  Pepperoni     | 10
  Salami        | 10
  Beef          | 10
  BBQ Sauce     | 9 
  Tomatoes      | 4
  Onions        | 4
  Peppers       | 4
  Tomato Sauce  | 4
  
  # Insights
- The most common extra is Bacon 
- The most common exclusion is Cheese 
- The most used ingredients are Mushrooms and Bacon
