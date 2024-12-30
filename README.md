# Analysis-of-World-Layoffs


## Table of Content

- [Project Overview](#project-overview)

- [Data Source](#data-source)

- [Tools](#tools)

- [Data Cleaning/Preparation](#data-cleaningpreparation)

- [Exploratory Data Analysis](#exploratory-data-analysis)

- [Insights/Results](#insightsresults)

- [Key Challenges](#key-challenges)
  
- [Conclusion](#conclusion)

- [References](#references)

### Project Overview

This is an in-depth examination of global job displacement trends across various industries and regions. This analysis typically identifies the primary factors driving layoffs, such as economic downturns, technological advancements, industry-specific challenges, or organizational restructuring. It also evaluates the short-term and long-term impacts of layoffs on employees, businesses, and the economy. The goal is to uncover patterns, provide actionable recommendations for mitigation, and forecast future trends in the labor market.

### Data Source

Layoffs data: The primary dataset used for this analysis is world_layoffs.xlsx, which contains detailed information about layoffs across industries and companies between 2020 and 2023.

### Tools

Excel is the source of the dataset in the XLSX file. Structured Query Language is used to clean the dataset, format, transform, group, and analyze.

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
```
![1](https://github.com/user-attachments/assets/25ed8967-5c11-49b2-a30c-39a8df83e8d6)
This shows that the highest total layoff is 12000 people which is 100% of the employees

```sql
SELECT *
FROM layoff_staging2
WHERE percentage_laid_off = 1
ORDER BY total_laid_off DESC;
```
![2](https://github.com/user-attachments/assets/f5359642-836c-4d9e-a979-eeb00359592f)
This shows the top 10 companies and industries with 100% layoffs, arranged according to their number of layoffs

```sql
SELECT company, SUM(total_laid_off) sum_total
FROM layoff_staging2
GROUP BY company
ORDER BY sum_total DESC;
```
![3](https://github.com/user-attachments/assets/b784b7c8-817e-4535-812d-780d6199d632)
This shows the top 5 companies with the highest total layoffs
```sql
SELECT *
FROM layoff_staging2
WHERE percentage_laid_off = 1
ORDER BY funds_raised_millions DESC;
```
![4](https://github.com/user-attachments/assets/09462660-d1ab-4659-967b-3187f6bfcf96)
This shows the top 10 companies and industries with 100% layoffs, arranged according to the number of millions made

```sql
SELECT industry, SUM(total_laid_off) sum_total
FROM layoff_staging2
GROUP BY industry
ORDER BY sum_total DESC;
```
![5](https://github.com/user-attachments/assets/556f3631-73aa-4fe2-a892-aea14c685fca)
This shows total layoffs by industries

```sql
SELECT country, SUM(total_laid_off) sum_total
FROM layoff_staging2
GROUP BY country
ORDER BY 2 DESC;
```
![6](https://github.com/user-attachments/assets/ad7f1ad7-ac86-48a6-a116-4bf4d5155db8)
This shows the total layoffs by countries

```sql
SELECT YEAR(`date`), SUM(total_laid_off) sum_total
FROM layoff_staging2
GROUP BY YEAR(`date`)
ORDER BY 1 DESC;
```
![7](https://github.com/user-attachments/assets/d896aed6-7234-4e91-9c4f-fd73bcfeab29)
This shows the total layoffs by years

```sql
SELECT stage, SUM(total_laid_off) sum_total
FROM layoff_staging2
GROUP BY stage
ORDER BY 2 DESC;
```
![8](https://github.com/user-attachments/assets/b7a36618-6c8c-4ff7-b53b-e584aaf33b2b)
This shows the stages of the companies by the total layoffs

```sql
SELECT company, YEAR(`date`), SUM(total_laid_off) 
FROM layoff_staging2
GROUP BY company, YEAR(`date`)
ORDER BY 3 DESC;
```
![10](https://github.com/user-attachments/assets/6a29e60e-ef31-4e8e-b06d-e2223f00b651)
This shows the companies with the highest total layoffs by years

```sql
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
```
![11](https://github.com/user-attachments/assets/f5e0950e-55d5-4a0a-8ee5-fc8ef1fc20a6)
This shows the ranking of total layoffs of companies by years

### Insights/Results

1. The top industries with 100% layoffs include Construction, Food, Retail, Transportation, Consumer, and Travel.
2. The top companies with 100% layoffs and their locations are Katerra—SF Bay Area, Butler Hospitality—New York City, Deliv—SF Bay Area, Jump—New York City, SEND—Sydney, HOOQ—Singapore, Stoqo—Jakarta, Stay Alfred—Spokane, Britishvolt—London, and Planetly—Berlin.
3. The top 5 companies with the highest total layoff overtime are Amazon, Google, Meta, Salesforce and Microsoft
4. Britishvolt Company had the highest funds raised, followed by Quibi, Deliveroo Australia, Katerra, BlockFi, Aura Financial and so on
5. THe top 10 industries with the highest total layoffs are Consumer, Retail, Others, Transportation, Finance, Healthcare, Food, Real Estate, Travel and Hardware
6. The top 10 countries where the highest layoffs were experienced are the United States, India, Netherlands, Sweden, Brazil, Germany, United Kingdom, Canada, Singapore, and China
7. The highest total layoffs were recorded in 2023, followed by 2022, 2021, and then 2020
8. The top 10 companies by their stages with the highest total layoff are Post-IPO, Unknown, Acquired, Series C, Series D, Series B, Series E, Series F, Private Equity, and Series H
9. Ranking companies according to the highest total layoff in each year shows that Uber, Booking.com, and Groupon were top 3 in 2020; Bytedance, Katerra, and Zillow in 2021; Meta, Amazon, and Cisco in 2022; Google, Microsoft, and Ericsson in 2023.

### Key Challenges
1. Some industries and countries appear multiple times with alterations in their names
2. The percentage layoff column was not useful as a lot of values are missing

### Conclusion
The phenomenon of layoffs worldwide reflects the dynamic interplay of economic, technological, and organizational factors. While layoffs are often necessary for businesses to remain agile and competitive, their ripple effects on individuals, communities, and economies are profound. Addressing these challenges requires a balanced approach—one that combines strategic workforce planning with initiatives to support displaced workers through reskilling, upskilling, and alternative employment opportunities. By fostering collaboration between governments, businesses, and educational institutions, we can mitigate the adverse impacts of layoffs and build a more resilient, inclusive global workforce.

### References
1. [ChatGPT](https://chat.openai.com)
2. [Google](https://www.google.com)














