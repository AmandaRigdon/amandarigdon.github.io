# Overview

This project was a practice in cleaning and exploring world layoff data in MySQL.

## Data Cleaning

To replicate a real-life scenario, I started with making a copy of the raw dataset, since we will be deleting and manipulating some of the data.

```sql
CREATE TABLE layoffs_staging
LIKE world_layoffs.layoffsconverted;
```
```sql
INSERT layoffs_staging
SELECT * 
FROM world_layoffs.layoffsconverted;
```

### Removing Duplicates

Because this dataset does not have a row with a unique ID, I inserted a row number to check for duplicates.

```sql
SELECT *,
ROW_NUMBER() OVER(
PARTITION BY company,industry, total_laid_off, percentage_laid_off, `date`) AS row_num
FROM layoffs_staging;
```


Now I can filter by row_num > 2 to identify duplicates by creating a CTE that will allow us to query from the above statement.

```sql
WITH duplicate_cte AS
(SELECT *,
ROW_NUMBER() OVER(
PARTITION BY company,industry, total_laid_off, percentage_laid_off, `date`) AS row_num
FROM layoffs_staging
)
SELECT *
FROM duplicate_cte
WHERE row_num > 1;
```

Below I am checking company to see if there's a true duplicate. In this case, Oda is not a duplicate, so I will have to revise my PARTITION BY statement.

```sql
SELECT*
FROM layoffs_staging
WHERE company = 'Oda';
```

```sql
WITH duplicate_cte AS
(SELECT *,
ROW_NUMBER() OVER(
PARTITION BY company,location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions) AS row_num
FROM layoffs_staging
)
SELECT *
FROM duplicate_cte
WHERE row_num > 1;
```

With that taken care of, I will be creating yet another table with the duplicates to delete from.

```sql
CREATE TABLE `layoffs_staging2` (
  `company` text,
  `location` text,
  `industry` text,
  `total_laid_off` text,
  `percentage_laid_off` text,
  `date` text,
  `stage` text,
  `country` text,
  `funds_raised_millions` text,
  `row_num` INT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

```

This is just an empty table, so I'll use the PARTITION BY statement from before to populate it.

```sql
INSERT INTO layoffs_staging2
SELECT *,
ROW_NUMBER() OVER(
PARTITION BY company,location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions) AS row_num
FROM layoffs_staging;
)
```

Finally deleting duplicates! 

```sql
DELETE 
FROM layoffs_staging2
WHERE row_num >1;
```

### Standardizing Data

I continued with using TRIM to remove spaces from the front and end of any strings.

```sql
SELECT company, (TRIM(company))
FROM layoffs_staging2;
```

```sql
UPDATE layoffs_staging2
SET company = TRIM(company);
```

Looking at industry

```sql
SELECT distinct industry
from layoffs_staging2
ORDER BY 1;
```

The "Crypto" industry has 3 different unique ID's under it, so I wrote the below statements to standardize anything in the industry column that starts with "Crypto" to just be "Crypto". 

```sql
SELECT *
from layoffs_staging2
WHERE industry LIKE 'Crypto%';
```

```sql
UPDATE layoffs_staging2
SET industry = 'Crypto'
WHERE industry LIKE 'Crypto%';
```

Looking at location

```sql
SELECT DISTINCT location
FROM layoffs_staging2
ORDER BY 1;
```

Somehow, a period got put onto one of the strings that has "United States" so I will use the TRAILING statement to remove the period.

```sql
SELECT *
FROM layoffs_staging2
WHERE country LIKE 'United States%'
ORDER BY 1;
```

```sql
SELECT DISTINCT country, TRIM(TRAILING '.' FROM country)
FROM layoffs_staging2
ORDER BY 1;
```

```sql
UPDATE layoffs_staging2
SET country = TRIM(TRAILING '.' FROM country)
WHERE country LIKE 'United States%';
```


