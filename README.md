

# Austin Bikeshare Data Analysis Portfolio

## Project Overview
This project uses SQL to solve 10 realistic operational and business challenges using Austin’s public bikeshare system.

Instead of basic SELECT * dumps, these queries tackle complex, real-world data constraints. You will see:
* **Common Table Expressions (CTEs) used to isolate high-traffic windows and calculate complex step-by-step metrics.**
* **Window Functions (DENSE_RANK(), LAG()) tracking bike migrations, running totals, and station idle times.**
* **Geospatial & Conditional Logic to clean up messy, mismatched station coordinates and categorize subscriber behavior.**

This portfolio is designed to showcase how database analytics directly translates to actionable operational insights—like finding where bikes sit hoarded or identifying which routes need physical capacity upgrades.

---

## The Data Source
Everything runs live on Google BigQuery's public datasets. You don't need to download huge CSVs or host a database locally.

* **Free Project Path:** `bigquery-public-data.austin_bikeshare`
* **Tables Queried:**
  * `bikeshare_trips`: Start/end times, durations, station IDs, and subscriber types for millions of rides.
  * `bikeshare_stations`: Geolocation data, station status (active/closed), and dock capacities.

---

## How to Run These Queries (Free)
Because the data is public, you can execute these queries in the cloud in instant at my shared queries:

1. Open the [my BigQuery SQL Workspace](https://console.cloud.google.com/bigquery?ws=!1m7!1m6!12m5!1m3!1sstrange-terra-500210-k8!2sus-central1!3scd9fd181-7ae3-4597-b196-29602d8daed6!2e1) (Ensure you are logged into a Google account).

---
## ERD Diagram of the dataset

![image alt](https://github.com/martarhezam/SQL_Austin_bikeshare_personal_project/blob/d3583ab5aa89f18fcc650c65e17ff200316d708d/Assets/Dataset_ERD/BIKESHARE_AUSTIN%20ERD.png)

---

## Insights & Executive Summary

Analyzing these systemic operational metrics reveals a clear behavioral dichotomy: a hyper-localized university commuter base contrasting with a highly volatile casual tourism market, exposing significant resource allocation bottlenecks.

### 1. The University Commuter Engine

* **The Insight:** **57.22% of all rides are short utility trips** lasting under 15 minutes, heavily driven by daily student transit patterns, you can look Q4 result.

* **The Insight:** A single high-traffic station—**21st & Speedway @PCL**—acts as the ultimate system bottleneck, handling over **179,000 historical departures** despite having a physical infrastructure limit of only **22 docks**. It routinely breaks past its capacity. check Q6 for the result.

* **The Insight:** The absolute most heavily trafficked route in the entire network flows directly through the campus core between **Dean Keeton & Whitis** and **21st & Speedway @PCL** (**25,891 trips**) you can look Q5 result.

### 2. High-Margin Casual Volatility

* **The Insight:** Casual "Walk Up" users display completely different spatial footprints, heavily favoring scenic and recreational hubs like **Riverside @ S. Lamar** (15,762 trips) and **Zilker Park** (13,606 trips)you can look Q1 result.

* **The Insight:** Late-night trip durations skyrocket to an average peak of **67.3 minutes at 3:00 AM**, dwarfing standard afternoon rush-hour commuter averages of **~28 minutes at 5:00 PM**, Check Q2 Result.

### 3. The "Hoarder Station" Rebalancing Problem

* **The Insight:** While high-demand zones face acute asset shortages, substantial deadweight exists within the system. Navigational sequence tracking caught massive logistical oversights—such as **Bike ID 19 sitting completely idle at the 26th/Nueces station for 18,086 consecutive hours (over 2 full years)** without a single user unlock, Check Q9 Result queries.

---

## The 10 Business Questions

* Below is a breakdown of the core problems solved. The full scripts are organized inside the `/Queries` folder, and complete result screenshot or CSV tables are located in `/Assets/queries_result`.
---

<details><summary><strong>Question 1</strong></summary>

### Q1: Find the top 5 most popular starting stations for casual "Walk Up" users,ignoring any test, repair, or staff stations (where the station name contains "test" or "status")
* **SQL Concepts:**  `COUNT(), GROUP BY, ORDER BY, LIMIT, LOWER(), NOT LIKE (Pattern Matching), WHERE (Filtering)`
```sql
SELECT 
  start_station_name,
  COUNT(trip_id) AS total_trips
FROM 
  `bigquery-public-data.austin_bikeshare.bikeshare_trips`
WHERE 
  subscriber_type = 'Walk Up'
  AND LOWER(start_station_name) NOT LIKE '%test%'
  AND LOWER(start_station_name) NOT LIKE '%status%'
GROUP BY 
  start_station_name
ORDER BY 
  total_trips DESC
LIMIT 5;
```
Result:

![image alt](
https://github.com/martarhezam/SQL_Austin_bikeshare_personal_project/blob/d3583ab5aa89f18fcc650c65e17ff200316d708d/Assets/queries_result/Q1_Top%205%20Popular%20Starting%20Stations%20for%20Walk%20Up%20Casual%20Users.png
)
</details>

---
<details><summary><strong>Question 2</strong></summary>
  
### Q2: Calculate the average trip duration (rounded to one decimal place) in minutes, broken down by the hour of the day the ride started

* **SQL Concepts:** `EXTRACT(HOUR FROM ...), AVG(), ROUND(), GROUP BY, ORDER BY ASC`
```sql
SELECT 
  EXTRACT(HOUR FROM start_time) AS start_hour,
  COUNT(trip_id) AS total_trips,
  ROUND(AVG(duration_minutes), 1) AS avg_duration_minutes
FROM 
  `bigquery-public-data.austin_bikeshare.bikeshare_trips`
GROUP BY 
  start_hour
ORDER BY 
  start_hour ASC;
```
Result:

[Show Link Data CSV](
https://github.com/martarhezam/SQL_Austin_bikeshare_personal_project/blob/d3583ab5aa89f18fcc650c65e17ff200316d708d/Assets/queries_result/Q2_Average_trip_durations.csv
) *(Ctrl + Click to open in a new tab)*


</details>

---
<details><summary><strong>Question 3</strong></summary>
  
### Q3: Identify all physically active bike stations status = 'active' that have recorded zero starting trips in the trips history

* **SQL Concepts:**  `LEFT JOIN, IS NULL (Null Handling), WHERE (Filtering)`
```sql
SELECT 
  s.station_id,
  s.name AS station_name
FROM 
  `bigquery-public-data.austin_bikeshare.bikeshare_stations` AS s
LEFT JOIN 
  `bigquery-public-data.austin_bikeshare.bikeshare_trips` AS t
  ON s.station_id = t.start_station_id
WHERE 
  s.status = 'active'
  AND t.trip_id IS NULL;
```
Result:

![image alt](
https://github.com/martarhezam/SQL_Austin_bikeshare_personal_project/blob/d3583ab5aa89f18fcc650c65e17ff200316d708d/Assets/queries_result/Q3_Active%20Bike%20Stations%20with%20Zero%20Trip%20History.png
)
</details>

---
<details><summary><strong>Question 4</strong></summary>

### Q4: Categorize all trips into three duration buckets
* "Short" (under 15 mins), "Medium" (15–45 mins) and "Long" (over 45 mins). Show the total count of trips and the percentage of total trips each Categories represents.

* **SQL Concepts:**  `CASE WHEN (Conditional Logic), BETWEEN, COUNT(*), SUM() OVER() (Window / Analytic Aggregate), Arithmetic Operations (Percentage math)`
```sql
SELECT 
  CASE 
    WHEN duration_minutes < 15 THEN 'Short (< 15 mins)'
    WHEN duration_minutes BETWEEN 15 AND 45 THEN 'Medium (15-45 mins)'
    ELSE 'Long (> 45 mins)'
  END AS trip_duration_category,
  COUNT(*) AS trip_count,
  ROUND(COUNT(*) * 100 / SUM(COUNT(*)) OVER(), 2) AS percentage_of_total
FROM 
  `bigquery-public-data.austin_bikeshare.bikeshare_trips`
GROUP BY 
  trip_duration_category;
```
Result:

![image alt](
https://github.com/martarhezam/SQL_Austin_bikeshare_personal_project/blob/d3583ab5aa89f18fcc650c65e17ff200316d708d/Assets/queries_result/Q4_Trip%20Count%20%26%20Percentage%20Breakdown%20by%20Duration%20Buckets.png
)
</details>

---
<details><summary><strong>Question 5</strong></summary>

### Q5: The top 10 most common "routes" (Start Station to End Station).

* **SQL Concepts:**  `JOINs, REGEXP_CONTAINS (Regular Expressions), SAFE_CAST (Data Type Conversion), Inequality Operators (!=), GROUP BY, LIMIT`
```sql
SELECT 
  t.start_station_id,
  s_start.name AS start_station_name,
  t.end_station_id,
  s_end.name AS end_station_name,
  COUNT(*) AS total_trips
FROM 
  `bigquery-public-data.austin_bikeshare.bikeshare_trips` AS t
JOIN 
  `bigquery-public-data.austin_bikeshare.bikeshare_stations` AS s_start
  ON t.start_station_id = s_start.station_id  
JOIN 
  `bigquery-public-data.austin_bikeshare.bikeshare_stations` AS s_end
  ON SAFE_CAST(t.end_station_id AS INT64) = s_end.station_id -- change the datatype
WHERE 
  -- 1. REGEX For make sure non number ID on stations id
  REGEXP_CONTAINS(t.end_station_id, r'^[0-9]+$')
  -- 2. Make sure bike is moved
  AND t.start_station_id != SAFE_CAST(t.end_station_id AS INT64)
GROUP BY 
  t.start_station_id, s_start.name, t.end_station_id, s_end.name
ORDER BY 
  total_trips DESC
LIMIT 10;
```
Result:

![image alt](
https://github.com/martarhezam/SQL_Austin_bikeshare_personal_project/blob/d3583ab5aa89f18fcc650c65e17ff200316d708d/Assets/queries_result/Q5_Top%2010%20Most%20Common%20Bike%20Routes.png
)
</details>

---
<details><summary><strong>Question 6</strong></summary>

### Q6: Find  stations  where  the  total  historical  number  of  trips  starting  there  is  greater  than  1,000  times and the  physical  dock  capacity  of  the  station. Exclude  stations  with  0  docks..

* **SQL Concepts:**  `JOIN, COUNT(), GROUP BY, HAVING (Grouped Filter), Arithmetic Expressions (multiplication inside HAVING)`
```sql
SELECT 
  s.station_id,
  s.name AS station_name,
  s.number_of_docks,
  COUNT(t.trip_id) AS total_trips_started
FROM 
  `bigquery-public-data.austin_bikeshare.bikeshare_stations` AS s
JOIN 
  `bigquery-public-data.austin_bikeshare.bikeshare_trips` AS t
  ON s.station_id = t.start_station_id
WHERE 
  s.number_of_docks > 0 
GROUP BY 
  s.station_id, s.name, s.number_of_docks
HAVING
  total_trips_started > (s.number_of_docks * 1000)
ORDER BY 
  total_trips_started DESC;
```
Result:

[Show Link Data CSV](
https://github.com/martarhezam/SQL_Austin_bikeshare_personal_project/blob/d3583ab5aa89f18fcc650c65e17ff200316d708d/Assets/queries_result/Q6_Station%20Capacity%20vs.%20Demand.csv
) *(Ctrl + Click to open in a new tab)*
</details>

---
<details><summary><strong>Question 7</strong></summary>

### Q7: Find the average number of daily trips taken during the busiest month adn year in the dataset. 

* **SQL Concepts:**  `WITH (Common Table Expressions - CTE), CROSS JOIN, EXTRACT(DATE/MONTH/YEAR), AVG(), LIMIT, GROUP BY`
```sql
WITH PeakMonth AS (
  SELECT 
    EXTRACT(YEAR FROM start_time) AS peak_year,
    EXTRACT(MONTH FROM start_time) AS peak_month,
    COUNT(*) AS total_trips
  FROM 
    `bigquery-public-data.austin_bikeshare.bikeshare_trips`
  GROUP BY peak_year, peak_month
  ORDER BY COUNT(*) DESC -- filter by top first
  LIMIT 1  -- select the MOST 
),
DailyTripsInPeak AS (
  SELECT 
    EXTRACT(DATE FROM t.start_time) AS trip_date,
    COUNT(*) AS daily_count
  FROM 
    `bigquery-public-data.austin_bikeshare.bikeshare_trips` AS t
  CROSS JOIN 
    PeakMonth AS p
  WHERE 
    EXTRACT(YEAR FROM t.start_time) = p.peak_year -- filtering 
    AND EXTRACT(MONTH FROM t.start_time) = p.peak_month -- filtering
  GROUP BY 
    trip_date
)
SELECT 
  AVG(daily_count) AS avg_daily_trips_in_peak_month
FROM 
  DailyTripsInPeak;
```
Result:

![image alt](
https://github.com/martarhezam/SQL_Austin_bikeshare_personal_project/blob/d3583ab5aa89f18fcc650c65e17ff200316d708d/Assets/queries_result/Q7_Peak%20Month%20Analysis%20Highest%20Trip%20Volume%20%26%20Daily%20Average.png
)
</details>

---
<details><summary><strong>Question 8</strong></summary>

### Q8: Peak Traffic Running Totals (Window Functions)
* For each unique bike station, find the top 3 busiest dates in history based on total daily departures. Display the station name, date, daily trip count, its dense rank, and a running total of trips accumulated across those top 3 days. 

* **SQL Concepts:**  `WITH (CTEs), REPLACE(), DENSE_RANK() OVER() (Ranking Window Function), SUM() OVER(ROWS BETWEEN ...) (Cumulative Window Frame), Ordinal Grouping (GROUP BY 1, 2)`
```sql
WITH DailyStationVolume AS (
  SELECT 
    REPLACE(start_station_name, '/', ' & ') AS start_station_name,
    EXTRACT(DATE FROM start_time) AS trip_date,
    COUNT(*) AS daily_trips
  FROM 
    `bigquery-public-data.austin_bikeshare.bikeshare_trips`
  WHERE 
    start_station_name IS NOT NULL
  GROUP BY 
    1, 2 
),
RankedDays AS (
  SELECT 
    start_station_name,
    trip_date,
    daily_trips,
    DENSE_RANK() OVER(PARTITION BY start_station_name ORDER BY daily_trips DESC) AS ranking
  FROM 
    DailyStationVolume
)
SELECT 
  start_station_name,
  trip_date,
  daily_trips,
  ranking,
  SUM(daily_trips) OVER(PARTITION BY start_station_name ORDER BY daily_trips DESC ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM 
  RankedDays
WHERE 
  ranking <= 3
ORDER BY 
  start_station_name, ranking;

```
Result:

[Show Link Data CSV](
https://github.com/martarhezam/SQL_Austin_bikeshare_personal_project/blob/d3583ab5aa89f18fcc650c65e17ff200316d708d/Assets/queries_result/Q8_Peak%20Traffic%20Running%20Totals.csv
)
</details>

---
<details><summary><strong>Question 9</strong></summary>

### Q9: Identifying "Hoarder" Docks

* Track individual bikes (bike_id) chronologically to find instances where a bike was returned to a station, but sat completely idle and unused for more than 72 consecutive hours before the next user unlocked it.

* **SQL Concepts:** `WITH (CTEs), LAG() OVER(PARTITION BY ...) (Navigational Window Function), TIMESTAMP_ADD(), TIMESTAMP_DIFF(), WHERE (Multi-condition Filtering)`
```sql
WITH OrderedTrips AS (
  SELECT 
    bike_id,
    start_station_name,
    start_time,
    --  Calculate the end_time by adding duration_minutes to start_time
    TIMESTAMP_ADD(start_time, INTERVAL duration_minutes MINUTE) AS calculated_end_time,
    --  Grab the previous trip end time and station name
    LAG(TIMESTAMP_ADD(start_time, INTERVAL duration_minutes MINUTE)) OVER (
      PARTITION BY bike_id ORDER BY start_time ASC
    ) AS previous_end_time,
    LAG(end_station_name) OVER (
      PARTITION BY bike_id ORDER BY start_time ASC
    ) AS previous_end_station
  FROM 
    `bigquery-public-data.austin_bikeshare.bikeshare_trips`
  WHERE 
    bike_id IS NOT NULL
)
SELECT 
  bike_id,
  previous_end_station AS idle_station,
  previous_end_time AS parked_time,
  start_time AS next_pickup_time,
  -- Calculate idle hours
  TIMESTAMP_DIFF(start_time, previous_end_time, HOUR) AS hours_sat_idle
FROM 
  OrderedTrips
WHERE 
  previous_end_station = start_station_name -- Confirms the bike not moved
  AND TIMESTAMP_DIFF(start_time, previous_end_time, HOUR) > 72
ORDER BY 
  hours_sat_idle DESC
LIMIT 10;
```
Result:

![image alt](
https://github.com/martarhezam/SQL_Austin_bikeshare_personal_project/blob/d3583ab5aa89f18fcc650c65e17ff200316d708d/Assets/queries_result/Q9_completely%20idle%20bike_stations%20and%20unused%20for%20more%20than%2072%20consecutive%20hours.png
)
</details>

---
<details><summary><strong>Question 10</strong></summary>

### Q10: Geospatial Rebalancing

* Calculate straight-line distance in kilometers between the physical starting and ending stations for all trips using latitude and longitude coordinates. Group by subscriber type to see if casual riders travel further than daily commuters.

* **SQL Concepts:**  `WITH (CTEs), TRIM() & SPLIT(), OFFSET(), SAFE_CAST(), ST_GEOGPOINT() (Geospatial constructor), ST_DISTANCE() (Geospatial measurement), AVG(), GROUP BY`
```sql
WITH TripGeometries AS (
  SELECT 
    t.subscriber_type,
    
    -- seperate latitude and longtitude with split trim
    ST_GEOGPOINT(
      SAFE_CAST(SPLIT(TRIM(s_start.location, "()"), ", ")[OFFSET(1)] AS FLOAT64), 
      SAFE_CAST(SPLIT(TRIM(s_start.location, "()"), ", ")[OFFSET(0)] AS FLOAT64)
    ) AS start_geo,

    ST_GEOGPOINT(
      SAFE_CAST(SPLIT(TRIM(s_end.location, "()"), ", ")[OFFSET(1)] AS FLOAT64), 
      SAFE_CAST(SPLIT(TRIM(s_end.location, "()"), ", ")[OFFSET(0)] AS FLOAT64)
    ) AS end_geo
    
  FROM 
    `bigquery-public-data.austin_bikeshare.bikeshare_trips` AS t
  JOIN 
    `bigquery-public-data.austin_bikeshare.bikeshare_stations` AS s_start 
    ON t.start_station_id = s_start.station_id
  JOIN 
    `bigquery-public-data.austin_bikeshare.bikeshare_stations` AS s_end 
    ON SAFE_CAST(t.end_station_id AS INT64) = s_end.station_id
  WHERE 
    s_start.location IS NOT NULL 
    AND s_end.location IS NOT NULL
    AND SAFE_CAST(t.end_station_id AS INT64) IS NOT NULL
    AND t.start_station_id != SAFE_CAST(t.end_station_id AS INT64)
)
SELECT 
  subscriber_type,
  COUNT(*) AS total_trips,
  ROUND(AVG(ST_DISTANCE(start_geo, end_geo) / 1000), 2) AS avg_distance_km
FROM 
  TripGeometries
GROUP BY 
  subscriber_type
ORDER BY 
  avg_distance_km DESC;
```
Result:
[Show Link Data CSV](
https://github.com/martarhezam/SQL_Austin_bikeshare_personal_project/blob/5e157aac5e2c0de72cb7aa30e74bbde018b53133/Assets/queries_result/Q10_Spatial%20Trip%20Distance%20(km)%20by%20Subscriber%20Type.csv
)
</details>

---

*Thank you for exploring the Austin Bikeshare Data Analysis Portfolio.*

