# Companies Layoffs Analysis

## Table of Contents

- [Project Overview](#project-overview)
- [Data Source](#data-source)
- [Data Cleaning](#data-cleaning)
- [Exploratory Data Analysis](#exploratory-data-analysis)

### Project Overview

This data analysis aims to identify trends on companies layoffs around the world during covid pandemic from 2020 to 2023

### Data Source

The dataset used for this project is 'layoffs.csv', you can download it [here.](https://microsoft.com)

### Data Cleaning
1. Remove duplicates
2. Standardize the data to make sure it has consistent data type and data format
3. Remove Null values or Blank values
4. Remove any columns or rows that aren't necessary or irrelevant

#### Remove duplicates
Create Staging

> Staging is use for modifying data so we dont mess with raw data

```sql
CREATE TABLE layoffs_staging LIKE layoffs;

SELECT*FROM layoffs_staging;

INSERT layoffs_staging SELECT*FROM layoffs;
```
Identifying Duplicates

> Use row number so we can see if there are more than one row it means it is duplicated
```
WITH duplicate_cte AS
(
SELECT*,
ROW_NUMBER() OVER(
PARTITION BY company, industry, total_laid_off, percentage_laid_off, `date`) as row_num
FROM layoffs_staging
)
SELECT*FRON duplicate_cte
WHERE row_num > 1;
```
> Check the duplicates, in this case we pick company name 'Oda'
```
SELECT*FROM layoffs_staging WHERE company = 'Oda';
```
> Upon checking, it wasn't a duplicate because the country location is different, the query earlier not complete because the columns is not fully included

> Correction of previous query
```
WITH duplicate_cte AS
(
SELECT*,
ROW_NUMBER() OVER(
PARTITION BY company, location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions) as row_num
FROM layoffs_staging
)
SELECT*FROM duplicate_cte
WHERE row_num > 1;
```

> Since in MySQL you cannot update a CTE, the workaround is to create another staging table to delete the duplicates.
Use create statement to create a table, then insert 'row_num' column as 'INT'
```
CREATE TABLE `layoffs_staging2` (
  `company` text,
  `location` text,
  `industry` text,
  `total_laid_off` int DEFAULT NULL,
  `percentage_laid_off` text,
  `date` text,
  `stage` text,
  `country` text,
  `funds_raised_millions` int DEFAULT NULL,
  `row_num` int
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```
After creating table 'layoffs_staging2', we insert the data from 'layoffs_staging'
```
INSERT INTO layoffs_staging2
SELECT*,
ROW_NUMBER() OVER(
PARTITION BY company, location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions) as row_num
FROM layoffs_staging;
```
Then DELETE the data where the row number greater than 1, which indicate it is duplicated
```
DELETE from layoffs_staging2
WHERE row_num > 1;
```

#### Standardizing data
Formatting

> There's a space in front of the companies name, we need to trim it
```
SELECT company, TRIM(company)
FROM layoffs_staging2;

UPDATE layoffs_staging2
SET company = trim(company);
```
> Check table industry, some industry mention couple times, even though its the same meaning. Also there's a blank and null, need to get rid of that.
```
SELECT distinct industry from layoffs_staging2
ORDER BY 1;
```
> Check further, for example, the crypto industry, to see if there are multiple names with the same meaning
```
SELECT*FROM layoffs_staging2
WHERE industry LIKE 'Crypto%';

UPDATE layoffs_staging2
SET industry = 'Crypto'
WHERE industry LIKE 'Crypto%';
```

> Check table country, there's two United States
```
SELECT DISTINCT country FROM layoffs_staging2 ORDER BY country;
```
> To remove the dot, we can use trailing
```
SELECT DISTINCT country, trim(trailing '.' FROM country) FROM layoffs_staging2 order by country;
UPDATE layoffs_staging2 SET country = trim(trailing '.' FROM country) WHERE country LIKE 'United States%';
```
> Currently date showing as text data type, the format is not good for time series data analysis, so we need to change that format
```
SELECT `date`, str_to_date(`date`, '%m/%d/%Y') FROM layoffs_staging2;
UPDATE layoffs_staging2 SET `date` = str_to_date(`date`, '%m/%d/%Y');
ALTER TABLE layoffs_staging2 modify column `date` DATE;
```
#### Removing blank and null values
> populate industry where column is null, for example airbnb actually have industry, but for some reason there is one that dont have industry
```
SELECT * FROM layoffs_staging2 WHERE industry IS NULL or industry = '';
SELECT * FROM layoffs_staging2 WHERE company = 'Airbnb';
```
> Because we have 'Airbnb' that has 'industry' and 'Airbnb' that is 'blank', we are going to join the table itself to fill in the blank with the same value that has 'industry'
```
-- Remove the blank first, because it cause an error
UPDATE layoffs_staging2 SET industry = NULL where industry = '';

UPDATE layoffs_staging2 t1
JOIN layoffs_staging2 t2
	on t1.company = t2.company
    SET t1.industry = t2.industry
    WHERE t1.industry is null
    AND t2.industry is not null;
```
#### Remove any columns, rows, or data that aren't necessary
> Some of the companies didn't have data about how many they laid off, so we can delete that
```
DELETE FROM layoffs_staging2
WHERE total_laid_off is null
AND percentage_laid_off is null;
```
> Lastly, we want to drop the row_num column because we didn't need it anymore after we cleanup the duplicates
```
ALTER TABLE layoffs_staging2
DROP COLUMN row_num;
SELECT*FROM layoffs_staging2;
```
### Exploratory Data Analysis
EDA involved exploring the layoffs data to answer questions, such as:
- Which company had the most employee laid off?
- Which company whose laid off all of its employee?
- What is the total employees laid off through the year?
- What is the trend of employees laid off each month?

