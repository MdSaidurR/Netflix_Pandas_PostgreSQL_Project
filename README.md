# Netflix_Pandas_PostgreSQL_Project

## Overview
This demo project involves a comprehensive analysis of Netflix movies and TV shows using Pandas and PostgreSQL. The goal is to extract valuable insights and answer key business questions based on the dataset. Pandas is used to prepare and clean the dataset, and then the cleaned data is loaded into PostgreSQL for further analysis.

## Objectives

- Analyze the distribution of content types (Movies vs. TV Shows).
- Identify the most frequently assigned ratings across different content categories.
- Examine content trends by release year, country, and duration.
- Explore and categorize content using specific criteria, filters, and keywords to uncover deeper insights.

##  Data Cleaning & Preparation (Pandas)

- Loaded and explored the dataset using:
  - `df.head()`
  - `df.info()`
  - `df.isnull().sum()`

- Replaced missing values:
  - `director` → `"Unknown"`
  - `cast` → `"Not Listed"`
  - `country` → `"Unknown"`

- Removed rows with missing `date_added` to maintain accurate time-based analysis.

- Filled missing values in:
  - `rating` using the mode (most frequent value)
  - `duration` with `"Not Mention"`

- Renamed columns for clarity:
  - `"cast"` → `"casts"`
  - `"type"` → `"type_of_show"`

**Result:** A clean, consistent, and analysis-ready dataset ready for further exploration and visualization.

## Load Data into PostgreSQL

1. **Connect to PostgreSQL** using SQLAlchemy:
   ```python
from sqlalchemy import create_engine
username = "postgres"
password = "******"
host = "localhost"
port = "5432"
database = "netflix_db"
engine = create_engine(f"postgresql+psycopg2://{username}:{password}@{host}:{port}/{database}")
table_name = "movie_show"         
```
## Business Problems and Solutions

### 1. Count the number of Movies vs TV Shows
```sql
SELECT 
    type_of_show,
    COUNT(type_of_show) AS Number
FROM movie_show
GROUP BY type_of_show;
```

### 2. Find the most common rating for movies and TV shows

```sql
SELECT type_of_show, rating, rating_count
FROM (
    SELECT 
        type_of_show,
        rating,
        COUNT(*) AS rating_count,
        ROW_NUMBER() OVER (
            PARTITION BY type_of_show 
            ORDER BY COUNT(*) DESC
        ) AS rn
    FROM movie_show
    GROUP BY type_of_show, rating
) t
WHERE rn = 1;
```

### 3. List all movies released in a specific year (e.g., 2020)

```sql
SELECT *
FROM movie_show
WHERE release_year = 2020
  AND type_of_show = 'Movie';
```

### 4. Find the top 5 countries with the most content on Netflix

```sql
SELECT c AS country, COUNT(*) AS movie_number
FROM movie_show,
LATERAL unnest(string_to_array(country, ',')) AS c
GROUP BY c
ORDER BY movie_number DESC
LIMIT 5;
```

### 5. Identify the longest movie

```sql
(
  SELECT type_of_show, title, duration
  FROM movie_show
  WHERE duration LIKE '%min%'
  ORDER BY SPLIT_PART(duration, ' ', 1)::INT DESC
  LIMIT 1
)

UNION ALL

(
  SELECT type_of_show, title, duration
  FROM movie_show
  WHERE duration LIKE '%Season%'
  ORDER BY SPLIT_PART(duration, ' ', 1)::INT DESC
  LIMIT 1
);
```

### 6. Find content added in the last 5 years

```sql
SELECT *
FROM movie_show
WHERE TO_DATE(date_added, 'Month DD, YYYY') >= CURRENT_DATE - INTERVAL '5 years';
```

### 7. Find all the movies/TV shows by director 'Rajiv Chilaka'!

```sql
SELECT *
FROM (
 SELECT *,
 TRIM(UNNEST(STRING_TO_ARRAY(director, ','))) AS director_name
 FROM movie_show
) AS d
WHERE director_name = 'Rajiv Chilaka';
```

### 8. List all TV shows with more than 5 seasons

```sql
  SELECT *
  FROM movie_show
  WHERE type_of_show = 'TV Show'
  AND SPLIT_PART(duration, ' ', 1)::INT >5
  ORDER BY SPLIT_PART(duration, ' ', 1)::INT DESC;
```
### 9. Count the number of content items in each genre

```sql
SELECT genre, COUNT(*) AS total
FROM (
    SELECT TRIM(UNNEST(STRING_TO_ARRAY(listed_in, ','))) AS genre
    FROM movie_show
) AS g
GROUP BY genre
ORDER BY total DESC;
```
### 10. Find each year and the average numbers of content release by India on netflix. Return top 5 year with highest avg content release !

```sql
SELECT 
	country,
	release_year,
	COUNT(show_id) as total_release,
	ROUND(
		COUNT(show_id)::numeric/
		(SELECT COUNT(show_id) FROM movie_show WHERE country = 'India')::numeric * 100 
		,2
	) as avg_release
FROM movie_show
WHERE country = 'India' 
GROUP BY country, 2
ORDER BY avg_release DESC 
LIMIT 5;
```

### 11. List all movies that are documentaries

```sql
SELECT *
FROM movie_show
WHERE listed_in ILIKE 'documentaries';
```

### 12. Find all content without a director OR unknown

```sql
SELECT *
FROM movie_show
WHERE director IS NULL OR director = 'Unknown';
```

### 13. Find how many movies actor 'Salman Khan' appeared in last 20 years!

```sql
SELECT * 
FROM movie_show
WHERE casts ILIKE '%Salman Khan%'
AND release_year >= EXTRACT(YEAR FROM CURRENT_DATE) - 20;
```

### 14. Find the top 10 actors who have appeared in the highest number of movies produced in India.

```sql
SELECT 
	TRIM(UNNEST(STRING_TO_ARRAY(casts, ','))) as actor,
	COUNT(*) AS Total
FROM movie_show
WHERE country = 'India'
GROUP BY 1
ORDER BY 2 DESC
LIMIT 10;
```

### 15.Question  
Categorize the content based on the presence of the keywords 'kill' and 'violence' in 
the description field. Label content containing these keywords as 'Bad' and all other 
content as 'Good'. Count how many items fall into each category.


```sql
SELECT 
    category,
	type_of_show,
    COUNT(*) AS content_count
FROM (
    SELECT 
		*,
        CASE 
            WHEN description ILIKE '%kill%' OR description ILIKE '%violence%' THEN 'Bad'
            ELSE 'Good'
        END AS category
    FROM movie_show
) AS c
GROUP BY 1,2
ORDER BY 2
```
