# Google Data Analytics Capstone: Cyclistic Case Study

# Introduction

*You are a junior data analyst working at Cyclistic, a bike-share company in Chicago. Until now, Cyclistic’s marketing strategy relied on building general awareness and appealing to broad consumer segments. One approach that helped make these things possible was the flexibility of its pricing plans: single-ride passes, full-day passes, and annual memberships. Customers who purchase single-ride or full-day passes are referred to as* ***Casuals***. *Customers who purchase annual memberships are* ***Members***.

*Although the pricing flexibility helps Cyclistic attract more customers, Director of Marketing Lily Moreno believes that maximizing the number of annual members will be key to future growth. She has set a clear goal: design marketing strategies aimed at converting Casuals into Members. In order to do that, however, the marketing analyst team needs to better understand how Members and Casuals differ. Moreno and her team are interested in analyzing the Cyclistic historical bike trip data to identify trends.*

# I. Ask

*You will produce a report with the following deliverables:*
1. *A clear statement of the business task*
2. *A description of all data sources used*
3. *Documentation of any cleaning or manipulation of data*
4. *A summary of your analysis*
5. *Supporting visualizations and key findings*
6. *Your top three recommendations based on your analysis*

Deliverables 5-6 can be found in [the accompanying Tableau visualization](https://public.tableau.com/app/profile/paulbrianross/viz/GoogleDataAnalyticsCapstoneCyclisticCaseStudy/story). Deliverables 1-4 can be found below.

We begin with the first deliverable, a statement of the business task, which is: Analyze the Cyclistic historical bike trip data for Director of Marketing Lily Moreno to identify trends and answer the question, **"How do Members and Casuals use Cyclistic bikes differently?"**

# II. Prepare

*You will use Cyclistic’s historical trip data to analyze and identify trends. Download the previous 12 months of Cyclistic trip data. This is public data that you can use to explore how different customer types are using Cyclistic bikes. But note that data-privacy issues prohibit you from using riders’ personally identifiable information. This means that you won’t be able to connect pass purchases to credit card numbers to determine if casual riders live in the Cyclistic service area or if they have purchased multiple single passes.*

##### 2.1-2.2 Where is your data located? How is the data organized?

Data was downloaded from [here](https://divvy-tripdata.s3.amazonaws.com/index.html). Data is organized into monthly CSV files, and we will analyze the last 12 months.

##### 2.3 Are there issues with bias or credibility in this data? Does your data ROCC?

* Reliable - data includes all rides over the last 12 months (not just a sample) and so it's 100% representative of the population of Cyclistic riders over that period.
* Original - we are using an original data source.
* Current - this dataset is updated monthly. It was last updated Jul 16, 2021.
* Comprehensive - there's no personal identity info, so we can't identify individual riders or how many rides each has made. Nevertheless, we have plenty of useful data (see 2.6 below).

##### 2.4 How are you addressing licensing, privacy, and security?

The data has been made available by Motivate International Inc. under [this license](https://www.divvybikes.com/data-license-agreement). This is public data with no personally identifiable info.

##### 2.5 How did you verify the data’s integrity?

Integrity checks can be found in 3.3 below, including:
* No duplicates of Primary Key (ride_id) in the cleaned dataset.
* Both Members and Casuals have 365 days of data.

##### 2.6 How does the data help you answer your question?

Each row is flagged as Member/Casual (with no exceptions) which is exactly what we need to identify differences between the two groups. Potential differences include:

* Usage patterns by month of year, day of week, and hour of day.
* Duration of rides (derived from start and end timestamps).
* Preference for Round Trip rides (back to the originating station) vs. One Way rides.
* Relative popularity of different stations, especially during peak usage hours.
* Relative popularity of the three different rideable types (although this is abandoned due to issues described in 3.3.3 below).

##### 2.7 Are there any problems with the data?

I identified problem data in the following categories:

* Duplicate ride_ids - A small number (209) of ride_ids appeared in both the 202011 and 202012 files.
* Rides with negative duration - indicate an error with data collection.
* Rides lasting less than 60s - can be excluded per [this link](https://www.divvybikes.com/system-data).
* Rides lasting more than 24h - can be excluded per [this link](https://help.divvybikes.com/hc/en-us/articles/360033484791-What-if-I-keep-a-bike-out-too-long-).
* Rides to or from TEST stations - can be excluding per [this link](https://www.divvybikes.com/system-data).
* Rides to or from NULL stations - indicate an error with data collection.

In 3.2 below, we filter out rides with durations outside of 60s-24h, and those rides to or from TEST or NULL stations, thereby:

* solving all of the above problems.
* eliminating ~500K rows of bad data (11% of the raw dataset).
* keeping the same ratio of Member : Casual rides in the clean dataset as we find in the raw dataset (57% Member rides in both).

# III. Process

##### 3.1 What tools are you choosing and why?

I chose Google BigQuery for data preparation because that was the SQL dialect taught in this course. The first step was to download the 12 most recent monthly CSV files and import them as tables. Each was imported with the following schema in BigQuery:

``` sql
ride_id,            STRING
rideable_type,      STRING
started_at,         TIMESTAMP
ended_at,           TIMESTAMP
start_station_name, STRING
end_station_name,   STRING
start_station_id,   INTEGER/STRING (see ** below)
end_station_id,     INTEGER/STRING (see ** below)
start_lat,          FLOAT
start_lng,          FLOAT
end_lat,            FLOAT
end_lng,            FLOAT
member_casual,      STRING
```

** The station_id fields were integers in the six files from 2020, but strings in the six files from 2021. This would have been straightforward to solve by CASTing the integers as strings before UNIONing the tables, however this proved to be unnecessary because wherever we have a station_id we also have a station_name. Since station_names are what we ultimately want to use for our analysis, we can discard the station_id fields.

With that in mind, we UNION the 12 monthly tables into a single table (and add duration_seconds as a calculated field) as follows:

``` sql
-- 4,460,151 rows
CREATE OR REPLACE TABLE cap.union_tripdata AS
SELECT
    DATETIME_DIFF(ended_at, started_at, SECOND) AS duration_seconds
    ,*
FROM (
SELECT '202007' AS filename, * EXCEPT (start_station_id, end_station_id) FROM cap.202007_tripdata UNION ALL
SELECT '202008' AS filename, * EXCEPT (start_station_id, end_station_id) FROM cap.202008_tripdata UNION ALL
SELECT '202009' AS filename, * EXCEPT (start_station_id, end_station_id) FROM cap.202009_tripdata UNION ALL
SELECT '202010' AS filename, * EXCEPT (start_station_id, end_station_id) FROM cap.202010_tripdata UNION ALL
SELECT '202011' AS filename, * EXCEPT (start_station_id, end_station_id) FROM cap.202011_tripdata UNION ALL
SELECT '202012' AS filename, * EXCEPT (start_station_id, end_station_id) FROM cap.202012_tripdata UNION ALL
SELECT '202101' AS filename, * EXCEPT (start_station_id, end_station_id) FROM cap.202101_tripdata UNION ALL
SELECT '202102' AS filename, * EXCEPT (start_station_id, end_station_id) FROM cap.202102_tripdata UNION ALL
SELECT '202103' AS filename, * EXCEPT (start_station_id, end_station_id) FROM cap.202103_tripdata UNION ALL
SELECT '202104' AS filename, * EXCEPT (start_station_id, end_station_id) FROM cap.202104_tripdata UNION ALL
SELECT '202105' AS filename, * EXCEPT (start_station_id, end_station_id) FROM cap.202105_tripdata UNION ALL
SELECT '202106' AS filename, * EXCEPT (start_station_id, end_station_id) FROM cap.202106_tripdata )
```

##### 3.2 What steps have you taken to ensure that your data is clean?

First, we check for NULL and TEST values. [This link](https://www.divvybikes.com/system-data) explains that TEST trips were taken by staff to service and inspect the system, and should be excluded.

``` sql
-- check all fields for NULL values, plus station_names for TEST values
SELECT
    SUM(CASE WHEN ride_id IS NULL THEN 1 ELSE 0 END) AS ride_id_null
    ,SUM(CASE WHEN rideable_type IS NULL THEN 1 ELSE 0 END) AS rideable_type_null
    ,SUM(CASE WHEN started_at IS NULL THEN 1 ELSE 0 END) AS started_at_null
    ,SUM(CASE WHEN ended_at IS NULL THEN 1 ELSE 0 END) AS ended_at_null
    ,SUM(CASE WHEN start_station_name IS NULL THEN 1 ELSE 0 END) AS start_station_null
    ,SUM(CASE WHEN end_station_name IS NULL THEN 1 ELSE 0 END) AS end_station_null
    ,SUM(CASE WHEN start_lat IS NULL THEN 1 ELSE 0 END) AS start_lat_null
    ,SUM(CASE WHEN end_lat IS NULL THEN 1 ELSE 0 END) AS end_lat_null
    ,SUM(CASE WHEN start_lng IS NULL THEN 1 ELSE 0 END) AS start_lng_null
    ,SUM(CASE WHEN end_lng IS NULL THEN 1 ELSE 0 END) AS end_lng_null
    ,SUM(CASE WHEN member_casual IS NULL THEN 1 ELSE 0 END) AS member_casual_null
    ,SUM(CASE WHEN UPPER(start_station_name) LIKE '%TEST%' THEN 1 ELSE 0 END) AS start_station_test
    ,SUM(CASE WHEN UPPER(end_station_name) LIKE '%TEST%' THEN 1 ELSE 0 END) AS end_station_test
FROM
    cap.union_tripdata
```

All sums returned zero except the following:

| start_station_null | end_station_null | end_lat_null  | end_lng_null | start_station_test | end_station_test |
| --- | --- | --- | --- | --- | --- |
| 282,068 | 315,109 | 5,286 | 5,286 | 3,103 | 3,213 |

We also take this opportunity to filter out rides below 60 seconds in duration or exceeding 24 hours:

* [This link](https://www.divvybikes.com/system-data) explains why we remove trips below 60s (potentially false starts or users trying to re-dock a bike).
* [This link](https://help.divvybikes.com/hc/en-us/articles/360033484791-What-if-I-keep-a-bike-out-too-long-) explains that bikes not returned with 24h are considered lost or stolen.

And so we can clean up our working dataset by combining all of the filters above:

``` SQL
-- 3,960,755 rows
CREATE OR REPLACE TABLE cap.clean_tripdata AS
SELECT
    CAST(started_at AS DATE) AS started_date
    ,TIME(EXTRACT(HOUR FROM started_at),0,0) AS started_hour
    ,CASE WHEN UPPER(start_station_name) = UPPER(end_station_name) THEN 1 ELSE 0 END AS flag_round_trip
    ,TRIM(ride_id) AS ride_id
    ,TRIM(start_station_name) AS start_station_name
    ,TRIM(end_station_name) AS end_station_name
    ,INITCAP(TRIM(member_casual)) AS member_casual
    ,* EXCEPT (ride_id, start_station_name, end_station_name, member_casual)
FROM
    cap.union_tripdata
WHERE
    start_station_name IS NOT NULL
    AND end_station_name IS NOT NULL
    AND end_lat IS NOT NULL
    AND end_lng IS NOT NULL
    AND UPPER(start_station_name) NOT LIKE '%TEST%'
    AND UPPER(end_station_name) NOT LIKE '%TEST%'
    AND duration_seconds BETWEEN 60 AND 60*60*24
```

##### 3.3 How can you verify that your data is clean and ready to analyze?

With our clean dataset, we conduct the following integrity checks:

###### 3.3.1 Primary Key (ride_id) duplicates

There were 209 ride_id duplicates in the original UNION above, however zero rows returned from the following query confirms that all duplicates were filtered out in the preceding step, and so each ride_id in our clean dataset is unique:

``` SQL
-- return duplicate ride_ids, or zero rows if all ride_ids are unique
SELECT ride_id, COUNT(1) FROM `cap.clean_tripdata` GROUP BY ride_id HAVING COUNT(1) > 1
```

###### 3.3.2 Member/Casual

BigQuery lacks a median function, so we define one as follows for the remaining analysis:

``` SQL
-- define MEDIAN function
CREATE TEMP FUNCTION median (arr ANY type) AS (
  IF(MOD(ARRAY_LENGTH(arr), 2) = 0,
    ( arr[OFFSET(DIV(ARRAY_LENGTH(arr), 2) - 1)] +
      arr[OFFSET(DIV(ARRAY_LENGTH(arr), 2))])  / 2,
      arr[OFFSET(DIV(ARRAY_LENGTH(arr), 2))] )
);
-- group by member/casual to analyze aggregates
SELECT
    member_casual
    ,COUNT(1) AS count_rides
    ,SUM(flag_round_trip) AS count_round
    ,ROUND(SAFE_DIVIDE(SUM(flag_round_trip),COUNT(1)),2) AS rate_round
    ,COUNT(DISTINCT started_date) AS count_dates
    ,ROUND(AVG(duration_seconds)/60,1) as mean_mins
    ,ROUND(median(ARRAY_AGG(duration_seconds ORDER BY duration_seconds))/60,1) AS median_mins
FROM
    cap.clean_tripdata
GROUP BY
    member_casual
```

| member_casual | count_rides    | count_round | rate_round | count_dates | mean_mins | median_mins |
|---------------|----------------|------------------|-----------------|-------------|---------------|-----------------|
|     Casual    |     1,704,687    |        256,519    |         0.15    |      365    |       34.5    |         20.1    |
|     Member    |     2,256,068    |         85,824    |         0.04    |      365    |       14.9    |         11.1    |

* Here we verify that we have 365 days of rides for both Members and Casuals.
* We also observe that Casuals are 3-4x more likely to make round trips, but most rides are one way, all of which seems reasonable.
* We note also that the mean duration is greater than the median. Further investigation reveals that the distribution has a long right tail, and so we will use the median rather than the mean in our analysis because the median is less sensitive to skewed data and outliers.

###### 3.3.3 Rideable Type

``` SQL
-- GROUP BY member/casual then rideable_type to analyze aggregates
SELECT
    member_casual
    ,rideable_type
    ,COUNT(1) AS count_rows
    ,COUNT(DISTINCT started_date) AS count_dates
    ,MIN(started_date) as from_date
    ,MAX(started_date) AS to_date
FROM
    cap.clean_tripdata
GROUP BY
    member_casual
    ,rideable_type
ORDER BY
    member_casual
    ,MIN(started_date)
```

| member_casual | rideable_type | count_rows | count_dates | from_date  | to_date    |
|---------------|---------------|------------|-------------|------------|------------|
| Casual        | docked_bike        |     956,615 |         365 | 2020-07-01 | 2021-06-30 |
| Casual        | electric_bike      |     301,218 |         337 | 2020-07-29 | 2021-06-30 |
| Casual        | classic_bike       |     446,854 |         211 | 2020-12-02 | 2021-06-30 |
| Member        | docked_bike        |    1,052,947 |         158 | 2020-07-01 | 2021-01-13 |
| Member        | electric_bike      |     391,855 |         337 | 2020-07-29 | 2021-06-30 |
| Member        | classic_bike       |     811,266 |         211 | 2020-12-02 | 2021-06-30 |

This table illustrates the different bikes used by Members and Casuals over the 12 months. Only Docked bikes were rideable on Jul 1, 2020, with Electric bikes becoming available on Jul 29, and Classic bikes not until Dec 2. Casuals continued to use all 3 types until the end of Jun 2021. Members, however, stopped using Docked bikes on Jan 13, 2021, and appear only to have used Classic and Electric bikes after that date. Thus, a comparison of Member and Casual preferences for rideable_type is troubled for these reasons:
* Bike types available changed 4 times over the year (from 1 option, to 2, to 3, and then to different options for each group).
* All 3 bike types were available to both Members and Casuals for only 42 of the 365 days.
* For the final 5.5 months, Casuals had 3 bike types to choose from, while Members had only 2 (or they were coded differently).
* Because of these issues, we exclude rideable_type from further analysis so we can focus on less muddied data.

###### 3.3.4 Latitude & Longitude

Latitude & Longitude data are in need of a clean. For example, the following query returns more than 100 stations, each with more than 1,000 different values for latitude AND longitude. Consolidating these will allow for more precise mapping in Tableau.

``` SQL
-- start_stations with 1000 or more different latitude values AND 1000 or more different longitude values
SELECT      start_station_name
            ,count(distinct start_lat) as count_lat
            ,count(distinct start_lng) as count_lng
FROM        cap.clean_tripdata
GROUP BY    start_station_name
HAVING      count(distinct start_lat) > 1000
            AND count(distinct start_lng) > 1000
```

One solution is to UNION all the start_station coordinates with the end_station coordinates, and then calculate the median latitude & longitude for each station_name. Here again, using the median will reduce the impact of any erroneous outliers.

``` SQL
-- 7,921,510 rows, which is 2x COUNT of `cap.clean_tripdata` as expected
CREATE OR REPLACE TABLE cap.union_stations AS
SELECT  *
FROM    (
    SELECT  start_station_name AS station_name
            ,start_lat AS station_lat
            ,start_lng AS station_lng
    FROM    cap.clean_tripdata UNION ALL
    SELECT  end_station_name AS station_name
            ,end_lat AS station_lat
            ,end_lng AS station_lng
    FROM    cap.clean_tripdata)

-- GROUP stations to calculate median latitude & longitude
CREATE OR REPLACE TABLE cap.group_stations AS
SELECT      station_name
            ,COUNT(1) AS count_rides
            ,median(ARRAY_AGG(station_lat ORDER BY station_lat)) AS median_lat
            ,median(ARRAY_AGG(station_lng ORDER BY station_lng)) AS median_lng
FROM        cap.union_stations
GROUP BY    station_name
```

The final step is to LEFT JOIN the grouped stations to add the corrected coordinates to the export dataset:

``` SQL

CREATE OR REPLACE TABLE cap.export_tripdata AS
SELECT
    a.* EXCEPT (start_lat,start_lng,end_lat,end_lng)
    , s.median_lat as start_lat
    , s.median_lng as start_lng
    , e.median_lat as end_lat
    , e.median_lng as end_lng
FROM
    cap.clean_tripdata a
    LEFT JOIN cap.group_stations s on a.start_station_name = s.station_name
    LEFT JOIN cap.group_stations e on a.end_station_name = e.station_name
```

# IV. Summary of Analysis

To see how this analysis unfolds, please follow the story in [the accompanying Tableau visualization](https://public.tableau.com/app/profile/paulbrianross/viz/GoogleDataAnalyticsCapstoneCyclisticCaseStudy/story).

Although Members and Casuals have some similarities:

* Both groups enjoy riding on weekends in roughly even numbers (although Casual Rides outnumber Member Rides during the warmer months, and vice versa).

* Both groups enjoy slightly longer rides on weekends compared to weekdays. 
  
Key differences between the two groups include:

* The median Casual Ride is a leisurely 20 minutes, while the median Member Ride is a more focused 11 minutes.

* While the majority of rides from both groups are from one station to another, Casuals are 3-4x more likely to make a Round Trip back to their point of origin.

* Some Members continue to ride through the coldest Winter months (a rough estimate is 20%) but Casual Rides dwindle to just 5% of their Summer peak.

* Almost two-thirds of weekday rides are made by Members, and Member Rides spike twice during each weekday: once in the morning as they ride to work, and again in the evening between 3-9 PM as they ride home. By contrast, Casual Rides spike only once each weekday, around the same time as the Member PM spike.

# V. Recommendations

To understand how these recommendations were reached, please follow the story in [the accompanying Tableau visualization](https://public.tableau.com/app/profile/paulbrianross/viz/GoogleDataAnalyticsCapstoneCyclisticCaseStudy/story).

1. Every weeknight evening between 3-9 PM, the day's largest surge of Members start mostly One Way Rides home from work. During the same hours (at least during the warmer months), the day's largest surge of Casuals also start mostly One Way Rides, many from the same stations that Members are using, and we can identify those stations! These are the Casuals who likely have the most to gain from membership, and so it these Casuals who we highly recommend as a target market.
  
2. Some of these Casuals might already be riding enough each week to make membership financially compelling. Others might be occasional riders who could be shown the financial (and health!) benefits of riding more regularly. Trial memberships could be a valuable tool in this regard, giving membership-curious Casuals a free taste of the benefits. Alternately, Cyclistic could raise Casual pricing during peak weekday hours to create a reverse financial incentive for membership.
  
3. Time is of the essence! Monthly Casual Rides just hit a 12-month high, but there are only 3 more months before the next Winter decline begins. This is a valuable window to convert Casuals into Members, and hopefully boost the numbers of the roughly 20% of Members who continue to ride through the coldest months.
