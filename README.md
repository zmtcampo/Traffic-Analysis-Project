# Traffic Analysis Project

## Project Overview

This data analysis project aims to provide insights into the traffic performance of EWG metropolitan region, located between Missouri and Illinois. By examining different facets of the traffic data, we aim to uncover trends and develop a clearer understanding of the region’s traffic performance to support data-driven, goal-oriented, and performance-based decision-making.

## Data Sources

The primary dataset used for this analysis is the National Performance Management Research Data Set (NPMRDS). It provides detailed travel time data at 5-minute intervals, covering a 10-year period across 10 key roadways, in our case. Due to its high temporal resolution and extended time span, the resulting data file is exceptionally large (over 12 million rows). The analytical process involved using SQL to compile and query multiple data files, followed by data exploration, cleaning, and summarization, with final visualization and analysis performed in Python.

## Tools
### 1. SQL - Data Analysis
  
##### --1. Time Range and TMC Count 
```sql
SELECT 
    MIN(date) AS min_date,
    MAX(date) AS max_date,
    COUNT(DISTINCT tmc) AS unique_tmcs
FROM travel_data;
```
#### Result

#### --2. Miles Summary by Quantiles

#### --3. Daily Avg Travel Time

#### --4. Hourly Travel Time Trend

#### --5. Peak Hour Travel Time by Period

#### --6. Summary Data Creation














### 2. Python - Created dashboard 

## Exploratory Data Analysis
The raw data file has the following columns:

![link3_data_head](https://github.com/user-attachments/assets/1c8bcfc0-ee93-45cf-8836-b937906bf884)

 
EDA involved exploring the traffic data to answer key questions, such as:
-	What is the overall travel time trend?
-	What are the peak period travel times?
-	What is the mile distribution by quantiles?

```sql
WITH base AS (
  SELECT *,
         strftime('%Y', date) AS year,
         CAST(strftime('%H', date) AS INTEGER) AS hour,
         strftime('%w', date) AS dow
  FROM travel_data
  WHERE travel_time IS NOT NULL
),
peak_filtered AS (
  SELECT *,
    CASE
      WHEN dow IN ('1','2','3','4','5') AND hour BETWEEN 5 AND 8 THEN 'AM'
      WHEN dow IN ('1','2','3','4','5') AND hour BETWEEN 15 AND 17 THEN 'PM'
    END AS peak
  FROM base
  WHERE (dow BETWEEN '1' AND '5') AND (hour BETWEEN 5 AND 8 OR hour BETWEEN 15 AND 17)
),
ranked AS (
  SELECT *,
         ROW_NUMBER() OVER (PARTITION BY year, peak ORDER BY travel_time) AS rn,
         COUNT(*) OVER (PARTITION BY year, peak) AS total
  FROM peak_filtered
),
percentiles AS (
  SELECT year, peak,
         MAX(CASE WHEN rn = ROUND(0.50 * total) THEN travel_time END) AS p50,
         MAX(CASE WHEN rn = ROUND(0.85 * total) THEN travel_time END) AS p85,
         MAX(CASE WHEN rn = ROUND(0.95 * total) THEN travel_time END) AS p95,
         AVG(miles) AS avg_miles
  FROM ranked
  GROUP BY year, peak
),
metrics AS (
  SELECT year, peak,
         ROUND(p85 * 1.0 / p50, 3) AS tti,
         ROUND(p95 * 1.0 / p50, 3) AS pti,
         ROUND(p95 * 1.0 / p85, 3) AS va,
         ROUND(avg_miles * (p85 * 1.0 / p50), 3) AS imp,
         ROUND(((p85 * 1.0 / p50) + (p95 * 1.0 / p50)) / 2.0, 3) AS sev
  FROM percentiles
)
SELECT * FROM metrics
LIMIT 5;
```
