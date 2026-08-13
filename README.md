# Hospital_Appointment_No_Show_Analysis
This is the documentation of my Data Analysis Bootcamp Project1 with @INCUBATOR HUB.

## Project Title
Sales Performance Analysis for a Retail Store 
 
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

The original dataset contained 110,517 rows and 15 columns after cleaning and preparation.

The first step of the project was to understand the dataset and clearly define each variable. For example, I examined the difference between ScheduledDay and AppointmentDay:

- ScheduledDay — the date when the appointment was booked.
- AppointmentDay — the date when the patient was expected to attend the appointment.

I also clarified the meaning of the No_show variable:

- Yes — the patient did not attend the scheduled appointment.
- No — the patient attended the appointment.

Understanding these variables and their relationships provided the foundation for the subsequent data cleaning, SQL analysis, risk classification, and Power BI visualization carried out in the project.


### Data Source
---
The Medical Appointment No-Show dataset was downloaded from Kaggle (https://www.kaggle.com/datasets/joniarroba/noshowappointments) and used for this project. The original dataset contained 110,527 rows and 14 columns before data cleaning and preparation.
### Tools Used
---
-  SQL - Structured Query Language for Querying of Data
  1. For Data Cleaning
  2. For Analysis
  3. For Visualization
      
- Power BI - Power Business Intelligence for Data Visualization

- Github for
  1. Portfolio Building
  
### Data Cleaning and Preparations
---
Data cleaning was done using SQL. 
1. I first loaded the CSV file into SQL Server
2. Changed inconsistent column names and data types
3. cleaned 2 date columns in multiple steps. Created new columns, updated the new columns with cleaned dates, deleted the old columns and renamed the new ones with the old name.
4. checked data validity e.g there shoudnt be a negative age value
5. checked for null and droped the rows with null patientId
6. Added a new column (LeadTime) for the interval between schedule date and actual appointment date
7. checked leadtime validity and removed 5 rows with negative lead time value.

NOTE: In a work setting, there must be discussions with the data team or managerary officer before deleting any row or information from the data base. I went ahead and deleted because it is a personal project and the data is from open source.

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
KPI
1. Total number of appointments
2. Total number of No_shows
3. Overall percentage of No_shows
4. Average lead time

- What is the overall percentage(rate) of patients that didnt show up for their appointment?
- Does the day of the week affect appointment No_show?
- Does appointment lead time have an effect on appointment No_shows?
- Does age affect appointment No_shows?
- What is the effect of SMS reminder on No_shows?
- Which neighbourhood has the heighest risk of missing appointment (No_show)?
- Patients risk scoring (patients that are likely going to miss appointments)?
Created a predictive Risk_tier View using CTEs and table functions.


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

![Screenshot 2024-11-03 224816](https://github.com/user-attachments/assets/22e80175-5561-46b2-94fd-228622b87214)

![Screenshot 2024-11-03 224440](https://github.com/user-attachments/assets/2679ebaf-7922-445d-b229-07420d1d0e21)

![Screenshot 2024-11-03 224609](https://github.com/user-attachments/assets/6b286016-9469-4ff9-b91f-7616275f3045)

![Power BI Visual](https://github.com/user-attachments/assets/2a424d7a-cac5-454c-8fe0-40e6f2578625)

### Finding and Recommendation
- One of the factors that I identified was lead time. The longer the lead time, the higher the risk of a patient not showing up.
Lead time is simply the interval between the day an appointment was scheduled and the actual appointment day. 

- More goods should be supplied to the South since it generates more revenue, this will help to increase sales turnover, more branches can also be established in the South.
- Different brands of shoes should be supplied to all regions, because customers seem to purchase more of it, and their purchasing interest should be sustained with multi-brand choice.
- I suggest that more investigation should be carried out on the West store to know the cause of its low sales.
- it could be that people don't wear Socks much, which causes the sales to be low. If the revenue is not more than the capital, I suggest it should be removed from the stores.
