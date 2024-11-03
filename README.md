## Tools Used - MYSQL, Power BI

## Table of Contents
1. [Project Objective](#project-objective)
2. [ETL Framework](#etl-framework)
3. [Dataset Description](#dataset-description)
4. [Project Workflow](#project-workflow)
5. [Power BI Dashboard](#power-bi-dashboard)
6. [Actionable Strategies to Reduce Churn](#actionable-strategies-to-reduce-churn)



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

## ETL Framework

Our ETL framework utilizes the following components:

1. **CSV File**: Serves as the source file for our data.
2. **MySQL Workbench**: We'll use the inbuilt Import Wizard to handle data transformation and loading.
3. **MySQL Database**: The final data will be loaded here, serving as our data warehouse with tables structured for analysis and reporting.


### Dataset Description

 - **Total Rows**: 2362
 - **Total Columns**: 9
 - **Dataset Overview**: This dataset contains layoffs information for a U.S.-based company from 2020 to 2023, including details such as the company name, location, industry, total employees laid off, percentage laid off, date of layoffs, stage of layoffs, country, and funds raised in millions.

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
Recommendations:

Delete Rows: Consider deleting rows with NULL values in the total_laid_off and percentage_laid_off columns to ensure the dataset is complete and to reduce potential bias in your analysis. This approach is preferred over imputation given the high percentages of missing data.

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
   FROM world_layoffs.layoffs_staging2
   WHERE  percentage_laid_off IS NOT NULL;
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
