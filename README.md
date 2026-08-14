# Hospital_Appointment_No_Show_Analysis
Why patients miss their scheduled appointments?
 
[Project Overview](#project-overview)

[Data Source](#data-source)

[Tools Used](#tools-used)

[Data Cleaning and Preparations](#data-cleaning-and-preparations)

[Exploratory Data Analysis](#exploratory-data-analysis)

[Data Visualization](#data-visualization)

[Finding and Recommendation](#finding-and-recommendation)



### Project Overview
---
This personal project analyses a hospital appointment no-show dataset to identify factors that may influence medical appointment no-show rates and generate insights that could support better healthcare decision-making.
The original dataset contained 110,517 rows and 18 columns after cleaning and preparation.
The first step of the project was to understand the dataset and clearly define each variable. For example, I examined the difference between ScheduledDay and AppointmentDay:

- ScheduledDay — the date when the appointment was booked.
- AppointmentDay — the date when the patient was expected to attend the appointment.

I also clarified the meaning of the No_show variable:
- Yes — the patient did not attend the scheduled appointment.
- No — the patient attended the appointment.

Understanding these variables and their relationships provided the foundation for the subsequent data cleaning, SQL analysis, risk classification, and Power BI visualization carried out in the project.


### Data Source
---
The Medical Appointment No-Show dataset was downloaded from Kaggle(https://www.kaggle.com/datasets/joniarroba/noshowappointments) and used for this project. The original dataset contained 110,527 rows and 14 columns before data cleaning and preparation.

### Tools Used
---
-  SQL - Structured Query Language
  1. For Data Cleaning
  2. For Analysis
  3. For Visualization
      
- Power BI - Power Business Intelligence for Data Visualization

- Github for
  1. Portfolio Building
  
### Data Cleaning and Preparations
---
Data cleaning and preparation were carried out using SQL Server.

The following steps were performed:
1. Loaded the CSV file into SQL Server.
2. Standardized column names and data types to ensure consistency.
3. Cleaned the two date columns through multiple steps. I created temporary columns with the appropriate date data types, populated them with the cleaned dates, removed the original columns, and renamed the temporary columns using the original column names.
4. Checked data validity, including identifying invalid values such as negative ages.
5. Checked for NULL values and removed rows with missing PatientId values, as PatientId was required as the primary key.
6. Created a new LeadTime column to calculate the interval between the scheduled date and the actual appointment date.
7. Validated the LeadTime values and removed five records with negative lead times.

Note: In a real-world work setting, deleting records or modifying source data should be discussed and approved by the relevant data team, manager, or data governance officer. For this personal project, I made these decisions because the dataset was obtained from an open-source platform and was being used for learning and portfolio purposes.


```SQL
CREATE DATABASE healthcare

SELECT * FROM hospital_appointments

---------------------------------------------------------
---DATA CLEANING
---------------------------------------------------------

--- Changed column names and data type-----------
sp_rename 'hospital_appointments.Hipertension', 'Hypertension', 'COLUMN';

sp_rename 'hospital_appointments.Handcap', 'No_of_disability', 'COLUMN';

ALTER TABLE hospital_appointments
ALTER COLUMN No_show VARCHAR (4) 

--- Cleaned date columns -------------------Created new columms for dates----
ALTER TABLE hospital_appointments
ADD ScheduledDay_temp DATETIME,
	AppointmentDay_temp DATE;
			                           ------Filled the new columns with cleaned dates-----  
UPDATE hospital_appointments
SET ScheduledDay_temp = TRY_CONVERT(DATETIME, REPLACE(REPLACE(ScheduledDay, 'T', ' '), 'Z', ''), 120),
    AppointmentDay_temp = TRY_CONVERT(DATE, REPLACE(REPLACE(AppointmentDay, 'T', ' '), 'Z', ''), 120);
	                                  --------Droped the old date columns------
ALTER TABLE hospital_appointments
DROP COLUMN ScheduledDay, AppointmentDay;
                                     ---------Renamed the new cleaned columns-------
SP_RENAME 'hospital_appointments.ScheduledDay_temp', 'ScheduledDay', 'COLUMN';

SP_RENAME 'hospital_appointments.AppointmentDay_temp', 'AppointmentDay', 'COLUMN';

                                     ---------Checked if the age column is clean--------
SELECT MIN(Age), MAX(Age)
FROM hospital_appointments
                                     ---------Checked null values-----
SELECT COUNT(*) AS Null_Count
FROM hospital_appointments
WHERE PatientId IS NULL;

SELECT * FROM hospital_appointments
WHERE PatientId IS NULL;
                                     --------Cleaned the null values in PatientId as it's the primary key---
DELETE FROM hospital_appointments 
WHERE PatientId IS NULL
									 --------Added a new column for schedule and appointment Date interval---- 
ALTER TABLE hospital_appointments
ADD LeadTime AS DATEDIFF(DAY, CAST(ScheduledDay AS DATE), AppointmentDay)

                                     --------Checked data validity for the new column-----
SELECT MIN(LeadTime), MAX(LeadTime)
FROM  hospital_appointments

SELECT * FROM hospital_appointments
WHERE LeadTime < 0
                                     --------Removed 5 rows with negative leadtime value-----
DELETE FROM hospital_appointments
WHERE LeadTime < 0
```



### Exploratory Data Analysis
---
The exploratory analysis focused on understanding appointment attendance patterns and identifying factors associated with medical appointment no-shows.

Key Performance Indicators (KPIs)
- Total Number of Appointments
- Total Number of No-Shows
- Overall No-Show Rate (%)
- Average Lead Time

Key Questions that the analysis sought to answer:
- What percentage of patients did not show up for their scheduled appointments?
- Does the day of the week affect appointment no-show rates?
- Does appointment lead time influence the likelihood of a no-show?
- Does patient age affect appointment attendance?
- What effect do SMS reminders have on no-show rates?
- Which neighbourhoods have the highest no-show rates?
- Can patients be classified based on their likelihood of missing an appointment?
- Patient Risk Classification

As part of the analysis, I created a View_AppointmentRisk using CTEs and window functions in SQL Server.

The view uses a patient's previous appointment history, previous no-shows, and current appointment lead time to assign a Risk_Tier. This type of risk classification could help healthcare teams identify patients who may need additional follow-up or reminders or follow-up before their appointments.

```SQL
---------------EXPLORATORY DATA ANALYSIS-----------------------------
----1: What is the overall percentage(rate) of patients that didnt show up for their appointment?-----

SELECT No_show, COUNT(No_show) AS TotalCount,
ROUND(COUNT(No_show) * 100.0 / (SELECT COUNT(No_show) FROM hospital_appointments), 2) AS PercentageCount
FROM hospital_appointments
GROUP BY No_show
ORDER BY TotalCount DESC

----2: Does the day of the week affect appointment No_show?
SELECT DATENAME(WEEKDAY, AppointmentDay) AS AppointmentDayName, COUNT(AppointmentDay) AS TotalAppointments,
SUM(CASE WHEN No_show = 'Yes' THEN 1 ELSE 0 END) AS No_shows,
ROUND(SUM(CASE WHEN No_show = 'Yes' THEN 1 ELSE 0 END) * 100.0 / COUNT(AppointmentDay), 2) AS No_show_rate
FROM hospital_appointments
GROUP BY DATENAME(WEEKDAY, AppointmentDay)
ORDER BY No_show_rate DESC

----3: Does appointment lead time have an effect on appointment No_shows?
SELECT CASE WHEN LeadTime = 0 THEN 'Same Day'
			WHEN LeadTime BETWEEN 1 AND 3 THEN 'Short (1-3 Days)'
			WHEN LeadTime BETWEEN 4 AND 7 THEN 'Within a week'
			ELSE 'Long (8+ days)'
		END AS LeadTimeBucket,
		COUNT (AppointmentDay) AS TotalAppointment,
		ROUND(SUM(CASE WHEN No_show = 'Yes' THEN 1 ELSE 0 END) * 100.0 / COUNT(AppointmentDay), 2) AS No_show_rate
FROM hospital_appointments
GROUP BY CASE WHEN LeadTime = 0 THEN 'Same Day'
			WHEN LeadTime BETWEEN 1 AND 3 THEN 'Short (1-3 Days)'
			WHEN LeadTime BETWEEN 4 AND 7 THEN 'Within a week'
			ELSE 'Long (8+ days)'
		END
ORDER BY No_show_rate DESC                                
                                                ---- Or use CTEs (Common Table Expression)----

WITH LeadTimeGroups AS (
    SELECT *,
        CASE 
            WHEN LeadTime = 0 THEN 'Same Day'
            WHEN LeadTime BETWEEN 1 AND 3 THEN 'Short (1-3 Days)'
            WHEN LeadTime BETWEEN 4 AND 7 THEN 'Within a week'
            ELSE 'Long (8+ days)'
        END AS LeadTimeBucket
    FROM hospital_appointments
)

SELECT 
    LeadTimeBucket,
    COUNT(AppointmentDay) AS TotalAppointments,
	SUM(CASE WHEN No_show = 'Yes' THEN 1 ELSE 0 END) AS Total_No_shows,
    CAST(
        SUM(CASE WHEN No_show = 'Yes' THEN 1 ELSE 0 END) * 100.0 
        / COUNT(AppointmentDay) AS DECIMAL (5,2)
    ) AS No_show_rate
FROM LeadTimeGroups
GROUP BY LeadTimeBucket
ORDER BY No_show_rate DESC;

----4: Does age affect appointment No_shows?
WITH AgeGroup AS(
				SELECT *,
				CASE WHEN Age BETWEEN 0 AND 12 THEN 'Child'
					 WHEN Age BETWEEN 13 AND 19 THEN 'Teenager'
					 WHEN Age BETWEEN 20 AND 39 THEN 'Young Adult'
					 WHEN Age BETWEEN 40 AND 59 THEN 'Adult'
					 ELSE 'Older Adult'
				END AS AgeBucket
FROM hospital_appointments)
SELECT AgeBucket,
COUNT (AppointmentDay) AS TotalAppointment,
ROUND(SUM(CASE WHEN No_show = 'Yes' THEN 1 ELSE 0 END) * 100.0 / COUNT(AppointmentDay), 2) AS No_show_rate
FROM AgeGroup
GROUP BY AgeBucket
ORDER BY No_show_rate DESC

----5: What is the effect of SMS reminder on No_shows?
SELECT CASE WHEN SMS_received = 1 THEN 'SMS'
			ELSE 'No SMS'
			END AS SMS_status,
COUNT (AppointmentDay) AS TotalAppointment,
ROUND(SUM(CASE WHEN No_show = 'Yes' THEN 1 ELSE 0 END) * 100.0 / COUNT(AppointmentDay), 2) AS No_show_rate
FROM hospital_appointments
GROUP BY CASE WHEN SMS_received = 1 THEN 'SMS'
			ELSE 'No SMS'
			END
ORDER BY No_show_rate DESC

----6: Which neighbourhood has the heighest risk of missing appointment (No_show)?
SELECT TOP 20 Neighbourhood, COUNT(*) AS TotalAppointments,
ROUND(SUM(CASE WHEN No_show = 'Yes' THEN 1 ELSE 0 END) * 100 / COUNT(*), 2) AS No_show_rate,
RANK() OVER (ORDER BY ROUND(SUM(CASE WHEN No_show = 'Yes' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) DESC) AS Risk_rank
FROM hospital_appointments
GROUP BY Neighbourhood
HAVING COUNT(*) >= 100
ORDER BY Risk_rank ASC

----7: Patients risk scoring (patients that are likely going to miss appointments)?
SELECT
	PatientId,
	AppointmentID,
	AppointmentDay,
	No_show,
COUNT(*) OVER(PARTITION BY PatientId 
			  ORDER BY AppointmentDay
			  ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING) AS Previous_appointments,
SUM(CASE WHEN No_show = 'Yes' THEN 1 ELSE 0 END) 
		 OVER(PARTITION BY PatientId 
			  ORDER BY AppointmentDay
			  ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING) AS Previous_no_shows
FROM hospital_appointments
ORDER BY PatientId, AppointmentDay

----8: Created a view with our findings

CREATE VIEW View_AppointmentRisk AS
WITH Patient_history AS(
SELECT PatientId,
	   AppointmentId,
	   AppointmentDay,
	   Neighbourhood,
	   LeadTime,
	   SMS_received,
	   Scholarship,
	   No_show,
	   COUNT(*) OVER(
				PARTITION BY PatientId
				ORDER BY AppointmentDay
				ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING) AS Previous_appointments,
	   SUM(CASE WHEN No_show = 'Yes' THEN 1 ELSE 0 END) OVER(
				PARTITION BY PatientId
				ORDER BY AppointmentDay
				ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING) AS Previous_no_shows
FROM hospital_appointments
),
Patient_rate AS(
SELECT *,
		CAST(Previous_no_shows * 1.0 / NULLIF(Previous_appointments, 0) AS DECIMAL (5,2)) 
		AS Previous_no_shows_rate
FROM Patient_history
)
SELECT PatientId,
	   AppointmentId,
	   AppointmentDay,
	   Neighbourhood,
	   LeadTime,
	   Previous_appointments,
	   Previous_no_shows,
	   Previous_no_shows_rate,
	   CASE WHEN Previous_appointments = 0 THEN 'New Patient - Monitor'
			WHEN Previous_no_shows_rate >= 0.5
				 OR LeadTime >= 8 THEN 'High Risk'
			WHEN Previous_no_shows_rate >= 0.2
				 OR LeadTime BETWEEN 4 AND 7 THEN 'Medium Risk'
			ELSE 'Low Risk'
		END AS Risk_tier
FROM Patient_rate
----------------------------------------------------------------------------------------

SELECT * FROM hospital_appointments

SELECT * FROM View_AppointmentRisk	   
	
```



### Data Visualization
---

<img width="1513" height="861" alt="Screenshot 2026-08-14 130139" src="https://github.com/user-attachments/assets/f448d817-ee24-47cc-8f80-43a1b510bc3c" />

<img width="1536" height="918" alt="Screenshot 2026-08-14 130311" src="https://github.com/user-attachments/assets/955c86c3-1ac8-435a-9ecb-23a6cb6fd9b1" />


### Finding and Recommendation
---
1. Appointment Lead Time
One of the key factors identified was lead time. The analysis showed that longer lead times were associated with higher no-show rates.
Lead time refers to the interval between the date an appointment was scheduled and the actual appointment date. A longer waiting period may give patients more time to feel better, seek care elsewhere, or become unavailable by the time of their appointment.

- Recommendation:
The hospital should explore ways to reduce appointment lead times where possible. For appointments that require longer waiting periods, timely reminders and follow-up could also help reduce missed appointments.

2. Age and No-Show Rates
Teenagers and young adults recorded the highest no-show rates in the analysis.
This may be related to lifestyle and other factors that could make it easier for younger patients to forget or deprioritize appointments. It is also possible that some patients feel better before their appointment, particularly when there is a long gap between scheduling and the appointment date, and therefore decide not to attend.

- Recommendation:
Consider targeted reminders and follow-up strategies for younger patients, particularly for appointments with longer lead times.

3. Effect of SMS Reminders
In this analysis, patients who received SMS reminders did not appear to have a lower no-show rate. However, this finding should be interpreted carefully.
SMS reminders were not necessarily sent to every patient. Patients with longer lead times were more likely to receive an SMS, while same-day appointments may not require or receive a reminder.
Therefore, the analysis does not establish that SMS reminders are ineffective. The difference could partly be explained by the characteristics of the patients who received the reminders.

- Recommendation:
The hospital could further evaluate the effectiveness of SMS reminders by comparing patients with similar appointment characteristics, particularly lead time, and considering a controlled or targeted reminder strategy.

- Key learning: An analytical finding should always be interpreted within the context of the underlying data and other influencing factors rather than viewed in isolation.

4. Gender and No-Show Rates
The analysis showed little difference in no-show rates between male and female patients.
The relatively small difference suggests that gender was not a major factor influencing appointment attendance in this dataset.

- Recommendation:
Gender may not need to be a primary factor when designing appointment reminder or follow-up strategies. Resources could instead be focused on factors that showed stronger associations with no-shows, such as lead time and age.

5. Neighbourhood had noticeable differences in appointment no-show rates. Some neighbourhoods recorded higher no-show rates than others, suggesting that location may be associated with appointment attendance patterns
I ranked neighbourhoods based on their no-show rates, considering only neighbourhoods with at least 100 appointments to avoid drawing conclusions from areas with very small sample sizes.

- Recommendation
The hospital could investigate the reasons behind higher no-show rates in specific neighbourhoods and consider targeted reminder or follow-up strategies where appropriate.

6. Patient Risk Classification
As part of the analysis, I created a View_AppointmentRisk using CTEs and window functions in SQL Server.
The view uses a patient's previous appointment history, previous no-shows, and current appointment lead time to assign a Risk_Tier.
This type of risk classification could help healthcare teams identify appointments that may require additional reminders or follow-up, allowing limited resources to be directed toward patients who may be more likely to miss their appointments.
Rather than relying on a single factor, healthcare teams should consider a combination of patient history, lead time, age, and other relevant characteristics when developing strategies to reduce appointment no-shows.
