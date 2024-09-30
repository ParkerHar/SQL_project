# What issues will you address by cleaning the data?

>I will attempt to adjust incorrect data, remove duplicates, and use only the required amount of data to answer questions.
>

**Queries:**

>`Divide price values by 1000000 in sessions table`
```sql
SELECT	total_transaction_revenue/1000000 AS total_transaction_revenue,
	product_revenue/1000000 AS product_revenue,
	product_price/1000000 AS price
FROM	sessions
```
>`Changing country value to null when no country is available in sessions table`
```sql
SELECT	CASE 
		WHEN country LIKE '(not set)' THEN NULL
		ELSE country
		END AS country
FROM	sessions
WHERE	total_transaction_revenue IS NOT NULL	--only including countries
	AND	total_transaction_revenue > 0			--with revenue over zero
```

>`Changing city value to null when no city is available in sessions table`
```sql
SELECT	CASE 
		WHEN city IN ('(not set)','not available in demo dataset') THEN NULL
		ELSE city
		END AS city
FROM	sessions
WHERE	total_transaction_revenue IS NOT NULL 	--only including cities with
	AND	total_transaction_revenue > 0		 	--revenue over zero
```

>`Sessions table has almost all unique visit_ids. Removing row with duplicates on visitor id`
```sql
SELECT	COUNT(DISTINCT(visit_id)), --has duplicates on visitor id
		COUNT(visit_id)
FROM	sessions
--------------Created view clean_sessions to remove duplicates and store new clean clean_sessions table
CREATE VIEW clean_sessions AS
(
WITH dist_sessions AS(
SELECT	*,
		ROW_NUMBER() OVER (PARTITION BY visit_id) AS count_row
FROM	sessions
)

SELECT	*
FROM	dist_sessions
WHERE		count_row = 1
)
```

>`For average products ordered I only wanted to uses rows with products ordered > 0`
```sql
SELECT	*
FROM	clean_sessions
WHERE	product_quantity IS NOT NULL
```

>`combined (not count) and ${esccattitle} categories to N/A category in clean_sessions table`
```sql
SELECT	CASE
			WHEN	product_category_2 IN ('(not set)','${escCatTitle}')
			THEN	'N/A'
			ELSE	product_category_2
		END AS product_category
FROM	clean_sessions
```

