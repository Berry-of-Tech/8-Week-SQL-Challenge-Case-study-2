# Data Cleaning

## Customer_orders table

-- Changing empty strings and 'null' to NULL data type in the exclusions and extras columns

```SQL
ALTER TABLE customer_orders
ALTER COLUMN exclusions TYPE varchar(4)
USING NULLIF(exclusions, '') :: varchar(4);

ALTER TABLE customer_orders
ALTER COLUMN exclusions TYPE varchar(4)
USING NULLIF(exclusions, 'null') :: varchar(4);

ALTER TABLE customer_orders
ALTER COLUMN extras TYPE varchar(4)
USING NULLIF(extras, '') :: varchar(4);

ALTER TABLE customer_orders
ALTER COLUMN extras TYPE varchar(4)
USING NULLIF(extras, 'null') :: varchar(4);

```


**Old vs Altered Table**
	Old table	  	       | 	Altered table
:-------------------------------------:|:----------------------------------------------:
![](https://github.com/Berry-of-Tech/8-Week-SQL-Challenge-Case-study-2/blob/main/Old%20customer%20order%20table.png)|![](https://github.com/Berry-of-Tech/8-Week-SQL-Challenge-Case-study-2/blob/main/clean%20customer%20orders%20table.png)
---


--I created a temp table called temp_customer_orders to seperate the delimiter

```sql
DROP TABLE IF EXISTS temp_customer_orders;

CREATE TABLE temp_customer_orders (
  order_id INT,
  customer_id INT,
  pizza_id INT,
  exclusions VARCHAR(20),
  extras VARCHAR(20),
  order_time TIMESTAMP WITHOUT TIME ZONE
);


INSERT INTO temp_customer_orders
(order_id, customer_id, pizza_id, exclusions, extras, order_time)
SELECT 
  order_id, customer_id, pizza_id,
  UNNEST(STRING_TO_ARRAY(exclusions, ',')) AS exclusions,
  UNNEST(STRING_TO_ARRAY(extras, ',')) AS extras,
  order_time	
FROM customer_orders;

```

-- changing the exclusions and extras data type from varchar(20) to INT

```sql
ALTER TABLE temp_customer_orders
ALTER COLUMN exclusions TYPE INTEGER 
USING (exclusions::integer);

ALTER TABLE temp_customer_orders
ALTER COLUMN extras TYPE INTEGER 
USING (extras::integer);

```

-- Viewing the cleaned table

```sql
SELECT *
FROM temp_customer_orders
```
![](https://github.com/Berry-of-Tech/8-Week-SQL-Challenge-Case-study-2/blob/main/Temp%20customer%20orders%20table.png)
---

## Runner_orders table

--changing empty strings and "null" to NULL data type in various columns in the runner_orders table
```sql
ALTER TABLE runner_orders
ALTER COLUMN pickup_time TYPE varchar(19) 
USING NULLIF(pickup_time, 'null')::varchar(19),
ALTER COLUMN distance TYPE varchar(7)
USING NULLIF(distance, 'null')::varchar(7),
ALTER COLUMN duration TYPE varchar(10) 
USING NULLIF(duration, 'null')::varchar(10);

ALTER TABLE runner_orders 
ALTER COLUMN cancellation TYPE varchar(23) 
USING CASE 
         WHEN cancellation = 'Nan'
		 OR cancellation = 'null' THEN NULL
         ELSE NULLIF(cancellation, '') 
       END::varchar(23);
```

--changing pickup_time column to the correct data type
```sql
ALTER TABLE runner_orders
ALTER COLUMN pickup_time TYPE timestamp 
USING to_timestamp(pickup_time, 'YYYY-MM-DD HH24:MI:SS');
```

-- I need to remove non-numeric values in the duration and distance columns to make it consistent
```sql
UPDATE runner_orders
SET duration = REPLACE(REPLACE(REPLACE(duration, 'mins', ''), 'minutes', ''), 'minute', '')

UPDATE runner_orders
SET distance = REPLACE(distance, 'km', '')
```

-- Now I need to rename the duration and distance columns

```SQL
ALTER TABLE runner_orders
RENAME COLUMN duration TO duration_mins

ALTER TABLE runner_orders
RENAME COLUMN distance TO distance_km
```

-- I trimmed the extra spaces in both columns

```sql
UPDATE runner_orders
SET duration_mins = TRIM(duration_mins), 
	distance_km = TRIM(distance_km)

```

-- Changing the both columns data types to their appropriate data type

```SQL
ALTER TABLE runner_orders
ALTER COLUMN duration_mins TYPE INTEGER
USING duration_mins::INTEGER;

ALTER TABLE runner_orders
ALTER COLUMN distance_km TYPE FLOAT
USING distance_km::FLOAT;
```

-- Viewing the clean runner_orders table

~~~sql
SELECT *
FROM runner_orders
~~~

**Old vs Altered Table** 
		Old Table                                       | 	Altered Table
:--------------------------------------------------:|:------------------------------------------------:
![](https://github.com/Berry-of-Tech/8-Week-SQL-Challenge-Case-study-2/blob/main/Old%20runner%20orders%20table.png)|![](https://github.com/Berry-of-Tech/8-Week-SQL-Challenge-Case-study-2/blob/main/clean%20runner%20orders%20table.png)
