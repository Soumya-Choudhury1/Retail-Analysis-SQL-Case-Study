# Retail-Analysis-SQL-Case-Study

-- SQL Case Study Project: Retail Analytics

-- 1. Initial Setup
-- Show and use database
SHOW DATABASES;
SELECT DATABASE();
CREATE DATABASE IF NOT EXISTS case_study;
USE case_study;

-- 2. Creating Tables

-- Customers Table
CREATE TABLE IF NOT EXISTS customers (
    CustomerID DECIMAL(38, 0) NOT NULL,
    Age DECIMAL(38, 0) NOT NULL,
    Gender VARCHAR(6) NOT NULL,
    Location VARCHAR(5),
    JoinDate VARCHAR(8) NOT NULL
);

-- Products Table
CREATE TABLE IF NOT EXISTS products (
    ProductID DECIMAL(38, 0) NOT NULL,
    ProductName VARCHAR(50) NOT NULL,
    Category VARCHAR(15) NOT NULL,
    StockLevel DECIMAL(38, 0) NOT NULL,
    Price DECIMAL(38, 2) NOT NULL
);

-- Sales Table
CREATE TABLE IF NOT EXISTS sales (
    TransactionID DECIMAL(38, 0) NOT NULL,
    CustomerID DECIMAL(38, 0) NOT NULL,
    ProductID DECIMAL(38, 0) NOT NULL,
    QuantityPurchased DECIMAL(38, 0) NOT NULL,
    TransactionDate VARCHAR(8) NOT NULL,
    Price DECIMAL(38, 2) NOT NULL
);

-- 3. Data Cleaning and Null Checks
-- Check for NULLs in products table
SELECT 
    SUM(CASE WHEN Price IS NULL THEN 1 ELSE 0 END) AS missing_price,
    SUM(CASE WHEN ProductName IS NULL THEN 1 ELSE 0 END) AS missing_productname,
    SUM(CASE WHEN Category IS NULL THEN 1 ELSE 0 END) AS missing_category,
    SUM(CASE WHEN StockLevel IS NULL THEN 1 ELSE 0 END) AS missing_stocklevel
FROM products;

-- Replace blank Location with 'BLANK'
UPDATE customers
SET Location = 'BLANK'
WHERE Location = '' OR Location IS NULL;

-- 4. Product Insights
-- Highest and Lowest Price Products
SELECT * FROM products ORDER BY Price DESC LIMIT 1;
SELECT * FROM products ORDER BY Price ASC LIMIT 1;

-- Product with highest and lowest quantity purchased
SELECT p.ProductName, s.QuantityPurchased, s.Price
FROM products p JOIN sales s ON p.ProductID = s.ProductID
ORDER BY s.QuantityPurchased DESC LIMIT 1;

SELECT p.ProductName, s.QuantityPurchased, s.Price
FROM products p JOIN sales s ON p.ProductID = s.ProductID
ORDER BY s.QuantityPurchased ASC LIMIT 1;

-- Best/Worst Products by Category
WITH RankedProducts AS (
    SELECT p.Category, p.ProductName, s.QuantityPurchased,
           ROW_NUMBER() OVER (PARTITION BY p.Category ORDER BY s.QuantityPurchased DESC) AS rn
    FROM products p JOIN sales s ON p.ProductID = s.ProductID
)
SELECT Category, ProductName, QuantityPurchased FROM RankedProducts WHERE rn = 1;

-- 5. Duplicate Transactions Cleanup
SET SQL_SAFE_UPDATES = 0;
DELETE s1 FROM sales s1
JOIN (
    SELECT MIN(TransactionID) AS keep_id, ProductID, CustomerID, TransactionDate
    FROM sales
    GROUP BY ProductID, CustomerID, TransactionDate
) s2
ON s1.ProductID = s2.ProductID
AND s1.CustomerID = s2.CustomerID
AND s1.TransactionDate = s2.TransactionDate
AND s1.TransactionID <> s2.keep_id;

-- 6. Demographic Insights
-- Age Category by Total Quantity and Spending
SELECT CASE
    WHEN Age BETWEEN 0 AND 18 THEN 'Teen'
    WHEN Age BETWEEN 19 AND 35 THEN 'Young Adult'
    WHEN Age BETWEEN 36 AND 55 THEN 'Adult'
    ELSE 'Senior'
  END AS AgeCategory,
  SUM(s.QuantityPurchased) AS TotalQuantity,
  SUM(s.QuantityPurchased * s.Price) AS TotalSpending
FROM customers c JOIN sales s ON c.CustomerID = s.CustomerID
GROUP BY AgeCategory
ORDER BY TotalSpending DESC;

-- 7. Customer Behavior
-- Repeat Orders with Age, Product Info
WITH RepeatCustomers AS (
    SELECT CustomerID, ProductID, COUNT(*) AS purchase_count
    FROM sales
    GROUP BY CustomerID, ProductID
    HAVING COUNT(*) > 1
)
SELECT rc.CustomerID, c.Age, rc.ProductID, p.ProductName, p.Category, rc.purchase_count
FROM RepeatCustomers rc
JOIN products p ON rc.ProductID = p.ProductID
JOIN customers c ON rc.CustomerID = c.CustomerID
ORDER BY rc.purchase_count DESC;

-- 8. Customer Segmentation
WITH CustomerSummary AS (
    SELECT CustomerID,
           COUNT(DISTINCT TransactionID) AS total_orders,
           SUM(QuantityPurchased * Price) AS total_spent,
           SUM(QuantityPurchased) AS total_quantity
    FROM sales
    GROUP BY CustomerID
),
CustomerSegments AS (
    SELECT *,
           CASE 
               WHEN total_spent >= 10000 THEN 'High Spender'
               WHEN total_spent BETWEEN 5000 AND 9999 THEN 'Medium Spender'
               WHEN total_spent BETWEEN 1000 AND 4999 THEN 'Low Spender'
               ELSE 'Occasional Buyer'
           END AS Segment
    FROM CustomerSummary
),
RankedSegments AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY Segment ORDER BY total_spent DESC) AS rnk
    FROM CustomerSegments
)
SELECT CustomerID, total_orders, total_spent, total_quantity, Segment
FROM RankedSegments
WHERE rnk <= 3
ORDER BY Segment, rnk;

-- 9. Location Insights
-- Total Sales per Location
SELECT Location, SUM(s.QuantityPurchased * s.Price) AS Total_Sales
FROM customers c JOIN sales s ON c.CustomerID = s.CustomerID
WHERE Location IS NOT NULL AND Location <> ''
GROUP BY Location
ORDER BY Total_Sales DESC;

-- Location-wise Highest and Lowest Selling Products
WITH ProductSalesByLocation AS (
    SELECT c.Location, s.ProductID, SUM(s.QuantityPurchased * s.Price) AS Total_Sales
    FROM sales s JOIN customers c ON s.CustomerID = c.CustomerID
    GROUP BY c.Location, s.ProductID
),
RankedProducts AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY Location ORDER BY Total_Sales DESC) AS rank_high,
           ROW_NUMBER() OVER (PARTITION BY Location ORDER BY Total_Sales ASC) AS rank_low
    FROM ProductSalesByLocation
)
SELECT 'Highest' AS SaleType, rp.Location, rp.ProductID,
       GROUP_CONCAT(DISTINCT s.CustomerID) AS CustomerIDs
FROM RankedProducts rp
JOIN sales s ON rp.ProductID = s.ProductID
JOIN customers c ON s.CustomerID = c.CustomerID AND c.Location = rp.Location
WHERE rp.rank_high = 1
GROUP BY rp.Location, rp.ProductID
UNION ALL
SELECT 'Lowest' AS SaleType, rp.Location, rp.ProductID,
       GROUP_CONCAT(DISTINCT s.CustomerID) AS CustomerIDs
FROM RankedProducts rp
JOIN sales s ON rp.ProductID = s.ProductID
JOIN customers c ON s.CustomerID = c.CustomerID AND c.Location = rp.Location
WHERE rp.rank_low = 1
GROUP BY rp.Location, rp.ProductID;
