Answer the following questions and provide the SQL queries used to find the answer.

    
**Question 1: Which cities and countries have the highest level of transaction revenues on the site?**

**SQL Queries:**

>`Countries with highest transaction revenues`

```sql
WITH	raw_country_rev AS 	--Find sums for all three revenue options (total_transaction_revenue, product_revenue, and
							    -- product_quantity * product_price) seperately
		(SELECT	CASE
		 --Changing country value to null when no city is available
					WHEN country LIKE '(not set)' THEN NULL
					ELSE country
				END AS country,
				SUM(total_transaction_revenue/1000000) AS trev,
				SUM(product_revenue/1000000) AS prev,
				SUM((product_quantity * product_price)/1000000) AS orev
		FROM	clean_sessions
		GROUP BY	country),

		clean_country_rev AS 
		        --convert null sums from above to zeros for final summation of revenue
		(SELECT	country,
				COALESCE(trev,0) AS total_rev,
				COALESCE(prev,0) AS prod_rev,
				COALESCE(orev,0) AS ord_rev
		FROM	raw_country_rev)

SELECT 	country,
		total_rev + prod_rev + ord_rev AS revenue
FROM	clean_country_rev
ORDER BY	revenue DESC
LIMIT 	10
```

>`Cities with highest transaction revenues`
```sql
WITH	raw_city_rev AS 	--Find sums for all three revenue options (total_transaction_revenue, product_revenue, and
						    	-- (product_quantity * product_price) seperately

		(SELECT	CASE 
		        --Changing city value to null when no city is available
					WHEN city IN ('(not set)','not available in demo dataset') THEN NULL
					ELSE city
				END AS city,
				SUM(total_transaction_revenue/1000000) AS trev,
				SUM(product_revenue/1000000) AS prev,
				SUM((product_quantity * product_price)/1000000) AS orev
		FROM	clean_sessions
		GROUP BY	city),

		clean_city_rev AS --convert nulls to zeros for final summation of revenue
		
		(SELECT	city,
				COALESCE(trev,0) AS total_rev,
				COALESCE(prev,0) AS prod_rev,
				COALESCE(orev,0) AS ord_rev
		FROM	raw_city_rev)

SELECT 	city,
		total_rev + prod_rev + ord_rev AS revenue
FROM	clean_city_rev
ORDER BY	revenue DESC
LIMIT 	10
```

**Answer:**

The top 3 countries by revenue are:
>1. United States | $18632.32
2. Israel | $602.00
3. Australia | $358.00
>
The top 3 cities by revenue are:
>1. San Francisco | $1832.32
2. Sunnyvale | $1430.23
3. New York | $1188.91
>

***Observations:***

-With no clear path for with column to use to calculate revenue a combination approach was taken. 

-NULL cities would have placed first on the cities list with $6092.56 which shows that
 Which shows that the null values make up a significant portion of the dataset.


**Question 2: What is the average number of products ordered from visitors in each city and country?**

**SQL Queries:**

>`Average number of products by country`
```sql
SELECT	CASE 
            --Changing country value to null when no country is available
			WHEN country LIKE '(not set)' THEN NULL
			ELSE country
		END AS country,
		AVG(product_quantity) AS avg_prod_ordered
FROM	clean_sessions
WHERE	product_quantity IS NOT NULL
GROUP BY	country
ORDER BY	avg_prod_ordered DESC
```

>`Average number of products by city`
```sql
SELECT	CASE 
            --Changing city value to null when no city is available
			WHEN city IN ('(not set)','not available in demo dataset') THEN NULL
			ELSE city
		END AS city,
		AVG(product_quantity) AS avg_prod_ordered
FROM	clean_sessions
WHERE	product_quantity IS NOT NULL
GROUP BY	city
ORDER BY	avg_prod_ordered DESC
```

**Answer:**

For countries, Spain has the highest average with 10, followed by the United States and Colombia with 4.1 and 1.0 respectively.

For cities, Madrid had the highest average with 10 items per order. Salem and Atlanta followed with 8 and 4 respectively.

***Observations:***

Spain had one order with 10 items ordered. This caused it to have a high relative average. More data needed to determine if Spain will stay ranked #1 over time.

There are many null city names in this data causing the results to be in-comprehensive. The null cities would have ranked third on this list with 
6.75 average items per order.


**Question 3: Is there any pattern in the types (product categories) of products ordered from visitors in each city and country?**


**SQL Queries:**

`Product types by country`
```sql
WITH country_product_count AS
		(SELECT	CASE 
					WHEN country LIKE '(not set)' THEN NULL
					ELSE country
				END AS country,
				CASE
					WHEN	product_category_2 IN ('(not set)','${escCatTitle}')
					THEN	'N/A'
					ELSE	product_category_2
				END AS product_category,
				SUM(product_quantity) AS number_ordered,
				RANK() OVER (PARTITION BY country ORDER BY SUM(product_quantity) DESC) AS order_rank
				--Ranked rows by number of products ordered in that category
				FROM	clean_sessions
		WHERE	product_quantity IS NOT NULL
		GROUP BY	country,
					product_category_2
		ORDER BY	country,
					number_ordered DESC)

SELECT	*
FROM	country_product_count
--rank column used here
WHERE	order_rank = 1 
ORDER BY number_ordered DESC
```


`Product types by city`
```sql
WITH city_product_ordered AS
		(SELECT	CASE --Changing city value to null when no city is available
					WHEN city IN ('(not set)','not available in demo dataset') THEN NULL
					ELSE city
				END AS city,
				CASE
					WHEN	product_category_2 IN ('(not set)','${escCatTitle}')
					THEN	'N/A'
					ELSE	product_category_2
				END AS product_category,		
				SUM(product_quantity) AS number_ordered,
				RANK() OVER (PARTITION BY city ORDER BY SUM(product_quantity) DESC) AS order_rank
				--Ranked rows by number of products ordered in that category
		FROM	clean_sessions
		WHERE	product_quantity IS NOT NULL
		GROUP BY	city,
					product_category_2
		ORDER BY	city,
					number_ordered)

SELECT	*
FROM	city_product_ordered
--rank column used here
WHERE	order_rank = 1 
ORDER BY number_ordered DESC
```


**Answer:**

The USA is by far our largest market with many purchases of in the notebooks & journals, and Bags categories. Data is insufficient to determine trends for other countries.

City data is too incomplete to develop patterns on at this point and I would like to further explore how to pull in city data if we had more time.



**Question 4: What is the top-selling product from each city/country? Can we find any pattern worthy of noting in the products sold?**


**SQL Queries:**

>`Top product by Country`
```sql
WITH country_order_count AS
(SELECT	CASE 
		--Changing city value to null when no city is available
			WHEN country LIKE '(not set)' THEN NULL
			ELSE country
		END AS country,
		SUM(product_quantity) AS num_bought,
		RANK() OVER (PARTITION BY country ORDER BY SUM(product_quantity) DESC) AS order_rank,
				--order_rank column added to be used in next select statement
		product_name_2
FROM	clean_sessions
GROUP BY	country,
			product_name_2
HAVING	SUM(product_quantity) IS NOT NULL	
ORDER BY	order_rank,
			country,
			num_bought DESC)

SELECT	*
FROM	country_order_count
--rank column used here
WHERE	order_rank = 1 
ORDER BY num_bought DESC
```

>`Top Product by city`
```sql
WITH city_order_count AS
(SELECT	CASE 
		--Changing city value to null when no city is available
			WHEN city IN ('(not set)','not available in demo dataset') THEN NULL
			ELSE city
		END AS city,
		SUM(product_quantity) AS num_bought,
		RANK() OVER (PARTITION BY city ORDER BY SUM(product_quantity) DESC) AS order_rank,
				--order_rank column added to be used in next select statement
		product_name_2
FROM	clean_sessions
GROUP BY	city,
			product_name_2
HAVING	SUM(product_quantity) IS NOT NULL	
ORDER BY	order_rank,
			city,
			num_bought DESC)

SELECT	*
FROM	city_order_count
WHERE	order_rank = 1
ORDER BY	num_bought DESC
```


**Answer:**
There is not much of a pattern to draw with this data set for either countries or cities
The only countries with multiple product orders were the United States and Spain. The US has a demand for Leatherette Journals, and Spain has demand for dress socks.

The incomplete city data present in the original data set meant the city data was also difficult to find any patterns in.
Madrid does have a demand for Dress socks, and Salem has a demand for Red Spiral Google Notebooks.


**Question 5: Can we summarize the impact of revenue generated from each city/country?**

**SQL Queries:**
>`Revenue impact by Country`
```sql
WITH	raw_country_rev AS 	--Find sums for all three revenue options (total_transaction_revenue, product_revenue, and
							-- product_quantity * product_price) seperately
		(SELECT	CASE 
					WHEN country LIKE '(not set)' THEN NULL
					ELSE country
				END AS country,
				SUM(total_transaction_revenue/1000000) AS trev,
				SUM(product_revenue/1000000) AS prev,
				SUM((product_quantity * product_price)/1000000) AS orev
		FROM	clean_sessions
		GROUP BY	country),

		clean_country_rev AS --convert null sum from above to zeros for final summation of revenue
		(SELECT	country,
				COALESCE(trev,0) AS total_rev,
				COALESCE(prev,0) AS prod_rev,
				COALESCE(orev,0) AS ord_rev
		FROM	raw_country_rev)

SELECT 	country,
		total_rev + prod_rev + ord_rev AS revenue,
		(total_rev + prod_rev + ord_rev)/(SUM(total_rev + prod_rev + ord_rev) OVER ()) * 100 AS percent_revenue
		--finding a countries revenue as a percentage of total revenue
FROM	clean_country_rev
ORDER BY	revenue DESC
```

>`Revenue impact by City`
```sql
WITH	raw_city_rev AS 	--Find sums for all three revenue options (total_transaction_revenue, product_revenue, and
							-- (product_quantity * product_price) seperately

		(SELECT	CASE --Changing city value to null when no city is available
					WHEN city IN ('(not set)','not available in demo dataset') THEN NULL
					ELSE city
				END AS city,
				SUM(total_transaction_revenue/1000000) AS trev,
				SUM(product_revenue/1000000) AS prev,
				SUM((product_quantity * product_price)/1000000) AS orev
		FROM	clean_sessions
		GROUP BY	city),

		clean_city_rev AS --convert nulls to zeros for final summation of revenue
		
		(SELECT	city,
				COALESCE(trev,0) AS total_rev,
				COALESCE(prev,0) AS prod_rev,
				COALESCE(orev,0) AS ord_rev
		FROM	raw_city_rev)

SELECT 	city,
		total_rev + prod_rev + ord_rev AS revenue,
		(total_rev + prod_rev + ord_rev)/(SUM(total_rev + prod_rev + ord_rev) OVER ()) * 100 AS percent_revenue
		--finding a countries revenue as a percentage of total revenue
FROM	clean_city_rev
ORDER BY	revenue DESC
LIMIT 	10
```

**Answer:**

For countries, the united states has by far the largest impact on revenue with a %92.34 share of revenue generation. Israel and Australia follow as a distant 2nd and 3rd, with %2.98 and %1.78 respectively.

For cities, San Fransisco had the largest impact on revenue with %9.01. Sunnyvale and New York followed with %7.09 and %5.90 respectively.

***Observations:***

The null city data again made it hard to have confidence in analysis by city. The NULLs made up %42.9 of revenue generated.


