-- SQL Project - Data Cleaning

-- Data Source: https://www.kaggle.com/datasets/swaptr/layoffs-2022

-- Initial Query to display data
SELECT * 
FROM world_layoffs.layoffs;

-- Step 1: Create a staging table in SQL Server
-- SQL Server doesn’t support `CREATE TABLE ... LIKE`, so we manually define the table structure.
CREATE TABLE world_layoffs_layoffs_staging (
    company NVARCHAR(255),
    industry NVARCHAR(255),
    total_laid_off INT,
    [date] DATE
    -- Add any other columns based on the schema of `layoffs` table
);

-- Step 2: Insert data into staging table
-- This creates a working copy of the data for cleaning without altering the raw dataset.
INSERT INTO world_layoffs_layoffs_staging
SELECT * FROM world_layoffs.layoffs;

-- Step 3: Check for duplicates
-- Identifying duplicates ensures data quality by removing redundant records.
SELECT company, industry, total_laid_off, [date],
       ROW_NUMBER() OVER (PARTITION BY company, industry, total_laid_off, [date]) AS row_num
FROM world_layoffs_layoffs_staging;

-- Step 4: Remove duplicates based on row number
-- Retains the first occurrence of each duplicate group and removes others to avoid duplicate records in the dataset.
DELETE FROM world_layoffs_layoffs_staging
WHERE row_num > 1;

-- Step 5: Standardize Data
-- Standardizing data, such as converting text to uppercase, ensures consistency in the dataset.
UPDATE world_layoffs_layoffs_staging
SET company = UPPER(company);

-- Step 6: Handle NULL Values
-- Handling NULLs avoids issues in analysis. Here, NULLs in critical columns are replaced with default values.
UPDATE world_layoffs_layoffs_staging
SET total_laid_off = ISNULL(total_laid_off, 0);

-- Step 7: Drop Unnecessary Columns
-- Removing unused columns streamlines the dataset and improves storage efficiency.
-- Example: Assume there's an unnecessary column `redundant_column`.
ALTER TABLE world_layoffs_layoffs_staging
DROP COLUMN redundant_column;

-- Step 8: Final Cleaned Data Selection
-- After cleaning, we retrieve data from the staging table for analysis or further processing.
SELECT *
FROM world_layoffs_layoffs_staging;
