# :convenience_store: Case Study #5: Data Mart

## Business Task:

Data Mart is an online supermarket that specialises in fresh produce. After running its international operations, Danny wants to analyse his sales performance.
Also, in June 2020, the company shifted to using sustainable packaging methods in every single step of the production. Danny needs help to quantify the impact of this change on the sales performance for Data Mart and its separate business areas. view the case study [Link](https://8weeksqlchallenge.com/case-study-5/)

## :memo: Solution 1. Data Cleansing Steps

*First, we perform some data cleaning before we start to answer Danny's key business questions in depth.*

**(a) Convert the week_date to a DATE format.**

````sql
ALTER TABLE dbo.weekly_sales
ALTER COLUMN week_date DATE;
````

**(b) Add week_number as the second column for each week_date value, for example any value from the 1st of January to 7th of January will be 1, 8th to 14th will be 2 etc.**

````sql
ALTER TABLE weekly_sales
ADD week_number AS DATEPART(week,week_date);
````
- Here, we take help of a computed column to add `week_number` to the existing table `weekly_sales`. This works in SQL Server.
- To position the column after week_date, we can use `AFTER` directive in the `ADD` statement. This works in MySQL Workbench.
However, it doesn't work in SQL Server. So, we need to use the System Interface to reorder the columns as desired.

**(c) Add a month_number with the calendar month for each week_date value as the 3rd column.**

````sql
ALTER TABLE weekly_sales
ADD month_number AS DATEPART(month,week_date);
````

**(d) Add a calendar_year column as the 4th column containing either 2018, 2019 or 2020 values.**

````sql
ALTER TABLE weekly_sales
ADD calendar_year AS DATEPART(year,week_date);
````

**(e) Add a new column called age_band after the original segment column using the following mapping on the number inside the segment value:**

![Screenshot 2023-03-02 122335](https://user-images.githubusercontent.com/96012488/222353329-4be01369-83b9-4e62-903a-c1c57f260dc2.png)

````sql
ALTER TABLE weekly_sales
ADD age_band AS CASE WHEN segment LIKE '_1' THEN 'Young Adults'
                     WHEN segment LIKE '_2' THEN 'Middle Aged'
                     WHEN segment LIKE '_3' OR segment LIKE '_4' THEN 'Retirees'
                     ELSE NULL
                     END;
````

**(f) Add a new demographic column using the following mapping for the first letter in the segment values:**

![Screenshot 2023-03-02 122458](https://user-images.githubusercontent.com/96012488/222353614-f13702f7-00cb-4734-848f-beb73e457a58.png)


````sql
ALTER TABLE weekly_sales
ADD demographic AS CASE WHEN segment LIKE 'C_' THEN 'Couples'
                        WHEN segment LIKE 'F_' THEN 'Families'
                        ELSE NULL
                        END;
````

**(g) Ensure all null string values with an "unknown" string value in the original segment column as well as the new age_band and demographic columns.**

````sql
UPDATE weekly_sales
SET segment = 'unknown'
WHERE segment = 'null';
````

- I tried updating all the 3 columns using a single UPDATE statemnt, however, computed columns cannnot be modified like this.
- To change the 2 computed columns, age_band and demographic, we need to drop these columns and recreate them.

````sql
ALTER TABLE weekly_sales
DROP COLUMN age_band,demographic;
````

````sql
ALTER TABLE weekly_sales
ADD age_band AS CASE WHEN segment LIKE '_1' THEN 'Young Adults'
                     WHEN segment LIKE '_2' THEN 'Middle Aged'
                     WHEN segment LIKE '_3' OR segment LIKE '_4' THEN 'Retirees'
                     ELSE 'unknown'
                     END;
````
````sql
ALTER TABLE weekly_sales
ADD demographic AS CASE WHEN segment LIKE 'C_' THEN 'Couples'
	                      WHEN segment LIKE 'F_' THEN 'Families'
	                      ELSE 'unknown'
	                      END;
````

**(h) Generate a new avg_transaction column as the sales value divided by transactions rounded to 2 decimal places for each record.**

- `avg_transaction` is computed by the division of two integer columns sales and transactions. 
- Integer division results in truncation of fractional part, whereas we want the value rounded to 2 decimal places.
- So, let's change the data type of one of the columns, say sales, to FLOAT.(DECIMAL/NUMERIC keep the insignificant zeros even afer rounding off, so we'll go with Float).

````sql
ALTER TABLE weekly_sales
ALTER COLUMN sales FLOAT;
````

- Now, Let's add the column `avg_transaction` by dividing sales by transactions and using the ROUND function to set it upto 2 decimal places.

````sql
ALTER TABLE weekly_sales
ADD avg_transaction AS ROUND(sales/transactions,2);
````

- Now, we need to generate a new table clean_weekly_sales which contains data with all the above steps performed.

*For this, I'm gonna rename this table to clean_weekly_sales. Then, I'll recreate the old table and name it weekly_sales. This is the opposite way of doing it. The straight way would be to create a copy of the original table, name it clean_weekly_sales and make given changes to it.*

***

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


## 📝 Solution 2. Data Exploration

*Now that we're done with cleaning, let's explore the data and answer some business questions.*

### 1. What day of the week is used for each week_date value?

````sql
SELECT DISTINCT
  week_date,DATENAME(weekday,week_date) AS day_of_week
FROM clean_weekly_sales
ORDER BY week_date;
````
**Answer:**

![Screenshot 2023-06-14 123730](https://github.com/PriyaPalak/8-Week-SQL-Challenge/assets/96012488/8be514df-7d81-40d9-9967-fb5c25550eb2)

- Monday is used to represent the week for each week_date value.

### 2. What range of week numbers are missing from the dataset?

````sql
SELECT DISTINCT 
  week_number,calendar_year
FROM clean_weekly_sales
ORDER BY week_number,calendar_year;
````
**Answer:**

![Screenshot 2023-06-14 124208](https://github.com/PriyaPalak/8-Week-SQL-Challenge/assets/96012488/84a4514f-b458-4130-8a19-566a0f622952)


![Screenshot 2023-06-14 124249](https://github.com/PriyaPalak/8-Week-SQL-Challenge/assets/96012488/936964ab-f2e1-4301-a882-cb4484f59e21)

- Thus, we have data for the week numbers 13 to 36 for the 3 years. The missing week numbers are 1 to 12 and 37 to 52.

### 3. How many total transactions were there for each year in the dataset?

````sql
SELECT 
  calendar_year,FORMAT(SUM(transactions),'0,,M') AS total_transactions
FROM clean_weekly_sales
GROUP BY calendar_year
ORDER BY total_transactions;
````
**Answer:**

![Screenshot 2023-06-14 124525](https://github.com/PriyaPalak/8-Week-SQL-Challenge/assets/96012488/f98efd9a-8035-46df-8297-560cef3e0f48)

````sql
/*
The FORMAT function has a way of trimming the thousands.

Each comma reduces the displayed value by 1000.

e.g.

select format(3000000,'$0,,,.000B')
select format(3000000,'$0,,M')
select format(3000000,'$0,K')

(note that I had to use decimals to show 3 million in Billions)

Output:
$0.003B
$3M
$3000K

*/
````

- Year 2020 accounts for the maximum number of transactions.
### 4. What is the total sales for each region for each month?

````sql
SELECT 
  region,month_number,calendar_year,FORMAT(SUM(sales),'$0,,,.00B') total_sales
FROM clean_weekly_sales
GROUP BY region,month_number,calendar_year
ORDER BY total_sales DESC;
````
**Answer:** 

![Screenshot 2023-06-14 124742](https://github.com/PriyaPalak/8-Week-SQL-Challenge/assets/96012488/7e85ada8-6f34-4c3b-87f4-ff336821aed9)

- Maximum sales value is for the region Oceania during August 2020.

### 5. What is the total count of transactions for each platform?

````sql
SELECT 
  platform,FORMAT(SUM(transactions),'0,,M') count_transactions
FROM clean_weekly_sales
GROUP BY platform
ORDER BY SUM(transactions);
````
**Answer:**

![Screenshot 2023-06-14 124852](https://github.com/PriyaPalak/8-Week-SQL-Challenge/assets/96012488/623a9153-a84c-4f8d-ba78-d590566662fc)

- Thus, the total count of transactions through the retail stores is much larger than  that of online Shopify stores.

### 6. What is the percentage of sales for Retail vs Shopify for each month?

````sql
WITH monthly_sales_cte AS
(
SELECT 
  calendar_year,month_number,platform, SUM(sales) monthly_sales
FROM clean_weekly_sales
GROUP BY calendar_year,month_number,platform

)
SELECT 
  calendar_year,month_number,
  ROUND(MAX(CASE WHEN  platform = 'Retail' THEN monthly_sales ELSE NULL END)*100/SUM(monthly_sales),2) retail_percent,
  ROUND(MAX(CASE WHEN  platform = 'Shopify' THEN monthly_sales ELSE NULL END)*100/SUM(monthly_sales),2) shopify_percent
FROM monthly_sales_cte
GROUP BY calendar_year,month_number;
````
**Answer:**

![Screenshot 2023-06-14 125623](https://github.com/PriyaPalak/8-Week-SQL-Challenge/assets/96012488/522cede5-0bad-4e56-92c4-d8634045d50b)

````sql
/* 
Approach: We created a CTE for monthly sales of each platform by grouping sales by platform, month and year. 
Then, we created 2 new columns as follows: 
- If the platform is 'Retail' take the monthly sales and divide by sum of monthly sales for that group(month,year),else null. 
And we take the Maximum of the percentage and null which gives the retail percentage.
- Similarly, we calculate the Shopify percentage.
*/
````
### 7. What is the percentage of sales by demographic for each year in the dataset?

````sql
WITH yearly_sales_cte AS
(
SELECT 
  calendar_year,demographic,SUM(sales) yearly_sales
FROM clean_weekly_sales
GROUP BY calendar_year,demographic
)
SELECT 
  calendar_year,
  ROUND(MAX(CASE WHEN demographic = 'Families' THEN yearly_sales ELSE NULL END)*100/SUM(yearly_sales),2) families_percent,
  ROUND(MAX(CASE WHEN demographic = 'Couples' THEN yearly_sales ELSE NULL END)*100/SUM(yearly_sales),2) couples_percent,
  ROUND(MAX(CASE WHEN demographic = 'unknown' THEN yearly_sales ELSE NULL END)*100/SUM(yearly_sales),2) unknown_percent
FROM yearly_sales_cte
GROUP BY calendar_year;
````

**Answer:**

![Screenshot 2023-06-14 130026](https://github.com/PriyaPalak/8-Week-SQL-Challenge/assets/96012488/511f920e-084d-4eb2-8925-29115a20e32e)

### 8. Which age_band and demographic values contribute the most to Retail sales?

````sql
SELECT 
  age_band,demographic,SUM(sales) AS retail_sales, ROUND(100*SUM(sales)/SUM(SUM(sales)) OVER(),2) AS contribution_percent
	-- Cannot apply aggregate function to an aggregate value, need to use the OVER() clause with SUM(SUM(sales))
FROM clean_weekly_sales                                  
WHERE platform = 'Retail'
GROUP BY age_band,demographic
ORDER BY retail_sales DESC;
````
**Answer:** 

![Screenshot 2023-06-14 130104](https://github.com/PriyaPalak/8-Week-SQL-Challenge/assets/96012488/d9bc2c17-5712-4b9b-8e5a-dca0ac8cc021)

- The maximum contribution to the retail sales is made by 'unknown' age_band and demographic at 42% followed by retired families with 16.7% and retired couples with 16.07%.

### 9. Can we use the avg_transaction column to find the average transaction size for each year for Retail vs Shopify? If not - how would you calculate it instead?

````sql
SELECT 
  calendar_year,platform,
  ROUND(AVG(avg_transaction),0) AS avg_avg_transaction_size,
  ROUND(SUM(sales)/SUM(transactions),0) AS avg_transaction_size
FROM clean_weekly_sales
GROUP BY calendar_year,platform
ORDER BY calendar_year,platform; 
````
**Answer:**

![Screenshot 2023-06-14 130222](https://github.com/PriyaPalak/8-Week-SQL-Challenge/assets/96012488/7b07b5b6-5d56-4286-8032-a5128fafec80)

````sql
/*
- Average of averages(of each group) is equal to the average of all the groups combined only when the number of elements in 
each group is same. Otherwise, the two calculations give different vales.
- The more accurate answer to the average of all the groups combined is achieved by the second way, as we calculate the 
average as per its definition in this way by taking the sum and dividing by the number of elements.
- Attempting to average existing averages without knowing the number of values contained in each value leads to statistical 
errors. 
- Either use the original values or keep hold of the number of values included in the average in order to keep your numbers 
accurate.
*/
````

***
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


## :memo: Solution 3. Before & After Analysis

This technique is usually used when we inspect an important event and its impact on the business performance.

*Here, the event is introduction of sustainable packaging methods in every single step from the farm all the way to the customer. Danny needs help to quantify the impact of this 
change on the sales performance for Data Mart and it’s separate business areas.*

*Taking the week_date value of 2020-06-15 as the baseline week where the Data Mart sustainable packaging changes came into effect,we 
would include all week_date values for 2020-06-15 as the start of the period after the change and the previous week_date values would be before.*

So, let's first see which week the changes were introduced.

````sql
SELECT DATEPART(week,'2020-06-15') AS week_number;
````
![Screenshot 2023-06-16 111852](https://github.com/PriyaPalak/8-Week-SQL-Challenge/assets/96012488/f68ea47e-676f-4ff6-8d0f-cefbd578b50b)

- The changes were introduced in `week number 25`.
- 12 weeks before 25th week : period `before` change
- 12 weeks including and after 25th week: period `after` change

### 1. What is the total sales for the 4 weeks before and after 2020-06-15? What is the growth or reduction rate in actual values and percentage of sales?

````sql
WITH sales_8_weeks AS
(
SELECT 
  week_date,week_number,SUM(sales) AS total_sales
FROM clean_weekly_sales
WHERE (week_number BETWEEN 21 AND 28)
AND (calendar_year = 2020)
GROUP BY week_date,week_number
)
, before_after_change AS
(
SELECT 
	SUM(CASE WHEN week_number BETWEEN 21 AND 24 THEN total_sales END) AS sales_before_change,
	SUM(CASE WHEN week_number BETWEEN 25 AND 28 THEN total_sales END) AS sales_after_change
 FROM sales_8_weeks
)
SELECT 
	(sales_after_change - sales_before_change) AS impact_on_sales,
	ROUND(100*((sales_after_change - sales_before_change)/sales_before_change),2) AS impact_percentage
FROM before_after_change;
````
**Answer:**

![Screenshot 2023-06-16 112638](https://github.com/PriyaPalak/8-Week-SQL-Challenge/assets/96012488/b688d3a4-0b1a-436b-925c-72216ab36b0f)


- Thus, the total sales reduced in the period after change by an amount of $26,884,188, reflecting a negative change by 1.15%.
- The possible reasons could be : customers didn't recognise the product on the shelves due to new packaging,
 or they opted for a cheaper alternative as sustainable changes often lead to higher prices.

### 2. What about the entire 12 weeks before and after?

A similar approach can be used to address this question.

````sql
WITH sales_24_weeks AS
(
SELECT 
  week_date,week_number,SUM(sales) AS total_sales
FROM clean_weekly_sales
WHERE (week_number BETWEEN 13 AND 36)
AND (calendar_year = 2020)
GROUP BY week_date,week_number
)
, before_after_change AS
(
SELECT 
	SUM(CASE WHEN week_number BETWEEN 13 AND 24 THEN total_sales END) AS sales_before_change,
	SUM(CASE WHEN week_number BETWEEN 25 AND 36 THEN total_sales END) AS sales_after_change
 FROM sales_24_weeks
)
SELECT 
	(sales_after_change - sales_before_change) AS impact_on_sales,
	ROUND(100*((sales_after_change - sales_before_change)/sales_before_change),2) AS impact_percentage
FROM before_after_change;
````

**Answer:**

![Screenshot 2023-06-16 112742](https://github.com/PriyaPalak/8-Week-SQL-Challenge/assets/96012488/403b9124-8641-4739-b045-27d5b9d9272f)

- If we consider the entire 12 weeks before and after the change, the decline in sales in the after period is even higher at a negative of 2.14%.

### 3. How do the sales metrics for these 2 periods before and after compare with the previous years in 2018 and 2019?

 Let's first compare the sales metrics for `4 weeks` before and after with the previous years 2018 and 2019.

- The same approach can be used here. We just need to add `calendar_year` into the context.

````sql
WITH sales_8_weeks AS
(
SELECT 
  calendar_year,week_date,week_number,SUM(sales) AS total_sales
FROM clean_weekly_sales
WHERE (week_number BETWEEN 21 AND 28)
GROUP BY calendar_year,week_date,week_number
)
, before_after_week_25 AS
(
SELECT calendar_year,
	SUM(CASE WHEN week_number BETWEEN 21 AND 24 THEN total_sales END) AS sales_before_change,
	SUM(CASE WHEN week_number BETWEEN 25 AND 28 THEN total_sales END) AS sales_after_change
 FROM sales_8_weeks
 GROUP BY calendar_year
)
SELECT calendar_year,
	(sales_after_change - sales_before_change) AS impact_on_sales,
	ROUND(100*((sales_after_change - sales_before_change)/sales_before_change),2) AS impact_percentage
FROM before_after_week_25;
````
**Answer:**

![Screenshot 2023-06-16 112908](https://github.com/PriyaPalak/8-Week-SQL-Challenge/assets/96012488/489a1e98-47a2-4ce0-af6c-b3b7942c9f44)


- In 2018, the sales increased by an amount of $ 4,102,105, indicating a positive change of 0.19% in the period after changes were introduced.
- Similarly, in 2019, the sales saw a rise amounting to $ 2,336,594, corresponding to a positive change of 0.1% in the period after packaging change.

- However, in 2020, there has been a significant drop in sales following the introduction of sustainable packaging. The sales declined by $ 26,884,188, reflecting a percentage drop of 1.15%.This reduction shows a significant impact on sales compared to the previous years 2018 and 2019.

Now, let's repeat this analysis for a period of `12 weeks`.



- For this, we just change the week number to `13 to 24` for before period and `25 to 36` for after period.

````sql
WITH sales_24_weeks AS
(
SELECT 
  calendar_year, week_date,week_number,SUM(sales) AS total_sales
FROM clean_weekly_sales
WHERE (week_number BETWEEN 13 AND 36)
GROUP BY calendar_year,week_date,week_number
)
, before_after_change_week AS
(
SELECT 
  calendar_year,
	SUM(CASE WHEN week_number BETWEEN 13 AND 24 THEN total_sales END) AS sales_before_25,
	SUM(CASE WHEN week_number BETWEEN 25 AND 36 THEN total_sales END) AS sales_after_25
 FROM sales_24_weeks
 GROUP BY calendar_year
)
SELECT calendar_year,
	(sales_after_25 - sales_before_25) AS impact_on_sales,
	ROUND(100*((sales_after_25 - sales_before_25)/sales_before_25),2) AS impact_percentage
FROM before_after_change_week
ORDER BY calendar_year;
````
**Answer:**

![Screenshot 2023-06-16 113410](https://github.com/PriyaPalak/8-Week-SQL-Challenge/assets/96012488/7e183564-975e-43b2-8a3a-7ae5be118338)

- As we can see above, in 2018, the sales increased by 1.63% in the period after change. While in 2019, a slight drop of 0.3% was observed.
- In 2020, however, the drop is even more significant at a staggerring 2.14%!

- Thus, there has been a significant impact on sales in 2020 due to packaging changes compared to the previous years, 2018 and 2019.

- When we compare the best year (2018) wih the worst year (2020), the variation in sales becomes even more apparent with a drop of 3.77% (1.63% + 2.14%).

***

*I thoroughly enjoyed solving this section of the case study - `Before-After analysis`.*
*Hadn't done this kind of analysis before. Got to learn something new and completely applicable in real world. Will use it as a guide when faced with a similar problem at workplace.*

***

-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## 📝 Solution 4. Bonus Question

### Which areas of the business have the highest negative impact in sales metrics performance in 2020 for the 12 week before and after period?

- region
- platform
- age_band
- demographic
- customer_type

 The approach is similar to comparing 12 weeks period before and after change, with the focus on a specific area of business.
 So, we'll just break down the numbers by that specific area of business, e.g., region.

### 1. Region

````sql
WITH sales_24_weeks AS
(
SELECT 
  region,week_date,week_number,SUM(sales) AS total_sales
FROM clean_weekly_sales
WHERE (week_number BETWEEN 13 AND 36)
AND (calendar_year = 2020)
GROUP BY region,week_date,week_number
)
, before_after_change AS
(
SELECT 
  region,
	SUM(CASE WHEN week_number BETWEEN 13 AND 24 THEN total_sales END) AS sales_before_change,
	SUM(CASE WHEN week_number BETWEEN 25 AND 36 THEN total_sales END) AS sales_after_change
 FROM sales_24_weeks
 GROUP BY region
)
SELECT 
  region,
	(sales_after_change - sales_before_change) AS impact_on_sales,
	ROUND(100*((sales_after_change - sales_before_change)/sales_before_change),2) AS impact_percentage
FROM before_after_change
ORDER BY impact_percentage;
````
**Answer:**

![Screenshot 2023-06-16 123043](https://github.com/PriyaPalak/8-Week-SQL-Challenge/assets/96012488/b3dfc587-c543-489b-94f8-3c4ab0e28eb1)


- Asia saw the highest negative impact on sales in 2020, with a decline of 3.26% in sales after packaging change.
- The metrics show that the sales declined in  every region except Europe. In fact, the sales saw a growth of 4.73% in Europe after the sustainable changes were introduced.
- This might be possible due to more inclination of Europeans toward sustainability. Maybe, they prefer sustainable products and don't mind paying slightly more in return.


### 2. Platform

````sql
WITH sales_24_weeks AS
(
SELECT 
  platform,week_date,week_number,SUM(sales) AS total_sales
FROM clean_weekly_sales
WHERE (week_number BETWEEN 13 AND 36)
AND (calendar_year = 2020)
GROUP BY platform,week_date,week_number
)
, before_after_change AS
(
SELECT 
  platform,
	SUM(CASE WHEN week_number BETWEEN 13 AND 24 THEN total_sales END) AS sales_before_change,
	SUM(CASE WHEN week_number BETWEEN 25 AND 36 THEN total_sales END) AS sales_after_change
 FROM sales_24_weeks
 GROUP BY platform
)
SELECT 
   platform,
	(sales_after_change - sales_before_change) AS impact_on_sales,
	ROUND(100*((sales_after_change - sales_before_change)/sales_before_change),2) AS impact_percentage
FROM before_after_change
ORDER BY impact_percentage;
````
**Answer:**

![Screenshot 2023-06-16 123202](https://github.com/PriyaPalak/8-Week-SQL-Challenge/assets/96012488/c53d7094-2b75-42e0-9c3f-67641f236472)


- Offline Retail stores saw a drop in sales by 2.43%, while Online Shopify stores observed a rise in sales by 7.18%.
- A possible explanation for this could be that Online customers of Data Mart are more inclined toward sustainable products compared to offline customers.




### 3. Age_band

````sql
WITH sales_24_weeks AS
(
SELECT 
  age_band,week_date,week_number,SUM(sales) AS total_sales
FROM clean_weekly_sales
WHERE (week_number BETWEEN 13 AND 36)
AND (calendar_year = 2020)
GROUP BY age_band,week_date,week_number
)
, before_after_change AS
(
SELECT 
  age_band,
	SUM(CASE WHEN week_number BETWEEN 13 AND 24 THEN total_sales END) AS sales_before_change,
	SUM(CASE WHEN week_number BETWEEN 25 AND 36 THEN total_sales END) AS sales_after_change
 FROM sales_24_weeks
 GROUP BY age_band
)
SELECT 
  age_band,
	(sales_after_change - sales_before_change) AS impact_on_sales,
	ROUND(100*((sales_after_change - sales_before_change)/sales_before_change),2) AS impact_percentage
FROM before_after_change
ORDER BY impact_percentage;
````

**Answer:**

![Screenshot 2023-06-16 123253](https://github.com/PriyaPalak/8-Week-SQL-Challenge/assets/96012488/8b81ca1d-e750-4c9a-b71a-0a37c05e597b)


- There has been a decline in sales across all the age groups after the packaging changes were introduced. However, the decline is highest in the unknown
- age group at a negative of 3.34%, whereas the 'Young Adults' group saw the lowest drop in sales due to packaging change at a negative of 0.92%.

### 4. Demographic

````sql
WITH sales_24_weeks AS
(
SELECT 
  demographic,week_date,week_number,SUM(sales) AS total_sales
FROM clean_weekly_sales
WHERE (week_number BETWEEN 13 AND 36)
AND (calendar_year = 2020)
GROUP BY demographic,week_date,week_number
)
, before_after_change AS
(
SELECT 
  demographic,
	SUM(CASE WHEN week_number BETWEEN 13 AND 24 THEN total_sales END) AS sales_before_change,
	SUM(CASE WHEN week_number BETWEEN 25 AND 36 THEN total_sales END) AS sales_after_change
 FROM sales_24_weeks
 GROUP BY demographic
)
SELECT 
  demographic,
	(sales_after_change - sales_before_change) AS impact_on_sales,
	ROUND(100*((sales_after_change - sales_before_change)/sales_before_change),2) AS impact_percentage
FROM before_after_change
ORDER BY impact_percentage;
````

**Answer:**

![Screenshot 2023-06-16 123336](https://github.com/PriyaPalak/8-Week-SQL-Challenge/assets/96012488/3685a931-bc1e-4a01-89af-5bd7b61cdeb0)

- Again, there has been a decline in sales across all demographics after the packaging changes, with the highest decline in the unknown demographic at a negative of 3.34%.
- While the 'families' demographic saw a drop of 1.82%, the drop in 'Couples' demographic was only 0.87%. 
- This might be because Families usually have more expenses and responsibilities compared to Couples. Hence, they find it difficult to pay a higher price for sustainable products and thus opt for cheaper alternatives.Couples, having less expenses and responsibilities, find it easier to afford slightly expensive sustainable products and thus stay loyal to the brand.

### 5. Customer_type

````sql
WITH sales_24_weeks AS
(
SELECT 
  customer_type,week_date,week_number,SUM(sales) AS total_sales
FROM clean_weekly_sales
WHERE (week_number BETWEEN 13 AND 36)
AND (calendar_year = 2020)
GROUP BY customer_type,week_date,week_number
)
, before_after_change AS
(
SELECT 
  customer_type,
	SUM(CASE WHEN week_number BETWEEN 13 AND 24 THEN total_sales END) AS sales_before_change,
	SUM(CASE WHEN week_number BETWEEN 25 AND 36 THEN total_sales END) AS sales_after_change
 FROM sales_24_weeks
 GROUP BY customer_type
)
SELECT 
  customer_type,
	(sales_after_change - sales_before_change) AS impact_on_sales,
	ROUND(100*((sales_after_change - sales_before_change)/sales_before_change),2) AS impact_percentage
FROM before_after_change
ORDER BY impact_percentage;
````
**Answer:**

![Screenshot 2023-06-16 123430](https://github.com/PriyaPalak/8-Week-SQL-Challenge/assets/96012488/1efba620-91c4-46e1-9ef1-45f12957e0f0)

- Sales reduced by 3% in the Guest Customers category, and by 2.27% in Existing Customers category. Meanwhile, 'New Customer' Category saw a rise in sales by 1.01%.
- A possible explanation could be that the brand, after making these changes, attracted new customers who are more into sustainable products and thus the sales increased in this category.

***





