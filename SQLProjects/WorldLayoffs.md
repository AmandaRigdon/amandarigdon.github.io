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

The date column in this dataset has the wrong format as well as the wrong data type which is fixed with the below statements:

```sql
SELECT `date`,
str_to_date(`date`, '%m/%d/%Y')
FROM layoffs_staging2;
```

```sql
UPDATE layoffs_staging2
SET `date` = 
    CASE
        WHEN `date` IS NOT NULL AND `date` != 'NULL' THEN STR_TO_DATE(`date`, '%m/%d/%Y')
        ELSE NULL  
    END;
```

```sql
ALTER TABLE layoffs_staging2
MODIFY COLUMN `date` DATE;
```

With all this completed, the next step will be removing null values. In my case, I had to convert the .csv to .json, and in doing that, it turned all NULL values to literally say "NULL". Surprise! I had to update those nulls before I could continue.

```sql
UPDATE layoffs_staging2
SET total_laid_off = NULL
WHERE total_laid_off = 'NULL';
```

```sql
UPDATE layoffs_staging2
SET percentage_laid_off = NULL
WHERE percentage_laid_off = 'NULL';
```

In this example, I'm looking for null values in both percentage_laid_off and total_laid_off since those won't be useful for any analysis if there's no data in either column.

```sql
SELECT *
FROM layoffs_staging2
WHERE total_laid_off IS NULL
AND percentage_laid_off IS NULL;
```

Deleting row where this was applicable:

```sql
DELETE FROM layoffs_staging2
WHERE total_laid_off IS NULL
AND percentage_laid_off IS NULL;
```

Just at a cursory glance, there's a null value for industry for the company 'Airbnb'. We can check to see if other Airbnb rows has the industry, and in this case they do!

```sql
SELECT *
FROM layoffs_staging2
WHERE company = 'Airbnb';
```

Now it's time to correct the blank row by using JOIN. But first, I will also update the industry column to have NULL instead of 'NULL' just like above.

```sql
UPDATE layoffs_staging2
SET industry = NULL
where industry = 'NULL';
```

```sql
SELECT t1.industry, t2.industry
FROM layoffs_staging2 as t1
JOIN layoffs_staging2 as t2
	ON t1.company = t2.company
    AND t1.location = t2.location
WHERE (t1.industry = NULL 
and t2.industry != NULL;
```

This table joined the same two tables together, showing blank and populated values side by side. Now the blanks can be updated.

```sql
UPDATE layoffs_staging2 AS t1
JOIN layoffs_staging2 AS t2
	ON t1.company = t2.company
SET t1.industry = t2.industry
WHERE t1.industry IS NULL
and t2.industry IS NOT NULL;
```

This did not work because the blank values need to be set to null, not ''.

```sql
UPDATE layoffs_staging2
SET industry = NULL
where industry = '';
```

After doing that, the update statement worked! The last couple of things that need to be done are dropping the row_num column since it will no longer be used.

```sql
ALTER TABLE layoffs_staging2
DROP COLUMN row_num;
```

I also realized as I started exploring the data that the total_laid_off column and funds_raised_millions column became text somehow when they should be integer. So I fixed it before I started doing any analysis.

```sql
ALTER TABLE layoffs_staging2
MODIFY COLUMN total_laid_off INTEGER;
```

The NULL strikes again. I have to update 'NULL' to NULL before I alter the funds_raised_millions columns.

```sql
UPDATE layoffs_staging2
SET funds_raised_millions = NULL
WHERE funds_raised_millions = 'NULL';
```

```sql
ALTER TABLE layoffs_staging2
MODIFY COLUMN funds_raised_millions INTEGER;
```

## Data Exploration

From here I'll look at the MAX laid off:

```sql
SELECT MAX(total_laid_off)
FROM layoffs_staging2;
```

The max is 12,000...running another statement to see the percentage of employees that is.

```sql
SELECT MAX(total_laid_off), MAX(percentage_laid_off)
FROM layoffs_staging2;
```

This returned a MAX of 1, which means 100% of the company was laid off. Let's look at more companies that were 100% laid off:

```sql
SELECT *
FROM layoffs_staging2
WHERE percentage_laid_off = 1
ORDER BY total_laid_off DESC;
```
<img width="752" alt="image" src="https://github.com/AmandaRigdon/amandarigdon.github.io/assets/137234405/9d6cf508-8996-43f8-a621-3060496fd091">


The table above just shows a short summary of the data, there's 116 rows in total with companies that completely went under during 2020-2023. We can look at funds raised for companies that totally went under to see how much money they received.

```sql
SELECT *
FROM layoffs_staging2
WHERE percentage_laid_off = 1
ORDER BY funds_raised_millions DESC;
```

<img width="791" alt="image" src="https://github.com/AmandaRigdon/amandarigdon.github.io/assets/137234405/efaacc7b-3bbe-462a-93e8-dd8fbaad80ae">

The numbers under funds_raised_millions would come out to 2.4 million, 1.8 million, etc...

Now let's see which companies laid off the most employees.

```sql
SELECT company, SUM(total_laid_off)
FROM layoffs_staging2
GROUP BY company
ORDER BY 2 DESC;
```

<img width="181" alt="image" src="https://github.com/AmandaRigdon/amandarigdon.github.io/assets/137234405/8d44fbbf-f62b-46ad-a37f-8695407b8678">

Amazon had the most layoffs at 18,150, followed by other tech companies. This lines up with all the news about a bunch of layoffs happening in tech.

Double checking the date range to see the exact time period this data is covering.

```sql
SELECT MIN(`date`), MAX(`date`)
FROM layoffs_staging2;
```

The date range is from 3/11/2020 until 3/6/2023, so almost exacly 3 years from when the pandemic started. 

What country was impacted the most during this time?

```sql
SELECT country, SUM(total_laid_off)
FROM layoffs_staging2
GROUP BY country
ORDER BY 2 DESC;
```

<img width="214" alt="image" src="https://github.com/AmandaRigdon/amandarigdon.github.io/assets/137234405/2d46f832-1572-4a44-83e2-301a0ea5e165">

The U.S. at the top of this list wasn't very surprising, but it was surprising to see the Netherlands and Brazil on this list.

When did the most layoffs happen?

```sql
SELECT YEAR(`date`), SUM(total_laid_off)
FROM layoffs_staging2
GROUP BY YEAR(`date`)
ORDER BY 1 DESC;
```
<img width="171" alt="image" src="https://github.com/AmandaRigdon/amandarigdon.github.io/assets/137234405/a644d5ca-c189-4cce-8036-799b971df402">

2022 had the most layoffs, but it was probably beat by 2023, since the data for 2023 was for the first 3 months so far. I would have expected more maybe in 2020 when the pandemic hit and a lot of places closed down.

Looking at the total percentage laid off in each company would be hard to observe because we don't have the company size vs. the percentage that was laid off. Let's instead look at a rolling progression of layoffs.

```sql
SELECT SUBSTRING(`date`,1,7) AS `MONTH`, SUM(total_laid_off)
FROM layoffs_staging2
WHERE SUBSTRING(`date`,1,7) IS NOT NULL
GROUP BY `MONTH`
ORDER BY 1 ASC;
```
<img width="150" alt="image" src="https://github.com/AmandaRigdon/amandarigdon.github.io/assets/137234405/f31d4c1f-73e6-4bd2-a780-7208f580dce0">

This is a good start! It clearly shows the layoffs of each month. Now we can put this into a CTE to show a rolling total:

```sql
WITH Rolling_Total AS 
(
SELECT SUBSTRING(`date`,1,7) AS `MONTH`, SUM(total_laid_off) AS total_off 
FROM layoffs_staging2
WHERE SUBSTRING(`date`,1,7) IS NOT NULL
GROUP BY `MONTH`
ORDER BY 1 ASC
)
SELECT `MONTH`, total_off,
SUM(total_off) OVER(ORDER BY `MONTH`) AS rolling_total
FROM Rolling_Total;
```
<img width="164" alt="image" src="https://github.com/AmandaRigdon/amandarigdon.github.io/assets/137234405/0980c5bc-4ba6-4ba5-b2c2-2e25b79f3c36">

From the table above, we can see that the rolling total column is adding the previous month's layoff total as it progresses through the months.

Let's finish with making a rank of the top companies by year that laid off the most employees. 

```sql
WITH Company_Year (company, years, total_laid_off) AS
(
SELECT company, YEAR(`date`), SUM(total_laid_off)
FROM layoffs_staging2
GROUP BY company, YEAR(`date`)
), Company_Year_Rank AS 
(SELECT *, DENSE_RANK() OVER(PARTITION BY years ORDER BY total_laid_off DESC) as Ranking
FROM Company_Year
WHERE years IS NOT NULL
)
SELECT *
FROM Company_Year_Rank
WHERE Ranking <= 5;
```
<img width="229" alt="image" src="https://github.com/AmandaRigdon/amandarigdon.github.io/assets/137234405/3f46b799-03a1-4a66-b95f-245d043955dd">

Now there is a much clearer picture of which company laid off the most people each year.


