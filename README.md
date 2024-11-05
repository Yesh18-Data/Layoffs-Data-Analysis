## Tools Used - MySQL, Power BI Desktop, Power BI Service

## Table of Contents
1. [Project Objective](#project-objective)
2. [ETL Framework](#etl-framework)
3. [Dataset Description](#dataset-description)
4. [Project Workflow](#project-workflow)
5. [Power BI Dashboard](#power-bi-dashboard)
6. [Findings and Insights](#findings-and-insights)

**Disclaimer**: The insights presented in this analysis may not reflect the complete picture of layoffs from 2020 to 2023. The dataset used was downloaded from Kaggle and may not capture all industry or regional trends comprehensively.

### Live Dashboard
[View Live Dashboard](https://app.powerbi.com/view?r=eyJrIjoiZjYxYjYyYzAtMmNiYy00Mzk3LWEzOGUtMDNhY2Y5NGFlNWU3IiwidCI6ImRmODY3OWNkLWE4MGUtNDVkOC05OWFjLWM4M2VkN2ZmOTVhMCJ9)


## Project Objective:  
To create a comprehensive ETL (Extract, Transform, Load) process in a MySQL database and develop Power BI reports utilizing layoffs data to achieve the following goals:

1. **Data Wrangling**: 
   - Cleaning, transforming, and standardizing data.
   - Removing unnecessary features.
   - Handling null values effectively.

2. **Comprehensive Exploratory Data Analysis (EDA)**: 
   - Analyzing layoffs data across multiple dimensions, including:
     - Years
     - Company
     - Industry
     - Stages
     - Location
3. **Power BI Dashboard**
   - Creating Interactive Dashboard to Analysis      

## ETL Framework

Our ETL framework utilizes the following components:

1. **CSV File**: Serves as the source file for our data.
2. **MySQL Workbench**: We'll use the inbuilt Import Wizard to handle data transformation and loading.
3. **MySQL Database**: The final data will be loaded here, serving as our data warehouse with tables structured for analysis and reporting.


### Dataset Description

 - **Total Rows**: 2362
 - **Total Columns**: 9
 - **Dataset Overview**: This dataset contains layoffs information for a U.S.-based company from 2020 to 2023, including details such as the company name, location, industry, total employees laid off, percentage laid off, date of layoffs, stage of layoffs, country, and funds raised in millions.

### Note: Please refer `Dataset` folder above for dataset.


## Project Workflow
### MySQL

#### Data Wrangling
Select all data from the original `layoffs` table
```bash
SELECT * 
FROM world_layoffs.layoffs;
```
Create a staging table Inserting the Data as a backup for data cleaning
```bash
CREATE TABLE world_layoffs.layoffs_staging 
LIKE world_layoffs.layoffs;

INSERT INTO layoffs_staging 
SELECT * FROM world_layoffs.layoffs;
```
- Steps to follow for Data Wrangling:
  - I. Remove duplicates
  - II. Standardize data and fix errors
  - III. Handle null values
  - IV. Remove unnecessary columns and rows


##### I. Remove Duplicates

1.Check for duplicates in the staging table
 ```bash
SELECT company, industry, total_laid_off, `date`,
	ROW_NUMBER() OVER (
		PARTITION BY company, industry, total_laid_off, `date`) AS row_num
	FROM world_layoffs.layoffs_staging;
 ```
2.Identify rows that are duplicates
 ```bash
SELECT *
FROM (
	SELECT company, industry, total_laid_off, `date`,
		ROW_NUMBER() OVER (
			PARTITION BY company, location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions
			) AS row_num
	FROM world_layoffs.layoffs_staging
) duplicates
WHERE row_num > 1;
 ```
3.Delete duplicates based on specific criteria
 ```bash
WITH DELETE_CTE AS 
(
	SELECT company, location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions,
		ROW_NUMBER() OVER (
			PARTITION BY company, location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions
			) AS row_num
	FROM world_layoffs.layoffs_staging
	WHERE row_num > 1
)
DELETE FROM world_layoffs.layoffs_staging
WHERE (company, location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions, row_num) 
IN (
	SELECT company, location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions, row_num
	FROM DELETE_CTE
);
 ````
4. Optionally, add `row_num` column for further duplicate handling
```bash
ALTER TABLE world_layoffs.layoffs_staging ADD row_num INT;
````
```bash
-- Create a new table with added row numbers to easily delete duplicates
CREATE TABLE `world_layoffs`.`layoffs_staging2` (
  `company` TEXT,
  `location` TEXT,
  `industry` TEXT,
  `total_laid_off` INT,
  `percentage_laid_off` TEXT,
  `date` TEXT,
  `stage` TEXT,
  `country` TEXT,
  `funds_raised_millions` INT,
  row_num INT
);

INSERT INTO `world_layoffs`.`layoffs_staging2`
  (`company`, `location`, `industry`, `total_laid_off`, `percentage_laid_off`, `date`, `stage`, `country`, `funds_raised_millions`, `row_num`)
SELECT `company`, `location`, `industry`, `total_laid_off`, `percentage_laid_off`, `date`, `stage`, `country`, `funds_raised_millions`,
  ROW_NUMBER() OVER (PARTITION BY company, location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions) AS row_num
  FROM world_layoffs.layoffs_staging;
```
5. Delete duplicates based on `row_num`
```bash
DELETE FROM world_layoffs.layoffs_staging2
WHERE row_num >= 2;
```
### II. Standardize Data

6. Retrieve all records from the layoffs staging table 
```bash
SELECT * 
FROM world_layoffs.layoffs_staging2;
````

7. Standardize nulls and blank rows for the `industry` column
```bash
UPDATE world_layoffs.layoffs_staging2
SET industry = NULL
WHERE industry = '';
```

8. if we look at industry it looks like we have some null and empty rows, let's take a look at these
```bash
SELECT DISTINCT industry
FROM world_layoffs.layoffs_staging2
ORDER BY industry;
```
``` bash
SELECT *
FROM world_layoffs.layoffs_staging2
WHERE industry IS NULL 
OR industry = ''
ORDER BY industry;
````
```bash
-- let's take a look at these
SELECT *
FROM world_layoffs.layoffs_staging2
WHERE company LIKE 'Bally%';
```

9. nothing wrong here
```bash
SELECT *
FROM world_layoffs.layoffs_staging2
WHERE company LIKE 'airbnb%';
```
-- it looks like airbnb is a travel, but this one just isn't populated.
-- I'm sure it's the same for the others. What we can do is
-- write a query that if there is another row with the same company name, it will update it to the non-null industry values
-- makes it easy so if there were thousands we wouldn't have to manually check them all

10. we should set the blanks to nulls since those are typically easier to work with
```bash
UPDATE world_layoffs.layoffs_staging2
SET industry = NULL
WHERE industry = '';
```

11. now if we check those are all null
```bash
SELECT *
FROM world_layoffs.layoffs_staging2
WHERE industry IS NULL 
OR industry = ''
ORDER BY industry;
```

12. now we need to populate those nulls if possible
```bash
UPDATE layoffs_staging2 t1
JOIN layoffs_staging2 t2
ON t1.company = t2.company
SET t1.industry = t2.industry
WHERE t1.industry IS NULL
AND t2.industry IS NOT NULL;
```

13. and if we check it looks like Bally's was the only one without a populated row to populate this null values
```bash
SELECT *
FROM world_layoffs.layoffs_staging2
WHERE industry IS NULL 
OR industry = ''
ORDER BY industry;
```


14.  I also noticed the Crypto has multiple different variations. We need to standardize that - let's set all to Crypto
```bash
SELECT DISTINCT industry
FROM world_layoffs.layoffs_staging2
ORDER BY industry;
```

15.  Standardize variations in `industry` names (e.g., 'Crypto Currency' to 'Crypto')
```bash
UPDATE layoffs_staging2
SET industry = 'Crypto'
WHERE industry IN ('Crypto Currency', 'CryptoCurrency');
````

16.  now that's taken care of:
```bash
SELECT DISTINCT industry
FROM world_layoffs.layoffs_staging2
ORDER BY industry;
```
17. everything looks good except apparently we have some "United States" and some "United States." with a period at the end. Let's standardize this.
```bash
SELECT DISTINCT country
FROM world_layoffs.layoffs_staging2
ORDER BY country;
```
18.  Standardize `country` names to remove trailing periods
```bash
UPDATE layoffs_staging2
SET country = TRIM(TRAILING '.' FROM country);
```

19.  now if we run this again it is fixed
```bash
SELECT DISTINCT country
FROM world_layoffs.layoffs_staging2
ORDER BY country;
```

20. Let's also fix the date columns:
```bash
SELECT *
FROM world_layoffs.layoffs_staging2;
```

21. Convert `date` to a consistent date format
```bash
UPDATE layoffs_staging2
SET `date` = STR_TO_DATE(`date`, '%m/%d/%Y');
````
22.  Modify the `date` column to the DATE data type
```bash
ALTER TABLE layoffs_staging2
MODIFY COLUMN `date` DATE;
```

23.  Final check of all records in the table
```bash
SELECT *
FROM world_layoffs.layoffs_staging2;
```



### 3.Finding Null Values
24. Column wise Null percentage count 
```bash
 SELECT 
    COUNT(*) AS total_rows,
    SUM(CASE WHEN company IS NULL THEN 1 ELSE 0 END) / COUNT(*) * 100 AS percent_null_company,
    SUM(CASE WHEN location IS NULL THEN 1 ELSE 0 END) / COUNT(*) * 100 AS percent_null_location,
    SUM(CASE WHEN industry IS NULL THEN 1 ELSE 0 END) / COUNT(*) * 100 AS percent_null_industry,
    SUM(CASE WHEN total_laid_off IS NULL THEN 1 ELSE 0 END) / COUNT(*) * 100 AS percent_null_total_laid_off,
    SUM(CASE WHEN percentage_laid_off IS NULL THEN 1 ELSE 0 END) / COUNT(*) * 100 AS percent_null_percentage_laid_off,
    SUM(CASE WHEN `date` IS NULL THEN 1 ELSE 0 END) / COUNT(*) * 100 AS percent_null_date,
    SUM(CASE WHEN stage IS NULL THEN 1 ELSE 0 END) / COUNT(*) * 100 AS percent_null_stage,
    SUM(CASE WHEN country IS NULL THEN 1 ELSE 0 END) / COUNT(*) * 100 AS percent_null_country,
    SUM(CASE WHEN funds_raised_millions IS NULL THEN 1 ELSE 0 END) / COUNT(*) * 100 AS percent_null_funds_raised_millions
FROM world_layoffs.layoffs_staging2;
```

Summary of Missing Data:

Total Laid Off: 31.34% of rows are missing values, indicating significant missing data.
Percentage Laid Off: 33.25% of rows are also missing values, which is similarly concerning.
funds_raised_millions: 8.0986% of rows are also missing values, comparitively less with above features

Delete Rows: Consider deleting rows with NULL values in the total_laid_off and percentage_laid_off columns to ensure the dataset is complete and to reduce potential bias in your analysis. 


```bash

````
### 4. Remove Useless Data
25.  Delete rows with null values in both `total_laid_off` and `percentage_laid_off`
```bash
-- Delete rows with null values in both `total_laid_off` and `percentage_laid_off`
DELETE FROM world_layoffs.layoffs_staging2
WHERE total_laid_off IS NULL
AND percentage_laid_off IS NULL;
````
26.  Drop the `row_num` column now that duplicates are removed
```bash
ALTER TABLE layoffs_staging2
DROP COLUMN row_num;
```

27. After deleting rows with null values in both `total_laid_off` and `percentage_laid_off` , still we left with 18.8632% nulls for `total_laid_off` , 21.2777% for `percentage_laid_off`, 8.0986% for `funds_raised_millions` then decided to do the simple Median imputation technique ( becuase it is most suitable when outliers are present ) based on most appropriate dimension Industry in the dataset.

 - Medial Imputation for `total_laid_off`
 - Step 1: Calculate median `total_laid_off` for each industry

```bash
WITH RankedData AS (
    SELECT 
        industry, 
        total_laid_off,
        ROW_NUMBER() OVER (PARTITION BY industry ORDER BY total_laid_off) AS row_num,
        COUNT(*) OVER (PARTITION BY industry) AS total_rows
    FROM world_layoffs.layoffs_staging2
    WHERE total_laid_off IS NOT NULL
),
MedianValues AS (
    SELECT industry, AVG(total_laid_off) AS median_total_laid_off
    FROM RankedData
    WHERE row_num IN (FLOOR((total_rows + 1) / 2), CEIL((total_rows + 1) / 2))
    GROUP BY industry
)
````
-- Step 2: Update `total_laid_off` NULL values with calculated median values
````bash
UPDATE world_layoffs.layoffs_staging2 AS w
JOIN MedianValues AS m ON w.industry = m.industry
SET w.total_laid_off = COALESCE(w.total_laid_off, m.median_total_laid_off)
WHERE w.total_laid_off IS NULL;
````


 - Medial Imputation for `percentage_laid_off`
 - Step 1: Calculate median `percentage_laid_off` for each industry
 
```bash
WITH RankedData AS (
    SELECT 
        industry, 
        percentage_laid_off,
        ROW_NUMBER() OVER (PARTITION BY industry ORDER BY percentage_laid_off) AS row_num,
        COUNT(*) OVER (PARTITION BY industry) AS total_rows
    FROM world_layoffs.layoffs_staging2
    WHERE percentage_laid_off IS NOT NULL
),
MedianValues AS (
    SELECT industry, AVG(percentage_laid_off) AS median_percentage_laid_off
    FROM RankedData
    WHERE row_num IN (FLOOR((total_rows + 1) / 2), CEIL((total_rows + 1) / 2))
    GROUP BY industry
)
````
Step 2: Update `total_laid_off` NULL values with calculated median values
```bash
UPDATE world_layoffs.layoffs_staging2 AS w
JOIN MedianValues AS m ON w.industry = m.industry
SET w.percentage_laid_off = COALESCE(w.percentage_laid_off, m.median_percentage_laid_off)
WHERE w.percentage_laid_off IS NULL;
```

 - Medial Imputation for `funds_raised_millions`
````bash
UPDATE world_layoffs.layoffs_staging2 AS w
JOIN (
    SELECT industry, AVG(funds_raised_millions) AS avg_funds_raised
    FROM world_layoffs.layoffs_staging2
    WHERE funds_raised_millions IS NOT NULL
    GROUP BY industry
) AS g ON w.industry = g.industry
SET w.funds_raised_millions = COALESCE(w.funds_raised_millions, g.avg_funds_raised)
WHERE w.funds_raised_millions IS NULL;
````


## Exploratory Data Analysis (EDA)

Conduct Exploratory Data Analysis (EDA): Explore the dataset to identify trends, patterns, and outliers while maintaining an open-minded approach, allowing for the discovery of unexpected insights, guided by initial inquiries about the data.

Here’s a description for each query in the provided SQL code related to analyzing layoffs data:

1. **Select All Data**:
   ```sql
   SELECT * 
   FROM world_layoffs.layoffs_staging2;
   ```
   This query retrieves all records from the `layoffs_staging2` table to provide a complete view of the dataset.

2. **Maximum Total Laid Off**:
   ```sql
   SELECT MAX(total_laid_off)
   FROM world_layoffs.layoffs_staging2;
   ```
   This query finds the maximum number of employees laid off in a single incident, helping to identify the largest layoff event recorded.

3. **Maximum and Minimum Percentage of Layoffs**:
   ```sql
   SELECT MAX(percentage_laid_off), MIN(percentage_laid_off)
   FROM world_layoffs.layoffs_staging2;
   ```
   This query retrieves the highest and lowest percentage of employees laid off across all records, providing insights into the extent of layoffs relative to company size.

4. **Companies with 100% Layoffs**:
   ```sql
   SELECT *
   FROM world_layoffs.layoffs_staging2
   WHERE  percentage_laid_off = 1;
   ```
   This query selects companies where 100% of the employees were laid off, typically indicating company closures or severe financial difficulties.

5. **Companies with 100% Layoffs Ordered by Funds Raised**:
   ```sql
   SELECT *
   FROM world_layoffs.layoffs_staging2
   WHERE  percentage_laid_off = 1
   ORDER BY funds_raised_millions DESC;
   ```
   This query lists companies that laid off all their employees, ordered by the amount of funds they previously raised, which can highlight significant failures in the startup ecosystem.

6. **Companies with the Biggest Single Layoff**:
   ```sql
   SELECT company, total_laid_off
   FROM world_layoffs.layoffs_staging
   ORDER BY 2 DESC
   LIMIT 5;
   ```
   This query identifies the top five companies that experienced the largest single-day layoffs, useful for understanding extreme events in the data.

7. **Companies with Most Total Layoffs**:
   ```sql
   SELECT company, SUM(total_laid_off)
   FROM world_layoffs.layoffs_staging2
   GROUP BY company
   ORDER BY 2 DESC
   LIMIT 10;
   ```
   This query calculates and ranks the total number of layoffs for each company, showing which companies had the highest cumulative layoffs.

8. **Total Layoffs by Location**:
   ```sql
   SELECT location, SUM(total_laid_off)
   FROM world_layoffs.layoffs_staging2
   GROUP BY location
   ORDER BY 2 DESC
   LIMIT 10;
   ```
   This query summarizes layoffs by geographical location, helping to identify regions most affected by layoffs.

9. **Total Layoffs by Country**:
   ```sql
   SELECT country, SUM(total_laid_off)
   FROM world_layoffs.layoffs_staging2
   GROUP BY country
   ORDER BY 2 DESC;
   ```
   This query groups and sums layoffs by country to analyze the impact of layoffs on a national level.

10. **Total Layoffs by Year**:
    ```sql
    SELECT YEAR(date), SUM(total_laid_off)
    FROM world_layoffs.layoffs_staging2
    GROUP BY YEAR(date)
    ORDER BY 1 ASC;
    ```
    This query aggregates the total number of layoffs by year, providing a temporal view of layoff trends.

11. **Total Layoffs by Industry**:
    ```sql
    SELECT industry, SUM(total_laid_off)
    FROM world_layoffs.layoffs_staging2
    GROUP BY industry
    ORDER BY 2 DESC;
    ```
    This query calculates the total layoffs by industry, allowing for an understanding of which sectors are most impacted.

12. **Delete Rows with Null Stages**:
    ```sql
    DELETE FROM layoffs_staging2
    WHERE stage IS NULL;
    ```
    This query deletes records from the `layoffs_staging2` table where the stage is null, cleaning the dataset for further analysis.

13. **Total Layoffs by Stage**:
    ```sql
    SELECT stage, SUM(total_laid_off)
    FROM world_layoffs.layoffs_staging2
    GROUP BY stage
    ORDER BY 2 DESC;
    ```
    This query sums layoffs by the stage of the layoffs process, giving insights into how layoffs are distributed across different stages.

14. **Top Companies with Most Layoffs by Year**:
    ```sql
    WITH Company_Year AS 
    (
      SELECT company, YEAR(date) AS years, SUM(total_laid_off) AS total_laid_off
      FROM layoffs_staging2
      GROUP BY company, YEAR(date)
    ),
    Company_Year_Rank AS (
      SELECT company, years, total_laid_off, DENSE_RANK() OVER (PARTITION BY years ORDER BY total_laid_off DESC) AS ranking
      FROM Company_Year
    )
    SELECT company, years, total_laid_off, ranking
    FROM Company_Year_Rank
    WHERE ranking <= 3
    AND years IS NOT NULL
    ORDER BY years ASC, total_laid_off DESC;
    ```
    This query creates a Common Table Expression (CTE) to rank companies by total layoffs per year and retrieves the top three companies with the most layoffs for each year, providing a year-wise view of layoffs.

15. **Rolling Total of Layoffs Per Month**:
    ```sql
    SELECT SUBSTRING(date,1,7) as dates, SUM(total_laid_off) AS total_laid_off
    FROM layoffs_staging2
    GROUP BY dates
    ORDER BY dates ASC;
    ```
    This query aggregates total layoffs by month to analyze trends over time on a monthly basis.

16. **Rolling Total of Layoffs Using CTE**:
    ```sql
    WITH DATE_CTE AS 
    (
      SELECT SUBSTRING(date,1,7) as dates, SUM(total_laid_off) AS total_laid_off
      FROM layoffs_staging2
      GROUP BY dates
      ORDER BY dates ASC
    )
    SELECT dates, SUM(total_laid_off) OVER (ORDER BY dates ASC) as rolling_total_layoffs
    FROM DATE_CTE
    ORDER BY dates ASC;
    ```
    This query uses a CTE to calculate the rolling total of layoffs per month, providing insights into cumulative layoffs over time.



Each of these queries serves to explore different aspects of the layoffs data, from examining total layoffs and percentages to identifying trends over time and by various categories.
## Power BI Dashboard
![layoffs Data Analysis_page-0001](https://github.com/user-attachments/assets/53a27384-b4d8-4ad8-9940-93e22fa79826)


### Current Setup:
1. **Database Connection**: Connected to MySQL through ODBC in Power BI.
2. **Separate Measures Table**: Created a dedicated table for measures, which helps in organizing calculations and keeps your model structured.
3. **Calendar Table**: Built a calendar table, essential for time-based analysis and enabling features like time intelligence functions 
4. **Stage Lifecycle Hierarchy**: Defined stages of company lifecycle in Power Query using a custom column, grouping companies into `Early Stage Funding`, `Growth Stage Funding`, `Mature Stages`, `Post-IPO`, and `Other`.

1. **Total Layoffs**:
   ```bash
   Total_Layoffs = SUM('LayoffsData'[layoffs])
   ```

2. **Average Layoffs per Company**:
   ```bash
   Avg_Layoffs_Per_Company = DIVIDE([Total_Layoffs], DISTINCTCOUNT('layoffs_staging2'[company]))
   ```

4. **Layoffs by Stage Lifecycle**:
   ```bash
   Layoffs_By_Stage_Lifecycle = 
   CALCULATE([Total_Layoffs], ALLEXCEPT('layoffs_staging2', 'layoffs_staging2'[Stage_lifecycle]))
   ```
5. **Layoffs by Industry**:
   ```bash
   Layoffs_By_Country = 
   CALCULATE([Total_Layoffs], ALLEXCEPT('layoffs_staging2', 'layoffs_staging2'[industry]))
   ```
6. **Layoffs by Country**:
   ```bash
   Layoffs_By_Country = 
   CALCULATE([Total_Layoffs], ALLEXCEPT('layoffs_staging2', 'layoffs_staging2'[country]))
   ```
7. **Total Companies**:
   ```bash
   No of Companies = DISTINCTCOUNT(layoffs_staging2[company])
   ``` 
8. **Total Funds raised**
   ```bash
   Total funds raised = SUM(layoffs_staging2[Funds_Raised])
    ```
9. **Funds Raised by Country**
  ```bash
  Funds_raised_By_Country = CALCULATE([Total funds raised],ALLEXCEPT(layoffs_staging2,layoffs_staging2[country]))
  ```
10.**Funds Raised by Indsutry**
```bash
Funds_raised_By_Industry = CALCULATE([Total funds raised],ALLEXCEPT(layoffs_staging2,layoffs_staging2[industry]))
```
11.**Company Rank by Layoffs**
```bash
CompanyRankByLayoffs = 
RANKX(
    ALL(layoffs_staging2[company]),         
    CALCULATE([Total layoffs])
```
12.**Created Heirarchy for stage**
```bash
Stage_lifecycle = 
    IF [stage] = "Seed" || [stage] = "Series A" || [stage] = "Series B" 
    THEN "Early Stage Funding"
    ELSE IF [stage] = "Series C" || [stage] = "Series D" || [stage] = "Series E" 
    THEN "Growth Stage Funding"
    ELSE IF [stage] = "Acquired" || [stage] = "Private Equity" 
    THEN "Mature Stages"
    ELSE IF [stage] = "Post-IPO" 
    THEN "Post-IPO"
    ELSE "Other"
```



## Findings and Insights


### Key Findings

1. **Scope of Layoffs (2020-March 2023)**
   - **Total Companies**: 1,621 companies conducted layoffs over the past three years.
   - **Total Layoffs**: Approximately 441,000 employees were laid off, with each company laying off an average of 208 employees.
   - **Yearly Breakdown**:
     - **2020**: 86,000 layoffs
     - **2021**: 16,000 layoffs
     - **2022**: 174,000 layoffs
     - **2023**: 134,000 layoffs
   - Layoffs peaked in 2022, possibly due to global economic pressures, inflation, and post-pandemic adjustments, while layoffs in 2023 remained high but slightly lower.

2. **Corporate Funding Amid Layoffs**
   - **Total Funds Raised**: Despite layoffs, these companies raised around $1.8 trillion over the three-year period.
   - **Top Fundraising Countries**:
     - **United States**: $1,260 billion
     - **India**: $166 billion
     - **United Kingdom**: $56 billion
     - **Germany**: $51 billion
     - **China**: $51 billion

3. **Layoffs by Company Stage**
   - **Post-IPO Companies**: Accounted for 51% of layoffs, indicating that even publicly listed companies faced significant workforce challenges.
   - **Private Equity or Acquired Companies**: 19.07%, reflecting financial restructuring or acquisitions leading to downsizing.
   - **Growth Stage Companies**: 14.36%, likely facing adjustments to operational expenses.
   - **Mature Stage Companies**: 9.03%, despite their stability, indicating wider market pressures.
   - **Early Stage Companies**: 7%, showing layoffs even among newer startups, which could be due to funding constraints.

4. **Layoffs by Industry**
   - The **Retail** (47,000 layoffs) and **Consumer** (46,000 layoffs) sectors were most affected, likely due to changing consumer behavior and economic uncertainty.
   - Other major contributors:
     - **IT/Other**: 38,000 layoffs
     - **Transportation**: 36,000 layoffs
     - **Finance**: 31,000 layoffs
   - These industries combined accounted for around 65% of the total layoffs, suggesting that service-oriented and customer-driven sectors were more impacted.

5. **Geographic Distribution of Layoffs**
   - **United States**: 276,000 layoffs, by far the largest contributor.
   - **India**: 37,000 layoffs, highlighting the impact on the country’s tech and service sectors.
   - **Netherlands**: 17,000 layoffs
   - **Sweden** and **Brazil**: 11,000 layoffs each
   - **Germany**: 9,000 layoffs
   - **United Kingdom** and **Canada**: 7,000 layoffs each
   - **China**: 6,000 layoffs
   - **Indonesia**: 4,000 layoffs
   - The U.S. leads in both layoffs and funding, reflecting a high-risk, high-reward business environment where large-scale investments are often paired with substantial workforce adjustments.

6. **Top Companies by Layoffs**
   - Major tech companies led layoffs:
     - **Amazon**: 18,150
     - **Google**: 12,000
     - **Meta**: 11,000
     - **Salesforce**: 10,090
     - **Microsoft**: 10,000
     - Other notable contributors included **Philips** (10,000), **Ericsson** (8,500), **Uber** (7,585), **Dell** (6,650), and **Booking.com** (6,651).
   - These companies faced substantial layoffs, likely as a result of both cost-cutting and restructuring efforts in response to a rapidly changing market environment.

### Insights

1. **Economic Impact on Mature Companies**:  
   Layoffs were common among post-IPO and mature companies, likely due to market pressures and the need to meet high-growth expectations during economic slowdowns.

2. **Sectoral Vulnerability**:  
   Retail and consumer sectors saw the highest layoffs, driven by reduced spending and a shift to online shopping. Transportation and finance also faced challenges tied to consumer and economic shifts.

3. **High Investment, High Instability**:  
   Despite layoffs, companies, especially in the U.S., raised substantial funds, reflecting strong investor interest in tech and innovation but challenges in maintaining workforce stability.

4. **Regional Impact**:  
   Layoffs were concentrated in the U.S., while India's tech sector also faced significant job cuts, highlighting both countries’ exposure to global market shifts.

5. **Tech Sector Realignment**:  
   Major tech companies led layoffs, indicating an industry-wide move toward operational efficiency over rapid growth.



