#  Online Retail E-commerce Dashboard (SQL + Power BI)

**Advanced SQL and Power BI project delivering a comprehensive e-commerce sales and marketing intelligence system to drive data-driven business decisions.**

![Dashboard Demo](https://github.com/MoRMatipour/Online-Retail-E-commerce-Dashboard-Postgress-SQL-Power-BI-/blob/main/BI%20Dashboard/Recording%202025-08-21%20123659%20(3)%20(1).gif?raw=true)

---

##  Project Overview
This project consolidates **transactional, product, and customer data** to provide a **360° view of online retail performance**.

By analyzing **revenue trends, product and category performance, customer segments, and marketing channel contributions**, the dashboard enables actionable insights for:
- Sales optimization
- Targeted marketing
- Customer engagement strategies

Using **SQL for data modeling and KPI calculation**, combined with **interactive Power BI visualizations**, I created a professional BI solution that supports **real-world decision-making and operational efficiency**.

---

##  Objective
In this project, I designed and implemented a comprehensive **e-commerce sales and marketing intelligence dashboard** to provide actionable insights for business decision-making.

The project involved:
- Extracting and transforming **transactional and customer data** from the Online Retail dataset.
- Modeling the data into **fact and dimension tables** using SQL for efficient analysis.
- Calculating **key performance indicators (KPIs)** such as revenue trends, product/category performance, customer segmentation, and channel contribution.
- Creating **interactive visualizations in Power BI** to highlight sales trends, customer behavior, and marketing effectiveness.
- Delivering a **real-world BI solution** that enables data-driven decisions to optimize sales, marketing, and customer engagement strategies.

---

##  Tools & Technologies
- **SQL (PostgreSQL)** → Data cleaning, transformation, and fact/dimension modeling.
- **Power Query** → Preprocessing, handling missing values, and shaping raw transaction data.
- **Power BI** → Creating interactive KPI reports and the final multi-page dashboard.
- **Excel** → Initial exploration and dataset integrity validation.

---
## Table of Contents

- [Dataset](#Dataset)
- [Inserting the Dataset into PostgreSQL](#inserting-the-dataset-into-postgresql)
- [Data Cleaning & Modeling](#data-cleaning--modeling)
- [Business KPIs and Analytical Queries](#business-kpis-and-analytical-queries)
- [Phase 1 Completed Data Preparation and Modeling](#phase-1-completed-data-preparation-and-modeling)
- [Phase 2 Fast KPIs Report in Power BI](#phase-2-fast-kpis-report-in-power-bi)
- [Phase 3 Comprehensive BI Dashboard in Power BI](#phase-3-comprehensive-bi-dashboard-in-power-bi)
- [Conclusion](#conclusion)

---
## Dataset 
This project uses the **Online Retail Dataset** from the [UCI Machine Learning Repository](https://archive.ics.uci.edu/ml/datasets/Online+Retail).

**Key details:**
- **Records**: ~500,000 transactions
- **Features**: Invoice ID, Stock Code, Description, Quantity, Invoice Date, Unit Price, Customer ID, Country
- **Granularity**: Each row = one product purchased within an invoice (line-item level)

This dataset enables analysis of:
- Customer purchasing patterns
- Revenue trends and seasonality
- Product performance (top sellers, revenue drivers, returns)
- Country-level sales distribution

---

## Inserting the Dataset into PostgreSQL 

To process and analyze the Online Retail dataset efficiently, I first created a **PostgreSQL table** to store the raw data. This structured storage allows accurate querying and seamless integration with downstream analytics and Power BI dashboards.

###  Table Creation in PostgreSQL
The table schema was designed to reflect the dataset’s structure and ensure proper data types.

```sql
CREATE TABLE retails (
  InvoiceNo VARCHAR(100),
  StockCode VARCHAR(100),
  Description TEXT,
  Quantity INT,
  InvoiceDate TIMESTAMP,
  UnitPrice DECIMAL(10,2),
  CustomerID INT,
  Country VARCHAR(100)
);
```

###  Inserting the Dataset via psql

Once the table was created, the dataset was imported using PostgreSQL’s command-line tool psql.
```psql
psql -U your_username -d your_database
\COPY retails(InvoiceNo, StockCode, Description, Quantity, InvoiceDate, UnitPrice, CustomerID, Country)
FROM '/path/to/OnlineRetail.csv' DELIMITER ',' CSV HEADER;
```


## Data Cleaning & Modeling

After importing the **Online Retail dataset** into PostgreSQL, I performed a series of cleaning, transformation, and modeling steps to prepare the data for fast KPIs and the main dashboard. This ensures accuracy, consistency, and optimized query performance for analytics and reporting.

---
### Step 0: Initial Data Cleaning
Before building fact and dimension tables, I filtered out invalid and incomplete transactions to create a clean working dataset:

```sql
CREATE TABLE clear_retails AS
  SELECT * FROM retails 
  WHERE description IS NOT NULL 
    AND description != ''
    AND Quantity > 0
    AND UnitPrice > 0;
```
---
### Step 1: Fact Table Creation
The cleaned transactional data was loaded into a **fact table** to centralize key metrics like quantity, unit price, and total sales per invoice. This table serves as the foundation for all subsequent analyses.

```sql
-- FACT TABLE
CREATE TABLE IF NOT EXISTS fact_sales AS 
SELECT 	
	invoiceno,
	customerid,
	stockcode,
	quantity,
	unitprice,
	ROUND(quantity * unitprice, 2) AS total_price,
	invoicedate,
	DATE(invoicedate) AS invoice_date_key  -- For joining with dim_date
FROM clean_retails;

````

---

### Step 2: Dimension Tables Creation

To support a **dimensional data model**, I created dimension tables for customers, products, and dates. These tables provide context for the fact table, allowing for aggregation, segmentation, and trend analysis across multiple perspectives.

* **Customer Dimension**: Stores unique customer IDs and their countries.

```sql
-- DIMENSION TABLE: CUSTOMER
CREATE TABLE IF NOT EXISTS dim_customer AS	
	SELECT 
		customerid,
		MAX(country) AS country
	FROM clean_retails
	WHERE customerid IS NOT NULL
	GROUP BY customerid;

```

* **Product Dimension**: Stores unique product codes and descriptions.

```sql
-- DIMENSION TABLE: PRODUCT
CREATE TABLE IF NOT EXISTS dim_product AS	
	SELECT 
		stockcode,
		MAX(description) AS description
	FROM clean_retails
	WHERE stockcode IS NOT NULL
	GROUP BY stockcode;

```

* **Date Dimension**: Extracts day, month, year, weekday, month name, and week number from invoice dates for time-based analysis.

```sql
-- DIMENSION TABLE: DATE
CREATE TABLE IF NOT EXISTS dim_date AS	
SELECT DISTINCT
	DATE(invoicedate) AS full_date,
	EXTRACT(DAY FROM invoicedate) AS day,
  	EXTRACT(MONTH FROM invoicedate) AS month,
  	EXTRACT(YEAR FROM invoicedate) AS year,
  	TO_CHAR(invoicedate, 'Day') AS weekday,
  	TO_CHAR(invoicedate, 'Month') AS month_name,
  	EXTRACT(WEEK FROM invoicedate) AS week_number
FROM clean_retails;

```

---

### Step 3: Data Enrichment & Relationship Building

To ensure **referential integrity** and fast query performance, I verified primary keys for all dimension tables and created foreign key relationships with the fact table. Indexes were also added to optimize joins and aggregations.

```sql
-- ADD PRIMARY KEYS FOR JOINING RELATIONSHIPS
ALTER TABLE dim_customer ADD PRIMARY KEY (customerid);
ALTER TABLE dim_product ADD PRIMARY KEY (stockcode);
ALTER TABLE dim_date ADD PRIMARY KEY (full_date);

--Create Foreing keys for relationship between main table and dime tables
ALTER TABLE fact_sales 
ADD CONSTRAINT fk_customer 
FOREIGN KEY (customerid) 
REFERENCES dim_customer(customerid);

ALTER TABLE fact_sales 
ADD CONSTRAINT fk_product 
FOREIGN KEY (stockcode) 
REFERENCES dim_product(stockcode);

ALTER TABLE fact_sales 
ADD CONSTRAINT fk_date 
FOREIGN KEY (invoice_date_key) 
REFERENCES dim_date(full_date);

--Add Indexes in dim tables
CREATE INDEX idx_customer ON fact_sales(customerid);
CREATE INDEX idx_product ON fact_sales(stockcode);
CREATE INDEX idx_date ON fact_sales(invoice_date_key);

```
I also checked for duplicate and null keys in the dimension tables using SQL queries, confirming that `customerid`, `stockcode`, and `full_date` are unique, which ensures the correctness and consistency of the dimensional model.

```sql
SELECT
	customerid, COUNT(*) 
FROM dim_customer 
GROUP BY customerid
HAVING COUNT(*) > 1;


SELECT
	stockcode, COUNT(*) 
FROM dim_product 
GROUP BY stockcode
HAVING COUNT(*) > 1;


SELECT
	full_date, COUNT(*) 
FROM dim_date 
GROUP BY full_date
HAVING COUNT(*) > 1;

--Verify There Are No NULLs in the Primary Key Columns
SELECT customerid FROM dim_customer WHERE customerid IS NULL; 
SELECT stockcode FROM dim_product WHERE stockcode IS NULL; 
SELECT full_date FROM dim_date WHERE full_date IS NULL;

```

**Key Cleaning & Modeling Functions Used:**

* `AGGREGATIONS` and `DISTINCT`: Remove duplicates and summarize key metrics.
* `EXTRACT` & `TO_CHAR`: Transform invoice dates into multiple useful date components.
* **Primary & Foreign Keys**: Enforce data integrity between fact and dimension tables.
* **Indexes**: Optimize query performance for large dataset analytics.

This structured and relational data model enables **fast and accurate KPI calculations** and forms the backbone of the Fast KPIs report and the main dashboard, providing reliable insights for business decision-making.

---

## Business KPIs and Analytical Queries

After setting up the data model, I wrote analytical queries to extract the most important KPIs for business performance. These queries helped track revenue, sales trends, customer behavior, and product performance in a structured way.

---

### 1. Total Revenue, Orders, and Quantity

This query calculates the overall business performance by measuring total orders, total quantity sold, and total revenue. It provides a baseline understanding of the company’s scale and value generation.

```sql
SELECT 
  COUNT(DISTINCT invoiceno) AS total_orders,
  SUM(quantity) AS total_quantity,
  ROUND(SUM(total_price), 2) AS total_revenue
FROM fact_sales;
```

---

### 2. Monthly Sales Performance

By joining the sales fact table with the date dimension, this query tracks **monthly trends** in revenue, orders, and quantities. This highlights seasonality patterns and performance over time.

```sql
SELECT 
  d.year,
  d.month,
  d.month_name,
  COUNT(DISTINCT f.invoiceno) AS total_orders,
  SUM(f.quantity) AS total_quantity,
  ROUND(SUM(f.total_price), 2) AS total_revenue
FROM fact_sales f
JOIN dim_date d ON f.invoice_date_key = d.full_date
GROUP BY d.year, d.month, d.month_name
ORDER BY d.year, d.month;
```

---

### 3. Orders and Revenue by Country

Breaking down sales by country identifies the most valuable markets and regions contributing the highest revenue.

```sql
SELECT 
  c.country,
  COUNT(DISTINCT f.invoiceno) AS total_orders,
  SUM(f.quantity) AS total_quantity,
  ROUND(SUM(f.total_price), 2) AS total_revenue
FROM fact_sales f
JOIN dim_customer c ON f.customerid = c.customerid
GROUP BY c.country
ORDER BY total_revenue DESC;
```

---

### 4. Top Products by Revenue

This query identifies the **best-selling products**, which is essential for inventory management, marketing focus, and product promotion.

```sql
SELECT 
  p.description AS product,
  SUM(f.quantity) AS total_quantity,
  ROUND(SUM(f.total_price), 2) AS total_revenue
FROM fact_sales f
JOIN dim_product p ON f.stockcode = p.stockcode
GROUP BY p.description
ORDER BY total_revenue DESC
LIMIT 10;
```

---

### 5. Top Customers by Customer Lifetime Value

Aggregating total spending per customer reveals the **highest-value customers** and allows targeting retention strategies for segments generating the most revenue.

```sql
SELECT 
  f.customerid,
  c.country,
  ROUND(SUM(f.total_price), 2) AS customer_lifetime_value,
  COUNT(DISTINCT f.invoiceno) AS orders_count
FROM fact_sales f
JOIN dim_customer c ON f.customerid = c.customerid
GROUP BY f.customerid, c.country
ORDER BY customer_lifetime_value DESC
LIMIT 20;
```

---

## Phase 1 Completed: Data Preparation and Modeling

In this phase, I:

* Imported the Online Retail dataset into PostgreSQL
* Performed **data cleaning**
* Built a **dimensional model** with fact and dimension tables
* Added **relationships, primary keys, and indexes** to ensure high-performance querying
* Defined **key business metrics and analytical queries** to measure revenue, orders, top products, top customers, and sales trends

This prepared the foundation for connecting the dataset to Power BI and building an interactive dashboard for actionable insights.

## Next Step: Fast KPIs Report in Power BI
With the fact and dimension tables ready, the next step is to **import the tables into Power BI** and implement the defined analytical queries as **DAX measures**. These measures are then used to build a **Fast KPI Report**, providing a quick overview of core business performance metrics such as:
- Total Revenue
- Total Orders
- Quantity Sold
- Top Products
- Top Customers
- Monthly Sales Trends

This step ensures that stakeholders can immediately access critical KPIs through a visually appealing and interactive report, setting the foundation for the main dashboard development in **Phase 2**.

---

## Phase 2: Fast KPIs Report in Power BI

After preparing the fact and dimension tables in PostgreSQL, I connected Power BI directly to the database and imported the tables. The first goal was to build a **Fast KPIs Report**, a simple but effective report to quickly highlight the most important business metrics.

### Steps in this Phase

1. **Connecting Power BI to PostgreSQL**
   - Established a direct connection to the PostgreSQL server.
   - Imported the KPIs via SQL Queries
   
![Connected Queries](https://github.com/MoRMatipour/Online-Retail-E-commerce-Dashboard-Postgress-SQL-Power-BI-/blob/main/fast%20kpis/Screenshot%202025-06-24%20183158.png?raw=true) 

2. **Creating DAX Measures**
   To bring the queries written in SQL into Power BI, I translated them into **DAX measures**. These measures allow dynamic calculations that respond to user interactions, filters, and slicers inside the report.

   #### Examples of DAX Measures for KPI Reports:
```
  YearMonth = FORMAT([year], "0000") & "-" & FORMAT([month], "00")

  YearMonthLabel = FORMAT(DATE([year], [month], 1), "MMM yyyy")
```

![KPIs DAX](https://github.com/MoRMatipour/Online-Retail-E-commerce-Dashboard-Postgress-SQL-Power-BI-/blob/main/fast%20kpis/Screenshot%202025-06-24%20193602.png?raw=true)

3. **Building the Fast KPIs Report**

   * Placed **card visuals** for Total Revenue, Total Orders, and Quantity Sold.
   * Added a **column chart** for Monthly Revenue Trends.
   * Included simple **tables** for Top Products and Top Customers.
   * Ensured **slicers** (Date, Country) were available for quick filtering.

![Report 1](https://github.com/MoRMatipour/Online-Retail-E-commerce-Dashboard-Postgress-SQL-Power-BI-/blob/main/fast%20kpis/Screenshot%202025-08-21%20115537.png?raw=true)

![Report 2](https://github.com/MoRMatipour/Online-Retail-E-commerce-Dashboard-Postgress-SQL-Power-BI-/blob/main/fast%20kpis/Screenshot%202025-07-15%20222344.png?raw=true)

---

### Output of Phase 2

At the end of this phase, I had a **lightweight Power BI report** that provides quick access to the most important KPIs, helping stakeholders track business performance instantly.
You can find the KPIs File [here](https://github.com/MoRMatipour/Online-Retail-E-commerce-Dashboard-Postgress-SQL-Power-BI-/blob/main/fast%20kpis/E-commerce%20Sales.pbix).

This report serves as the **foundation for more advanced visualizations and insights in Phase 3**, where we will design a **comprehensive dashboard with deeper drill-down capabilities**.

---

## Phase 3: Comprehensive BI Dashboard in Power BI

With the Fast KPIs report completed, the next goal was to design a **full-scale BI dashboard**. Unlike the Phase 2 lightweight view, this phase focuses on **deep insights, comparisons, and drill-downs** to support decision-making.

### Steps in this Phase

1. **Connecting Power BI to PostgreSQL**
   - Established a direct connection to the PostgreSQL server.
   - Imported the `fact_sales` and related dimension tables.
   
![Table Imports](https://github.com/MoRMatipour/Online-Retail-E-commerce-Dashboard-Postgress-SQL-Power-BI-/blob/main/BI%20Dashboard/Screenshot%202025-06-27%20154549.png?raw=true)

   - Verified relationships between tables (fact-to-dim joins).

![Table Models](https://github.com/MoRMatipour/Online-Retail-E-commerce-Dashboard-Postgress-SQL-Power-BI-/blob/main/BI%20Dashboard/Screenshot%202025-08-21%20121252.png?raw=true)
2. **Designing the Dashboard Layout**

   * Built a **dedicated Power BI dashboard file** with interactive visuals.  
   * Included **slicers** for filtering, a **bar chart** for product analysis, a **map** for geographic insights, and a **line chart** for sales trends.  
   * Added **four KPI cards** to give an at-a-glance overview.  
   * Ensured a clean and minimal layout for easy navigation and storytelling.  


3. **Advanced DAX Measures**
   Expanded the measures library to enable richer analysis:

```DAX
  Revenue MoM % = 
VAR CurrentMonth = [Total Revenue]
VAR PreviousMont = 
    CALCULATE(
        [Total Revenue],
        DATEADD('dim_date'[full_date], -1, MONTH)
    )
RETURN 
    DIVIDE(CurrentMonth - PreviousMont, PreviousMont);

Cumulative Revenue = 
CALCULATE(
    [Total Revenue],
    FILTER(
        ALLSELECTED('dim_date'),
        'dim_date'[full_date] <= MAX('dim_date'[full_date])
    )
);
```
   

   These measures allowed the dashboard to track **growth trends, profitability ratios, and rankings dynamically**.

4. **Visuals Created**

   * **Sales Overview**: Cards for revenue, orders, avg order value; line chart for monthly trends.
   * **Customer Insights**: Bar chart of top customers by revenue; segmentation by customer country.
   * **Product Performance**: Top-selling products by revenue and quantity; scatter plot of revenue vs. order frequency.
   * **Geographic Distribution**: Map visualization showing sales by country.
   * **Growth Tracking**: KPI visual for monthly growth % with conditional formatting.

![Dashboard Picture](https://github.com/MoRMatipour/Online-Retail-E-commerce-Dashboard-Postgress-SQL-Power-BI-/blob/main/BI%20Dashboard/Screenshot%202025-08-21%20124334.png?raw=true)

5. **Interactivity & Usability**

   * Added slicers for **Date, Country, Product Category**.
   * Enabled **drill-down** from category → product → transaction details.
   * Implemented **bookmarks** for switching between Executive View and Analyst View.
   
![Dashboard Demo](https://github.com/MoRMatipour/Online-Retail-E-commerce-Dashboard-Postgress-SQL-Power-BI-/blob/main/BI%20Dashboard/Recording%202025-08-21%20123659%20(3)%20(1).gif?raw=true)

---

### Output of Phase 3

The result is a **comprehensive BI dashboard** that provides:

* **High-level overview for executives** (quick health check).
* **Granular insights for analysts** (detailed breakdowns).
* **Interactive exploration for stakeholders** (filter, drill, compare).

This dashboard turns raw transactional data into **actionable intelligence**, bridging the gap between data and business strategy.
You can find the Dashboard File [here](https://github.com/MoRMatipour/Online-Retail-E-commerce-Dashboard-Postgress-SQL-Power-BI-/blob/main/BI%20Dashboard/correct%20dashboard.pbix).

---

## Conclusion

This project demonstrated the complete **BI workflow**, from **data modeling and SQL-based KPI analysis** to **interactive Power BI reporting**.

By transforming the raw Online Retail dataset into a **structured star schema**, I was able to define clear fact and dimension tables that supported robust analytical queries.

Using SQL, I extracted critical business insights such as:

* Monthly revenue trends
* Country-level performance
* Top products by sales
* Customer lifetime value

These queries laid the foundation for a **self-service BI layer in Power BI**, where stakeholders can instantly explore revenue drivers, customer behavior, and product performance.

### Business Impact

This dashboard empowers stakeholders to make **data-driven decisions** that can lead to measurable improvements in business performance:

* **Revenue Optimization** – Identify top-performing products and customer segments to tailor marketing efforts, potentially increasing average order value (AOV) and overall revenue.
* **Cost Reduction** – Monitor inventory turnover and return rates to optimize stock levels and reduce holding costs.
* **Customer Retention** – Track purchase patterns to develop targeted retention strategies, enhancing customer lifetime value (CLV) and reducing churn rates.

### Stakeholder Utilization

The dashboard serves as a versatile tool for various stakeholders:

* **Marketing Teams** – Monitor campaign performance and customer acquisition costs (CAC) to refine targeting and budget allocation.
* **Sales Managers** – Identify sales trends and product performance to inform sales strategies and forecasting.
* **Operations Managers** – Assess inventory levels and fulfillment metrics to streamline supply chain processes.
* **Executives** – Gain a comprehensive overview of business health to guide strategic planning and decision-making.

Ultimately, the project highlights how a **BI developer can bridge the gap between raw transactional data and business-ready insights**. This **end-to-end approach**, combined with clear **quantified business impact**, ensures that decision-makers have access to **fast, reliable, and actionable KPIs**, enabling **data-driven growth strategies** across the organization.

---





