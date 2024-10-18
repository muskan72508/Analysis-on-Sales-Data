# Sales Data Analysis Project

## Overview
This project involves analyzing a sales dataset to derive insights and trends. Using SQL queries, various aspects of the sales data are explored, including total revenue by product line, monthly sales performance, customer segmentation, and product co-occurrence.

## Dataset
The dataset used in this analysis is a sample sales data table (`dbo.sales_data_sample`). It contains the following relevant fields:
- **ORDERNUMBER**: Unique identifier for each order
- **CUSTOMERNAME**: Name of the customer
- **PRODUCTCODE**: Code of the product sold
- **PRODUCTLINE**: Category of the product
- **SALES**: Revenue generated from the sale
- **STATUS**: Current status of the order (e.g., Shipped)
- **ORDERDATE**: Date when the order was placed
- **YEAR_ID**: Year of the order
- **MONTH_ID**: Month of the order
- **DEALSIZE**: Size of the deal
- **COUNTRY**: Country of the customer
- **TERRITORY**: Territory of the sale

## Analysis
To inspect the data and check unique values, the following SQL queries were executed:
```sql
-- Inspecting data
SELECT * FROM dbo.sales_data_sample;

-- Checking unique values
SELECT DISTINCT status FROM dbo.sales_data_sample;
SELECT DISTINCT YEAR_ID FROM dbo.sales_data_sample;
SELECT DISTINCT PRODUCTLINE FROM dbo.sales_data_sample;
SELECT DISTINCT COUNTRY FROM dbo.sales_data_sample;
SELECT DISTINCT DEALSIZE FROM dbo.sales_data_sample;
SELECT DISTINCT TERRITORY FROM dbo.sales_data_sample;
SELECT DISTINCT MONTH_ID FROM dbo.sales_data_sample WHERE YEAR_ID = 2003;

-- Sales by product line
SELECT PRODUCTLINE, SUM(sales) AS revenue
FROM dbo.sales_data_sample
GROUP BY PRODUCTLINE
ORDER BY revenue DESC;

-- Annual sales performance
SELECT YEAR_ID, SUM(sales) AS revenue
FROM dbo.sales_data_sample
GROUP BY YEAR_ID
ORDER BY revenue DESC;

-- Sales by deal size
SELECT DEALSIZE, SUM(sales) AS revenue
FROM dbo.sales_data_sample
GROUP BY DEALSIZE
ORDER BY revenue DESC;

-- Best month for sales in a specific year
SELECT MONTH_ID, SUM(sales) AS Revenue, COUNT(ORDERNUMBER) AS Frequency
FROM dbo.sales_data_sample
WHERE YEAR_ID = 2003 -- Change year to see results for other years
GROUP BY MONTH_ID
ORDER BY Revenue DESC;

-- Product sales in the best month
SELECT MONTH_ID, PRODUCTLINE, SUM(sales) AS Revenue, COUNT(ORDERNUMBER)
FROM dbo.sales_data_sample
WHERE YEAR_ID = 2004 AND MONTH_ID = 11 -- Change year to see results for other months
GROUP BY MONTH_ID, PRODUCTLINE
ORDER BY Revenue DESC;

-- RFM (Recency, Frequency, Monetary) analysis to identify customer segments
WITH rfm AS 
(
    SELECT 
        CUSTOMERNAME,	
        SUM(sales) AS MonetaryValue,
        AVG(sales) AS AvgMonetaryValue,
        COUNT(ORDERNUMBER) AS Frequency,
        MAX(ORDERDATE) AS last_order_date,
        (SELECT MAX(ORDERDATE) FROM dbo.sales_data_sample) AS max_order_date,
        DATEDIFF(DD, MAX(ORDERDATE), (SELECT MAX(ORDERDATE) FROM dbo.sales_data_sample)) AS Recency
    FROM dbo.sales_data_sample
    GROUP BY CUSTOMERNAME
),
rfm_calc AS
(
    SELECT r.*,
        NTILE(4) OVER (ORDER BY Recency DESC) AS rfm_recency,
        NTILE(4) OVER (ORDER BY Frequency) AS rfm_frequency,
        NTILE(4) OVER (ORDER BY MonetaryValue) AS rfm_monetary
    FROM rfm r
)
SELECT 
    c.*, 
    rfm_recency + rfm_frequency + rfm_monetary AS rfm_cell,
    CAST(rfm_recency AS VARCHAR) + CAST(rfm_frequency AS VARCHAR) + CAST(rfm_monetary AS VARCHAR) AS rfm_cell_string
INTO #rfm
FROM rfm_calc c;

SELECT CUSTOMERNAME, rfm_recency, rfm_frequency, rfm_monetary,
    CASE 
        WHEN rfm_cell_string IN (111, 112, 121, 122, 123, 132, 211, 212, 114, 141) THEN 'Lost Customers'
        WHEN rfm_cell_string IN (133, 134, 143, 244, 334, 343, 344, 144) THEN 'Slipping Away'
        WHEN rfm_cell_string IN (311, 411, 331) THEN 'New Customers'
        WHEN rfm_cell_string IN (222, 223, 233, 322) THEN 'Potential Churners'
        WHEN rfm_cell_string IN (323, 333, 321, 422, 332, 432) THEN 'Active'
        WHEN rfm_cell_string IN (433, 434, 443, 444) THEN 'Loyal'
    END AS rfm_segment
FROM #rfm;

-- Product co-occurrence analysis
SELECT DISTINCT OrderNumber, STUFF(
    (SELECT ',' + PRODUCTCODE
    FROM dbo.sales_data_sample p
    WHERE ORDERNUMBER IN 
        (SELECT ORDERNUMBER
        FROM 
            (SELECT ORDERNUMBER, COUNT(*) rn
            FROM dbo.sales_data_sample
            WHERE STATUS = 'Shipped'
            GROUP BY ORDERNUMBER) m
        WHERE rn = 3)
    AND p.ORDERNUMBER = s.ORDERNUMBER
    FOR XML PATH ('')), 1, 1, '') AS ProductCodes
FROM dbo.sales_data_sample s
ORDER BY ProductCodes DESC;

```

# Conclusion
This project demonstrates how to perform a comprehensive analysis of sales data using SQL, providing valuable insights into sales trends, customer behavior, and product performance.
