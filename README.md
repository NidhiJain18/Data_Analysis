---
title: "BellaBeat Analysis"
author: "Nidhi Jain"
date: "2023-10-06"
output: html_document
---

## Setting up my environment
### Installing the packages

```{r message=FALSE, warning=FALSE, include=FALSE}
install.packages('tidyverse')
install.packages('janitor')
install.packages('lubridate')
install.packages('skimr')
```

### Loading the packages
```{r}

library(tidyverse)
library(janitor)
library(lubridate)
library(skimr)
```

## Load the data
After this, we would need to import the datasets into RStudio 
```{r echo=TRUE, message=FALSE, warning=FALSE}
daily_activity <- read_csv("/cloud/project/CaseStudy2_Project/dailyActivity_merged.csv")
daily_sleep <- read.csv("/cloud/project/CaseStudy2_Project/sleepDay_merged.csv")
weight_log <- read.csv("/cloud/project/CaseStudy2_Project/weightLogInfo_merged.csv")
```

## Inspect data
Here, we will check the data types and columns of each data tables.
```{r}
str(daily_activity)
str(daily_sleep)
str(weight_log)

```

After reviewing the output, we found several issues:

* The naming of the column names is in camelCase

* daily_activity$ActivityDate — Is formatted as CHR not as a date format

* daily_sleep$SleepDay — Is formatted as CHR not as a date format

* weight_log$Date — Is formatted as CHR not as a date format

* weight_log$IsManualReport is formated as CHR not boolean


### Clean and format columns
```{r}
daily_activity <- clean_names(daily_activity)
daily_sleep <- clean_names(daily_sleep)
weight_log <- clean_names(weight_log)
daily_activity$activity_date <- as.Date(daily_activity$activity_date,'%m/%d/%y')
daily_sleep$sleep_day <- as.Date(daily_sleep$sleep_day, '%m/%d/%y')
weight_log$date <- parse_date_time(weight_log$date, '%m/%d/%y %H:%M:%S %p')
weight_log$is_manual_report <- as.logical(weight_log$is_manual_report)
```

## Analysing and share the data

It will tell total activity minutes

```{r}
daily_activity$day_of_week <- wday(daily_activity$activity_date, label = T, abbr = T)
daily_activity$total_active_hours = round((daily_activity$very_active_minutes + daily_activity$fairly_active_minutes + daily_activity$lightly_active_minutes)/60, digits = 2)
daily_activity$sedentary_hours = round((daily_activity$sedentary_minutes)/60, digits = 2)
```


It will tell us time taken to get sleep :

```{r}
daily_sleep$hours_in_bed = round((daily_sleep$total_time_in_bed)/60, digits = 2)
daily_sleep$hours_asleep = round((daily_sleep$total_minutes_asleep)/60, digits = 2)
daily_sleep$time_taken_to_sleep = (daily_sleep$total_time_in_bed - daily_sleep$total_minutes_asleep)

```


We will add a column in weight_log table 'bmi2' to check if the person is healthy, overweight or underweight.

```{r}
weight_log <- weight_log %>% 
  mutate(bmi2 = case_when(
    bmi > 24.9 ~ 'Overweight',
    bmi < 18.5 ~ 'Underweight',
    TRUE ~ 'Healthy'
  ))
```

we will add a new column in daily sleep and check who is having good, poor, over sleeping
```{r}
daily_sleep <- daily_sleep %>% 
  mutate(sleep_calculation = case_when(
    hours_asleep > 9 ~ 'Over Sleep',
    hours_asleep < 7 ~ 'Poor Sleep' ,
    TRUE ~ 'Good Sleep'
  ))
```

We will remove the zero rows for calories and total active hours.

```{r}
daily_activity_cl <- daily_activity[!(daily_activity$calories<=0),]
```


```{r}
daily_activity_cl <- daily_activity_cl[!(daily_activity_cl$total_active_hours<=0.00),]
```



We are now combining the tables for further analysis. We are merging (daily_sleep, daily_activity_cl) and (daily_activity_cl, weight_log) and then (activity_weight, daily_sleep)  for further analysis.

```{r}
merged_sleep_activity <- merge(daily_sleep, daily_activity_cl, by.x = "id", by.y = "id", all.x=TRUE, all.y= FALSE)
head(merged_sleep_activity)

```
```{r}
activity_weight <- merge(daily_activity_cl, weight_log, by=c('id'))

```
```{r}
activity_weight_sleep <- merge(activity_weight, daily_sleep, by=c('id'))
```
#### Saving files

write.csv(daily_activity_cl, file ='fitbit_daily_activity.csv')
write.csv(daily_sleep, file = 'fitbit_sleep_log.csv')
write.csv(weight_log, file = 'fitbit_weight_log.csv')
write.csv(activity_weight_sleep, file = 'fitbit_activity_weight_sleep_log.csv')




## Summarize the data and creating Plots

We can find the general trends in the data to get insights and answers to our business problem.
Here, are some of the insights we get from the data:

**Which days the users are most active**

Here we will check the relationship between totalsteps, very active minutes, calories. 
```{r}

ggplot(data = daily_activity_cl) +
  aes(x = day_of_week, y = total_steps) +
  geom_col(fill =  'blue') +
  labs(x = 'Day of week', y = 'Total steps', title = 'Total steps taken in a week', subtitle = 'Total number of steps taken by all the users in a day of the week')
ggsave('total_steps.png')

```
```{r}
ggplot(data = daily_activity_cl) +
  aes(x = day_of_week, y = very_active_minutes) +
  geom_col(fill =  'red') +
  labs(x = 'Day of week', y = 'Total very active minutes', title = 'Total activity in a week', subtitle = 'Total very active minutes of all the users in a day of the week')
ggsave('total_activity.png')

```
```{r}

  ggplot(data = daily_activity_cl) +
  aes(x = day_of_week, y = calories) +
  geom_col(fill =  'Green') +
  labs(x = 'Day of week', y = 'Calories burned', title = 'Total calories burned in a week', subtitle = 'Total calories burned by all the users in a day of the week')
ggsave('total_calories.png')

```

Here, we can see the most active days for the Fitbit users were on Sunday, with a slow decline throughout the week.

**Which days users are having good sleep and which days the users are most inactive**

```{r}
ggplot(data = activity_weight_sleep, aes(day_of_week,hours_asleep)) +
    geom_col(color="Blue")
  labs(x = 'Day of week', y = 'hours_asleeo', title = 'Total sleep hours in a day of a week', subtitle = 'Which days users have the best sleep in a day of the week')
ggsave('sleep_day.png')

```


```{r}
ggplot(data = activity_weight_sleep, aes(day_of_week,sedentary_hours)) +
    geom_col(color="Blue")
 labs(x = 'Day of week', y = 'Sedentary hours', title = 'Total sedantary hours in a day of a week', subtitle = 'Which days users are most inactive in a day of the week')
ggsave('Sedentary_day.png')
```

From this we can infer that at start of the week users are sleep best and are more inactive.


**The relationship between total active hours, total steps taken, sleep and sedentary hours against calories burned**

```{r}
ggplot(data = daily_activity_cl) +
  aes(x= total_active_hours, y = calories) +
  geom_point(color = 'red') +
  geom_smooth() +
  labs(x = 'Total active hours', y = 'Calories burned', title = 'Calories burned vs active hours')
ggsave('calories_burned_vs_active_hours.png')
```


```{r}
ggplot(data = daily_activity_cl) +
  aes(x= total_steps, y = calories) +
  geom_point(color = 'orange') +
  geom_smooth() +
  labs(x = 'Total steps', y = 'Calories burned', title = 'Calories burned vs total steps')
ggsave('calories_burned_vs_total_steps.png')
```


```{r}
ggplot(data = daily_activity_cl) +
  aes(x= sedentary_hours, y = calories) +
  geom_point(color = 'purple') +
  geom_smooth() +
  labs(x = 'Sedentary hours', y = 'Calories burned', title = 'Calories burned vs sedentary hours')
ggsave('sedentary_hours_vs_calories_burned.png')
```
```{r}
ggplot(data = merged_sleep_activity) +
   geom_point(mapping=aes(x= hours_asleep, y = calories), color = 'Purple')+
   geom_smooth(mapping=aes(x= hours_asleep, y = calories)) +
    labs(x = 'Hours asleep', y = 'Calories burned', title = 'Calories burned vs Hours Asleep')
ggsave('Hours_Asleep_vs_calories_burned.png')
```



We can tell that there is a positive correlation between calories burned and total steps taken/total active hours. However, in the last chart, we can see that the relationship between sedentary hours and calories burned was fairly positive up till about the 17-hour mark and when a person is not active for more than 17 hrs then the calorie burned is decreasing.
The graph between sleep hours and calorie burned also tells also that when people is not sleep for atleast 5 hours the calories burned is decreasing and from 5 hrs to 7hrs it is the maximum and till 10 hrs it is constant and when people is sleeping more than 10 hrs it again starts to fall.


**The relationship between weight, total active hours**

```{r}
ggplot(data = activity_weight,aes(x= total_active_hours, y = weight_kg)) +
  geom_point(color = 'Gold') +
  geom_smooth(orientation = "x")+
  labs(x = 'Total_active_hours', y = 'Weight(kg)', title = 'Relationship between weight and physical activity')
ggsave('weight_physical_activity.png')
```


We can infer that users weighing around 60kg & 85kg are the most active



**The number of overweight users**
```{r}
daily_activity_cl %>% distinct(id)
```


```{r}
activity_weight %>% distinct(id)
```


```{r}
activity_weight %>% count(bmi2,id)
```


Here we can conclude that out of the 33 users, only 8 submitted their responses regarding weight. 
5 users are overweight and only 3 are within the healthy BMI range of 18.5–24.9


**The relationship between good sleep, total activity and Health(BMI)**

```{r}
ggplot(data = activity_weight_sleep, aes(total_active_hours,bmi)) +
  geom_point(color = "Orange",size = 1) +
  geom_smooth() +
   labs(x = 'Total_active_hours', y = 'bmi', title = 'Relationship between BMI and physical activity')
ggsave('bmi_physical_activity.png')
```


```{r}
ggplot(data = activity_weight_sleep, aes(hours_asleep,bmi)) +
  geom_point(color = "Orange",size = 1) +
  geom_smooth()+
   labs(x = 'Hours asleep', y = 'BMI', title = 'Relationship between sleep and physical activity')
ggsave('sleep_BMI.png')
```


```{r}
ggplot(data = activity_weight_sleep, aes(time_taken_to_sleep,bmi2)) +
  geom_jitter(color = "Orange")+
labs(x = 'Time taken to sleep', y = 'BMI', title = 'Relationship between Time taken to sleep and BMI')
ggsave('time_taken_to_sleep_bmi.png')

```

From below we get find general trends in the data:


```{r}
summary(daily_activity_cl$total_steps)
   
summary(daily_activity_cl$very_active_minutes)

summary(daily_sleep$hours_asleep)
```

From here we can say, the average steps taken by active users are 8053 who are very active for around 7 hours and their average sleep 7.22.


# Act

In the previous section of Analyze & Share, we have covered the 1st and 2nd business task which are:

* What are some trends in smart device usage
* How could these trends apply to Bellabeat customers 

We have analysed and shared several trends in the smart device usage and which could be followed by bellabeat customers.

Based on my findings, I would like to share my views on this matter.

Users spend more time engaged in physical activity specifically on Sundays, which then proceeds to decrease throughout the week with a slight peak on Thursdays which then sees a slow climb on Saturdays.

I suspect that:

Motivation levels & free time are higher on the weekends, which would provide an opportunity for users to sneak in a workout.
As work load decreases, a window of opportunity to exercise would present itself in the midweek (Thursdays)
We see an alltime low of recorded activity on Friday’s and some on Saturdays due to the possibility of social engagement with friends/coworkers.

* Now to answer the final business task, I would like to share my recommendations based on my findings to help influence Bellabeat’s marketing strategy.

-Bellabeat could host events limited to those that are enrolled in their Bellabeat memberships which would reward users who engage in a healthy lifestyle(IE 8k steps a day, less than 7 hours sedentary etc.) with points. With enough points, users could then use points to purchase products that help supplement a healthy lifestyle.

-Bellabeat could partner with healthcare or sports brands to reward users who consistently engage in a healthy lifestyle with coupons/store discounts.

-Bellabeat could introduce some 5mins or 10 mins videos on easy but impactful workout that could help its inactive users to motivate them in doing some activities.

-Bellabeat could send notifications if inactivity is very less or sleep is not well 


**some general recommendations to further improve Bellabeat’s products:**

-Bellabeat could implement personalized milestones, to encourage users to slowly engage in a more healthy lifestyle. 

-Bellabeat could introduce some 5mins or 10 mins videos on easy but impactful workout that could help its inactive users to motivate them in doing some activities.

-Bellabeat could send notifications if inactivity is very less or sleep is not well 


Additional remarks:

Bellabeat should require users to input their height, weight and their activity levels so that BMR calculations and a more accurate.

Bellabeat should create devices that would track sleep more sophisticatedly (REM sleep tracking, deep sleep tracking) to provide more insights into sleep health.
