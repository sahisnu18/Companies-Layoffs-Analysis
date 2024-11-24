# Companies Layoffs Analysis - Data Cleaning

## Table of Contents

- [Project Overview](#project-overview)
- [Data Source](#data-source)
- [Data Cleaning](#data-cleaning)

### Project Overview

This data analysis aims to clean the data and present the output properly so we can do exploratory data analysis in the next step.

### Data Source

The dataset used for this project is 'layoffs.csv', you can download it [here.](layoffs.csv)

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

> Because we don't have unique row ID in our data,
We use row number so we can see if the row_num is greater than one row, it means it is duplicated.
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
![Check Duplicates](https://github.com/user-attachments/assets/3e9bb214-7b63-4fed-a87d-64749f654fff)

> Check the duplicates, in this case we pick company name 'Oda'
```
SELECT*FROM layoffs_staging WHERE company = 'Oda';
```
![Check Duplicates 2](https://github.com/user-attachments/assets/187e88db-48d5-421f-bb62-9d07b730db9f)

> Upon checking, it wasn't a duplicate because the country location is different. If we look at the 'duplicate_cte' query earlier, it is incomplete because the columns in the 'PARTITION BY' are not fully included.


> Correction of the previous query after we included every single column.
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
After we checked the data that duplicates, we need to delete it.
>  But, in MySQL you cannot update a CTE, the workaround is to create another staging table, copy the data, filter the row_num greater than 1, then delete the duplicates.

> Use create statement to create a table, then insert 'row_num' column as 'INT'
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
After creating table 'layoffs_staging2', we insert data from 'layoffs_staging'
```
INSERT INTO layoffs_staging2
SELECT*,
ROW_NUMBER() OVER(
PARTITION BY company, location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions) as row_num
FROM layoffs_staging;
```
Then DELETE the data where the row number is greater than 1, which indicates it is duplicated
```
DELETE from layoffs_staging2
WHERE row_num > 1;
```
<img width="772" alt="{219A2C27-1B67-4183-8381-35E5E4D3371B}" src="https://github.com/user-attachments/assets/4fc3ce72-5600-41ac-9a60-82afa594dbd0">

#### Standardizing data
Formatting

> There's a space in front of the companies name, we need to trim it
<img width="176" alt="{73630705-F3CD-49FA-B566-70488E424A51}" src="https://github.com/user-attachments/assets/c0ecb85a-26fd-48a0-9f86-23749faf66ce">

```
SELECT company, TRIM(company)
FROM layoffs_staging2;

UPDATE layoffs_staging2
SET company = trim(company);
```
> Check table industry, some industry mention couple times, even though its the same meaning. Also there's a blank and null, need to get rid of that.
<img width="141" alt="{2F6A0C79-FA37-465D-ABA1-D0556BBE2076}" src="https://github.com/user-attachments/assets/50fa3175-4c65-40aa-bc5c-b866a0a9f7f9">

```
SELECT distinct industry from layoffs_staging2
ORDER BY 1;
```
> Check further, for example, the crypto industry, to see if there are multiple names with the same meaning
```
SELECT*FROM layoffs_staging2
WHERE industry LIKE 'Crypto%';
```
<img width="763" alt="{5F062FFC-3A90-4DEC-B83D-39ACB32EECB0}" src="https://github.com/user-attachments/assets/1c1b9d05-07bc-4269-9205-b7550009d915">

Remove other than Crypto industry.
```
UPDATE layoffs_staging2
SET industry = 'Crypto'
WHERE industry LIKE 'Crypto%';
```

> Check table country, there's two United States
```
SELECT DISTINCT country FROM layoffs_staging2 ORDER BY country;
```
<img width="129" alt="{9CE9AABF-9FC6-4DB9-9F0E-014F7DCB6DFC}" src="https://github.com/user-attachments/assets/4ef67d02-7a87-4b2b-b4a1-9d279ca6242f">

> To remove the dot, we can use trailing
<img width="272" alt="{EA1A28AC-45DC-4089-8C89-940DDAC68A3F}" src="https://github.com/user-attachments/assets/8fa99f9e-a4de-4333-aa6a-ce5933a98571">

```
SELECT DISTINCT country, trim(trailing '.' FROM country) FROM layoffs_staging2 order by country;
UPDATE layoffs_staging2 SET country = trim(trailing '.' FROM country) WHERE country LIKE 'United States%';
```
Currently 'date' showing as 'TEXT' data type, the format is not good for time series data analysis, so we need to change that format

<img width="113" alt="{A53573E7-3452-4F71-AACE-E0B966426201}" src="https://github.com/user-attachments/assets/07148c03-d028-4253-9c3c-b0256acdd71c">

<img width="184" alt="{0D866BF7-FD66-4310-9620-72D027E15A42}" src="https://github.com/user-attachments/assets/96f31690-c67d-4ea9-9fc5-7eee2147e838">

```
SELECT `date`, str_to_date(`date`, '%m/%d/%Y') FROM layoffs_staging2;
UPDATE layoffs_staging2 SET `date` = str_to_date(`date`, '%m/%d/%Y');
ALTER TABLE layoffs_staging2 modify column `date` DATE;
```
#### Removing blank and null values
If we look at column 'industry', there are blank and null values.
<img width="728" alt="{77860CBF-26ED-48B0-9DF1-2810900215B1}" src="https://github.com/user-attachments/assets/88537f0d-d80e-4e88-a6ba-c1abd87291f2">

For example airbnb actually have industry, but there is one that dont have industry. Because we know that Airbnb is in the 'Travel' industry, so we try to populate or update the same data which is 'Travel' industry into the column that is blank.

<img width="711" alt="{19C7CC7B-9D18-4F3B-972A-4E4E6A9C64DB}" src="https://github.com/user-attachments/assets/e847db78-cf10-4eda-bbd3-9527807bc6d6">

> What i'm trying to do is, if we're trying to look at which industry were impacted the most, the blank row won't be in our output, and the result will be invalid because of miscalculated.
```
SELECT * FROM layoffs_staging2 WHERE industry IS NULL or industry = '';
SELECT * FROM layoffs_staging2 WHERE company = 'Airbnb';
```
To populate or update the same data into the other industry column, we are going to join the table itself to fill in the 'blank' with the same value that has 'industry'
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
<img width="713" alt="{B719E073-02CA-478A-A294-088DB294C05E}" src="https://github.com/user-attachments/assets/a7196180-f8a6-44ae-8dc3-d82d936a8489">

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

