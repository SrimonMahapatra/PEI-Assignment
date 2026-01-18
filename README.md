# PEI Assignment

**Data Analyst Task**

The sales team has the following data from various sources, which you have to download from the below drive:

[**https://drive.google.com/drive/folders/1cfVJx6IicqLNKwWUIJG-O-htpTK3R-tE?](https://drive.google.com/drive/folders/1cfVJx6IicqLNKwWUIJG-O-htpTK3R-tE?usp=sharing) [usp=sharing](https://drive.google.com/drive/folders/1cfVJx6IicqLNKwWUIJG-O-htpTK3R-tE?usp=sharing)**

## **As a Data Analyst, you are required to**

- Verify the accuracy, completeness, and reliability of source data. Show your results in a SQL or Python output.
- Based on your findings, define and outline the requirements for anticipated datasets, detailing the proposed domain model to support the reporting requirements noted below.
- Prepare a story with technical specifications for **one part of the data model** for a data engineer to include technical specifications and transformations. This story should give enough information for a Data Engineer to build table(s) and for a QA engineer to test it.

## **Business Reporting Requirements**

These reporting requirements should be enabled based on the tasks outlined for the Data Analyst above. The task is not to create the reports but to enable it.

1. *The total amount spent and the country for the Pending delivery status for each country.*
2. *the total number of transactions, total quantity sold, and total amount spent for each customer, along with the product details.*
3. *The maximum product purchased for each country.*
4. *The most purchased product based on the age category, less than 30 and above 30.*
5. *The country that had the minimum transactions and sales amount.*

### **Objective - 1**

Verify the accuracy, completeness, and reliability of source data. Show your results in a SQL or Python output.

### 2. Scope

The following tables were evaluated

- `customers`
- `orders`
- `shipping`

### 3. Summary of findings

| Table | Status |
| --- | --- |
| customers | No major issues |
| orders | No major issues |
| shipping | Referential Integrity issue found with orders table. |

### 4. Details of quality assessment on tables

### **4.1 Customers**

**Null check in customer_id (PK) :**

```sql
SELECT customer_id
FROM bronze.customers
WHERE customer_id IS NUll
-- No records found
```

**Duplicate check customer_id (PK) :**

```sql
SELECT customer_id, count(customer_id) as count
FROM bronze.customers
GROUP BY customer_id
HAVING count(customer_id) > 1
-- No records found
```

**Duplication check in table :** 

```sql
SELECT customer_id, first, last, age, country, count(*)
FROM bronze.customers
GROUP BY
customer_id, first, last, age, country 
HAVING 
count(*) > 1
-- No records found
```

**Null check in table :** 

```sql
SELECT *
FROM bronze.customers c
WHERE c.first IS NULL OR c.last IS NULL or country is null
-- No records found
```

**Age outlier check :**

```sql
SELECT *
FROM bronze.customers c
WHERE age < 0 or age > 100 or age is null
-- no records found
```

**Check for any outlier values and consistency in records :**

```sql
SELECT DISTINCT country
FROM bronze.customers c
```

**Output:**

"UK"
"USA"
"UAE"

### **4.2 Orders**

**Null check in order_id (PK) and customer_id (FK) :**

```sql
SELECT order_id
FROM bronze.orders
WHERE order_id IS NULL
-- no records found
```

```sql
SELECT customer_id
FROM bronze.orders
WHERE customer_id IS NULL
-- no records found
```

**Duplicate check order_id (PK) :**

```sql
SELECT order_id, COUNT(order_id)
FROM bronze.orders
GROUP BY order_id
HAVING COUNT(order_id) > 1
-- no records found
```

**Null and negative value check in amount :**

```sql
SELECT * FROM
bronze.orders
WHERE amount is null or amount < 0
-- no records found
```

**Null check in items :**

```sql
SELECT * FROM
bronze.orders
WHERE item is null
-- no records found
```

**Referential integrity check with customers table :** 

```sql
SELECT o.*, c.* FROM
bronze.orders o
LEFT JOIN bronze.customers c
ON o.customer_id = c.customer_id
WHERE c.customer_id is null
-- no records found
```

### **4.3 Shipping**

**Null check in shipping_id (PK) and customer_id(FK) :**

```sql
SELECT shipping_id, customer_id
FROM bronze.shipping
WHERE shipping_id IS NULl or customer_id IS NULL
-- no records found
```

**Duplicate check in shipping_id (PK)**

```sql
SELECT shipping_id, count(shipping_id) as count
FROM bronze.shipping
group by shipping_id
HAVING count(shipping_id) > 1
-- no records found
```

**Referential integrity check for customers present in shipping table but not in orders table**

```sql
SELECT distinct s.customer_id as shipping_customer_id, 
o.customer_id as orders_customer_id
FROM bronze.shipping s
LEFT JOIN bronze.orders o
ON s.customer_id = o.customer_id
WHERE o.customer_id IS NULL
-- 55 records found

```

[data-1768748629288.csv](data-1768748629288.csv)

```sql
SELECT COUNT(DISTINCT s.customer_id) as shipping_customer_count, 
COUNT(DISTINCT o.customer_id) as orders_customer_count
FROM bronze.shipping s
LEFT JOIN bronze.orders o
ON s.customer_id = o.customer_id
WHERE o.customer_id IS NULL
```

| shipping_customer_count | orders_customer_count |
| --- | --- |
| 55 | 0 |

**Null check in status :**

```sql
SELECT status
FROM bronze.shipping
WHERE status IS NULL
-- no records found
```

**Orphan records check with customer table :**

```sql
SELECT
s.customer_id, c.customer_id
FROM bronze.shipping s
LEFT JOIN bronze.customers c
ON s.customer_id = c.customer_id
WHERE c.customer_id IS NULL
-- no records found
```

### **Objective - 2**

Based on your findings, define and outline the requirements for anticipated datasets, detailing the proposed domain model to support the reporting requirements noted below.

### **Findings**

### **Relationship between customers, orders and shipping**

In the current data model both the shipping table and orders table are connected to the customers table through **customer_id**, these two doesn't share a direct relationship.

The customer table acts a bridge table for many to many relationship between orders and shipping table

It is important that they need to share a direct relationship as the shipping table should be dependent upon the orders table order-id not the customer tables customer-id

considering a scenario here -
one customer can have multiple orders with different delivery status now, the delivery status should be depend on the order_id for identification because

over the time the delivery status changes of a single order pending & delivered 

customer_id identifies a person and a person can have many orders and multiple deliveries over time

so backtracking the shipping id and status to a customer makes it ambiguous and not possible to get the correct data.

In case of order-id, it is a unique id attached to each order created by customer

so, if the status changes it will be backtracked by the order_id as the order-id is unique there will be no ambiguity.

**conclusion** 
The shipping table should contain the order_id(FK) connecting it to orders table order_id (PK),

creating a one to many relationship.

### Anticipated data sets

### **Introduction of order_items tables**

The orders tables stores all the records of the items that has been created by the customer

It is possible a single order can contain multiple items , in the current structure item level details are stored directly in the orders table which require a separate row.

This would result same **order_id** being repeated across multiple rows, which violates PK constraint

of the orders table.

To avoid such scenarios I propose an order items table which will contain the following-

## order_items

| Column | Data Type | Description |
| --- | --- | --- |
| item_id, order_id |  | Composite Primary key; unique  identifier |
| item_id | INT | Foreign Key `items.item_id` |
| order_id | VARCHAR(50) | Foreign key to `orders.order_id` |
| item | VARCHAR(50) | Name of the products |
| item_qty | INT | Quantity of items ordered |
| item_unit_price | FLOAT | unit price of items  |
| items_total_amount | FLOAT | item_qty * item_unit_price |

The **order_items** table will contain the details of the items attached to an order

as the **order_id** is a FK constraint even if the items are repeated it will not violate any constraints 

and for multiple items as single **order_id** will be stored in the **orders table.**

In the assignment there was requirement to fetch the total quantity sold but any specific column was not available in the given data 

Hence, the **item_qty** column is added in to the proposed model for a clear analytical purpose .

The **item_unit_price** and **items_total_amount** is added to support financial analysis.

This will help us understand the based on the total orders for a FY what is the total amount generated by each product and the product performance by the quantity sold and revenue generated by that product.

### **Introduction of items table**

To ensure the consistent and unambiguous product representation in **order_items** table,

it is important to maintain a standardized reference table for item information.

Storing the item names directly in order_items may lead to data quality issues such as 

spelling variation of same item, duplicate entries making it difficult to maintain the attributes.

To ensure the consistency an items table need to be introduced which will act as a master table for reference that will store unique product information

This table will consist of the following -

## **Items**

| Column | Data Type | Description |
| --- | --- | --- |
| item_id | INT | Composite Primary key; unique  identifier |
| item_name | VARCHAR(30) | Foreign Key `items.item_id` |

### **Introduction of payments table**

The differentiation between the amount and the payment has been done for the product has different definitions.
In the current model we are tracking the product amount that is shipped to the customer but we are not tracking the payment details from the customer such as payment amount, payment method, instalments, date of payment received

To accurately track all the financial transactions and to distinguish between the value of goods that has been shipped vs the amount paid by the customer, a separate payment table is need to be introduced

| Column | Data Type | Description |
| --- | --- | --- |
| payment_id | INT | Primary key; unique payment identifier |
| order_id | INT | Foreign key to `orders.order_id` |
| payment_installment | INT | Instalments number for the payment |
| payment_type | VARCHAR(30) | Mode of payment |
| payment_value | FLOAT | Amount paid in the transaction |
| payment_date | TIMESTAMP | Date and time of payment |
| payment_status | VARCHAR(30) | Status of the payment (success, pending, failed) |

### **Introduction of timestamps in orders table**

It has been observed, that the **orders** table doesn’t have any timestamps columns

this limits the analysis for time based performance analysis and delay analysis for the

orders, which is crucial to understand the any bottleneck in the orders shipping to delivery.

**Below is the updated orders table :** 

| Column | Data Type | Description |
| --- | --- | --- |
| order_id | INT | Primary key; unique order identifier |
| customer-id | INT | Foreign key to `customer.customer_id` |
| amount | INT | Total amount of orders |
| order_approved_date | TIMESTAMP | Approval date of order |
| order_shipping_date | TIMESTAMP | Shipping date of order |
| order_delivery_date | TIMESTAMP | Delivery date of order |
| order_estimated_delivery_date | TIMESTAMP | Estimated delivery date of order |

### Objective - 3

Prepare a story with technical specifications for one part of the data model for a data engineer to include technical specifications and transformations. This story should give enough information for a Data Engineer to build table(s) and for a QA engineer to test it.

## Entity Relationship Diagram

 

![Blank board.png](https://github.com/SrimonMahapatra/PEI-Assignment/blob/77b211245e5de82e18dd846a7e1eea52f099abab/Blank%20board%20test.png)

## customers

| Column | Data Type | Description |
| --- | --- | --- |
| customer_id | INT | Foreign key to `olist_orders.order_id` |
| first | VARCHAR(100) | Line item number within an order (composite primary key with order_id) |
| last | VARCHAR(100) | Foreign key to `olist_products.product_id` |
| age | INT | Foreign key to `olist_sellers.seller_id` |
| country | VARCHAR(30) | Last allowed date seller must ship item |

## orders

| Column | Data Type | Description |
| --- | --- | --- |
| order_id | INT | Primary key; unique order identifier |
| customer-id | INT | Foreign key to `customer.customer_id` |
| amount | INT | Total amount of orders |
| order_approved_date | TIMESTAMP | Approval date of order |
| order_shipping_date | TIMESTAMP | Shipping date of order |
| order_delivery_date | TIMESTAMP | Delivery date of order |
| order_estimated_delivery_date | TIMESTAMP | Estimated delivery date of order |

## shipping

| Column | Data Type | Description |
| --- | --- | --- |
| shipping_id | INT | Primary key; unique shipping identifier |
| order_id | INT | Foreign key to `orders.order_id` |
| status | VARCHAR(30) |  |

## order_items

| Column | Data Type | Description |
| --- | --- | --- |
| item_id, order_id |  | Composite Primary key; unique  identifier |
| item_id | INT | Foreign Key `items.item_id` |
| order_id | VARCHAR(50) | Foreign key to `orders.order_id` |
| item | VARCHAR(50) | Name of the products |
| item_qty | INT | Quantity of items ordered |
| item_unit_price | FLOAT | unit price of items |
| items_total_amount | FLOAT | item_qty * item_unit_price |

## items

| Column | Data Type | Description |
| --- | --- | --- |
| item_id | INT | Composite Primary key; unique  identifier |
| item_name | VARCHAR(30) | Foreign Key `items.item_id` |

## payments

| Column | Data Type | Description |
| --- | --- | --- |
| payment_id | INT | Primary key; unique payment identifier |
| order_id | INT | Foreign key to `orders.order_id` |
| payment_installment | INT | Instalments number for the payment |
| payment_type | VARCHAR(30) | Mode of payment |
| payment_value | FLOAT | Amount paid in the transaction |
| payment_date | TIMESTAMP | Date and time of payment |
| payment_status | VARCHAR(30) | Status of the payment (success, pending, failed) |

## **Business Reporting Requirements**

### Task-1 :

*The total amount spent and the country for the Pending delivery status for each country.*

```sql
SELECT
c.country,
s.status,
sum(o.amount) as total_amount
FROM
bronze.customers c
JOIN bronze.orders o
ON c.customer_id = o.customer_id
JOIN bronze.shipping s
ON c.customer_id = s.customer_id
WHERE s.status = 'Pending'
GROUP BY
c.country, s.status
```

[data-1768747904639.csv](data-1768747904639.csv)

### Task-2 :

*the total number of transactions, total quantity sold, and total amount spent for each customer, along with the product details.*

```sql
with t1 as
(
SELECT DISTINCT customer_id
FROM bronze.shipping
WHERE status = 'Delivered' 
-- This temp table is used to create a deduplicated list of customers who have at least one deliverd shipment
)

SELECT
o.customer_id as orders_customer_id,
o.item as product_name,
count(o.order_id) as transactions_count,
count(o.item) as total_qty_sold,
sum(o.amount) as total_amount
FROM T1
JOIN bronze.orders o  -- This make sure that the customer_id appeared in shipping table matches with the customer_id orders table correcting the refrential integrity issue
ON T1.customer_id = o.customer_id -- common customers from both table
GROUP BY
o.customer_id, o.item
ORDER BY
o.customer_id
```

[data-1768748172695.csv](data-1768748172695.csv)

### Task-3 :

*The maximum product purchased for each country.*

```sql
WITH t1 as
(
SELECT c.country, o.item as product_name,
count(order_id) as product_purchased, sum(o.amount) as total_amount
FROM bronze.orders o
JOIN bronze.customers c
ON o.customer_id = c.customer_id
GROUP BY
c.country, o.item
),

t2 as
(
SELECT country, product_name, product_purchased,
dense_rank() OVER (partition by country order by product_purchased DESC) as RANK
FROM T1
)

SELECT *
FROM T2
WHERE RANK = 1
```

[data-1768748303884.csv](data-1768748303884.csv)

### Task-4 :

*The country that had the minimum transactions and sales amount.*

```sql
WITH t1 as
(
SELECT o.item as product,
CASE
WHEN age < 30 THEN 'below 30'
WHEN age >= 30 THEN 'above 30'
END AS age_category,
count(o.order_id) as products_purchased
FROM bronze.customers c
JOIN bronze.orders o
ON o.customer_id = c.customer_id
GROUP BY
age_category , o.item
),

t2 as
(SELECT product, age_category, products_purchased , dense_rank() OVER (partition by age_category ORDER BY products_purchased DESC) as RANK
FROM T1)

SELECT product, age_category, products_purchased
FROM T2
WHERE RANK = 1
```

[data-1768748531152.csv](data-1768748531152.csv)
