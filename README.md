# SQL-customer_analysis

## Introduction 

The **dataset used** in this project are available for access [here](/database).

## Objective 


## Preliminary preparation 

```sql
CREATE VIEW cohort_analysis
AS WITH customer_revenue AS (
         SELECT s.customerkey,
            s.orderdate,
            sum(s.quantity::double precision * s.netprice * s.exchangerate) AS net_revenue,
            count(s.orderdate) AS num_orders,
            c.countryfull,
            c.age,
            c.givenname,
            c.surname
           FROM sales s
             LEFT JOIN customer c ON s.customerkey = c.customerkey
          GROUP BY c.countryfull, c.age, c.givenname, c.surname, c.customerkey, s.customerkey, s.orderdate
          ORDER BY s.customerkey
        )
 SELECT customerkey,
    orderdate,
    net_revenue,
    num_orders,
    countryfull,
    age,
    CONCAT(TRIM(givenname),' ',TRIM(surname)) AS cleaned_name,
    min(orderdate) OVER (PARTITION BY customerkey) AS first_purchase_date,
    EXTRACT(year FROM min(orderdate) OVER (PARTITION BY customerkey)) AS cohort_year
   FROM customer_revenue cr;
```
## Project 1 

```sql
SELECT
	cohort_year,
	COUNT(DISTINCT customerkey) AS total_customers,
	SUM(net_revenue) AS total_revenue,
	SUM(net_revenue) / COUNT(DISTINCT customerkey) AS customer_revenue
FROM
	cohort_analysis
GROUP BY
	cohort_year
```

<div align="center">
  <img src="screenshots/project_1.png" alt="image_10" width="600" height="100" />
</div>


## Project 2



```sql
WITH customer_ltv AS (
	SELECT
		customerkey,
		cleaned_name,
		SUM(net_revenue) AS total_ltv
	FROM cohort_analysis
	
	GROUP BY 
		customerkey,
		cleaned_name
	ORDER BY customerkey
), customer_segments AS (
SELECT
	PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY total_ltv) AS ltv_25th_percentile,
	PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY total_ltv) AS ltv_75th_percentile
FROM customer_ltv
), segment_values AS (
SELECT 
	c.*,
	CASE
		WHEN c.total_ltv < cs.ltv_25th_percentile THEN '1 - Low-Value '
		WHEN c.total_ltv > cs.ltv_75th_percentile THEN '3 - High-Value '
		ELSE '2 - Mid-Value'
	END AS customer_segment
	
FROM
	customer_ltv c,
	customer_segments cs
)

SELECT
	customer_segment,
	SUM(total_ltv) AS total_ltv,
	COUNT(customerkey) AS customer_count,
	SUM(total_ltv) / COUNT(customerkey) AS avg_ltv
FROM
	segment_values 
GROUP BY 
	customer_segment
```

<div align="center">
  <img src="screenshots/project_2.png" alt="image_10" width="600" height="100" />
</div>

## Project 3 



```sql
WITH customer_last_purchase AS (
SELECT
	customerkey,
	cleaned_name,
	orderdate,
	ROW_NUMBER() OVER( PARTITION BY customerkey ORDER BY orderdate DESC) AS rn,
	first_purchase_date,
	cohort_year
FROM
	cohort_analysis
), churned_customers AS (
	SELECT
		customerkey,
		cleaned_name,
		first_purchase_date,
		orderdate AS last_purchase_date,
		CASE
			WHEN orderdate < (SELECT MAX(orderdate) FROM sales) - INTERVAL '6 months'  THEN 'Churned'
			ELSE 'Active'
		END AS customer_status,
		cohort_year
	
	FROM customer_last_purchase
	WHERE rn = '1'
		AND first_purchase_date < (SELECT MAX(orderdate) FROM sales) - INTERVAL '6 months'
)

SELECT
	customer_status,
	COUNT(customerkey) AS num_customers,
	SUM(COUNT(customerkey)) OVER( PARTITION BY cohort_year) AS total_customerss,
	ROUND(COUNT(customerkey) / SUM(COUNT(customerkey)) OVER(PARTITION BY cohort_year), 2) AS percent_of_total,
	cohort_year
FROM
	churned_customers 
GROUP BY
		cohort_year,
		customer_status

```

<div align="center">
  <img src="screenshots/project_3_2.png" alt="image_10" width="600" height="100" />
</div>


<div align="center">
  <img src="screenshots/project_3_1.png" alt="image_10" width="700" height="500" />
</div>



## How to Use  

1. **Install dependencies**: pandas, matplotlib, numpy.
2. **Load datasets**: use pd.read_csv() and adjust the file path/directory as needed.


## Author  
Created by **Arsen Pankiv**  
- [LinkedIn](https://www.linkedin.com/in/arsen-pankiv-6082b4349/)  
- [GitHub](https://github.com/Arsen-Pankiv)
