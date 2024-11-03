## Tools Used - MYSQL, Power BI

## Table of Contents
1. [Project Objective](#project-objective)
2. [ETL Framework](#etl-framework)
3. [Dataset Description](#dataset-description)
4. [Project Workflow](#project-workflow)
5. [Power BI Dashboard](#power-bi-dashboard)
6. [Predictions](#predictions)
7. [Actionable Strategies to Reduce Churn](#actionable-strategies-to-reduce-churn)


-- Select all data from the original `layoffs` table
```bash
SELECT * 
FROM world_layoffs.layoffs;
```
-- Step 1: Create a staging table Inserting the Data as a backup for data cleaning
```bash
CREATE TABLE world_layoffs.layoffs_staging 
LIKE world_layoffs.layoffs;

INSERT INTO layoffs_staging 
SELECT * FROM world_layoffs.layoffs;
```
-- Steps to follow for Data Wrangling :
-- 1. Remove duplicates
-- 2. Standardize data and fix errors
-- 3. Handle null values
-- 4. Remove unnecessary columns and rows

1. Remove Duplicates

-- Check for duplicates in the staging table
 ```bash
SELECT company, industry, total_laid_off, `date`,
	ROW_NUMBER() OVER (
		PARTITION BY company, industry, total_laid_off, `date`) AS row_num
	FROM world_layoffs.layoffs_staging;
 ```
--Identify rows that are duplicates
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
-- Delete duplicates based on specific criteria
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
-- Optionally, add `row_num` column for further duplicate handling
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
-- Delete duplicates based on `row_num`
```bash
DELETE FROM world_layoffs.layoffs_staging2
WHERE row_num >= 2;
```
2. Standardize Data

-- Retrieve all records from the layoffs staging table 
```bash
SELECT * 
FROM world_layoffs.layoffs_staging2;
````

-- Standardize nulls and blank rows for the `industry` column
```bash
UPDATE world_layoffs.layoffs_staging2
SET industry = NULL
WHERE industry = '';
```

-- if we look at industry it looks like we have some null and empty rows, let's take a look at these
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

-- nothing wrong here
```bash
SELECT *
FROM world_layoffs.layoffs_staging2
WHERE company LIKE 'airbnb%';
```
-- it looks like airbnb is a travel, but this one just isn't populated.
-- I'm sure it's the same for the others. What we can do is
-- write a query that if there is another row with the same company name, it will update it to the non-null industry values
-- makes it easy so if there were thousands we wouldn't have to manually check them all

-- we should set the blanks to nulls since those are typically easier to work with
```bash
UPDATE world_layoffs.layoffs_staging2
SET industry = NULL
WHERE industry = '';
```

-- now if we check those are all null
```bash
SELECT *
FROM world_layoffs.layoffs_staging2
WHERE industry IS NULL 
OR industry = ''
ORDER BY industry;
```

-- now we need to populate those nulls if possible
```bash
UPDATE layoffs_staging2 t1
JOIN layoffs_staging2 t2
ON t1.company = t2.company
SET t1.industry = t2.industry
WHERE t1.industry IS NULL
AND t2.industry IS NOT NULL;
```

-- and if we check it looks like Bally's was the only one without a populated row to populate this null values
```bash
SELECT *
FROM world_layoffs.layoffs_staging2
WHERE industry IS NULL 
OR industry = ''
ORDER BY industry;
```


-- I also noticed the Crypto has multiple different variations. We need to standardize that - let's set all to Crypto
```bash
SELECT DISTINCT industry
FROM world_layoffs.layoffs_staging2
ORDER BY industry;
```

-- Standardize variations in `industry` names (e.g., 'Crypto Currency' to 'Crypto')
```bash
UPDATE layoffs_staging2
SET industry = 'Crypto'
WHERE industry IN ('Crypto Currency', 'CryptoCurrency');
````

-- now that's taken care of:
```bash
SELECT DISTINCT industry
FROM world_layoffs.layoffs_staging2
ORDER BY industry;
```
-- everything looks good except apparently we have some "United States" and some "United States." with a period at the end. Let's standardize this.
```bash
SELECT DISTINCT country
FROM world_layoffs.layoffs_staging2
ORDER BY country;
```
-- Standardize `country` names to remove trailing periods
```bash
UPDATE layoffs_staging2
SET country = TRIM(TRAILING '.' FROM country);
```

-- now if we run this again it is fixed
```bash
SELECT DISTINCT country
FROM world_layoffs.layoffs_staging2
ORDER BY country;
```

-- Let's also fix the date columns:
```bash
SELECT *
FROM world_layoffs.layoffs_staging2;
```

-- Convert `date` to a consistent date format
```bash
UPDATE layoffs_staging2
SET `date` = STR_TO_DATE(`date`, '%m/%d/%Y');
````
-- Modify the `date` column to the DATE data type
```bash
ALTER TABLE layoffs_staging2
MODIFY COLUMN `date` DATE;
```

-- Final check of all records in the table
```bash
SELECT *
FROM world_layoffs.layoffs_staging2;
```



3. Handle Null Values
```bash
SELECT 
    SUM(CASE WHEN company IS NULL THEN 1 ELSE 0 END) AS null_company,
    SUM(CASE WHEN location IS NULL THEN 1 ELSE 0 END) AS null_location,
    SUM(CASE WHEN industry IS NULL THEN 1 ELSE 0 END) AS null_industry,
    SUM(CASE WHEN total_laid_off IS NULL THEN 1 ELSE 0 END) AS null_total_laid_off,
    SUM(CASE WHEN percentage_laid_off IS NULL THEN 1 ELSE 0 END) AS null_percentage_laid_off,
    SUM(CASE WHEN `date` IS NULL THEN 1 ELSE 0 END) AS null_date,
    SUM(CASE WHEN stage IS NULL THEN 1 ELSE 0 END) AS null_stage,
    SUM(CASE WHEN country IS NULL THEN 1 ELSE 0 END) AS null_country,
    SUM(CASE WHEN funds_raised_millions IS NULL THEN 1 ELSE 0 END) AS null_funds_raised_millions
FROM world_layoffs.layoffs_staging2;
```
```bash

````
4. Remove Useless Data
```bash
-- Delete rows with null values in both `total_laid_off` and `percentage_laid_off`
DELETE FROM world_layoffs.layoffs_staging2
WHERE total_laid_off IS NULL
AND percentage_laid_off IS NULL;
````
-- Drop the `row_num` column now that duplicates are removed
```bash
ALTER TABLE layoffs_staging2
DROP COLUMN row_num;
```
-- Exploratory Data Analysis (EDA)
```bash
-- Find maximum layoffs
SELECT MAX(total_laid_off)
FROM world_layoffs.layoffs_staging2;
```
-- Identify the highest percentage of layoffs
```bash
SELECT MAX(percentage_laid_off), MIN(percentage_laid_off)
FROM world_layoffs.layoffs_staging2
WHERE percentage_laid_off IS NOT NULL;
```
-- Companies with 100% layoffs
```bash
SELECT *
FROM world_layoffs.layoffs_staging2
WHERE percentage_laid_off = 1;
```
-- Companies with the most layoffs
```bash
SELECT company, SUM(total_laid_off) AS total_laid_off
FROM world_layoffs.layoffs_staging2
GROUP BY company
ORDER BY total_laid_off DESC
LIMIT 10;
```
-- Summarize layoffs by `location`
```bash
SELECT location, SUM(total_laid_off)
FROM world_layoffs.layoffs_staging2
GROUP BY location
ORDER BY SUM(total_laid_off) DESC
LIMIT 10;
````
-- Total layoffs by year
```bash
SELECT YEAR(date), SUM(total_laid_off)
FROM world_layoffs.layoffs_staging2
GROUP BY YEAR(date)
ORDER BY YEAR(date);
```
-- Rolling total of layoffs by month
```bash
WITH DATE_CTE AS (
  SELECT DATE_FORMAT(date, '%Y-%m') AS month, SUM(total_laid_off) AS total_laid_off
  FROM layoffs_staging2
  GROUP BY month
)
SELECT month, SUM(total_laid_off) OVER (ORDER BY month) AS rolling_total_layoffs
FROM DATE_CTE
ORDER BY month;
````
