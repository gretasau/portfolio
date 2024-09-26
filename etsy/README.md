# General Information

## Dataset
Private dataset from real Etsy shop.

## Prepared by
Greta Šaumanė, 2024

## Goal
Answer basic business questions as well as draw more complex findings from the data.

## Files
- [Q&A Part 1](https://docs.google.com/spreadsheets/d/1KmHcqO6XddgsWjj-EcLLL8BEmWozgxDctK_l_zEW2BY/edit?usp=sharing)


## Contents

# Etsy Shop Data Analysis

This project involves analyzing sales data from an Etsy shop using BigQuery. The focus is on providing answers to specific business questions without drawing conclusions. The results can be viewed in this [Google Spreadsheet](https://docs.google.com/spreadsheets/d/1KmHcqO6XddgsWjj-EcLLL8BEmWozgxDctK_l_zEW2BY/edit?usp=sharing).

## Data Preparation

Multiple CSV files with yearly data have been appended in Google Sheets due to the low number of rows. Any personal information, such as names and addresses, has been eliminated from the table. Additional changes have been made in BigQuery.

### Data Preparation Steps

1. **Consistent Column Naming**

   Renamed a column to ensure consistent naming across the dataset.

   ```sql
   -- Changing one column name to have consistent naming
   CREATE OR REPLACE TABLE orders.etsy_orders AS 
   SELECT 
     _Sale_Date_ AS Sale_Date,
     * EXCEPT(_Sale_Date_)
   FROM minilab123.orders.etsy_orders2;

   -- Dropping the old table
   DROP TABLE orders.etsy_orders2;

   -- Renaming the new table
   ALTER TABLE orders.etsy_orders RENAME TO orders;

2. **Anonymizing Buyer User IDs**

   The `Buyer_User_ID` column may contain personal information but is necessary for identifying returning customers. A mapping table was created to assign new user IDs to non-null `Buyer_User_ID`s.

   ```sql
   -- Creating a mapping table to assign an ID based on the row to all distinct Buyer User IDs.
   CREATE OR REPLACE TABLE orders.buyer_mapping AS 
   WITH distinct_users AS (
     SELECT DISTINCT(Buyer_User_ID)
     FROM orders.orders
     WHERE Buyer_User_ID IS NOT NULL
   )
   SELECT 
     Buyer_User_ID,
     ROW_NUMBER() OVER(ORDER BY Buyer_User_ID) AS User_ID
   FROM distinct_users;

   -- Creating new table with all columns, including the new ID, except the previous Buyer User ID.
   CREATE OR REPLACE TABLE orders.orders_new AS
   SELECT
     * EXCEPT(Buyer_User_ID)
   FROM orders.orders AS o
   LEFT JOIN orders.buyer_mapping AS m USING(Buyer_User_ID);

   -- Dropping unnecessary tables.
   DROP TABLE orders.buyer_mapping;
   DROP TABLE orders.orders;

   -- Renaming the new table.
   ALTER TABLE orders.orders_new RENAME TO orders;

3. **Adding Date-Related Columns**

   Added `Sale_Year`, `Sale_Month`, and `Sale_Day` columns for easier data analysis.

   ```sql
   -- Adding date-related columns
   ALTER TABLE orders.orders 
   ADD COLUMN Sale_Year INT64,
   ADD COLUMN Sale_Month INT64,
   ADD COLUMN Sale_Day INT64;

   -- Populating the date-related columns
   UPDATE orders.orders AS o
   SET 
     Sale_Year = EXTRACT(YEAR FROM o.Sale_Date),
     Sale_Month = EXTRACT(MONTH FROM o.Sale_Date),
     Sale_Day = EXTRACT(DAY FROM o.Sale_Date)
   WHERE o.Sale_Date IS NOT NULL;
   ```

4. **Calculating Gross Revenue**

   The dataset contains various revenue-related columns, but none refer to gross revenue. A `Gross_Revenue` column was added for easier querying.

   ```sql
   -- Adding Gross_Revenue column
   ALTER TABLE orders.orders
   ADD COLUMN Gross_Revenue FLOAT64;

   -- Calculating Gross_Revenue
   UPDATE orders.orders AS o
   SET 
     Gross_Revenue = ROUND((o.Order_Value + o.Shipping - o.Discount_Amount), 2)
   WHERE o.Order_ID IS NOT NULL;
   ```

# Data Analysis

   Below are the SQL queries used to answer specific business questions.

1. **Date Range of the Data**

   ```sql
   -- The date range of the data.
   SELECT
     MIN(Sale_Date) AS min_date,
     MAX(Sale_Date) AS max_date
   FROM orders.orders;
   ```

2. **Yearly Sales (Count of Orders, Items, Revenue)**

   ```sql
   -- Yearly sales in count of orders, count of items, revenue. Full years only.
   SELECT
     Sale_Year,
     COUNT(*) AS order_count,
     SUM(Number_of_Items) AS num_of_items,
     ROUND(SUM(Gross_Revenue), 2) AS revenue
   FROM orders.orders 
   WHERE Sale_Year BETWEEN 2019 AND 2023
   GROUP BY Sale_Year
   ORDER BY Sale_Year ASC;
   ```

3. **Monthly Revenue Change (Last 12 Months Compared to Previous Year)**

   ```sql
   -- Monthly revenue change last 12 months, compared to previous year. Full months only.
   WITH sale_dates AS (
     SELECT
       MAX(DATE_TRUNC(Sale_Date, MONTH)) AS l_month,
       DATE_SUB(MAX(DATE_TRUNC(Sale_Date, MONTH)), INTERVAL 1 MONTH) AS l_full_month
     FROM orders.orders
   ),
   rev_prev_year AS (
     SELECT
       Sale_Year AS year,
       Sale_Month AS month,
       ROUND(SUM(Gross_Revenue), 2) AS monthly_revenue
     FROM orders.orders AS o, sale_dates AS d
     WHERE DATE_DIFF(d.l_full_month, Sale_Date, MONTH) BETWEEN 12 AND 23
       AND Sale_Date < (d.l_month - 12)
     GROUP BY Sale_Year, Sale_Month
     ORDER BY Sale_Year ASC, Sale_Month ASC
   ),
   rev_this_year AS (
     SELECT
       Sale_Year AS year,
       Sale_Month AS month,
       ROUND(SUM(Gross_Revenue), 2) AS monthly_revenue
     FROM orders.orders AS o, sale_dates AS d
     WHERE DATE_DIFF(d.l_full_month, Sale_Date, MONTH) < 12
       AND Sale_Date < d.l_month
     GROUP BY Sale_Year, Sale_Month
     ORDER BY Sale_Year ASC, Sale_Month ASC
   )
   SELECT
     t.year AS year,
     t.month AS month,
     t.monthly_revenue AS rev_this_year,
     p.monthly_revenue AS rev_prev_year,
     ROUND((t.monthly_revenue - p.monthly_revenue), 2) AS revenue_change_Eur,
     ROUND((t.monthly_revenue - p.monthly_revenue) / p.monthly_revenue * 100, 2) AS revenue_change_perc
   FROM rev_this_year AS t
   LEFT JOIN rev_prev_year AS p USING (month)
   ORDER BY year ASC, month ASC;
   ```

4. **Top 5 Countries by Revenue**

   ```sql
   -- Top 5 countries by revenue.
   SELECT 
     Ship_Country,
     ROUND(SUM(Gross_Revenue), 2) AS revenue
   FROM orders.orders
   GROUP BY Ship_Country
   ORDER BY revenue DESC
   LIMIT 5;
   ```

5. **Top 10 States by Revenue**

   ```sql
   -- Top 10 states by revenue.
   SELECT
     Ship_State,
     ROUND(SUM(Gross_Revenue), 2) AS revenue
   FROM orders.orders
   WHERE Ship_State IS NOT NULL
   GROUP BY Ship_State
   ORDER BY revenue DESC
   LIMIT 10;
   ```

6. **Average Shipping Lag by Year**

   ```sql
   -- Average shipping lag by year.
   SELECT 
     Sale_Year,
     ROUND(AVG(DATETIME_DIFF(Date_Shipped, Sale_Date, DAY)), 1) AS ship_lag_days
   FROM orders.orders 
   WHERE Date_Shipped IS NOT NULL
   GROUP BY Sale_Year
   ORDER BY Sale_Year ASC;
   ```

7. **Average Order Value by Year**

   ```sql
   -- Average order value by year.
   SELECT 
     Sale_Year,
     ROUND(SUM(Gross_Revenue) / COUNT(Order_ID), 2) AS AOV
   FROM minilab123.orders.orders 
   GROUP BY Sale_Year
   ORDER BY Sale_Year ASC;
   ```

8. **Average Items per Order by Year**

   ```sql
   -- Average items per order by year.
   SELECT 
     Sale_Year,
     ROUND(SUM(Number_of_Items) / COUNT(Order_ID), 2) AS items_per_order
   FROM minilab123.orders.orders 
   GROUP BY Sale_Year
   ORDER BY Sale_Year ASC;
   ```

9. **Percentage of Customers Who Made More Than One Order**

   ```sql
   -- Percentage of customers that made more than one order.
   WITH users AS (
     SELECT
       User_ID,
       COUNT(Order_ID) AS orders_per_user
     FROM orders.orders
     WHERE User_ID IS NOT NULL
     GROUP BY User_ID
   )
   SELECT
     ROUND((COUNTIF(users.orders_per_user > 1) / COUNT(*)) * 100, 2) AS perc_returning_users
   FROM users;
   ```
