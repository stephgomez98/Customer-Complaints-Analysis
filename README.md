# Customer Complaints Analysis

## Data Walkthrough/Column Description

- **Company**: Name of the company involved in the complaint.
- **Company public response**: Public statement by the company regarding the complaint.
- **Company response to consumer**: Specific response or action taken by the company for the consumer.
- **Complaint ID**: Unique identifier for each complaint.
- **Consumer consent provided?**: Indicates if the consumer consented to share their complaint details (yes/no).
- **Consumer disputed?**: Indicates if the consumer disputed the company's response (yes/no).
- **Date Received**: Date the complaint was received.
- **Date Submitted**: Date the complaint was submitted by the consumer.
- **Issue**: Main category of the problem reported.
- **Product**: Type of product or service involved.
- **State**: U.S. state of the consumer.
- **Sub-issue**: More detailed categorization of the issue.
- **Sub-product**: More detailed categorization of the product.
- **Submitted via**: Medium through which the complaint was submitted.
- **Tags**: Specific tags related to the complaint.
- **Timely response?**: Indicates if the company responded in a timely manner (yes/no).
- **ZIP code**: Consumer's postal ZIP code.
- **All Complaints (Selected)**: Indicates if the complaint is part of a selected subset.
- **Number of Complaints**: Number of complaints related to the consumer, company, or product.
- **Target**: Target resolution date or goal for handling the complaint.
- **Time to Receipt**: Time taken from submission to receipt of the complaint.

### Row Count: 75,514
### Complaint ID: Primary Key

## Data Cleaning

Execute the following command to check the data type to know what you need to change and for memory usage:

````sql
EXEC sp_help complaints_data;
````
Correcting Dates
Through inspection, it was found that the Date_Received and Date_Submitted columns return negative numbers, so we will make those dates NULL.

Counting Incorrect Values
````sql
SELECT COUNT(complaint_id) AS countcomplaint
FROM (
    SELECT complaint_id, date_received,
        CASE 
            WHEN date_submitted > date_received THEN NULL
            ELSE date_submitted
        END AS date_submitted
    FROM complaints_data
) AS subquery
WHERE date_submitted IS NULL;
````
Updating the Original Database
````sql
UPDATE complaints_data
SET 
    date_submitted = CASE 
                        WHEN date_submitted > date_received THEN NULL
                        ELSE date_submitted
                     END,
    date_received = CASE 
                        WHEN date_submitted > date_received THEN NULL
                        ELSE date_received
                    END;
````
KPI Requirements
Average Response Time
Description: The average time taken by the company to respond to consumer complaints.

````sql
SELECT AVG(DATEDIFF(day, date_submitted, date_received)) AS days_difference_avg
FROM complaints_data;
````

Percentage of Timely Responses
Description: The percentage of complaints that received a response within a designated time frame (e.g., within 30 days).

````sql
WITH TotalComplaints AS (
    SELECT COUNT(*) AS total_count 
    FROM complaints_data
),
OneDayDifference AS (
    SELECT COUNT(*) AS one_day_count 
    FROM complaints_data 
    WHERE DATEDIFF(day, date_submitted, date_received) IN (0, 1)
)
SELECT 
    CAST(one_day_count AS FLOAT) / CAST(total_count AS FLOAT) * 100 AS percentage
FROM 
    TotalComplaints, OneDayDifference;
````
Consumer Dispute Rate
Description: The percentage of complaints where the consumer disputed the company's response.
````sql
WITH TotalComplaintsDisputed AS (
    SELECT COUNT(*) AS total_dispute 
    FROM complaints_data 
    WHERE consumer_disputed = 'Yes'
),
Total AS (
    SELECT COUNT(*) AS total 
    FROM complaints_data
)
SELECT 
    CAST(total_dispute AS FLOAT) / CAST(total AS FLOAT) * 100 AS percentage
FROM 
    TotalComplaintsDisputed, Total;
````
Complaint Volume by Product
Description: The number of complaints received for each product category.

````sql
SELECT product, COUNT(product) AS count 
FROM complaints_data 
GROUP BY product 
ORDER BY count DESC;

````
Complaint Volume by State
Description: The number of complaints received from each state.

````sql
SELECT state, COUNT(state) AS count 
FROM complaints_data 
GROUP BY state 
ORDER BY count DESC;
````

Resolution Rate
Description: The percentage of complaints that received a company response to the consumer.

````sql
WITH TotalComplaints AS (
    SELECT COUNT(*) AS total 
    FROM complaints_data
),
TotalCompanyResponse AS (
    SELECT COUNT(*) AS totalresponse 
    FROM complaints_data 
    WHERE company_response_to_consumer = 'Closed with explanation'
)
SELECT 
    CAST(totalresponse AS FLOAT) / CAST(total AS FLOAT) * 100 AS percentage 
FROM 
    TotalComplaints, TotalCompanyResponse;
Average Time to Receipt

````
Description: The average time taken for complaints to be processed from the date submitted to the date received.

````sql
SELECT AVG(DATEDIFF(day, date_submitted, date_received)) 
FROM complaints_data;
````

Top Issues by Product
Description: The most common issues reported for each product category.

````sql
SELECT product, issue, COUNT(issue) AS countissues 
FROM complaints_data 
GROUP BY product, issue 
ORDER BY countissues DESC;
````
Complaints by Submission Method
Description: The distribution of complaints based on the submission method (e.g., Email, Phone, Website).

````sql
SELECT submitted_via, COUNT(submitted_via) AS countsubmittedvia 
FROM complaints_data 
GROUP BY submitted_via 
ORDER BY countsubmittedvia DESC;
````
Complaint Volume Over Time
Description: Analysis of complaint volumes over different time periods (e.g., monthly, quarterly).

````sql
SELECT YEAR(date_received) AS years, COUNT(*) AS yearcount 
FROM complaints_data 
GROUP BY YEAR(date_received) 
ORDER BY yearcount DESC;
````

Effectiveness of Company Responses
Description: Measure the effectiveness of the company's responses in resolving complaints without further escalation.

````sql
WITH TotalNoDispute AS (
    SELECT COUNT(consumer_disputed) AS totalno 
    FROM complaints_data 
    WHERE consumer_disputed = 'No'
),
Total AS (
    SELECT COUNT(*) AS totals 
    FROM complaints_data
)
SELECT 
    CAST(totalno AS FLOAT) / CAST(totals AS FLOAT) * 100 AS percentage_no_dispute 
FROM 
    TotalNoDispute, Total;
````

Complaint Recurrence Rate
Description: Average time taken to respond to complaints categorized by issue type.

````sql
WITH TotalNoDispute AS (
    SELECT COUNT(*) AS totalno 
    FROM complaints_data 
    WHERE consumer_disputed = 'No'
),
Total AS (
    SELECT COUNT(*) AS totals 
    FROM complaints_data
)
SELECT 
    (totalno * 1.0 / totals) * 100 AS percentage_no_dispute 
FROM 
    TotalNoDispute, Total;
````

Resolution Time by Product and State
Description: Average time taken to resolve complaints, broken down by product type and consumer state.

````sql
SELECT product, state, AVG(DATEDIFF(day, date_submitted, date_received)) AS avg_resolution_time 
FROM complaints_data 
GROUP BY product, state 
ORDER BY avg_resolution_time DESC;
````
Here are partical stored procedures that should be in place:


To return # of complaints for a specific year
````sql
USE [complaints]
GO
/****** Object:  StoredProcedure [dbo].[CountYear]    Script Date: 8/7/2024 2:09:12 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
ALTER PROCEDURE [dbo].[CountYear]
@Year INT
as
begin
	select count(*) as count
	from complaints_data
	where year(Date_Submitted) = @Year
end
````
To return # of complaints for years in between, if no dates specified then use min and max year in dataset
````sql
CREATE PROCEDURE IFDEFAULT2
@product varchar (255) = 'Mortgages',
@beginningYear INT = NULL,
@endingYear INT = NULL
AS
BEGIN
	IF @beginningYear is null
	begin
		Select @beginningYear = MIN(YEAR(date_submitted))
		from complaints_data;
	end

	IF @endingYear is null
	begin
		select @endingYear = MAX(year(date_submitted))
		from complaints_data;
	end

	select count(*) from complaints_data
	where product = @product and year(date_submitted) between @BeginningYear and @endingYear;
END
````
--had to change the data type for time_to_receipt because it was originally varchar
 alter table complaints_data
 alter column time_to_receipt int
 
subqueries:

return all the complaints that have more than average time to receipt
````sql
 select * from complaints_data where time_to_receipt>(
 select avg(time_to_receipt) from complaints_data ) group by product
````

