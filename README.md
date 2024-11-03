# Layoffs-Data-Analysis

-- Initial Query to display data
SELECT * 
FROM world_layoffs.layoffs;

Staging Table Creation: Creating a staging table ensures the raw data remains unchanged, which is crucial for data recovery if errors occur during cleaning

-- This command creates the `world_layoffs_layoffs_staging` table with the same structure and data from `world_layoffs.layoffs`
SELECT * INTO world_layoffs_layoffs_staging
FROM world_layoffs.layoffs;
