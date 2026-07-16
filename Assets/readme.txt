

# Austin Bikeshare Data Analysis Portfolio

## Project Overview
This project uses SQL to solve 10 realistic operational and business questions using Austin's public bikeshare data. 
Instead of just running basic aggregations, the queries in this repo handle messy real-world data constraints using:
* **Common Table Expressions (CTEs)** to isolate high-traffic windows.
* **Window Functions** (`RANK()`, `LEAD()`, `LAG()`) to track bike migrations and station volumes.
* **Conditional Logic & Date Functions** to clean up mismatched timestamps and categorize subscriber behaviors.

---

## The Data Source
Everything runs live on Google BigQuery's public datasets. You don't need to download huge CSVs or host a database locally.

* **Billing Project Path:** `bigquery-public-data.austin_bikeshare`
* **Tables Queried:**
  * `bikeshare_trips`: Start/end times, durations, station IDs, and subscriber types for millions of rides.
  * `bikeshare_stations`: Geolocation data, station status (active/closed), and dock capacities.

---

## How to Run These Queries (Free)
Because the data is public, you can execute these queries in the cloud in instant at my shared queries:

1. Open the [my BigQuery SQL Workspace](https://console.cloud.google.com/bigquery?ws=!1m7!1m6!12m5!1m3!1sstrange-terra-500210-k8!2sus-central1!3scd9fd181-7ae3-4597-b196-29602d8daed6!2e1)

---
## ERD Diagram of the dataset
```
![Alt text](/relative/path/to/img.jpg?raw=true "Optional Title")
ambil dari repo link - > copy permalink

```
## The 10 Business Questions & Insights

* Below is a breakdown of the core problems solved. The full scripts are organized inside the `/Queries` folder, and complete result screenshot or CSV tables are located in `/queries_result`.
---

<details><summary><strong>Question 1</strong></summary>
  
### Q1: Find the top 5 most popular starting stations for casual "Walk Up" users,ignoring any test, repair, or staff stations (where the station name contains "test" or "status")
* **SQL Concepts:**  `COUNT()`, `GROUP BY`,`LIMIT`,`WHERE`
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
```
copy perma link image
```

</details>

<details><summary><strong>Question 2</strong></summary>

---
### Q2: Calculate the average trip duration (rounded to one decimal place) in minutes, broken down by the hour of the day the ride started

* **SQL Concepts:**  `COUNT`, `ROUND`,`AVG`,`GROUP BY`
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
```
copy perma link image
```
</details>

---
<details><summary><strong>Question 3</strong></summary>
### Q3: Identify all physically active bike stations status = 'active' that have recorded zero starting trips in the trips history

* **SQL Concepts:**  `JOIN`, `WHERE`
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
```
copy perma link image
```
</details>

---
<details><summary><strong>Question 4</strong></summary>

### Q4: Categorize all trips into three duration buckets
* "Short" (under 15 mins), "Medium" (15–45 mins) and "Long" (over 45 mins). Show the total count of trips and the percentage of total trips each Categories represents.

* **SQL Concepts:**  `CASE WHEN`, `ROUND`, `COUNT`, `SUM`,`OVER`
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
```
copy perma link image
```
</details>

---
<details><summary><strong>Question 5</strong></summary>

### Q5: The top 10 most common "routes" (Start Station to End Station).

* **SQL Concepts:**  `COUNT`, `JOIN`, `LIMIT`, `GROUP BY`.
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
```
copy perma link image
```
</details>

---
<details><summary><strong>Question 6</strong></summary>

### Q6: Find  stations  where  the  total  historical  number  of  trips  starting  there  is  greater  than  1,000  times and the  physical  dock  capacity  of  the  station. Exclude  stations  with  0  docks..

* **SQL Concepts:**  `COUNT`, `JOIN`, `LIMIT`, `GROUP BY`,`HAVING`
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
```
copy perma link image
```
</details>

---
<details><summary><strong>Question 7</strong></summary>

### Q7: Find the average number of daily trips taken during the busiest month adn year in the dataset. 

* **SQL Concepts:**  `CTE`, `JOIN`, `LIMIT`, `GROUP BY`,`HAVING`,`EXTRACT`
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
```
copy perma link image
```
</details>

---
<details><summary><strong>Question 8</strong></summary>

### Q8: Peak Traffic Running Totals (Window Functions)
* For each unique bike station, find the top 3 busiest dates in history based on total daily departures. Display the station name, date, daily trip count, the density rank, and a running total of trips accumulated across those top 3 days. 

* **SQL Concepts:**  `CTE`, `JOIN`, `LIMIT`, `GROUP BY`,`HAVING`,`EXTRACT`
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
```
copy perma link image
```
</details>

---
<details><summary><strong>Question 9</strong></summary>

### Q9: Identifying "Hoarder" Docks

* Track individual bikes (bike_id) chronologically to find instances where a bike was returned to a station, but sat completely idle and unused for more than 72 consecutive hours before the next user unlocked it.

* **SQL Concepts:**  `CTE`, `JOIN`, `LIMIT`, `GROUP BY`,`OVER`,`LAG`,` TIMESTAMP_ADD`
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
```
copy perma link image
```
</details>

---
<details><summary><strong>Question 10</strong></summary>

### Q10: Geospatial Rebalancing

* Calculate straight-line distance in kilometers between the physical starting and ending stations for all trips using latitude and longitude coordinates. Group by subscriber type to see if casual riders travel further than daily commuters.

* **SQL Concepts:**  `CTE`, `JOIN`, `LIMIT`, `GROUP BY`,`OVER`,`LAG`,` TIMESTAMP_ADD`
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
```
copy perma link image
```
</details>
