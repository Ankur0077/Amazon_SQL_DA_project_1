---

# **Amazon USA Sales Analysis Project**

### **Difficulty Level: Advanced**

---

## **Project Overview**

I have worked on analyzing a dataset of over 20,000 sales records from an Amazon-like e-commerce platform. This project involves extensive querying of customer behavior, product performance, and sales trends using PostgreSQL. Through this project, I have tackled various SQL problems, including revenue analysis, customer segmentation, and inventory management.

The project also focuses on data cleaning, handling null values, and solving real-world business problems using structured queries.

An ERD diagram is included to visually represent the database schema and relationships between tables.

---

![ERD Scratch](https://github.com/najirh/amazon_usa_project5/blob/main/erd2.png)

## **Database Setup & Design**

### **Schema Structure**

```sql
Create Table category (
category_id Int Primary Key,
category_name Varchar(25)
);

-- Customers Table
Create Table customers(
Customer_ID	Int Primary key,
first_name	varchar(25),
last_name varchar(25),	
state varchar(25),
address varchar(20) Default ('xxxx') 
);

-- Inventory Table
Create Table inventory(
inventory_id Int primary key,
product_id int,
stock int,	
warehouse_id int,
last_stock_date Date,
CONSTRAINT products_fk_inventory FOREIGN KEY(product_id) REFERENCES products(product_id)
);

-- order_items table
Create Table order_items(
order_item_id	Int Primary key,
order_id int,  -- FK
product_id int,	 -- FK
quantity int,
price_per_unit Double,
CONSTRAINT orders_fk_order_items FOREIGN KEY(order_id) REFERENCES orders(order_id),
CONSTRAINT products_fk_order_items FOREIGN KEY(product_id) REFERENCES products(product_id)

);

-- orders table
Create table orders (
order_id int Primary key,
order_date Date,
customer_id int, -- FK
CONSTRAINT customer_fk_orders FOREIGN KEY(customer_id) REFERENCES customers(Customer_id) ,
seller_id int, -- FK
CONSTRAINT sellers_fk_orders FOREIGN KEY(seller_id) REFERENCES sellers(seller_id),
order_status Varchar(20));

-- payments table
Create Table payments (
payment_id	int Primary key,
order_id int,
payment_date	Date,
payment_status Varchar(25),
CONSTRAINT orders_fk_payments FOREIGN KEY(order_id) REFERENCES orders(order_id));

-- products table
Create table products(
product_id int Primary Key,
product_name varchar(50),
price double,
cogs double,
category_id int , -- Foreign key
CONSTRAINT product_fk_category FOREIGN KEY(category_id) References category(category_id) );

-- sellers table
Create table sellers(
seller_id int Primary Key,
seller_name Varchar(20),
origin varchar(20));

-- shipping table
Create table shipping(
shipping_id	 int primary key,
order_id int,
shipping_date Date,
return_date Date,
shipping_providers varchar(20),
delivery_status varchar(20),
CONSTRAINT orders_fk_shipping FOREIGN KEY(order_id) REFERENCES orders(order_id)
);

```

---

## **Task: Data Cleaning**

I cleaned the dataset by:
- **Removing duplicates**: Duplicates in the customer and order tables were identified and removed.
- **Handling missing values**: Null values in critical fields (e.g., customer address, payment status) were either filled with default values or handled using appropriate methods.

---

## **Handling Null Values**

Null values were handled based on their context:
- **Customer addresses**: Missing addresses were assigned default placeholder values.
- **Payment statuses**: Orders with null payment statuses were categorized as “Pending.”
- **Shipping information**: Null return dates were left as is, as not all shipments are returned.

---

## **Objective**

The primary objective of this project is to showcase SQL proficiency through complex queries that address real-world e-commerce business challenges. The analysis covers various aspects of e-commerce operations, including:
- Customer behavior
- Sales trends
- Inventory management
- Payment and shipping analysis
- product performance
  

## **Identifying Business Problems**

Key business problems identified:
1. Low product availability due to inconsistent restocking.
2. High return rates for specific product categories.
3. Significant delays in shipments and inconsistencies in delivery times.
4. High customer acquisition costs with a low customer retention rate.

---

## **Solving Business Problems**

### Solutions Implemented:
1. Top Selling Products
Query the top 10 products by total sales value.
Challenge: Include product name, total quantity sold, and total sales value.

```sql
select
  p.product_name,
  SUM(quantity) as total_qty_sold ,
  ROUND(SUM(price*quantity),2) as total_sales_value,
  COUNT(order_id) as total_orders
FROM order_items
RIGHT JOIN products p
on order_items.product_id = p.product_id
GROUP BY product_name
ORDER BY total_sales_value DESC
LIMIT 10;
```

2. Revenue by Category
Calculate total revenue generated by each product category.
Challenge: Include the percentage contribution of each category to total revenue.

```sql
With cte as
 (SELECT category_name
  ,ROUND(SUM(price * quantity),2) as total_sales
FROM category c
JOIN products p
ON c.category_id =p.category_id
INNER JOIN order_items oi
ON oi.product_id = p.product_id
GROUP BY category_name
ORDER BY total_sales desc),

cte2 as (
SELECT *,
  SUM(total_sales) OVER() as total_sales_from_portal FROM cte)

SELECT
  category_name, total_sales, CONCAT(ROUND((total_sales*100.0/total_sales_from_portal),2),"%")  as percent_contribution
from cte2;
```

3. Average Order Value (AOV)
Compute the average order value for each customer.
Challenge: Include only customers with more than 5 orders.

```sql
WITH cte as
(SELECT c.customer_id,
 first_name,
 last_name,
 round(avg(quantity * price_per_unit),2) as aov,
 count(oi.order_id) as total_orders
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
JOIN order_items oi ON oi.order_id  = o.order_id
GROUP BY customer_id)

SELECT
  customer_id,
  concat(first_name, " ", last_name) as Customer_name, aov,total_orders
FROM cte
WHERE total_orders >5 
ORDER BY aov desc;
```

4. Monthly Sales Trend
Query monthly total sales over the past year.
Challenge: Display the sales trend, grouping by month, return current_month sale, last month sale!

```sql
WITH cte as
 (Select EXTRACT(year from order_date) as year,
   EXTRACT(month from order_date) as month, 
   ROUND(SUM(quantity * price_per_unit),2) as total_sales
FROM orders o 
JOIN order_items oi 
ON o.order_id = oi.order_id
WHERE o.order_date >= current_date - Interval 1 year
GROUP BY year, month
ORDER BY year, month ),

cte2 as (
 select year,
   month, total_sales as current_month_sales, 
   LAG(total_sales,1,0) OEVR(order by year, month) as prev_month_sales
 FROM cte)
 
 select *,
   ROUND((current_month_sales -prev_month_sales)*100.0/prev_month_sales,2) as monthly_growth_percentage
 FROM cte2;

```


5. Customers with No Purchases
Find customers who have registered but never placed an order.
Challenge: List customer details and the time since their registration.

```sql
SELECT
 distinct c.customer_id,
 CONCAT(first_name, " ", last_name) as customer_name
FROM customers c
LEFT JOIN orders on 
c.customer_id = orders.customer_id
WHERE order_id is null;
```

6. BEST-Selling Categories by State
Identify the BEST-selling product category for each state.
Challenge: Include the total sales for that category within each state.

```sql
WITH cte as 
(SELECT state,
  category_name,
  SUM(price_per_unit * quantity) as total_sales,
  ROW_NUMBER() OVER(PARTITION BY state
                          ORDER BY SUM(price_per_unit * quantity) DESC) as rn
FROM category c
INNER JOIN products p ON c.category_id = p.category_id
INNER JOIN order_items oi ON oi.product_id = p.product_id
INNER JOIN orders o oON o.order_id = oi.order_id
INNER JOIN customers ON o.customer_id = customers.Customer_ID
GROUP BY state,category_name
ORDER BY state, total_sales desc)

SELECT
 state,
 category_name,
 total_sales from cte WHERE rn=1;
```

6. 2nd Part of BEST-Selling Categories by State
Identify the BEST-selling product category for each state along with top selling product in each category 
Challenge: Include the total sales for that category within each state.

```sql
WITH cte as (
SELECT
  state,
  category_name,
  SUM(price_per_unit * quantity) as total_sales,
  product_name,
  ROW_NUMBER() OVER(PARTITION BY state
				            ORDER BY SUM(price_per_unit * quantity) DESC) as rn 
FROM
category c
INNER JOIN products p ON c.category_id = p.category_id
INNER JOIN order_items oi ON oi.product_id = p.product_id
INNER JOIN orders o ON o.order_id = oi.order_id
INNER JOIN customers ON o.customer_id = customers.Customer_ID
GROUP BY state,category_name,product_name
ORDER BY state, total_sales desc

)
SELECT
  state,
  category_name,
  product_name,
  total_sales FROM cte WHERE rn=1;
```



7. Customer Lifetime Value (CLTV)
Calculate the total value of orders placed by each customer over their lifetime.
Challenge: Rank customers based on their CLTV.

```sql
SELECT
  customers.customer_id,
  CONCAT(first_name, " ",last_name) as Customer_name,
  ROUND(SUM(quantity * price_per_unit),2) as total_order_value,
  DENSE_RANK() OVER(order by SUM(quantity * price_per_unit) DESC) as cx_ranking
FROM customers 
INNER JOIN orders ON customers.Customer_ID = orders.customer_id
INNER JOIN order_items oi
ON orders.order_id = oi.order_id
GROUP BY customers.customer_id
ORDER BY total_order_value DESC;
```


8. Inventory Stock Alerts
Query products with stock levels below a certain threshold (e.g., less than 10 units).
Challenge: Include last restock date and warehouse information.

```sql
SELECT 
  inventory_id, 
  product_name, 
  stock as current_stock_left,
  last_stock_date, 
  warehouse_id
FROM inventory
INNER JOIN products on inventory.product_id = products.product_id
WHERE stock <=10;
```

9. Shipping Delays
Identify orders where the shipping date is after 3 days from the order date.
Challenge: Include customer, order details, and delivery provider.

```sql
SELECT
 customers.customer_id,
 concat(first_name, " ", last_name) as customer_name,
 orders.order_id,
 order_date,
 shipping_date,
 shipping_providers
FROM customers
INNER JOIN orders ON customers.customer_id = orders.customer_id
INNER JOIN shipping ON shipping.order_id = orders.order_id
WHERE shipping_date > order_date + interval 3 day;
```

10. Payment Success Rate 
Calculate the percentage of successful payments across all orders.
Challenge: Include breakdowns by payment status (e.g., failed, pending).

```sql
SELECT
   payment_status,
   COUNT(payment_status) as total_cnt, 
   COUNT(*) *100.0/ (SELECT COUNT(*) FROM payments) as prcnt_count
FROM payments
INNER JOIN orders ON payments.order_id = orders.order_id
GROUP BY payment_status;
```

11. Top Performing Sellers
Find the top 5 sellers based on total sales value.
Challenge: Include both successful,failed orders,returned,Inprogress order count

```sql
 WITH cte as
  (
  SELECT
    o.seller_id,
    s.seller_name,
    order_status,
    count(*) as order_count ,
    sum(quantity*price_per_unit) as total_sales
  FROM sellers s
  INNER JOIN orders  o ON  s.seller_id = o.seller_id
  INNER JOIN order_items oi ON oi.order_id = o.order_id
  GROUP BY s.seller_id, s.seller_name, order_status
  ORDER BY order_count desc),

 CTE2 as (
 SELECT
   o.seller_id,
   s.seller_name,
   count(*) as total_order_count
 FORM sellers s
 INNER JOIN orders  o on  s.seller_id =o.seller_id
 INNER JOIN order_items oi on oi.order_id = o.order_id
 GROUP BY s.seller_id, s.seller_name
 ORDER BY total_order_count desc)
 
SELECT
  seller_id,
  seller_name,
  MAX(CASE when order_status = "Completed" then order_count end) as completed_orders,
  COALESCE(MAX(case when order_status = "returned" then order_count end) ,0)as returned_order,
  COALESCE(MAX(case when order_status = "cancelled" then order_count end),0) as cancelled_orders,
  COALESCE(MAX(case when order_status = "Inprogress" then order_count end),0) as in_projress_orders,
  ROUND(MAX(CASE when order_status = "Completed" then order_count * total_sales end)/1000000,2) as completed_osv_in_million
 FROM cte
 GROUP BY seller_id, seller_name
 LIMIT 5;
```


12. Product Profit Margin
Calculate the profit margin for each product (difference between price and cost of goods sold).
Challenge: Rank products by their profit margin, showing highest to lowest.
*/


```sql
SELECT
   product_name,  
   round(sum((quantity * price) - (quantity * cogs)),2) as profit,
   round(sum((quantity * price) - (quantity * cogs))/sum((quantity*cogs))*100.0 ,2)    as profit_margin,
   dense_rank() over(order by round(sum((quantity * price) - (quantity * cogs)),2) desc) as rnk
FROM products
INNER JOIN order_items
ON products.product_id = order_items.product_id
INNER JOIN orders on order_items.order_id = orders.order_id
GROUP BY product_name;
```

13. Most Returned Products
Query the top 10 products by the number of returns.
Challenge: Display the return rate as a percentage of total units sold for each product.

```sql
with qty_sold as (
 SELECT
   product_name,
   sum(quantity)as qty_sold
 FROM shipping s
 INNER JOIN orders o on s.order_id = o.order_id
 INNER JOIN order_items oi on oi.order_id = o.order_id
 INNER JOIN products p on p.product_id = oi.product_id
 GROUP BY product_name),
 
 return_qty as (
 SELECT
  product_name,
  sum(quantity)as qty_returned
 FROM shipping s
 INNER JOIN orders o on s.order_id = o.order_id
 INNER JOIN order_items oi on oi.order_id = o.order_id
 INNER JOIN products p on p.product_id = oi.product_id
 WHERE return_date is not Null
 GROUP BY product_name)
 
 select
   return_qty.product_name,
   qty_sold,
   qty_returned,
   ROUND((qty_returned/qty_sold)*100.0,2) as return_pct
 FROM return_qty
 INNER JOIN qty_sold
 ON return_qty.product_name = qty_sold.product_name
 ORDER BY return_pct desc;
```

14. Inactive Sellers
Identify sellers who haven’t made any sales in the last 6 months.
Challenge: Show the last sale date and total sales from those sellers.

```sql
WITH cte1 -- as these sellers has not done any sale in last 6 month
AS
(SELECT * FROM sellers
WHERE seller_id NOT IN (SELECT seller_id FROM orders WHERE order_date >= CURRENT_DATE - INTERVAL '6 month')
)

SELECT 
o.seller_id,
MAX(o.order_date) as last_sale_date,
MAX(sum(oi.price_per_unit * oi.qty)) as last_sale_amount
FROM orders as o
JOIN 
cte1
ON cte1.seller_id = o.seller_id
JOIN order_items as oi
ON o.order_id = oi.order_id
GROUP BY seller.id
```


15. IDENTITY customers into returning or new
if the customer has done more than 5 return categorize them as returning otherwise new
Challenge: List customers id, name, total orders, total returns

```sql
with cte as
(
select
 customers.customer_id,
 CONCAT(first_name,last_name) as full_name,
 COUNT(*) as total_orders,
 SUM(case when order_status = "returned" then 1 else 0 end) as returned_order
FROM customers
INNER JOIN orders ON customers.customer_id = orders.customer_id
GROUP BY customers.customer_id
ORDER BY returned_order desc)

SELECT *,
  CASE WHEN returned_order > 5 THEN "returning" else "new" END as cx_category
FROM cte
```


16. Top 5 Customers by Orders in Each State
Identify the top 5 customers with the highest number of orders for each state.
Challenge: Include the number of orders and total sales for each customer.
```sql
WITH cte as
(
SELECT
 customers.Customer_ID,
 state,
 COUNT(*) as total_orders ,
 CONCAT(first_name, last_name) as customer_name ,
 ROUND(sum(price_per_unit * quantity),2) as total_sales,
 DENSE_RANK() OVER(partition by state
                   ORDER BY count(*) DESC) as rn
 FROM customers
 INNER JOIN orders ON orders.customer_id = customers.customer_id
 INNER JOIN order_items ON order_items.order_id = orders.order_id
 GROUP BY customer_id
 )
 
 SELECT * from cte
  WHERE rn <= 5;
```

17. Revenue by Shipping Provider
Calculate the total revenue handled by each shipping provider.
Challenge: Include the total number of orders handled and the average delivery time for each provider.

```sql
SELECT
 shipping_providers,
 COUNT(shipping.order_id) as total_orders,
 ROUND(SUM(price_per_unit * quantity),2) as total_revenue,
 AVG(shipping_date - order_date) as avg_delievery_time
FROM shipping 
INNER JOIN order_items ON order_items.order_id = shipping.order_id
INNER JOIN orders ON orders.order_id = order_items.order_id
WHERE delivery_status = "Delivered" 
GROUP BY shipping_providers;
```

18. Top 10 product with highest decreasing revenue ratio compare to last year(2022) and current_year(2023)
Challenge: Return product_id, product_name, category_name, 2022 revenue and 2023 revenue decrease ratio at end Round the result
Note: Decrease ratio = cr-ls/ls* 100 (cs = current_year ls=last_year)

```sql

WITH cte_2022
as
(    
select
 p.product_id,
 round(sum(price_per_unit * quantity),2) as sales,
 product_name,
 category_name,
 year(order_date)
FROM products p
INNER JOIN order_items oi ON oi.product_id = p.product_id
INNER JOIN orders ON orders.order_id = oi.order_id
INNER JOIN category ON category.category_id =  p.category_id
WHERE YEAR(order_date) = 2022
GROUP BY product_id, year(order_date)),

cte_2023
as 
(
SELECT
 p.product_id,
 round(sum(price_per_unit * quantity),2) as sales,
 product_name,
 category_name,
 year(order_date)
FROM products p
INNER JOIN order_items oi ON oi.product_id = p.product_id
INNER JOIN orders ON orders.order_id = oi.order_id
INNER JOIN category ON category.category_id =  p.category_id
WHERE YEAR(order_date) = 2023
GROUP BY product_id, year(order_date)
)
SELECT
 cte_2022.product_id,
 cte_2022.sales as 2022_sales,
 cte_2023.sales as 2023_sales,
 cte_2022.product_name,
 cte_2022.category_name,
 ROUND(((cte_2023.sales - cte_2022.sales)/cte_2022.sales)*100.0,2) as dec_ratio
FROM cte_2022
INNER JOIN cte_2023
ON cte_2022.product_id = cte_2023.product_id
ORDER BY dec_ratio 
LIMIT 10; 
```

---

## **Learning Outcomes**

This project enabled me to:
- Design and implement a normalized database schema.
- Clean and preprocess real-world datasets for analysis.
- Use advanced SQL techniques, including window functions, subqueries, and joins.
- Conduct in-depth business analysis using SQL.
- Optimize query performance and handle large datasets efficiently.

---

## **Conclusion**

This advanced SQL project successfully demonstrates my ability to solve real-world e-commerce problems using structured queries. From improving customer retention to optimizing inventory and logistics, the project provides valuable insights into operational challenges and solutions.

By completing this project, I have gained a deeper understanding of how SQL can be used to tackle complex data problems and drive business decision-making.

---

### **Entity Relationship Diagram (ERD)**
![ERD](https://github.com/najirh/amazon_usa_project5/blob/main/erd.png)

---
