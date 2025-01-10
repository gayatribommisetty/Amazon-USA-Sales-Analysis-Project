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
CREATE TABLE category
(
  category_id	INT PRIMARY KEY,
  category_name VARCHAR(20)
);

-- customers TABLE
CREATE TABLE customers
(
  customer_id INT PRIMARY KEY,	
  first_name	VARCHAR(20),
  last_name	VARCHAR(20),
  state VARCHAR(20),
  address VARCHAR(5) DEFAULT ('xxxx')
);

-- sellers TABLE
CREATE TABLE sellers
(
  seller_id INT PRIMARY KEY,
  seller_name	VARCHAR(25),
  origin VARCHAR(15)
);

-- products table
  CREATE TABLE products
  (
  product_id INT PRIMARY KEY,	
  product_name VARCHAR(50),	
  price	FLOAT,
  cogs	FLOAT,
  category_id INT, -- FK 
  CONSTRAINT product_fk_category FOREIGN KEY(category_id) REFERENCES category(category_id)
);

-- orders
CREATE TABLE orders
(
  order_id INT PRIMARY KEY, 	
  order_date	DATE,
  customer_id	INT, -- FK
  seller_id INT, -- FK 
  order_status VARCHAR(15),
  CONSTRAINT orders_fk_customers FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
  CONSTRAINT orders_fk_sellers FOREIGN KEY (seller_id) REFERENCES sellers(seller_id)
);

CREATE TABLE order_items
(
  order_item_id INT PRIMARY KEY,
  order_id INT,	-- FK 
  product_id INT, -- FK
  quantity INT,	
  price_per_unit FLOAT,
  CONSTRAINT order_items_fk_orders FOREIGN KEY (order_id) REFERENCES orders(order_id),
  CONSTRAINT order_items_fk_products FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- payment TABLE
CREATE TABLE payments
(
  payment_id	
  INT PRIMARY KEY,
  order_id INT, -- FK 	
  payment_date DATE,
  payment_status VARCHAR(20),
  CONSTRAINT payments_fk_orders FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

CREATE TABLE shippings
(
  shipping_id	INT PRIMARY KEY,
  order_id	INT, -- FK
  shipping_date DATE,	
  return_date	 DATE,
  shipping_providers	VARCHAR(15),
  delivery_status VARCHAR(15),
  CONSTRAINT shippings_fk_orders FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

CREATE TABLE inventory
(
  inventory_id INT PRIMARY KEY,
  product_id INT, -- FK
  stock INT,
  warehouse_id INT,
  last_stock_date DATE,
  CONSTRAINT inventory_fk_products FOREIGN KEY (product_id) REFERENCES products(product_id)
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
- Forecasting and product performance
  

## **Identifying Business Problems**

Key business problems identified:
1. Low product availability due to inconsistent restocking.
2. High return rates for specific product categories.
3. Significant delays in shipments and inconsistencies in delivery times.
4. High customer acquisition costs with a low customer retention rate.

---
-- EDA
SELECT * FROM category;
SELECT * FROM customer;
SELECT * FROM inventory;
SELECT * FROM order_item;
SELECT * FROM orders;
SELECT * FROM product;
SELECT * FROM seller;
SELECT * FROM shipping
WHERE return_date IS NOT NULL;

SELECT 
	DISTINCT payment_status
FROM payment;

--- BUSINESS PROBLEMS--
/*
1. Top Selling Products
Query the top 10 products by total sales value.
Challenge: Include product name, total quantity sold, and total sales value.
*/

--- join order_items with order table nad then join with product table 
-- Group by
--- total sales value = qty*price
--group by product id

---CREATING NEW COLUMN
ALTER TABLE order_item
ADD COLUMN total_sale_value FLOAT;

--Upadating total_sale_value 
UPDATE order_item
SET total_sale_value = quantity * price_per_unit;

SELECT * FROM order_item;
SELECT * FROM orders;

SELECT p.product_id,
	p.product_name,
	sum(oi.total_sale_value) as total_sale_value,
	count(o.order_id) as total_orders
FROM order_item oi
JOIN orders o ON oi.order_id =o.order_id 
JOIN product p ON oi.product_id = p.product_id 
GROUP BY 1,2 
ORDER BY total_sale_value DESC
LIMIT 10;

/*2. Revenue by Category
Calculate total revenue generated by each product category.
Challenge: Include the percentage contribution of each category to total revenue.
*/
SELECT * FROM category;
SELECT * FROM product;
SELECT * FROM order_item;

SELECT 
	c.category_id,
	c.category_name,
	ROUND(SUM(oi.total_sale_value)::NUMERIC, 2) as total_revenue,
	SUM(oi.total_sale_value)/ (SELECT SUM(total_sale_value) FROM order_item)* 100 as Contribution 
	FROM product p
LEFT JOIN category c ON p.category_id = c.category_id
JOIN order_item oi ON oi.product_id = p.product_id
GROUP BY 1,2
ORDER BY 2 DESC
;

/*
3. Average Order Value (AOV)
Compute the average order value for each customer.
Challenge: Include only customers with more than 5 orders.
*/
SELECT c.customer_id,
	CONCAT(c.first_name, ' ' , c.last_name) as customer_name,
	count(o.order_id) as total_orders,
	ROUND((sum(total_sale_value)/count(o.order_id))::NUMERIC,2) as avg_value
FROM customer c
JOIN orders o on c.customer_id = o.customer_id
JOIN order_item oi on oi.order_id = o.order_id
GROUP BY c.customer_id,customer_name
HAVING count(*) > 5 
ORDER BY 4 DESC;

/*
4. Monthly Sales Trend
Query monthly total sales over the past year.
Challenge: Display the sales trend, grouping by month, return current_month sale, last month sale!
*/
--last 1 year data
-- ecah month totaol sale, previous month sale 
WITH CTE_monthly as (
SELECT 
	EXTRACT (MONTH FROM o.order_date) as month_name,
	EXTRACT (YEAR FROM o.order_date) as year,
	ROUND(SUM(oi.total_sale_value)::numeric, 2) as last_month_sale
FROM orders o
JOIN order_item oi on oi.order_id = o.order_id
WHERE order_date >= CURRENT_DATE - INTERVAL '1 year'
GROUP BY 2,1
ORDER BY 2,1)
SELECT year,month_name,
lag(last_month_sale,1)over(order by year, month_name)
FROM CTE_monthly;

/*
5. Customers with No Purchases
Find customers who have registered but never placed an order.
Challenge: List customer details and the time since their registration.
*/
SELECT 
	c.customer_id,
	c.first_name,
	c.last_name,
	c.state
FROM customer c
LEFT JOIN orders o ON c.customer_id= o.customer_id
WHERE o.order_id IS NULL;

/*
6. Least-Selling Categories by State
Identify the least-selling product category for each state.
Challenge: Include the total sales for that category within each state.
*/
SELECT * FROM category;

SELECT c.category_name,
oi.quantity
FROM order_item oi
JOIN product p  ON oi.product_id = p.product_id
JOIN customer c ON oi.order_id = c.
JOIN category c  ON c.category_id = p.category_id 
;
SELECT * FROM orders;
SELECT * FROM product;


/*
7. Customer Lifetime Value (CLTV)
Calculate the total value of orders placed by each customer over their lifetime.
Challenge: Rank customers based on their CLTV.
*/
WITH CTE_CLTV AS (
SELECT 
	c.customer_id,
	CONCAT(c.first_name, ' ', c.last_name) as c_name,
	ROUND(SUM(oi.total_sale_value)::NUMERIC, 2) as Customer_Lifetime_Value
FROM customer c
JOIN orders o ON c.customer_id = o.customer_id
JOIN order_item oi ON oi.order_id = o.order_id
GROUP BY c.customer_id)

SELECT *,
DENSE_RANK()over(order by Customer_Lifetime_Value desc)
FROM CTE_CLTV;

/*
8. Inventory Stock Alerts
Query products with stock levels below a certain threshold (e.g., less than 10 units).
Challenge: Include last restock date and warehouse information.
*/
SELECT i.inventory_id,
	p.product_name,
	i.stock,
	i.last_stock_date,
	i.warehouse_id
FROM inventory i
JOIN product p ON p.product_id = i.product_id 
WHERE i.stock < 10
;

/*
9. Shipping Delays
Identify orders where the shipping date is later than 3 days after the order date.
Challenge: Include customer, order details, and delivery provider.
*/
SELECT 
	o.order_id,
	CONCAT(c.first_name, ' ', c.last_name) as c_name,
	s.shipping_date,
	o.order_date,
	o.order_status,
	s.shipping_providers
FROM 
	shipping s
JOIN 
	orders o ON o.order_id = s.order_id
JOIN 
	customer c ON c.customer_id = o.customer_id
WHERE 
	s.shipping_date > o.order_date + INTERVAL '3 days';

/*
10. Payment Success Rate 
Calculate the percentage of successful payments across all orders.
Challenge: Include breakdowns by payment status (e.g., failed, pending).
*/
SELECT payment_status,
COUNT(*) as total_cnt,
ROUND(COUNT(*):: NUMERIC/(SELECT count(*) from payment) :: numeric * 100 , 2) as percentage
FROM payment
GROUP BY payment_status
ORDER BY 1 DESC;
	
/*
11. Top Performing Sellers
Find the top 5 sellers based on total sales value.
Challenge: Include both successful and failed orders, and display their percentage of successful orders.
*/

WITH CTE_top_sellers AS (
SELECT 
	s.seller_id,
	s.seller_name,
	ROUND(SUM(oi.total_sale_value):: numeric, 2) as total_sale_value
FROM order_item oi
JOIN  orders o ON oi.order_id = o.order_id
JOIN seller s ON o.seller_id = s.seller_id
GROUP BY 1
ORDER BY 2 DESC
LIMIT 5),
seller_reports as (
SELECT 
	o.seller_id,
	so.seller_name,
	o.order_status,
	COUNT(*) as total_orders
FROM orders o
JOIN CTE_top_sellers so ON o.seller_id = so.seller_id
WHERE o.order_status NOT IN ( 'Inprogress', 'Returned')
GROUP BY 1,2,3
)
SELECT 
	seller_id,
	seller_name,
	SUM(CASE WHEN order_status = 'Completed' THEN total_orders ELSE 0 END) as completed_orders,
	SUM(CASE WHEN order_status = 'Cancelled' THEN total_orders ELSE 0 END) as Cancelled_orders,
	SUM(total_orders),
	ROUND(SUM(CASE WHEN order_status = 'Completed' THEN total_orders ELSE 0 END):: NUMERIC/SUM(total_orders) :: numeric * 100,2) as percen_completed_orders,
	ROUND(SUM(CASE WHEN order_status = 'Cancelled' THEN total_orders ELSE 0 END):: NUMERIC/SUM(total_orders) :: numeric * 100 ,2)as percen_Cancelled_orders
FROM seller_reports
GROUP BY 1,2
;

/*
12. Product Profit Margin
Calculate the profit margin for each product (difference between price and cost of goods sold).
Challenge: Rank products by their profit margin, showing highest to lowest.
*/

SELECT 
    p.product_id,
    p.product_name,
    p.price,
    p.cogs,
    SUM(oi.quantity) AS total_quantity,
    SUM((p.price - p.cogs) * oi.quantity) AS total_profit,
    (SUM((p.price - p.cogs) * oi.quantity) / SUM(p.price * oi.quantity)) * 100 AS profit_margin,
    DENSE_RANK() OVER (ORDER BY 
        SUM((p.price - p.cogs) * oi.quantity) DESC
    ) AS rank_profit_margin
FROM 
    product p
JOIN 
    order_item oi ON oi.product_id = p.product_id
GROUP BY 
    p.product_id, p.product_name, p.price, p.cogs
ORDER BY 
    rank_profit_margin;
/*
13. Most Returned Products
Query the top 10 products by the number of returns.
Challenge: Display the return rate as a percentage of total units sold for each product.
*/
WITH CTE_returns AS (
    SELECT
        p.product_id,
        p.product_name,
        COUNT(*) AS total_returns
    FROM shipping s
    JOIN orders o ON s.order_id = o.order_id
    JOIN order_item oi ON oi.order_id = o.order_id
    JOIN product p ON p.product_id = oi.product_id
    WHERE o.order_status = 'Returned'
    GROUP BY p.product_id, p.product_name
    ORDER BY total_returns DESC
    LIMIT 10
),
CTE_return_percentage AS (
    SELECT
        r.product_id,
        r.product_name,
        r.total_returns,
        SUM(oi.quantity) AS total_units_sold,
        (r.total_returns::NUMERIC / SUM(oi.quantity) * 100) AS return_rate
    FROM CTE_returns r
    JOIN order_item oi ON r.product_id = oi.product_id
    GROUP BY r.product_id, r.product_name, r.total_returns
)
SELECT
    product_id,
    product_name,
    total_returns,
    total_units_sold,
    ROUND(return_rate, 2) AS return_rate
FROM CTE_return_percentage
ORDER BY return_rate DESC;

/*
14. Orders Pending Shipment
Find orders that have been paid but are still pending shipment.
Challenge: Include order details, payment date, and customer information.
*/

SELECT 
	DISTINCT payment_status
FROM payment;
SELECT 
    o.order_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
     o.order_date,
    p.payment_date,
    o.order_status,
    s.delivery_status
FROM 
    orders o
JOIN 
    payment p ON o.order_id = p.order_id
JOIN 
    customer c ON o.customer_id = c.customer_id
JOIN 
    shipping s ON o.order_id = s.order_id
WHERE 
    p.payment_status = 'Payment Successed'
    
ORDER BY 
    o.order_date DESC;
/*
15. Inactive Sellers
Identify sellers who haven’t made any sales in the last 6 months.
Challenge: Show the last sale date and total sales from those sellers.
*/



WITH SellerSales AS (
    SELECT 
        s.seller_id,
        s.seller_name,
        MAX(o.order_date) AS last_sale_date,
        COUNT(o.order_id) AS total_sales
    FROM 
        seller s
    LEFT JOIN 
        orders o ON s.seller_id = o.seller_id
    WHERE 
        o.order_date <= NOW() - INTERVAL '6 months' OR o.order_date IS NULL
    GROUP BY 
        s.seller_id, s.seller_name
)
SELECT 
    seller_id,
    seller_name,
    last_sale_date,
    total_sales
FROM 
    SellerSales
WHERE 
    last_sale_date IS NULL OR last_sale_date <= NOW() - INTERVAL '6 months'
ORDER BY 
    last_sale_date ASC NULLS FIRST;

/*
16. IDENTITY customers into returning or new
if the customer has done more than 5 return categorize them as returning otherwise new
Challenge: List customers id, name, total orders, total returns
*/
WITH CustomerStats AS (
    SELECT 
        c.customer_id,
        CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
        COUNT(o.order_id) AS total_orders,
        SUM(CASE WHEN o.order_status = 'Returned' THEN 1 ELSE 0 END) AS total_returns
    FROM 
        customer c
    LEFT JOIN 
        orders o ON c.customer_id = o.customer_id
    GROUP BY 
        c.customer_id, c.first_name, c.last_name
)
SELECT 
    customer_id,
    customer_name,
    total_orders,
    total_returns,
    CASE 
        WHEN total_returns > 5 THEN 'Returning'
        ELSE 'New'
    END AS customer_type
FROM 
    CustomerStats
ORDER BY 
    total_returns DESC;

/*
17. Cross-Sell Opportunities
Find customers who purchased product A but not product B (e.g., customers who bought AirPods but not AirPods Max).
Challenge: Suggest cross-sell opportunities by displaying matching product categories.
*/
WITH ProductACustomers AS (
    SELECT DISTINCT o.customer_id
    FROM order_item oi
    JOIN orders o ON oi.order_id = o.order_id
    WHERE oi.product_id = 'A'  -- Replace 'A' with the actual product ID for Product A
),
ProductBCustomers AS (
    SELECT DISTINCT o.customer_id
    FROM order_item oi
    JOIN orders o ON oi.order_id = o.order_id
    WHERE oi.product_id = 'B'  -- Replace 'B' with the actual product ID for Product B
)
SELECT 
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    'Product B' AS suggested_product
FROM 
    ProductACustomers a
LEFT JOIN 
    ProductBCustomers b ON a.customer_id = b.customer_id
JOIN 
    customer c ON a.customer_id = c.customer_id
WHERE 
    b.customer_id IS NULL;


/*
18. Top 5 Customers by Orders in Each State
Identify the top 5 customers with the highest number of orders for each state.
Challenge: Include the number of orders and total sales for each customer.
*/
WITH RankedCustomers AS (
    SELECT 
        c.state,
        o.customer_id,
        CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
        COUNT(o.order_id) AS total_orders,
        SUM(oi.total_sale_value) AS total_sales,
        RANK() OVER (PARTITION BY c.state ORDER BY COUNT(o.order_id) DESC) AS rank
    FROM 
        orders o
    JOIN 
        customer c ON o.customer_id = c.customer_id
    JOIN 
        order_item oi ON o.order_id = oi.order_id
    GROUP BY 
        c.state, o.customer_id, c.first_name, c.last_name
)
SELECT 
    state,
    customer_id,
    customer_name,
    total_orders,
    total_sales
FROM 
    RankedCustomers
WHERE 
    rank <= 5
ORDER BY 
    state, rank;



/*
19. Revenue by Shipping Provider
Calculate the total revenue handled by each shipping provider.
Challenge: Include the total number of orders handled and the average delivery time for each provider.
*/
SELECT 
    s.shipping_providers,
    COUNT(o.order_id) AS total_orders_handled,
    SUM(oi.total_sale_value) AS total_revenue,
    AVG(s.shipping_date - o.order_date) AS avg_delivery_time
FROM 
    shipping s
JOIN 
    orders o ON s.order_id = o.order_id
JOIN 
    order_item oi ON o.order_id = oi.order_id
GROUP BY 
    s.shipping_providers
ORDER BY 
    total_revenue DESC;


/*
20. Top 10 product with highest decreasing revenue ratio compare to last year(2022) and current_year(2023)
Challenge: Return product_id, product_name, category_name, 2022 revenue and 2023 revenue decrease ratio at end Round the result

Note: Decrease ratio = cr-ls/ls* 100 (cs = current_year ls=last_year)
*/
WITH RevenueByYear AS (
    SELECT 
        p.product_id,
        p.product_name,
        c.category_name,
        EXTRACT(YEAR FROM o.order_date) AS year,
        SUM(oi.total_sale_value) AS total_revenue
    FROM 
        orders o
    JOIN 
        order_item oi ON o.order_id = oi.order_id
    JOIN 
        product p ON oi.product_id = p.product_id
    JOIN 
        category c ON p.category_id = c.category_id
    WHERE 
        EXTRACT(YEAR FROM o.order_date) IN (2022, 2023)
    GROUP BY 
        p.product_id, p.product_name, c.category_name, EXTRACT(YEAR FROM o.order_date)
),
RevenueComparison AS (
    SELECT 
        r2022.product_id,
        r2022.product_name,
        r2022.category_name,
        r2022.total_revenue AS revenue_2022,
        r2023.total_revenue AS revenue_2023,
        ROUND(((r2023.total_revenue - r2022.total_revenue) / r2022.total_revenue)::NUMERIC * 100, 2) AS revenue_decrease_ratio
    FROM 
        (SELECT * FROM RevenueByYear WHERE year = 2022) r2022
    LEFT JOIN 
        (SELECT * FROM RevenueByYear WHERE year = 2023) r2023
    ON 
        r2022.product_id = r2023.product_id
)
SELECT 
    product_id,
    product_name,
    category_name,
    revenue_2022,
    revenue_2023,
    revenue_decrease_ratio
FROM 
    RevenueComparison
ORDER BY 
    revenue_decrease_ratio ASC
LIMIT 10;


/*
Final Task
-- Store Procedure
create a function as soon as the product is sold the the same quantity should reduced from inventory table
after adding any sales records it should update the stock in the inventory table based on the product and qty purchased
-- 
*/

CREATE OR REPLACE FUNCTION update_inventory_and_add_sale(
    p_order_id INT,         -- Order ID
    p_customer_id INT,      -- Customer ID
    p_seller_id INT,        -- Seller ID
    p_order_item_id INT,    -- Order Item ID
    p_product_id INT,       -- Product ID
    p_quantity INT          -- Quantity Sold
)
RETURNS VOID AS $$
DECLARE 
    v_stock_remaining INT;        -- Variable to check stock availability
    v_price FLOAT;                -- Variable to fetch product price
    v_product_name VARCHAR(100);  -- Variable to fetch product name
BEGIN
    -- Fetch product details: price and name
    SELECT 
        price, product_name
    INTO 
        v_price, v_product_name
    FROM 
        products
    WHERE 
        product_id = p_product_id;

    --Check stock availability in inventory
    SELECT 
        stock_remaining 
    INTO 
        v_stock_remaining
    FROM 
        inventory
    WHERE 
        product_id = p_product_id;

    -- Validate stock availability
    IF v_stock_remaining >= p_quantity THEN
        -- Add the order to the 'orders' table
        INSERT INTO orders(order_id, order_date, customer_id, seller_id)
        VALUES (p_order_id, CURRENT_DATE, p_customer_id, p_seller_id);

        -- Add the order details to the 'order_items' table
        INSERT INTO order_items(order_item_id, order_id, product_id, quantity, price_per_unit, total_price)
        VALUES (p_order_item_id, p_order_id, p_product_id, p_quantity, v_price, v_price * p_quantity);

        -- Update the inventory table to reduce the stock
        UPDATE inventory
        SET stock_remaining = stock_remaining - p_quantity
        WHERE product_id = p_product_id;

        -- Success notice
        RAISE NOTICE 'Order processed successfully! Product: %, Quantity: %, Remaining Stock: %', 
                     v_product_name, p_quantity, v_stock_remaining - p_quantity;
    ELSE
        -- Failure notice for insufficient stock
        RAISE EXCEPTION 'Insufficient stock for Product: %. Requested: %, Available: %', 
                        v_product_name, p_quantity, v_stock_remaining;
    END IF;
END;
$$ LANGUAGE plpgsql;

SELECT update_inventory_and_add_sale(
    25005,  -- Order ID
    2,      -- Customer ID
    5,      -- Seller ID
    25004,  -- Order Item ID
    1,      -- Product ID (AirPods 3rd Gen)
    10      -- Quantity Sold
);
SELECT COUNT(*) 
FROM inventory
WHERE 
	product_id = 1
	AND 
	stock >= 56



call update_inventory_and_add_sale
(
25005, 2, 5, 25004, 1, 14
);


(
p_order_id INT,
p_customer_id INT,
p_seller_id INT,
p_order_item_id INT,
p_product_id INT,
p_quantity INT
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
