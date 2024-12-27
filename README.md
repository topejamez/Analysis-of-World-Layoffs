# Analysis-of-World-Layoffs

### Project Overview

This is an in-depth examination of global job displacement trends across various industries and regions. This analysis typically involves identifying the primary factors driving layoffs, such as economic downturns, technological advancements, industry-specific challenges, or organizational restructuring. It also evaluates the short-term and long-term impacts of layoffs on. The goal is to uncover patterns, provide actionable recommendations for mitigation, and forecast future trends in the labor market.

### Data Source

Layoffs data: The primary dataset used for this analysis is world_layoffs.xlsx, containing detailed information about the layoffs across industries and companies.

### Tools

Excel is the source of the dataset in the XLSX file. Structured Query Language is used to clean the dataset, format, transform, group, and analyze. Libraries used include 

### Data Cleaning/Preparation

It is important to ensure the dataset is clean enough to gain insights from a dataset. I ensured the dataset was thoroughly cleaned to ensure the accuracy and integrity of my findings. The data cleaning and preparation processes I carried out are shown in the codes below:

```sql
#Import Dataset
SELECT *
FROM layoffs;

#Create another table with the raw data
CREATE TABLE layoff_staging
LIKE layoffs;

INSERT layoff_staging
SELECT *
FROM layoffs;

#removing duplicates by partitioning the rows
SELECT *,
ROW_NUMBER() OVER(PARTITION BY company, location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions) AS row_num
FROM layoff_staging;

#Checking for rows with duplicates
WITH duplicate_cte AS 
(
SELECT *,
ROW_NUMBER() OVER(PARTITION BY company, location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions) AS row_num
FROM layoff_staging
)
SELECT *
FROM duplicate_cte
WHERE row_num  > 1 ;

CREATE TABLE `layoff_staging2` (
  `company` text,
  `location` text,
  `industry` text,
  `total_laid_off` int DEFAULT NULL,
  `percentage_laid_off` text,
  `date` text,
  `stage` text,
  `country` text,
  `funds_raised_millions` int DEFAULT NULL,
  `row_num` INT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

#View the new table
SELECT *
FROM layoff_staging2;

#Give numbers to each row
INSERT INTO layoff_staging2
SELECT *,
ROW_NUMBER() OVER(
PARTITION BY company, location, industry, total_laid_off, percentage_laid_off, `date` , stage
, country, funds_raised_millions) AS row_num
FROM layoff_staging;

#Delete rows with row numbers greater than 1
DELETE
FROM layoff_staging2
WHERE row_num > 1;

#Stardardizing data: finding issues in the data and fixing it
SELECT DISTINCT(company)
FROM layoff_staging2;

SELECT company, TRIM(company)
FROM layoff_staging2;

UPDATE layoff_staging2     
SET company = TRIM(company);

SELECT DISTINCT industry
FROM layoff_staging2
ORDER BY 1;

SELECT industry
FROM layoff_staging2
WHERE industry LIKE 'Crypto%';

#Renaming all crypto-related industries with just crypto 
UPDATE layoff_staging2
SET industry = 'Crypto'
WHERE industry LIKE 'Crypto%';

#Checking countries
SELECT DISTINCT country
FROM layoff_staging2
ORDER BY 1;

#Cleaning countries with '.'
SELECT DISTINCT country, TRIM(TRAILING '.' FROM country)
FROM layoff_staging2
ORDER BY 1;

#Removing '.' from United States
UPDATE layoff_staging2
SET country = TRIM(TRAILING '.' FROM country)
WHERE country LIKE 'United States%';

#Changing the date format
SELECT `date`,
STR_TO_DATE (`date`, '%m/%d/%Y')
FROM layoff_staging2;

#Updating the new date format
UPDATE layoff_staging2
SET date = STR_TO_DATE (`date`, '%m/%d/%Y');

#Checking the date
SELECT `date`
FROM layoff_staging2;

#Altering the date column
ALTER TABLE layoff_staging2
MODIFY COLUMN `date` DATE;

#Checking for null values on total and percentage columns
SELECT *
FROM layoff_staging2
WHERE total_laid_off IS NULL
AND percentage_laid_off IS NULL;

#Updating industries with a null value
UPDATE layoff_staging2
SET industry = null
WHERE industry = '';

#Checking to see if there are still number values on the industry column
SELECT *
FROM layoff_staging2
WHERE industry IS NULL
OR industry = '';

#Checking the Airbnb company
SELECT *
FROM layoff_staging2
WHERE company = 'Airbnb';

SELECT t1.industry, t2.industry
FROM layoff_staging2 t1
JOIN layoff_staging2 t2
	ON t1.company = t2.company
WHERE(t1.industry IS NULL OR t2.industry = '')
AND t2.industry IS NOT NULL;

#Updating null companies with values from not-null companies
UPDATE layoff_staging2 t1
JOIN layoff_staging2 t2
	ON t1.company = t2.company
SET t1.industry = t2.industry
WHERE t1.industry IS NULL
AND t2.industry IS NOT NULL;

#Deleting rows of total and percentage laid-off columns with null
DELETE 
FROM layoff_staging2
WHERE total_laid_off IS NULL
AND percentage_laid_off IS NULL;

#Drop the altered table
ALTER TABLE layoff_staging2
DROP COLUMN row_num;
```

### Exploratory Data Analysis

```sql
SELECT MAX(total_laid_off), MAX(percentage_laid_off)
FROM layoff_staging2;

SELECT *
FROM layoff_staging2
WHERE percentage_laid_off = 1
ORDER BY total_laid_off DESC;

SELECT company, SUM(total_laid_off) sum_total
FROM layoff_staging2
GROUP BY company
ORDER BY sum_total DESC;

SELECT *
FROM layoff_staging2
WHERE percentage_laid_off = 1
ORDER BY funds_raised_millions DESC;

SELECT industry, SUM(total_laid_off) sum_total
FROM layoff_staging2
GROUP BY industry
ORDER BY sum_total DESC;

SELECT country, SUM(total_laid_off) sum_total
FROM layoff_staging2
GROUP BY country
ORDER BY 2 DESC;

SELECT YEAR(`date`), SUM(total_laid_off) sum_total
FROM layoff_staging2
GROUP BY YEAR(`date`)
ORDER BY 1 DESC;

SELECT stage, SUM(total_laid_off) sum_total
FROM layoff_staging2
GROUP BY stage
ORDER BY 2 DESC;

SELECT company, YEAR(`date`), SUM(total_laid_off) 
FROM layoff_staging2
GROUP BY company, YEAR(`date`)
ORDER BY 3 DESC;

WITH company_year (company, years, total_laid_off) AS
(
SELECT company, YEAR(`date`), SUM(total_laid_off) 
FROM layoff_staging2
GROUP BY company, YEAR(`date`)
), company_year_rank AS
(SELECT *, DENSE_RANK() OVER(PARTITION BY years ORDER BY total_laid_off DESC) AS ranking
FROM company_year
WHERE years IS NOT NULL
)
SELECT *
FROM Company_Year_Rank
WHERE ranking <= 5;
