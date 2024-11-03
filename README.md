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

-- 1. Remove Duplicates

-- Check for duplicates in the staging table
 ```bash
SELECT company, industry, total_laid_off, `date`,
	ROW_NUMBER() OVER (
		PARTITION BY company, industry, total_laid_off, `date`) AS row_num
	FROM world_layoffs.layoffs_staging;
 ```
-- Identify rows that are duplicates
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
-- 2. Standardize Data
```bash
-- Standardize nulls and blank rows for the `industry` column
UPDATE world_layoffs.layoffs_staging2
SET industry = NULL
WHERE industry = '';
```
```bash
-- Populate missing values in the `industry` column based on existing values for the same company
UPDATE layoffs_staging2 t1
JOIN layoffs_staging2 t2
ON t1.company = t2.company
SET t1.industry = t2.industry
WHERE t1.industry IS NULL
AND t2.industry IS NOT NULL;
```
-- Standardize variations in `industry` names (e.g., 'Crypto Currency' to 'Crypto')
```bash
UPDATE layoffs_staging2
SET industry = 'Crypto'
WHERE industry IN ('Crypto Currency', 'CryptoCurrency');
````
-- Standardize `country` names to remove trailing periods
```bash
UPDATE layoffs_staging2
SET country = TRIM(TRAILING '.' FROM country);
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
-- 3. Handle Null Values
```bash
-- Keep `NULL` values in `total_laid_off`, `percentage_laid_off`, and `funds_raised_millions` for easier analysis during EDA.
````
-- 4. Remove Useless Data
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

-- Find maximum layoffs
SELECT MAX(total_laid_off)
FROM world_layoffs.layoffs_staging2;

-- Identify the highest percentage of layoffs
SELECT MAX(percentage_laid_off), MIN(percentage_laid_off)
FROM world_layoffs.layoffs_staging2
WHERE percentage_laid_off IS NOT NULL;

-- Companies with 100% layoffs
SELECT *
FROM world_layoffs.layoffs_staging2
WHERE percentage_laid_off = 1;

-- Companies with the most layoffs
SELECT company, SUM(total_laid_off) AS total_laid_off
FROM world_layoffs.layoffs_staging2
GROUP BY company
ORDER BY total_laid_off DESC
LIMIT 10;

-- Summarize layoffs by `location`
SELECT location, SUM(total_laid_off)
FROM world_layoffs.layoffs_staging2
GROUP BY location
ORDER BY SUM(total_laid_off) DESC
LIMIT 10;

-- Total layoffs by year
SELECT YEAR(date), SUM(total_laid_off)
FROM world_layoffs.layoffs_staging2
GROUP BY YEAR(date)
ORDER BY YEAR(date);

-- Rolling total of layoffs by month
WITH DATE_CTE AS (
  SELECT DATE_FORMAT(date, '%Y-%m') AS month, SUM(total_laid_off) AS total_laid_off
  FROM layoffs_staging2
  GROUP BY month
)
SELECT month, SUM(total_laid_off) OVER (ORDER BY month) AS rolling_total_layoffs
FROM DATE_CTE
ORDER BY month;
