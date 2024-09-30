# What are your risk areas? Identify and describe them.

There is a glaring issue with this data set. There is no way to use sales data from either the sales_report table or the sales_by_sku table to break down sales by country or city. This caused me to use only the sessions table for the questions asked which proved to be unreliable. 

# QA Process:

To start I wanted to identify a primary key to be used. Sku looked to be a good candidate and I determined the sku from products would be the PK as they were all unique. Any other table containing sku would have sku as a foreign key 

```sql
SELECT	COUNT(DISTINCT(sku)),
		COUNT(sku)
FROM	products
```

I then looked into breaking down the sales tables by city and country to answer the provided questions. This proved to be a blocker as I could not find a way to break down the sales data by anything other than sku. Example query shown below:

```sql
SELECT	cs.sku,
		cs.country,
		cs.city,
		cs.product_price,
		sbs.total_ordered
FROM	clean_sessions 	cs
JOIN	sales_by_sku 	sbs
ON		cs.sku = sbs.sku
```
This lead me to use the sessions table for the remainder of this project. 

I then wanted to compare the total products sold in sales_report to the total products sold in the sessions table. 

```sql
SELECT	SUM(total_ordered)
FROM	sales_report

SELECT	SUM(product_quantity)
FROM	clean_sessions
```

The totals were very far apart (5519 from sales_report and 188 from sessions). This indicates that the sessions table is very incomplete leading to skewed and unreliable answers.
Knowing that the sessions table was incomplete, I still chose to use it moving forward as I could not see a better way to break down the data by country or city.

