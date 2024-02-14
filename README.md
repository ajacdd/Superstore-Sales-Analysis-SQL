# Superstore-Analysis
- Data source: Tableau Sample Superstore
- Tools: SQL (MySQL), Tableau

## Introduction
Superstore is a U.S. based small retail business that sells furniture, office supplies, and technology products. Their customer segments are mass Consumers, Corporate, and Home Offices. The project aims to provide a thorough analysis of Superstore sales performance and trends to reveal key insights that can be used to guide data-driven decision-making. Using a detailed dataset, the project explores multiple facets of the business, including sales trends, product popularity, regional market performance, and more. This report highlights key metrics, analyzes category and product performance, examines regional sales trends, and offers strategic insights and recommendations to optimize sales and maintain a competitive edge in the industry.

## Requirements
The Superset dataset used in this project was obtained from Tableau and converted from an XSLX file to a CSV file for easier handling. The analysis was performed using MySQL and complemented using Tableau to create an interactive dashboard for data visualization.

The dataset contains 3 tables: orders, returns, and people. The “orders” table includes the following columns:
-	row_id: unique ID of each row
-	order_id: ID of each order placed
-	order_date: date on which an order was placed
-	ship_date: date on which an order shipped
-	ship_mode: Second Class, Standard Class, First Class, Same Day
-	customer_id: unique ID of a customer
-	customer_name: first and last name
-	segment: Consumer, Corporate, Home Office
-	country: United States
-	city: name of city
-	state: name of state
-	postal_code: 5-digit zip code
-	region: Central, East, South, West
-	product_id: unique ID of each product
-	category: Furniture, Office Supplies, Technology
-	subcategory: Accessories, Appliances, Art, Binders, Bookcases, Chairs, Copiers, Envelopes, Fasteners, Furnishings, Labels, Machines, Paper, Phones, Storage, Supplies, Tables
-	product_name: short description of product name
-	sales: revenue generated by the sale of goods
-	quantity: number of individual units sold
-	discount: discounts applied to units in an order
-	profit: profit generated from the sale of goods

The “returns” table includes the following columns:
-	returned: denotes 'Yes' where order was returned
-	order_id: ID of each order returned

The “people” table includes the following columns:
-	person: first and last name of regional sales person
-	region: Central, East, South, West

## Business Questions
1.	What is the yearly total sales, profit, and year-over-year (YOY) % variance?
2.	What is the total sales, average sales, and percent of total sales for each category?
3.	Which subcategories have the highest and lowest total profit overall?
4.	Which customer segments generated the most and least profits?
5.	What are the top 3 spending customers?
6.	How many orders were placed compared to orders returned each year?
7.	What is the total sales, value of returned orders, and net sales by region along with the associated regional salesperson?
8.	How many orders shipped 7+ days after the order date?

## Data Cleaning & Preparation
#### Duplicate values
After creating the database, tables, and importing values from the .csv file in MySQL, I checked for duplicate values. Duplicate values in a dataset can lead to skewed or inaccurate analyses by artificially inflating the size of the data and/or creating misleading results.

```
SELECT
	customer_name, COUNT(customer_name),
	order_date, COUNT(order_date),
	order_id, COUNT(order_id),
	customer_id, COUNT(customer_id),
	product_id, COUNT(product_id),
	sales, COUNT(sales),
	discount, COUNT(discount),
	quantity, COUNT(quantity)
FROM
	superstore.orders
GROUP BY
	customer_name, order_date, order_id, customer_id, product_id, sales, discount, quantity
HAVING
	COUNT(customer_name) > 1 
	AND COUNT(order_date) > 1 
	AND COUNT(order_id) > 1 
	AND COUNT(customer_id) > 1 
	AND COUNT(product_id) > 1 
	AND COUNT(sales) > 1 
	AND COUNT(discount) > 1 
	AND COUNT(quantity) > 1;
```

There appears to be 1 exact duplicate entry across all columns for customer "Laurel Beltran" (row_id 3406 and 3407). For the purpose of this project, I assumed this was an error and deleted the second entry. (Note: in a real-life business situation, I'd first want to confirm with the Order Management Team before removing.)

```
DELETE FROM superstore.orders
WHERE row_id = 3407;
```
#### Null values
Checking for null values in a dataset involves identifying any missing or undefined data in each column. Understanding the presence and distribution of null values helps in making informed decisions about how to handle them. There were no null values found in the dataset.

#### Update table
Next, I decided to update the orders table schema to add a new “order_year" column by extracting the year from order_date. This allows for easier filtering by year.
```
ALTER TABLE superstore.orders
ADD COLUMN order_year INT AFTER order_id;

UPDATE superstore.orders
SET order_year = YEAR(order_date);
```

## Data Analysis & Insights
1. Superstore experienced strong year-over-year (YOY) sales growth of 29.5% (+138.7K) in 2016 and 20.4% (+124K) in 2017, however, it had a YOY decline of 2.8% (-$13.4K) in 2015. 
```
WITH yearly_totals AS 
	(SELECT order_year, SUM(sales) AS total_sales, SUM(profit) AS total_profit
	FROM superstore.orders
	GROUP BY 1
	ORDER BY 1)

SELECT
	order_year, total_sales, total_profit,
	(total_sales/LAG(total_sales) OVER(ORDER BY order_year) - 1)*100 AS yoy_sales_perc_var,
	(total_profit/LAG(total_profit) OVER(ORDER BY order_year) - 1)*100 AS yoy_profit_perc_var
FROM yearly_totals;
```
| order_year | total_sales | total_profit | yoy_sales_perc_var | yoy_profit_perc_var |
|------------|-------------|--------------|--------------------|---------------------|
| 2014       | 483966.19   | 49556.06     |                    |                     |
| 2015       | 470532.46   | 61618.66     | -2.775758          | 24.341322           |
| 2016       | 609205.86   | 81795.23     | 29.471591          | 32.744253           |
| 2017       | 733215.19   | 93439.73     | 20.355899          | 14.23616            |

2.	Technology is the largest category with total sales of $836.2K, accounting for about 36% of Superstore's total business. The Furniture and Office Supplies categories each account for 32% and 31% of total sales, respectively. Average sales in the Technology category are roughly 1.3x that of the Furniture category and 3.8x that of the Office Supplies category. 
```
SELECT
	category, SUM(sales) AS total_sales, AVG(sales) AS avg_sales,
	SUM(sales)/(SELECT SUM(sales) FROM superstore.orders)*100 AS perc_of_total
FROM superstore.orders
GROUP BY 1
ORDER BY 2 DESC;
```
| category        | total_sales | avg_sales  | perc_of_total |
|-----------------|-------------|------------|---------------|
| Technology      | 836154.1    | 452.709312 | 36.40328      |
| Furniture       | 741718.61   | 349.867269 | 32.291882     |
| Office Supplies | 719046.99   | 119.324094 | 31.304838     |

3.	The most profitable subcategory is Copiers generating total profits of $55.6K. The least profitable subcategory is Tables which has a total profit loss of -$17.7K. The Tables subcategory is negatively contributing to Superstore's overall profitability and should be investigated as a potential subcategory to exit if profitability cannot be improved through more favorable cost structures.
```
(
	SELECT subcategory, SUM(profit) AS total_profit
	FROM superstore.orders
	GROUP BY 1
	ORDER BY 2 DESC
	LIMIT 1
)
UNION
(
	SELECT subcategory, SUM(profit) AS total_profit
	FROM superstore.orders
	GROUP BY 1
	ORDER BY 2 ASC
	LIMIT 1
);
```
| subcategory | total_profit |
|-------------|--------------|
| Copiers     | 55617.9      |
| Tables      | -17725.59    |

4.	The most profitable segment is mass Consumers with a total profit of $134.1K, while the least profitable segment is Home Office with $60.3K in profits. Profits in the Consumer segment are about 2.2x that of Home Office; this is a strong indicator that mass Consumers should continue to be a strategic focus for Superstore.
```
SELECT segment, SUM(profit) AS total_profit
FROM superstore.orders
GROUP BY 1
ORDER BY 2 DESC;
```
| segment     | total_profit |
|-------------|--------------|
| Consumer    | 134119.23    |
| Corporate   | 91979.41     |
| Home Office | 60311.04     |

5.	The top 3 spending customers are Sean Miller ($25K), Tamara Chand ($19K), and Raymond Buch ($15K). Superstore's Sales & Marketing teams can strengthen customer retention and increase customer lifetime value through personalized marketing campaigns and loyalty programs that reward top spenders with highly desirable incentives. 
```
SELECT *
FROM 
	(SELECT customer_id, customer_name, SUM(sales) AS total_spend,
		DENSE_RANK() OVER(ORDER BY SUM(sales) DESC) AS top_rank_customers
	FROM superstore.orders
	GROUP BY 1, 2) AS t1
WHERE top_rank_customers <= 3;
```
| customer_id | customer_name | total_spend | top_rank_customers |
|-------------|---------------|-------------|--------------------|
| SM-20320    | Sean Miller   | 25043.07    | 1                  |
| TC-20980    | Tamara Chand  | 19052.22    | 2                  |
| RB-19360    | Raymond Buch  | 15117.35    | 3                  |

6.	While it's important to acknowledge the increasing number of orders each year, it's also important to look at how many of those orders were returned. In three years, "order return rate" jumped from 5.4% in 2014 to 6.2% in 2017. Further investigation should be done to determine if the increase in the rate of returned orders is due to product quality issues, unclear/misleading advertisements, incorrect products, lack of customer support, increased competition, or other factors.
```
SELECT
	order_year,
    COUNT(DISTINCT o.order_id) AS num_orders_placed, 
    COUNT(DISTINCT r.order_id) AS num_orders_returned,
    COUNT(DISTINCT r.order_id)/COUNT(DISTINCT o.order_id)*100 AS perc_returned
FROM superstore.orders o
LEFT JOIN superstore.returns r ON o.order_id = r.order_id
GROUP BY 1
ORDER BY 1;
```
| order_year | num_orders_placed | num_orders_returned | perc_returned |
|------------|-------------------|---------------------|----------------|
| 2014       | 969               | 53                  | 5.4696         |
| 2015       | 1038              | 61                  | 5.8767         |
| 2016       | 1315              | 77                  | 5.8555         |
| 2017       | 1687              | 105                 | 6.2241         |

7.	East Regional Sales Manager, Chuck Magee, achieved the highest net sales after returns ($636.8K). While West Regional Sales Manager, Anna Andreadi, is #1 in gross sales, she also has the most returned orders ($107.5K) accounting for 14.8% of her total sales. 
```
SELECT
	o.region, p.person, SUM(sales) AS total_sales,
	SUM(CASE WHEN r.returned = 'Yes' THEN sales END) AS total_return_value,
    SUM(sales)-SUM(CASE WHEN r.returned = 'Yes' THEN sales END) AS net_sales,
    SUM(CASE WHEN r.returned = 'Yes' THEN sales END)/SUM(sales)*100 AS percent_of_total_sales_returned
FROM superstore.orders o
LEFT JOIN superstore.returns r ON o.order_id = r.order_id
LEFT JOIN superstore.people p ON o.region = p.region
GROUP BY 1, 2
ORDER BY net_sales DESC;
```
| region  | person            | total_sales | total_return_value | net_sales | percent_of_total_sales_returned |
|---------|-------------------|-------------|--------------------|-----------|---------------------------------|
| East    | Chuck Magee       | 678499.99   | 41705.12           | 636794.87 | 6.146665                        |
| West    | Anna Andreadi     | 725457.93   | 107483.06          | 617974.87 | 14.815892                       |
| Central | Kelly Williams    | 501239.88   | 14006.99           | 487232.89 | 2.794468                        |
| South   | Cassandra Brandow | 391721.9    | 17309.13           | 374412.77 | 4.418729                        |

8.	Shipping and delivering orders on time is essential to a thriving business. When shipments are late, customers are much less likely to continue purchasing orders and may switch to a competitor that can guarantee timely deliveries. At Superstore, 6% of orders left the warehouse 7+ days after the original order was placed. Shipping this late doesn't create a positive customer experience. A deeper root cause analysis should be performed to uncover contributing factors such as inventory in-stock levels, labor shortage, or other reasons.
```
WITH long_orders AS
	(SELECT order_year, order_id, product_id, order_date, ship_date, DATEDIFF(ship_date, order_date) AS order_to_ship_time
    FROM superstore.orders)
    
SELECT
    (SELECT COUNT(DISTINCT order_id) FROM superstore.orders) AS total_orders,
    COUNT(DISTINCT order_id) AS num_long_orders,
    COUNT(DISTINCT order_id)/(SELECT COUNT(DISTINCT order_id) FROM superstore.orders)*100 AS perc_long_orders
FROM long_orders
WHERE order_to_ship_time >= 7;
```
| total_orders | num_long_orders | perc_long_orders |
|--------------|-----------------|------------------|
| 5009         | 308             | 6.1489           |

## Data Visualization
As a complement to this project, I created a [Superstore Sales Dashboard in Tableau Public linked here](https://public.tableau.com/views/SuperstoreSalesOverviewDashboard_17055191173420/SuperstoreSalesDashboard?:language=en-US&:display_count=n&:origin=viz_share_link). This dashboard provides a graphical representation of the key metrics most important to stakeholders at Superstore.

![image of Superstore Sales Overview Dashboard](https://github.com/ajacdd/Superstore-Analysis/blob/main/Superstore%20Sales%20Dashboard.png)

