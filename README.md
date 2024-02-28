# Wellness

## About this project
Main task of the project is to analyze smart device usage data in order to gain insight into how consumers use non-Bellabeat smart
devices and apply these insights into one Bellabeat product. Discovered insights will help guide marketing strategy for the company. 

**Task:** explore user’s data to uncover trends and patterns in daily activities and identify key wellness indicators of healthy lifestyle.

**Questions:**
 - What are some trends in smart device usage?
 - What specific metrics indicate a more active lifestyle?
 - How could these trends impact Bellabeat marketing strategy?

## About the data
Dataset “FitBit Fitness Tracker Data” was generated by respondents to a distributed survey via Amazon Mechanical Turk between 04.12.2016-05.12.2016 and  kindly provided through [Mobius](https://www.kaggle.com/datasets/arashnic/fitbit?select=Fitabase+Data+4.12.16-5.12.16). Dataset consists of 18 csv files of 33 users who consented to provide personal tracker data that includes output for physical activity, heart rate, sleep monitoring, and  steps. Some tables are organized in long format, some organized in wide format. Upon closer examination we discovered that not all tables of the dataset have the same number of unique user Ids. For example, 'sleep_day' table has 24, 'daily_activity' - 33, and 'heart_rate_seconds' has only 8 unique users. These findings will limit some aspects of further analysis. To answer the above mentioned questions we decided to study the following metrics recorded daily: calories, total steps, total distance, different intensity minutes, time asleep, time in bed, heart rate.

**Tables**: daily_activity.csv, daily_calories.csv, heart_rate_seconds.csv, sleep_day.csv


## Cleaning process 
Considering that the number of observations in some tables reach over 1 millions rows, we decided to use BigQuery Studio to perform our data manipulations. We created a new dataset named ‘fitbit’ and uploaded all csv files while assigning new names following standards of naming conventions. Ensuring that the dataset is clean we checked Field and Type of data in the SCHEMA of selected tables. 
1. We identified that the ‘sleep_day’ table has a column named ‘SleepDay’ that is formatted as STRING instead of TIMESTAMP. So, we changed the format of that column, renamed columns to better reflect the data they describe. Also we added another column ‘sleep_inertia_min’ that reflects the difference between ‘TotalTimeInBed’  and ‘TotalMinutesAsleep’. Sleep inertia will be used as an important metrics that relates to a daily activity of a user:

```
SELECT Id,
     PARSE_TIMESTAMP('%m/%d/%y %I:%M %p', SleepDay) AS date,
     ROUND((TotalMinutesAsleep/60), 1) AS total_time_asleep_h,
     ROUND((TotalTimeInBed/60), 1) AS total_time_in_bed_h,
     (TotalTimeInBed - TotalMinutesAsleep) AS sleep_inertia_min
FROM `project-bellabeat-414921.fitbit.sleep_day`
;
```

Then, this query was saved as a new table named ‘sleep_inertia’ and downloaded to the local folder that contains the dataset.

2. The same formatting (STRING into TIMESTAMP) was done for column ‘Time’ in ‘heart_rate_seconds’ and used CAST function to separate date and time values and saved as new csv file named  ‘clean_heart_rate’ in local folder:

```
WITH heart_rate_1 AS
      (SELECT Id,
              PARSE_TIMESTAMP('%m/%d/%y %I:%M %p', Time) AS time,
              Value
       FROM `project-bellabeat-414921.fitbit.heart_rate_seconds`)

SELECT Id,
       CAST(time AS date) AS date,
       CAST(time AS time) AS time,
       Value AS hr_bpm
FROM `heart_rate_1`
;
```


3. In order to perform analysis more efficiently we decided to combine columns with data we need in a single table from the following tables: ‘daily_activity’, ‘daily_calories’, and ‘sleep_inertia’. Using FULL OUTER JOIN clause allowed us  to include all data from all tables:

```
SELECT daily_activity.Id, 
       daily_calories.calories AS calories,
       daily_activity.TotalSteps AS total_steps,
       ROUND(daily_activity.TotalDistance, 2) AS total_distance,
       daily_activity.VeryActiveMinutes AS active_min,
       daily_activity.FairlyActiveMinutes AS mod_min,
       daily_activity.LightlyActiveMinutes AS light_min,
       daily_activity.SedentaryMinutes AS sed_min,
       sleep_inertia.total_time_asleep_h AS time_asleep_h,
       sleep_inertia.total_time_in_bed_h AS time_in_bed_h,
       sleep_inertia.sleep_inertia_min AS sleep_inertia_min

FROM project-bellabeat-414921.fitbit.daily_activity
FULL OUTER JOIN project-bellabeat-414921.fitbit.daily_calories ON daily_activity.Id = daily_calories.Id
FULL OUTER JOIN project-bellabeat-414921.fitbit.sleep_inertia ON daily_activity.Id = sleep_inertia.Id
```

Output of this query was saved as ‘filtered_data’ csv file in the local folder.


## Descriptive statistics. 
- Finding average, minimun and maximum heart rate values of users. 

```
SELECT DISTINCT(user) as Id, 
     ROUND(AVG(hr_bpm), 1) AS avg_bpm,
     MIN(hr_bpm) AS min_bpm,
     MAX(hr_bpm) AS max_bpm
FROM `project-bellabeat-414921.fitbit.clean_heart_rate`
GROUP BY Id
;
```

![summary of heart rate](summary_hr.png)

Only 7 distinct Ids were returned. This table shows that the highest heart rate was  203 bpm and lowest 38 bpm, which overall tells about a very good fitness shape of a user with Id 2022484408. 
Based on this data almost all users lead an active lifestyle.


- Average values of steps, distance and calories for top 10 users sorted by steps in descending order:

```
SELECT DISTINCT(Id),
       ROUND(AVG(total_steps), 1) AS avg_steps,
       ROUND(AVG(total_distance), 1) AS avg_distance,
       ROUND(AVG(calories), 1) AS avg_calories
FROM `project-bellabeat-414921.fitbit.COMBINED_DATA`
GROUP BY Id
ORDER BY avg_steps DESC
LIMIT 10
;
```
![summary_steps](summary_steps_dist.png)

The above table shows no relation between calories and steps taken (see rows 3, 6 and 7 in of the table). We can assume that calories spent per day may relate to personal basal metabolic rate (BMR) which indicates the number of calories a person burns as the body performs basic (basal) life-sustaining function. The higher the BMR the more calories the burns without  physical activities.


- Daily Average values of intensity minutes:

```
SELECT  DISTINCT(Id),
       ROUND(AVG(active_min), 1) AS avg_active_min,
       ROUND(AVG(mod_min), 1) AS avg_moderate_min,
       ROUND(AVG(light_min), 1) AS avg_light_min,
       ROUND(AVG(sed_min), 1) as avg_sedentary_min

FROM `project-bellabeat-414921.fitbit.COMBINED_DATA`
GROUP BY Id
ORDER BY avg_active_min DESC
LIMIT 10
;
```



