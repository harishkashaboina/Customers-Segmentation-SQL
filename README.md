# Customer Segmentation

## Project Title : 
Customer Segmentation - RFM Analysis

## Problem Statement:
We will be performing a RFM analysis for a chain of retail stores that sells a lot of different items and categories.

The stores need to adjust their marketing budget and have better targeting of customers so they need to know which customers to focus on and how important they are to the
business.

## What is RFM Score?

We all know that valuing customers based on a single parameter is not correct. The biggest value customer may have only purchased once or twice in a year, or the most frequent
purchaser may have a value so low that it is almost not profitable to service them.

One parameter will never give you an accurate view of your customer base, and you’ll ignore customer lifetime value.

We calculate the RFM score by attributing a numerical value for each of the criteria.
The customer gets more points -
- if they bought in the recent past,
- bought many times
- if the purchase value is larger.
Combine these three values to create the RFM score


This RFM score can then be used to segment your customer data platform (CDP).
- Ultimately, we will end up with 5 bands for each of the R, F and M-values, this can be reduced to bands of 3 if the variation of your data values is narrow.
- The larger the score for each value the better it is. A final RFM score is calculated simply by combining individual RFM score numbers.
- There are many different permutations of the R,F & M scores, 125 in total, which is too many to deal with on an individual basis and many will require similar
marketing responses.

## Data Preview

Attribute Information:
- InvoiceNo: Invoice number. Nominal, a 6-digit integral number uniquely assigned to each transaction. If this code starts with letter 'c', it indicates a cancellation.
- StockCode: Product (item) code. Nominal, a 5-digit integral number uniquely assigned to each distinct product.
- Description: Product (item) name. Nominal.
- Quantity: The quantities of each product (item) per transaction. Numeric.
- InvoiceDate: Invoice Date and time. Numeric, the day and time when each transaction was generated.
- UnitPrice: Unit price. Numeric, Product price per unit in sterling.
- CustomerID: Customer number. Nominal, a 5-digit integral number uniquely assigned to each customer.
- Country: Country name. Nominal, the name of the country where each customer resides.

## RFM Segmentation in BigQuery

The RFM Segmentation can be executed using these five steps:
1. Data processing
2. Compute for recency, frequency, and monetary values per customer
3. Determine quantiles for each RFM metric
4. Assign scores for each RFM metric
5. Define the RFM segments using the scores in step 4

## Prerequisites
- Access to BigQuery
- Data set (Which will be available in this repo.)
- SQL Knowledge

## Data Cleaning

- Checking Null values / Missing values
- Checking duplicate values
Code is available in data-cleaning.ipynb file

## Data Preprocessing

#### Adding the data to BigQuery:
Create a new dataset and upload ‘sales_final.csv’ as a new table.
We created a dataset named `customer_segmentation` and the table name is `sales`.

Now if we look at the data we can see that there are products that have been bought in quantities more than one and we have unit price for those products but we do not have
the total cost of that product. So the first thing we’re gonna do is find the total cost for that product i.e., quantity * unit price -

```
SELECT *, (Quantity * UnitPrice) As Amount FROM `customer_segmentation.sales`
```

Output
<img src="./Images/ss1.png" alt="result"/>

Now that we have got the total cost for each product we need to find out the amount spent on each visit. For each invoice id there may be different products, and till now we have calculated the total for each product, but we do not have the total bill amount for individual invoice ids. For this we use the above query and create a CTE. Then group it by invoice id and sum the total cost, getting the actual bill amount.

```
WITH bills AS (
SELECT *, (Quantity * UnitPrice) As Amount FROM `customer_segmentation.sales`
)

SELECT InvoiceNo, ROUND(SUM(bills.Amount),1) AS total FROM bills GROUP BY InvoiceNo;
```

Output
<img src="./Images/ss2.png" alt="result"/>

Save this data as a `bill` table in the same dataset by using the save button below the query editor.

<img src="./Images/ss3.png" alt="result"/>

### Compute for recency, frequency and monetary values per customer :

- For monetary, this is just a simple sum of sales,
- while for frequency, this is a count of distinct invoice numbers per customer for the time they have been a customer ie: the number of separate purchases/ num
of months they have been a customer. So we will get the first and last purchase for all customers and also the number of purchases
- For calculating recency we will first get the last purchase for each customer

We will join the `bill` table that we saved with the `sales` table and add the total cost on the customer level for monetary value.

```
SELECT S.CustomerID,
Date(MAX(S.InvoiceDate)) AS Last_purchase_date,
Date(MIN(S.InvoiceDate)) AS First_purchase_date,
COUNT(DISTINCT(S.InvoiceNo)) AS Num_of_purchases,
ROUND(SUM(total),1) AS Monetory
FROM customer_segmentation.sales AS S 
LEFT JOIN customer_segmentation.bill AS B 
ON S.InvoiceNo = B.InvoiceNo
GROUP BY S.CustomerID
```

Output
<img src="./Images/ss4.png" alt="result"/>

For recency, we chose a reference date, which is the most recent purchase in the dataset. In other situations, one may select the date when the data was analyzed
instead. 
After choosing the reference date, we get the date difference between the reference date and the last purchase date of each customer. This is the recency value for that
particular customer.
For frequency we calculate the months the person has been a customer by difference in first and last purchase +1 ( for when first and last month are same and the customer should be
considered a customer for at least 1 month)

```
WITH rfm AS (
SELECT S.CustomerID,
Date(MAX(S.InvoiceDate)) AS Last_purchase_date,
Date(MIN(S.InvoiceDate)) AS First_purchase_date,
COUNT(DISTINCT(S.InvoiceNo)) AS Num_of_purchases,
ROUND(SUM(total),1) AS Monetory
FROM customer_segmentation.sales AS S 
LEFT JOIN customer_segmentation.bill AS B 
ON S.InvoiceNo = B.InvoiceNo
GROUP BY S.CustomerID
)

SELECT *, DATE_DIFF( reference_date,Last_purchase_date, day) AS recency,
ROUND(Num_of_purchases/month_cnt,1) AS frequency
FROM (
SELECT *, MAX(rfm.Last_purchase_date) OVER()+1 AS reference_date,
DATE_DIFF(rfm.Last_purchase_date,rfm.First_purchase_date,month)+1 AS month_cnt
FROM rfm
);
```

Output
<img src="./Images/ss5.png" alt="result"/>

Now that we have the RFM data we can save it as another table named `RFM`.


