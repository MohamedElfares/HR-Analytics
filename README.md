# HR Analytics | Employee Attrition Analysis
### Power BI Case Study — DataCamp

---

## Table of Contents

- [Project Overview](#project-overview)
- [Dataset & Data Model](#dataset--data-model)
- [Data Preparation](#data-preparation)
  - [Calculated Tables](#calculated-tables)
  - [DAX Measures](#dax-measures)
- [Dashboard Pages](#dashboard-pages)
  - [Overview](#1-overview)
  - [Demographics](#2-demographics)
  - [Performance Tracker](#3-performance-tracker)
  - [Attrition](#4-attrition)
- [Key Findings](#key-findings)
- [Tools Used](#tools-used)

---

## Project Overview

This project is an end-to-end HR analytics case study focused on investigating and understanding employee attrition at a fictional company, **Atlas Labs**. The goal is to help HR stakeholders identify patterns driving attrition and make informed, data-backed decisions to improve employee retention.

The analysis is structured across four report pages: a high-level **Overview**, a **Demographics** breakdown, an individual **Performance Tracker**, and a dedicated **Attrition** analysis page.

---

## Dataset & Data Model

The dataset was provided by **DataCamp** as part of a guided case study. It follows a star schema consisting of one fact table and four dimension tables:

| Table | Type | Description |
|---|---|---|
| `FactPerformanceRating` | Fact | Performance review records per employee per year |
| `DimEmployee` | Dimension | Employee profile attributes |
| `DimEducationLevel` | Dimension | Lookup table for education level labels |
| `DimRatingLevel` | Dimension | Lookup table for performance rating labels |
| `DimSatisfiedLevel` | Dimension | Lookup table for satisfaction level labels |

![Data Model](./assets/data_model.png)

The key relationships established in the model connect `DimEmployee` to `FactPerformanceRating` via `EmployeeID`, with dimension lookup tables joined to the fact table using their respective ID columns. Several inactive relationships were defined and activated selectively in DAX measures using `USERELATIONSHIP()`.

---

## Data Preparation

### Calculated Tables

Two calculated tables were created to support the analysis.

**1. DimDate** — A calendar table generated dynamically from the hire date range in `DimEmployee`. It provides a full date hierarchy (year, quarter, month, week, day) and fiscal period support, enabling time-based filtering and trend analysis across the report.

```dax
DimDate = 
VAR _minYear = YEAR(MIN(DimEmployee[HireDate]))
VAR _maxYear = YEAR(MAX(DimEmployee[HireDate]))
VAR _fiscalStart = 4 

RETURN
ADDCOLUMNS(
    CALENDAR(
        DATE(_minYear,1,1),
        DATE(_maxYear,12,31)
    ),
    "Year",                 YEAR([Date]),
    "Year Start",           DATE(YEAR([Date]),1,1),
    "YearEnd",              DATE(YEAR([Date]),12,31),
    "MonthNumber",          MONTH([Date]),
    "MonthStart",           DATE(YEAR([Date]), MONTH([Date]), 1),
    "MonthEnd",             EOMONTH([Date],0),
    "DaysInMonth",          DATEDIFF(DATE(YEAR([Date]), MONTH([Date]), 1), EOMONTH([Date],0), DAY)+1,
    "YearMonthNumber",      INT(FORMAT([Date],"YYYYMM")),
    "YearMonthName",        FORMAT([Date],"YYYY-MMM"),
    "DayNumber",            DAY([Date]),
    "DayName",              FORMAT([Date],"DDDD"),
    "DayNameShort",         FORMAT([Date],"DDD"),
    "DayOfWeek",            WEEKDAY([Date]),
    "MonthName",            FORMAT([Date],"MMMM"),
    "MonthNameShort",       FORMAT([Date],"MMM"),
    "Quarter",              QUARTER([Date]),
    "QuarterName",          "Q"&FORMAT([Date],"Q"),
    "YearQuarterNumber",    INT(FORMAT([Date],"YYYYQ")),
    "YearQuarterName",      FORMAT([Date],"YYYY")&" Q"&FORMAT([Date],"Q"),
    "QuarterStart",         DATE(YEAR([Date]), (QUARTER([Date])*3)-2, 1),
    "QuarterEnd",           EOMONTH(DATE(YEAR([Date]), QUARTER([Date])*3, 1),0),
    "WeekNumber",           WEEKNUM([Date]),
    "WeekStart",            [Date]-WEEKDAY([Date])+1,
    "WeekEnd",              [Date]+7-WEEKDAY([Date]),
    "FiscalYear",           IF(_fiscalStart=1, YEAR([Date]), YEAR([Date])+QUOTIENT(MONTH([Date])+(13-_fiscalStart),13)),
    "FiscalQuarter",        QUARTER(DATE(YEAR([Date]),MOD(MONTH([Date])+(13-_fiscalStart)-1,12)+1,1)),
    "FiscalMonth",          MOD(MONTH([Date])+(13-_fiscalStart)-1,12)+1
)
```

**2. _Measures** — A dedicated empty table used exclusively as a container for all DAX measures. This keeps the data model clean and clearly distinguishes calculated measures from raw data columns.

---

### DAX Measures

All measures are stored in the `_Measures` table. Below is the full set of measures created for this report.

**Core Employee Counts**

```dax
TotalEmployees = DISTINCTCOUNT(DimEmployee[EmployeeID])
```
```dax
ActiveEmployees = 
    CALCULATE(
        COUNT(DimEmployee[EmployeeID]),
        FILTER(DimEmployee, DimEmployee[Attrition] = "No")
    )
```
```dax
InactiveEmployees = 
    CALCULATE(
        [TotalEmployees],
        FILTER(DimEmployee, DimEmployee[Attrition] = "Yes")
    )
```
```dax
TotalEmployeesDate = 
    CALCULATE(
        DISTINCTCOUNT(DimEmployee[EmployeeID]),
        USERELATIONSHIP(DimEmployee[HireDate], DimDate[Date])
    )
```
```dax
InactiveEmployeesDate = 
    CALCULATE(
        [InactiveEmployees],
        USERELATIONSHIP(DimEmployee[HireDate], DimDate[Date])
    )
```

**Attrition**

```dax
% Attrition Rate = [InactiveEmployees] / [TotalEmployees]
```
```dax
% Attrition Rate Date = DIVIDE([InactiveEmployeesDate], [TotalEmployeesDate])
```

**Salary**

```dax
AverageSalary = AVERAGE(DimEmployee[Salary])
```

**Performance & Satisfaction** — These measures activate inactive relationships to correctly resolve satisfaction and rating lookups from the dimension tables.

```dax
EnvironmentSatisfaction = 
    CALCULATE(
        MAX(FactPerformanceRating[EnvironmentSatisfaction]),
        USERELATIONSHIP(DimSatisfiedLevel[SatisfactionID], FactPerformanceRating[EnvironmentSatisfaction])
    )
```
```dax
JobSatisfaction = MAX(FactPerformanceRating[JobSatisfaction])
```
```dax
RelationshipSatisfaction = 
    CALCULATE(
        MAX(FactPerformanceRating[RelationshipSatisfaction]),
        USERELATIONSHIP(DimSatisfiedLevel[SatisfactionID], FactPerformanceRating[RelationshipSatisfaction])
    )
```
```dax
WorkLifeBalance = 
    CALCULATE(
        MAX(FactPerformanceRating[WorkLifeBalance]),
        USERELATIONSHIP(DimSatisfiedLevel[SatisfactionID], FactPerformanceRating[WorkLifeBalance])
    )
```
```dax
SelfRating = 
    CALCULATE(
        MAX(FactPerformanceRating[SelfRating]),
        USERELATIONSHIP(DimRatingLevel[RatingID], FactPerformanceRating[SelfRating])
    )
```
```dax
ManagerRating = 
    CALCULATE(
        MAX(FactPerformanceRating[ManagerRating]),
        USERELATIONSHIP(DimRatingLevel[RatingID], FactPerformanceRating[ManagerRating])
    )
```

**Review Dates**

```dax
LastReviewDate = 
    IF(
        ISBLANK(LASTDATE(FactPerformanceRating[ReviewDate])),
        "No Review Yet",
        FORMAT(LASTDATE(FactPerformanceRating[ReviewDate]), "mm/dd/yyyy")
    )
```
```dax
NextReviewDate = 
    VAR reviewOrHire =
        IF(
            ISBLANK(LASTDATE(FactPerformanceRating[ReviewDate])),
            MAX(DimEmployee[HireDate]),
            LASTDATE(FactPerformanceRating[ReviewDate])
        )
    RETURN
        FORMAT(reviewOrHire + 365, "mm/dd/yyyy")
```

---

## Dashboard Pages

### 1. Overview

![Overview Page](./assets/overview.png)

The overview page provides a high-level snapshot of the workforce. Atlas Labs has a total of **1,470 employees**, of which **1,233 are currently active** and **237 have left**, resulting in an overall **attrition rate of 16.1%**.

The **Employee Hiring Trends** chart breaks down annual hires by attrition status. The cohorts hired in **2012 and 2022** recorded the highest absolute number of attritions, while the **2017 cohort** had the lowest, with only 11 employees having left. The Technology department dominates headcount, with Software Engineers (247) and Sales Executives (269) being the two largest job roles across the organization.

---

### 2. Demographics

![Demographics Page](./assets/demographics.png)

The demographics page examines the workforce composition across age, gender, marital status, and ethnicity.

- **Age Distribution:** The workforce skews young, with approximately **60% of employees falling in the 20–29 age bracket**. The youngest employee is 18 and the oldest is 51.
- **Marital Status:** The majority of employees are married (42.45%), followed by single (37.35%), and divorced (20.2%).
- **Ethnicity & Salary:** White employees represent the largest group at approximately **58.5% of the workforce** and hold the highest average salary at around **$115,000**. Black or African American and Mixed or multiple ethnicity employees are comparable in headcount, yet differ noticeably in average salary — African American employees average **$112,117**. A notable finding is that **Native Hawaiian employees**, despite being the second-smallest ethnic group, receive an average salary comparable to the White employee group.

---

### 3. Performance Tracker

![Performance Tracker Page](./assets/performance_tracker.png)

The performance tracker is an employee-level drill-down page. Using the employee selector, HR managers can view any individual's hire date, last review date, and projected next review date (calculated as 365 days from the last review or hire date if no review exists yet).

Six time-series charts track annual trends for: **Job Satisfaction, Relationship Satisfaction, Environment Satisfaction, Work-Life Balance, Self Rating,** and **Manager Rating** — each scored on a 1–5 scale using their respective lookup dimension tables.

Reference tables for both the satisfaction scale (Very Dissatisfied to Very Satisfied) and performance rating scale (Unacceptable to Above and Beyond) are displayed at the bottom of the page for context.

---

### 4. Attrition

![Attrition Page](./assets/attrition.png)

The attrition page surfaces the key drivers behind the company's 16.1% attrition rate.

**By Job Role:**
Attrition is not evenly distributed across roles. **Recruiters** in the HR department and **Sales Representatives** in the Sales department exhibit the highest attrition rates, both approaching **~40%**. Within the Technology department, **Data Scientists** show a relatively elevated rate of **24%**, while Software Engineers stand at **13%**.

**By Hire Year:**
Employees hired in **2016 and 2020** show the highest attrition rates at **22%** and **21%** respectively, suggesting potential issues tied to those specific cohorts or the conditions at the time of hiring.

**By Business Travel:**
Travel frequency shows a clear correlation with attrition. Among the **277 Frequent Travellers**, the attrition rate is **25%**. This drops to **15%** for the **1,043 employees** in the "Some Travel" category, and falls sharply to just **8%** among the **150 employees** who do not travel at all.

**By Overtime:**
Employees required to work overtime have an attrition rate of **30%**, roughly double that of employees without an overtime requirement (~15%). This is a strong signal that workload and work-life balance are significant contributors to voluntary departures.

**By Tenure:**
The tenure chart reveals the most critical retention window. Employees with **one year or less at the company account for ~66% attrition**, indicating that early-career engagement and onboarding experience are pivotal. Attrition rates decline steadily as tenure increases, which suggests that employees who survive the first two years are substantially more likely to remain.

---

## Key Findings

| Area | Finding |
|---|---|
| Overall Attrition | 16.1% — 237 out of 1,470 employees have left |
| Highest-Risk Roles | Recruiters (~40%) and Sales Representatives (~40%) |
| Travel Impact | Frequent Travellers attrite at 3x the rate of non-travellers |
| Overtime | Employees on overtime have a 30% attrition rate |
| Tenure Risk | Employees with under 1 year of tenure show ~66% attrition |
| Age Profile | 60% of the workforce is aged 20–29, amplifying early-tenure risk |
| Salary Equity | Notable average salary disparities across ethnic groups |

---

## Tools Used

- **Power BI Desktop** — Data modeling, DAX, and report development
- **DAX** — Calculated tables, measures, and inactive relationship handling
- **DataCamp** — Dataset and case study framework

---

*This project was completed as part of the DataCamp HR Analytics in Power BI case study track.*
