# SQL_IMDB_DATA_ANALYSIS
This project analyzes the IMDB movie dataset to reveal movie trends, audience preferences, and rating patterns. It delivers actionable insights to identify top-rated movies, genre leaders, and optimize data-driven decisions in the entertainment industry.


# DATA CLEANING(MySQL Workbench)

# movies_messy (table-1)

CREATE TABLE movies
LIKE movies_messy;

INSERT INTO movies
SELECT * 
FROM movies_messy;

SELECT * 
FROM movies;

-- DATA CLEANING

-- 1. DUPLICATES

-- Detect movieId duplicates

SELECT movie_Id, COUNT(*) AS cnt
FROM movies
GROUP BY movie_Id
HAVING cnt > 1
ORDER BY cnt DESC;

-- 2. MISSING VALUES (NULLs & empty strings)

-- Detect release_date missing values (NULLs & empty strings)

SELECT *
FROM movies
WHERE (release_date IS NULL OR release_date = '');

-- REMOVING MISSING VALUES (NULLs & empty strings)

DELETE 
FROM movies
WHERE (release_date IS NULL OR release_date = '');

SET SQL_SAFE_UPDATES = 0;   
-- SET SQL_SAFE_UPDATES = 1;    

-- 3. INCONSISTENT TEXT & TYPOS

-- Inspect variants of titles/genres/languauge

SELECT DISTINCT title FROM movies ORDER BY title LIMIT 200;
SELECT DISTINCT genre FROM movies ORDER BY genre LIMIT 200;
SELECT DISTINCT language FROM movies ORDER BY language LIMIT 200;

-- REMOVING INCONSISTENT TEXT & TYPOS

UPDATE movies
SET title = TRIM(title),
    genre = LOWER(TRIM(genre)), 
    language = LOWER(TRIM(language)),
    duration = CONCAT(TRIM(REPLACE(duration, 'min', '')), ' min');
    
-- SELECT DISTINCT 
--        CASE 
--            WHEN genre = 'drma' THEN 'drama'
--            ELSE genre
--        END AS genre
-- FROM movies;

UPDATE movies
SET genre = 'drama'
WHERE genre = 'drma';
SELECT DISTINCT genre FROM movies;

-- SET SQL_SAFE_UPDATES = 0;  
-- SET SQL_SAFE_UPDATES = 1; 

	
-- 4. INCORRECT DATA TYPES & FORMAT ISSUES

-- Detect non-date release_date strings (expected YYYY-MM-DD)

SELECT movie_Id, release_date
FROM movies
WHERE release_date IS NOT NULL
AND STR_TO_DATE(release_date, '%Y-%m-%d') IS NULL;
  
  -- REMOVING INCORRECT DATA TYPES & FORMAT ISSUES
  
UPDATE movies
SET release_date = CASE
  
  -- yyyy-mm-dd
  WHEN release_date REGEXP '^[0-9]{4}-[0-9]{2}-[0-9]{2}$'
    THEN DATE_FORMAT(STR_TO_DATE(release_date, '%Y-%m-%d'), '%Y-%m-%d')

  -- yyyy/mm/dd
  WHEN release_date REGEXP '^[0-9]{4}/[0-9]{2}/[0-9]{2}$'
    THEN DATE_FORMAT(STR_TO_DATE(release_date, '%Y/%m/%d'), '%Y-%m-%d')

  -- dd-mm-yyyy
  WHEN release_date REGEXP '^[0-9]{2}-[0-9]{2}-[0-9]{4}$'
    THEN DATE_FORMAT(STR_TO_DATE(release_date, '%d-%m-%Y'), '%Y-%m-%d')

  -- dd/mm/yyyy (day > 12 means definitely day first)
  WHEN release_date REGEXP '^[0-9]{2}/[0-9]{2}/[0-9]{4}$'
   AND CAST(SUBSTRING_INDEX(release_date, '/', 1) AS UNSIGNED) > 12
    THEN DATE_FORMAT(STR_TO_DATE(release_date, '%d/%m/%Y'), '%Y-%m-%d')

  -- mm/dd/yyyy (if first number <= 12, treat as month)
  WHEN release_date REGEXP '^[0-9]{2}/[0-9]{2}/[0-9]{4}$'
   AND CAST(SUBSTRING_INDEX(release_date, '/', 1) AS UNSIGNED) <= 12
    THEN DATE_FORMAT(STR_TO_DATE(release_date, '%m/%d/%Y'), '%Y-%m-%d')

  -- Full month name with comma, e.g. March 05, 2018 (allow spaces and letters)
  WHEN release_date REGEXP '^[A-Za-z]+ [0-9]{1,2}, [0-9]{4}$'
    THEN DATE_FORMAT(STR_TO_DATE(release_date, '%M %d, %Y'), '%Y-%m-%d')

  -- dd-Mon-yyyy (e.g. 28-Aug-1975)
  WHEN release_date REGEXP '^[0-9]{1,2}-[A-Za-z]{3}-[0-9]{4}$'
    THEN DATE_FORMAT(STR_TO_DATE(release_date, '%d-%b-%Y'), '%Y-%m-%d')

  ELSE release_date
END
WHERE release_date IS NOT NULL
  AND TRIM(release_date) <> '';
  
  SELECT * From movies;
  

# ratings_messy (table-2)

CREATE TABLE ratings
LIKE ratings_messy;

INSERT INTO ratings
SELECT * 
FROM ratings_messy;

SELECT * 
FROM ratings;

-- DATA CLEANING

-- 1. DUPLICATES

-- Detect ratingId, movie_id, user_name duplicates

SELECT rating_Id, COUNT(*) AS cnt
FROM ratings
GROUP BY rating_Id
HAVING cnt > 1
ORDER BY cnt DESC;

SELECT movie_id, user_name, COUNT(*) AS cnt
FROM ratings
GROUP BY movie_id, user_name
HAVING cnt > 1;

-- 2. MISSING VALUES (NULLs & empty strings)

-- Detect review missing values (NULLs & empty strings)

SELECT *
FROM ratings
WHERE (review IS NULL OR review = '');

-- REMOVING MISSING VALUES (NULLs & empty strings)

DELETE 
FROM ratings
WHERE (review IS NULL OR review = '');

SET SQL_SAFE_UPDATES = 0;   
SET SQL_SAFE_UPDATES = 1;    

-- 3. INCONSISTENT TEXT & TYPOS

-- Inspect variants of ratings

SELECT DISTINCT review FROM ratings ORDER BY review LIMIT 200;
SELECT DISTINCT user_name FROM ratings ORDER BY user_name LIMIT 200;

-- REMOVING INCONSISTENT TEXT & TYPOS

UPDATE ratings
SET review = TRIM(review),
    user_name = REPLACE(user_name, '@', '');
    
-- 4. INCORRECT DATA TYPES & FORMAT ISSUES

-- Detect non-numeric rating values

SELECT *
FROM ratings
WHERE rating IS NOT NULL
  AND rating NOT REGEXP '^[0-9]+(\\.[0-9]+)?$';

  -- REMOVING INCORRECT DATA TYPES & FORMAT ISSUES
  
 UPDATE ratings_messy
SET rating = NULL
WHERE rating NOT REGEXP '^[0-9]+(\\.[0-9]+)?$';

UPDATE ratings_messy
SET rating = CAST(rating AS DECIMAL(3,1))
WHERE rating REGEXP '^[0-9]+(\\.[0-9]+)?$';

ALTER TABLE ratings_messy MODIFY COLUMN rating DECIMAL(3,1);

-- 5. OUTLIERS & SUSPICIOUS VALUES

-- Detect ratings outside expected bounds (e.g., 0.5-5.0)

SELECT *
FROM ratings
WHERE rating < 0.5 OR rating > 5;

-- 6. REFERENTIAL INTEGRITY (ratings → movies)

-- Ratings referencing non-existent movies:

SELECT r.*
FROM ratings AS r
LEFT JOIN movies m ON r.movie_Id = m.movie_Id
WHERE m.movie_Id IS NULL;

-- Delete orphan ratings

DELETE r
FROM ratings r
LEFT JOIN movies m ON r.movie_id = m.movie_id
WHERE m.movie_id IS NULL;

-- Add foreign key (only after cleaning and ensuring all movieIds exist and primary key present)

ALTER TABLE movies
ADD CONSTRAINT pk_movies PRIMARY KEY (movie_Id);

ALTER TABLE ratings
ADD CONSTRAINT fk_ratings_movies
FOREIGN KEY (movie_Id) REFERENCES movies(movie_Id)
ON DELETE SET NULL
ON UPDATE CASCADE;

SELECT * FROM ratings;


# My Presentation

Click below to view/download the presentation:

[View Presentation](./PPT_IMDB_Movie_Data_Analysis%20.pdf)
