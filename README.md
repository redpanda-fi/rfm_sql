# SQL RFM Analysis For Customer Segmentation

In this project, I want to demonstrate my ability to perform RFM analysis to segment customers using SQL. The analysis is primarily done in BigQuery. Visualizations will be created using a combination of Google Sheets and Power BI to better present the insights derived from the data.

## Description of Dataset

Global Super Store is a fictional dataset, which contains data on around 50000 orders placed from the year 2011 to 2014.
Source: https://powerbidocs.com/wp-content/uploads/2019/11/Global-Superstore.csv

## RFM Analysis

RFM can help to identify customers who are more likely to respond to promotions by segmenting them into various categories based on their orders history.

### Data Preparation

I've loaded the .csv file into BigQuery, creating a new table. This dataset consists of 23 columns. However, for RFM analysis, I only need the following columns:

* Order_ID,
* Customer_ID,
* Order_Date,
* Order_Amount: the results of multiplying the "Sales" column by the "Quantity" column for the same customers on the same date.

To optimize resources, I have focused exclusively on the most recent year, 2014, within our dataset. I did this by adding a WHERE clause. This timeframe looks good enough to identify lost clients and order frequency patterns.

Here's the SQL query I used to create a view in BigQuery:

```SQL
SELECT
Order_ID,
Customer_ID,
Order_Date,
Round(SUM(Sales*Quantity),2) as Order_Amount
FROM `global-superstore-415215.superstore.superstore`
WHERE EXTRACT(Year FROM Order_Date) = 2014
GROUP BY Order_ID, Customer_ID, Order_Date
```

I'll continue RFM analysis using this view.

### Calculation of Relevant Values

Now I need to compute RFM (Recency, Frequency, Monetary) metrics

* Recency: The difference in days between the last day of 2014 and each customer's latest order date.
* Frequency: The total number of orders per customer.
* Monetary: The average order amount per customer.

SQL-query:

```SQL
SELECT
Customer_ID,
DATE_DIFF('2014-12-31',MAX(Order_Date), DAY) AS Recency,
COUNT(Order_ID) AS Frequency,
ROUND(AVG(Order_Amount),2) AS Monetary
FROM `global-superstore-415215.superstore.rfm`
GROUP BY Customer_ID
```

Executing the query provides us with the necessary data to establish customer segmentation. Now, let's consider how we can define the boundaries of these segments.

### Cutoff Values Calculation

To define the borders of segments, I calculated approximate quantile cutoff values for Recency, Frequency, and Monetary using the approxQuantile() function.

Here is SQL-query for Recency-cutoff:

```SQL
WITH segmentation AS  (
 SELECT
Customer_ID,
DATE_DIFF('2014-12-31',MAX(Order_Date), DAY) AS Recency,
COUNT(Order_ID) AS Frequency,
ROUND(AVG(Order_Amount),2) AS Monetary
FROM `global-superstore-415215.superstore.rfm`
GROUP BY Customer_ID
)

SELECT
percentiles[offset(20)] as p20,
percentiles[offset(40)] as p40,
percentiles[offset(60)] as p60,
percentiles[offset(80)] as p80,
percentiles[offset(100)] as p100
FROM (
SELECT approx_quantiles(segmentation.Recency, 100) AS percentiles
FROM segmentation
```
CHARTS
![Recency Cutoff]([Recency Cutoff Points.png](https://drive.google.com/file/d/1mwvvofQH6BNIcKJDyUkH9a_3UqXL1rTF/view?usp=drive_link))

### Assigning scores

Based on the cutoff points I created intervals and use them to assign scores. I assigned a higher score to those who visit more frequently and spend more money and a lower score to those who visit less frequently and spend less money. For recency, a higher score is assigned to one with a lower recency value and a lower score to one who visited longer ago.

TABLE

The below query assigns the individual scores of Recency, Frequency, and Monetary as well as computes the RFM score using the formula: RFM Score = Recency Score * 100 + Frequency Score * 10 + Monetary Score

```SQL
WITH rfm_raw AS (
 SELECT
Customer_ID,
DATE_DIFF('2014-12-31',MAX(Order_Date), DAY) AS Recency,
COUNT(Order_ID) AS Frequency,
ROUND(AVG(Order_Amount),2) AS Monetary
FROM `global-superstore-415215.superstore.rfm`
GROUP BY Customer_ID),
rfm_scores AS (
 SELECT
r.*,
CASE WHEN Recency <= 9 THEN 5
    WHEN Recency <= 23 THEN 4
    WHEN Recency <= 43 THEN 3
    WHEN Recency <= 103 THEN 2
    ELSE 1 END AS R_Score,
CASE WHEN Frequency > 10 THEN 5
    WHEN Frequency > 7 THEN 4
    WHEN Frequency > 4 THEN 3
    WHEN Frequency > 2 THEN 2
    ELSE 1 END AS F_Score,
CASE WHEN Monetary > 3090.34 THEN 5
    WHEN Monetary > 1815.48 THEN 4
    WHEN Monetary > 1025.33 THEN 3
    WHEN Monetary > 315.33 THEN 2
    ELSE 1 END AS M_Score
FROM rfm_raw r)
SELECT rs.*,
R_Score*100 + F_Score*10 + M_Score AS RFM
FROM rfm_scores rs
```
### Segments Definition

Now, after calculating the RFM scores, I can classify the customers based on that into 11 groups to understand their behavioral patterns. For this classification, I use the table from bloomreach RFM-Guide (https://documentation.bloomreach.com/engagement/docs/rfm-segmentation). 

TABLE

SQL-query:

```SQL
WITH rfm_raw AS (
...),

rfm_scores AS (
...)

SELECT all_s.*,
CASE WHEN RFM IN (555, 554, 544, 545, 454, 455, 445) THEN 'Champions'
     WHEN RFM IN (543, 444, 435, 355, 354, 345, 344, 335) THEN 'Loyal'
     WHEN RFM IN (553, 551, 552, 541, 542, 533, 532, 531, 452, 451, 442, 441, 431, 453, 433, 432, 423, 353, 352, 351, 342, 341, 333, 323) THEN 'Potential Loyalists'
     WHEN RFM IN (512, 511, 422, 421, 412, 411, 311) THEN 'New Customers'
     WHEN RFM IN (525, 524, 523, 522, 521, 515, 514, 513, 425,424, 413,414,415, 315, 314, 313) THEN 'Promising'
     WHEN RFM IN (535, 534, 443, 434, 343, 334, 325, 324) THEN 'Need Attention'
     WHEN RFM IN (331, 321, 312, 221, 213, 231, 241, 251) THEN 'About To Sleep'
     WHEN RFM IN (155, 154, 144, 214,215,115, 114, 113) THEN 'Cannot Lose Them But Losing'
     WHEN RFM IN (255, 254, 245, 244, 253, 252, 243, 242, 235, 234, 225, 224, 153, 152, 145, 143, 142, 135, 134, 133, 125, 124) THEN 'At Risk'
     WHEN RFM IN (332, 322, 233, 232, 223, 222, 132, 123, 122, 212, 211) THEN 'Hibernating Customers'
     ELSE 'Lost Customers' END AS Segment
FROM all_scores all_s
```

## Customer Segments

After performing the RFM analysis, we've categorized our customers into 11 different groups. However, some of these groups may be very small, or they may have very similar characteristics. So it might be a good idea to merge some of them.

TABLE

CHART


Now that we have a clearer understanding of our customer segments, we can begin to develop personalized strategies and tactics for each group and drive growth for our Global Superstore.
