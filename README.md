# JEWELRY_AMAZON Data Analytics Project

## Overview

This data analytics project encompasses the entire ETL process, incorporating Python, Excel, SQL, and Tableau. The key steps involved in the project include:

1. **Extraction:** Utilizing Python libraries, such as BeautifulSoup, to extract product data from Amazon.

2. **Transformation and Cleaning:** Employing SQL and Excel for data cleaning and transformation processes.

3. **Consolidation and Loading:** Loading the cleaned data into Tableau for comprehensive analysis and visualization.

## Extraction

### Amazon Scraper Script

The `amazon_scraper.py` script serves the purpose of scraping vital product information from Amazon. The extracted variables include:

- **Product Title**
- **Rating**
- **Reviews**
- **Price**
- **Brand/Store**
- **Availability**
```python
# from concurrent.futures import ThreadPoolExecutor
from bs4 import BeautifulSoup
import requests
import pandas as pd

def get_title(new_soup):
    try:
        title = new_soup.find("span", attrs={"id": 'productTitle'})
        title_value = title.text
        title_string = title_value.strip()
    except AttributeError:
        title_string = None
    return title_string

def get_price(new_soup):
    try:
        price_element = new_soup.find("span", attrs={'class': 'a-price-whole'})
        price = price_element.text.strip()
    except AttributeError:
        price = None
    return price

def get_rating(new_soup):
    try:
        rating = new_soup.find("span", attrs={'class': 'a-size-base a-color-base'})
        rating_value = rating.text.strip()
    except AttributeError:
        rating_value = None
    return rating_value

def get_no_rating(new_soup):
    try:
        no_rating = new_soup.find("span", attrs={'id': 'acrCustomerReviewText'})
        no_rating_value = no_rating.text.strip()
    except AttributeError:
        no_rating_value = None
    return no_rating_value

def get_available(new_soup):
    try:
        available = new_soup.find("span", attrs={'class': 'a-size-medium a-color-success'})
        available_value = available.text.strip()
    except AttributeError:
        available_value = None
    return available_value

def get_brand(new_soup):
    try:
        brand = new_soup.find("a", attrs={'id': 'bylineInfo'})
        brand_value = brand.text.strip()
    except AttributeError:
        brand_value = None
    return brand_value

def scrape_product_data(link):
    try:
        HEADERS = {
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
            'Accept-Language': 'en-US, en;q=0.5'
        }

        new_webpage = requests.get("https://www.amazon.sa" + link, headers=HEADERS)
        new_soup = BeautifulSoup(new_webpage.content, "html.parser")

        title = get_title(new_soup)
        price = get_price(new_soup)
        rating = get_rating(new_soup)
        no_rating = get_no_rating(new_soup)
        available = get_available(new_soup)
        brand = get_brand(new_soup)

        return {"title": title, "price": price, "rating": rating, "reviews": no_rating, "availability": available, "brand": brand}

    except Exception as e:
        print(f"Error processing link {link}: {e}")
        return None

if __name__ == '__main__':
    # List of URLs to scrape
    urls = [
        'https://www.amazon.ae/s?i=fashion&rh=n%3A11995892031&fs=true&page=63',
        'https://www.amazon.ae/s?i=fashion&rh=n%3A11995892031&fs=true&page=64',
        # Add other URLs here
    ]

    links_list = []

    with ThreadPoolExecutor(max_workers=5) as executor:
        # Extract links from all URLs
        for url in urls:
            webpage = requests.get(url, headers={'User-Agent': 'Mozilla/5.0'})
            soup = BeautifulSoup(webpage.content, 'html.parser')
            links = soup.find_all('a', attrs={'class': 'a-link-normal s-underline-text s-underline-link-text s-link-style a-text-normal'})
            links_list.extend([link.get('href') for link in links])

        # ThreadPoolExecutor to scrape data concurrently
        scraped_data = list(executor.map(scrape_product_data, links_list))

    # Remove None values from the list
    scraped_data = [data for data in scraped_data if data is not None]

    # Create DataFrame from the scraped data
    amazon_df = pd.DataFrame(scraped_data)

    # Drop rows with missing titles
    amazon_df.dropna(subset=['title'], inplace=True)

    # Save DataFrame to CSV
    amazon_df.to_csv("amazon_data.csv", sep=';', header=True, index=False, encoding='utf-8')
```
## Transform
### PostgreSQL Script for Data Cleaning and Analysis

#### Overview
In this phase, PostgreSQL scripts were utilized for data cleaning and analysis. The key focus was on three primary files: `products`, `products_ae`, and `fixtures`. The `products` and `products_ae` files contain product data, while `fixtures` comprises Amazon UAE reviews.

**Uploaded Files:**
1. `products`: Amazon KSA product data file.
2. `products_ae`: Amazon UAE product data.
3. `fixtures`: Amazon UAE reviews (for products_ae).

During the process, a specific SQL script was employed to address the accidental deletion of a column from `products_ae`. The script involves joining tables to combine review columns for comprehensive analysis of `products_ae`.

#### 1) Update & Cleaning:
```sql
-- Remove unwanted words/text
UPDATE products_ae
SET reviews = UPPER(TRIM(REPLACE(reviews, 'S', '')));

-- Segregate title based on category type
ALTER TABLE products_ae
ADD COLUMN categories VARCHAR(255));

UPDATE products_ae
SET categories =
    CASE
        WHEN LOWER(title) LIKE '%bracelet%' THEN 'Bracelets'
        WHEN LOWER(title) LIKE '%necklace%' THEN 'Necklaces'
        WHEN LOWER(title) LIKE '%brooche%' THEN 'Brooches'
        WHEN LOWER(title) LIKE '%earring%' THEN 'Earrings'
        WHEN LOWER(title) LIKE '%bracelet%' THEN 'Bracelet'
        WHEN LOWER(title) LIKE '%charm%' THEN 'Charms'
        WHEN LOWER(title) LIKE '%ring%' THEN 'Rings'
        ELSE 'others'
    END;
```
#### 1) Exploratory Data Analysis:
Category Analysis:
```sql

-- Show cats with average rating, average product price, cat count, and number of reviews
SELECT
    categories,
    ROUND(AVG(rating), 1) AS average_rating,
    SUM(reviews) AS number_of_reviews,
    ROUND(AVG(price), 0) AS avg_product_price,
    COUNT(categories) AS cat_count
FROM
    cat_rating
WHERE
    rating IS NOT NULL
GROUP BY
    categories
ORDER BY
    number_of_reviews DESC;

-- Check category distribution between UAE and KSA
SELECT
    categories,
    SUM(CASE WHEN source = 'ae' THEN count ELSE 0 END) AS count_ae,
    SUM(CASE WHEN source = 'sa' THEN count ELSE 0 END) AS count_sa
FROM (
         SELECT categories, COUNT(*) AS count, 'ae' AS source
         FROM products_ae
         GROUP BY categories

         UNION ALL

         SELECT categories, COUNT(*) AS count, 'sa' AS source
         FROM products
         GROUP BY categories
     ) AS subquery
GROUP BY
    categories;
```
Word Analysis:
```sql
-- Show words with average high rating and highest word occurrence
SELECT
    word,
    ROUND(AVG(rating), 1) AS average_rating,
    COUNT(word) AS word_count
FROM
    temp_words
WHERE
    rating IS NOT NULL
GROUP BY
    word
HAVING
    AVG(rating) >= 4.5
ORDER BY
    word_count DESC,
    average_rating;

-- Fetch the top 20 most common words found in the product titles.
SELECT
    word,
    COUNT(word) AS word_count
FROM
    temp_words
GROUP BY
    word
ORDER BY
    word_count DESC
LIMIT 20;

-- Check material distribution
SELECT
    word,
    COUNT(word) AS word_count
FROM
    temp_words
WHERE
    word IN ('bronze', 'gold', 'silver')
GROUP BY
    word
ORDER BY
    word_count DESC;

-- Check gender segment
SELECT
    word,
    COUNT(word) AS word_count
FROM
    temp_words
WHERE
    word IN ('men', 'women')
GROUP BY
    word
ORDER BY
    word_count DESC;

-- Check category distribution
SELECT
    word,
    COUNT(word) AS word_count
FROM
    temp_words
WHERE
    word IN ('ring', 'bracelet', 'necklace', 'brooch', 'earrings', 'charm')
GROUP BY
    word
ORDER BY
    word_count DESC;
```
Brand Analysis:

```sql
-- Check brand summary
SELECT
    brand,
    ROUND(AVG(rating), 1),
    SUM(reviews),
    COUNT(brand),
    ROUND(AVG(price), 1)
FROM
    products
WHERE
    brand <> 'Generic'
GROUP BY
    brand
ORDER BY
    COUNT DESC;

-- Fetch distinct brands with reviews, listings, average rating, and average price
SELECT
    DISTINCT brand,
    SUM(reviews) AS reviews,
    COUNT(brand) AS listings,
    ROUND(AVG(rating), 1) AS avg_rating,
    ROUND(AVG(price), 1) AS avg_price
FROM
    products
WHERE
    brand <> 'Generic'
GROUP BY
    brand
ORDER BY
    reviews DESC,
    avg_price
LIMIT 10;

-- Compare UAE and KSA brand reviews
SELECT
    sa.brand AS KSA_brands,
    SUM(sa.reviews) AS ksa_reviews,
    ROUND(AVG(sa.rating), 1) AS ksa_rating,
    ae.brand AS UAE_brands,
    ae.reviews,
    ROUND(AVG(uae.rating)::numeric, 1) AS uae_rating
FROM
    products AS sa
INNER JOIN ae_reviews AS ae ON sa.brand = ae.brand
INNER JOIN products_ae AS uae ON uae.brand = sa.brand
WHERE
    ae.reviews >= 100
GROUP BY
    sa.brand,
    ae.brand,
    ae.reviews
HAVING
    SUM(sa.reviews) <> 0
    AND AVG(sa.rating) >= 4.5
    AND AVG(uae.rating) >= 4.5
ORDER BY
    ksa_reviews DESC;
```
Additional Analysis:
```sql
-- Compare brand presence across the two markets
SELECT
    brand,
    SUM(CASE WHEN source = 'ae' THEN count ELSE 0 END) AS COUNTAE,
    SUM(CASE WHEN source = 'sa' THEN count ELSE 0 END) AS COUNTSA
FROM (
         SELECT
             brand,
             COUNT(*) AS count,
             'ae' AS source
         FROM products_ae
         GROUP BY brand

         UNION ALL

         SELECT
             brand,
             COUNT(*) AS count,
             'sa' AS source
         FROM products
         GROUP BY brand
     ) AS subquery

GROUP BY
    brand

HAVING
    SUM(CASE WHEN source = 'ae' THEN count ELSE 0 END) >= 10
    AND SUM(CASE WHEN source = 'sa' THEN count ELSE 0 END) >= 10

ORDER BY
    COUNTSA DESC;
```

#### 2) Data Joining and Integration:
Data Joining:
```sql
-- Inner join reviews and product title
SELECT
    products_ae.title,
    fixtures.reviews,
    products_ae.id,
    products_ae.reviews
FROM
    products_ae
RIGHT JOIN
    fixtures ON id = re_id
ORDER BY
    fixtures.re_id DESC;

-- Inner join brand reviews
SELECT
    DISTINCT (products_ae.brand),
    SUM(fixtures.reviews) AS reviews
FROM
    products_ae
LEFT JOIN fixtures ON id = re_id
WHERE
    brand IS NOT NULL
GROUP BY
    brand
ORDER BY
    reviews DESC;

-- Fix brand names
SELECT
    DISTINCT (brand)
FROM
    products_ae
WHERE
    brand LIKE 'YU%'
    AND brand LIKE '%B%';
```
Temp Table and Comparison:
```sql
-- Make a temporary table to join with SA products
CREATE TEMPORARY TABLE ae_reviews AS
SELECT
    products_ae.brand,
    SUM(fixtures.reviews) AS reviews
FROM
    products_ae
LEFT JOIN fixtures ON id = re_id
WHERE
    brand IS NOT NULL
GROUP BY
    brand
ORDER BY
    reviews DESC;

-- Compare UAE and KSA brand reviews
SELECT
    sa.brand AS KSA_brands,
    SUM(sa.reviews) AS ksa_reviews,
    ROUND(AVG(sa.rating), 1) AS ksa_rating,
    ae.brand AS UAE_brands,
    ae.reviews,
    ROUND(AVG(uae.rating)::numeric, 1) AS uae_rating
FROM
    products AS sa
INNER JOIN ae_reviews AS ae ON sa.brand = ae.brand
INNER JOIN products_ae AS uae ON uae.brand = sa.brand
WHERE
    ae.reviews >= 100
GROUP BY
    sa.brand,
    ae.brand,
    ae.reviews
HAVING
    SUM(sa.reviews) <> 0
    AND AVG(sa.rating) >= 4.5
    AND AVG(uae.rating) >= 4.5
ORDER BY
    ksa_reviews DESC;
```









