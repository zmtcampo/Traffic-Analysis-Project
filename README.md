# Traffic Analysis Project

## Project Overview

This data analysis project aims to provide insights into the traffic performance of EWG metropolitan region, located between Missouri and Illinois. By examining different facets of the traffic data, we aim to uncover trends and develop a clearer understanding of the region’s traffic performance to support data-driven, goal-oriented, and performance-based decision-making.

## Data Sources

The primary dataset used for this analysis is the National Performance Management Research Data Set (NPMRDS). It provides detailed travel time data at 5-minute intervals, covering a 10-year period across 10 key roadways, in our case. Due to its high temporal resolution and extended time span, the resulting data file is exceptionally large (over 12 million rows). The analytical process involved using SQL to compile and query multiple data files, followed by data exploration, cleaning, and summarization, with final visualization and analysis performed in Python. Below is the sample of the raw data:

![link3_data_head](https://github.com/user-attachments/assets/1c8bcfc0-ee93-45cf-8836-b937906bf884)

## Tools
### a. SQL - Data Analysis
  
##### --1. Time Range and TMC Count 
```sql
SELECT 
    MIN(date) AS min_date,
    MAX(date) AS max_date,
    COUNT(DISTINCT tmc) AS unique_tmcs
FROM congestion;
```
#### Result
![sql1](https://github.com/user-attachments/assets/e32e1471-86f2-4020-9dc3-b45e22b44284)

#### --2. Miles Summary by Quantiles
```sql
WITH ordered_miles AS (
    SELECT miles,
           ROW_NUMBER() OVER (ORDER BY miles) AS rn,
           COUNT(*) OVER () AS total
    FROM congestion
)
SELECT 
    MIN(miles) AS min_miles,
    MAX(miles) AS max_miles,
    ROUND(AVG(miles), 2) AS avg_miles,
    (
        SELECT miles 
        FROM (
            SELECT miles,
                   ROW_NUMBER() OVER (ORDER BY miles) AS rn,
                   COUNT(*) OVER () AS total
            FROM congestion
        )
        WHERE rn = total / 2
    ) AS median_miles
FROM congestion;
```
#### Result
![sql2](https://github.com/user-attachments/assets/5d83bb14-f81d-4d6f-995c-9974839a9381)

#### --3. Daily Avg Travel Time
```sql
SELECT 
    DATE(date) AS day,
    ROUND(AVG(travel_time), 2) AS avg_tt
FROM congestion
GROUP BY day
ORDER BY day
LIMIT 5;
```
#### Result
![sql3](https://github.com/user-attachments/assets/a6e1a305-6c98-4f01-a3a1-54b66dd6e206)

#### --4. Hourly Travel Time Trend
```sql
WITH hourly_avg AS (
    SELECT 
        strftime('%Y-%m-%d %H:00', date) AS hour_block,
        ROUND(AVG(travel_time), 2) AS avg_tt,
        ROW_NUMBER() OVER (ORDER BY strftime('%Y-%m-%d %H:00', date)) AS rn
    FROM congestion
    GROUP BY hour_block
)
SELECT * FROM hourly_avg
LIMIT 5;
```
#### Result
![sql4](https://github.com/user-attachments/assets/d65039cc-93e2-4541-8061-0443f1c9e382)

#### --5. Peak Hour Travel Time by Period
```sql
SELECT 
    CASE 
        WHEN CAST(strftime('%H', date) AS INTEGER) BETWEEN 5 AND 8 THEN 'AM Peak'
        WHEN CAST(strftime('%H', date) AS INTEGER) BETWEEN 15 AND 17 THEN 'PM Peak'
        ELSE 'Off Peak'
    END AS period,
    ROUND(AVG(travel_time), 2) AS avg_tt,
    COUNT(*) AS records
FROM congestion
GROUP BY period;
```
#### Result
![sql5](https://github.com/user-attachments/assets/0d915fbb-3072-42f9-8ca4-c6b84b4b3601)

#### --6. Summary Data Creation
To support in-depth analysis and visualization, we derived a summary dataset from the raw data by leveraging advanced SQL techniques. These included Window Functions, Common Table Expressions (CTEs), Conditional Aggregation, Date and Time Functions, and CASE Expressions, which enabled us to transform the original variables into novel metrics that offer valuable insights into travel time reliability.

Specifically, we calculated:
-	Travel Time Index (TTI): the ratio of the 85th percentile travel time to the 50th percentile travel time.
-	Planning Time Index (PTI): the ratio of the 95th percentile travel time to the 50th percentile travel time.
-	Variability: the ratio of PTI to TTI, capturing the fluctuation in travel times.
-	Impact: the product of miles traveled and TTI, reflecting the effect of travel time variability on distance.
-	Severity: the average of TTI and PTI, indicating the overall reliability of travel times.
-	Peak: categorized into AM and PM values, highlighting variations during different times of day.

The resulting summary dataset includes the following columns: YEAR, TTI, PTI, PEAK (with AM and PM values), SEV (Severity), VAR (Variability), and IMP (Impact). This transformed dataset is better suited for in-depth analysis and visualization.

This is the SQL code for the above narrative:
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

-- Final results
SELECT * FROM metrics
LIMIT 5;
```
#### Result
![sql6](https://github.com/user-attachments/assets/f32184d3-dace-43aa-90f7-03dae0a8cdb4)

### b. Python - Created dashboard 
The summarized data was used to generate two interactive charts via Python's Dash framework:
- Line Graph: Trends over time/metric
- Bar Chart: Comparative analysis

#### Dashboard Preview
Attached are static snapshots of the dashboard. For the full interactive experience, access the live version here: [Insert Link]

![dash_f1_line](https://github.com/user-attachments/assets/15fe4bbe-4bec-42df-bffa-fec7065bb0b5)
![dash_f2_bar](https://github.com/user-attachments/assets/1d130928-efa3-444c-af2a-8bdc2bedfafc)


```sql

```
