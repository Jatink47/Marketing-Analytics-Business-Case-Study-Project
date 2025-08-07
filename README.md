# Marketing-Analytics-Business-Case-Study-Project

**Project Overview**
Conducted a detailed marketing analysis for ShopEasy, an online retail platform facing declining engagement and conversions. Analyzed key KPIs such as conversion rate, engagement rate, AOV, and customer feedback to uncover drop-off points, optimize content strategy, and deliver actionable insights for improving marketing ROI and customer satisfaction.

**Goals**
• **Increase Conversion Rates:**
 • Goal: Identify factors impacting the conversion rate and provide recommendations to improve it.
 • Insight: Highlight key stages where visitors drop off and suggest improvements to optimize the conversion funnel.
 • **Enhance Customer Engagement:**
 • Goal:Determine which types of content drive the highest engagement. 
 • Insight:Analyze interaction levels with different types of marketing content to inform better content strategies.
 •**Improve Customer Feedback Scores:**
 • Goal:Understand common themes in customer reviews and provide actionable insights.
 • Insight:Identify recurring positive and negative feedback to guide product and service improvements

**Project Structure**

**Product Categorization**
 Used the Products table to categorize products based on their price into high medium lowusing SQL CASE statements.
```sql
 SELECT 
    c.CustomerID,  
    c.CustomerName, 
    c.Email,  
    c.Gender, 
    c.Age,  
    g.Country,  
    g.City  
FROM 
    dbo.customers as c  
LEFT JOIN
-- RIGHT JOIN
-- INNER JOIN
-- FULL OUTER JOIN
    dbo.geography g 
ON 
    c.GeographyID = g.GeographyID;
 ```
**Customer-Geography Merge**
 Merged Customers table with Geography using a LEFT JOIN to enrich customer records with geographical information
 ```sql
SELECT 
    ProductID, 
    ProductName,  
    Price,
    Category,

    CASE
        WHEN Price < 50 THEN 'Low'  
        WHEN Price BETWEEN 50 AND 200 THEN 'Medium'  
        ELSE 'High'  
    END AS PriceCategory  

FROM 
    dbo.products;
```
**Cleaning Customer Reviews**
 Cleaned the customer reviewstable by removing extra spaces from the review text

```sql
SELECT 
    ReviewID,  
    CustomerID,  
    ProductID, 
    ReviewDate, 
    Rating, 
    REPLACE(ReviewText, '  ', ' ') AS ReviewText
FROM 
    dbo.customer_reviews;
```
**Engagement Data Normalization**
Cleaned and normalized the Engagement_datatable to standardize engagement metrics (e.g., scaling, null handling, and formatting)
```sql
SELECT 
    EngagementID, 
    ContentID,  
	CampaignID, 
    ProductID,  
    UPPER(REPLACE(ContentType, 'Socialmedia', 'Social Media')) AS ContentType, 
    LEFT(ViewsClicksCombined, CHARINDEX('-', ViewsClicksCombined) - 1) AS Views, 
    RIGHT(ViewsClicksCombined, LEN(ViewsClicksCombined) - CHARINDEX('-', ViewsClicksCombined)) AS Clicks, 
    Likes, 
    
    FORMAT(CONVERT(DATE, EngagementDate), 'dd.MM.yyyy') AS EngagementDate  
FROM 
    dbo.engagement_data  
WHERE 
    ContentType != 'Newsletter'; 
```
**Handling Duplicates**
 In the customer journey table, afterselecting uniquerowsanotherissue sof null values in duration column and which is solved using the sub query
```sql
 WITH DuplicateRecords AS (
    SELECT 
        JourneyID, 
        CustomerID,  
        ProductID,  
        VisitDate, 
        Stage,  
        Action,  
        Duration,  
        -- Use ROW_NUMBER() to assign a unique row number to each record within the partition defined below
        ROW_NUMBER() OVER (
          
            PARTITION BY CustomerID, ProductID, VisitDate, Stage, Action  
           
            ORDER BY JourneyID  
        ) AS row_num  
    FROM 
        dbo.customer_journey 
)

-- Select all records from the CTE where row_num > 1, which indicates duplicate entries
    
SELECT *
FROM DuplicateRecords
WHERE row_num > 1  
ORDER BY JourneyID
    
SELECT 
    JourneyID,  
    CustomerID, 
    ProductID,  
    VisitDate, 
    Stage,  
    Action,  
    COALESCE(Duration, avg_duration) AS Duration 
FROM 
    (
        -- Subquery to process and clean the data
        SELECT 
            JourneyID, 
            CustomerID, 
            ProductID, 
            VisitDate, 
            UPPER(Stage) AS Stage,  
            Action, 
            Duration, 
            AVG(Duration) OVER (PARTITION BY VisitDate) AS avg_duration,  
            ROW_NUMBER() OVER (
                PARTITION BY CustomerID, ProductID, VisitDate, UPPER(Stage), Action  
                ORDER BY JourneyID  
            ) AS row_num  
        FROM 
            dbo.customer_journey  
    ) AS subquery  
WHERE 
    row_num = 1; 
    ```
**Sentiment Analysis**
 Exported customer reviews to Jupyter Notebook and performed sentiment analysis using Python libraries to classify reviews as Positive, Neutral, or Negative
```python
# pip install pandas nltk pyodbc sqlalchemy

import pandas as pd
import pyodbc
import nltk
from nltk.sentiment.vader import SentimentIntensityAnalyzer


# Download the VADER lexicon for sentiment analysis if not already present.
nltk.download('vader_lexicon')

# Define a function to fetch data from a SQL database using a SQL query
def fetch_data_from_sql():
    # Define the connection string with parameters for the database connection
    conn_str = (
        "Driver={SQL Server};"  # Specify the driver for SQL Server
        "Server=ALI-LT2024\\SQLEXPRESS;"  # Specify your SQL Server instance
        "Database=PortfolioProject_MarketingAnalytics;"  # Specify the database name
        "Trusted_Connection=yes;"  # Use Windows Authentication for the connection
    )
    # Establish the connection to the database
    conn = pyodbc.connect(conn_str)
    
    # Define the SQL query to fetch customer reviews data
    query = "SELECT ReviewID, CustomerID, ProductID, ReviewDate, Rating, ReviewText FROM fact_customer_reviews"
    
    # Execute the query and fetch the data into a DataFrame
    df = pd.read_sql(query, conn)
    
    # Close the connection to free up resources
    conn.close()
    
    # Return the fetched data as a DataFrame
    return df

# Fetch the customer reviews data from the SQL database
customer_reviews_df = fetch_data_from_sql()

# Initialize the VADER sentiment intensity analyzer for analyzing the sentiment of text data
sia = SentimentIntensityAnalyzer()

# Define a function to calculate sentiment scores using VADER
def calculate_sentiment(review):
    # Get the sentiment scores for the review text
    sentiment = sia.polarity_scores(review)
    # Return the compound score, which is a normalized score between -1 (most negative) and 1 (most positive)
    return sentiment['compound']

# Define a function to categorize sentiment using both the sentiment score and the review rating
def categorize_sentiment(score, rating):
    # Use both the text sentiment score and the numerical rating to determine sentiment category
    if score > 0.05:  # Positive sentiment score
        if rating >= 4:
            return 'Positive'  # High rating and positive sentiment
        elif rating == 3:
            return 'Mixed Positive'  # Neutral rating but positive sentiment
        else:
            return 'Mixed Negative'  # Low rating but positive sentiment
    elif score < -0.05:  # Negative sentiment score
        if rating <= 2:
            return 'Negative'  # Low rating and negative sentiment
        elif rating == 3:
            return 'Mixed Negative'  # Neutral rating but negative sentiment
        else:
            return 'Mixed Positive'  # High rating but negative sentiment
    else:  # Neutral sentiment score
        if rating >= 4:
            return 'Positive'  # High rating with neutral sentiment
        elif rating <= 2:
            return 'Negative'  # Low rating with neutral sentiment
        else:
            return 'Neutral'  # Neutral rating and neutral sentiment

# Define a function to bucket sentiment scores into text ranges
def sentiment_bucket(score):
    if score >= 0.5:
        return '0.5 to 1.0'  # Strongly positive sentiment
    elif 0.0 <= score < 0.5:
        return '0.0 to 0.49'  # Mildly positive sentiment
    elif -0.5 <= score < 0.0:
        return '-0.49 to 0.0'  # Mildly negative sentiment
    else:
        return '-1.0 to -0.5'  # Strongly negative sentiment

# Apply sentiment analysis to calculate sentiment scores for each review
customer_reviews_df['SentimentScore'] = customer_reviews_df['ReviewText'].apply(calculate_sentiment)

# Apply sentiment categorization using both text and rating
customer_reviews_df['SentimentCategory'] = customer_reviews_df.apply(
    lambda row: categorize_sentiment(row['SentimentScore'], row['Rating']), axis=1)

# Apply sentiment bucketing to categorize scores into defined ranges
customer_reviews_df['SentimentBucket'] = customer_reviews_df['SentimentScore'].apply(sentiment_bucket)

# Display the first few rows of the DataFrame with sentiment scores, categories, and buckets
print(customer_reviews_df.head())

# Save the DataFrame with sentiment scores, categories, and buckets to a new CSV file
customer_reviews_df.to_csv('fact_customer_reviews_with_sentiment.csv', index=False)
```

**Data Visualization**
Identified key issues including fluctuating conversion rates, declining social media engagement, and subpar customer satisfaction—each of which is explored in detail below.

![image alt]()
 **Decreased Conversion Rates:** The conversion rate demonstrated a strong rebound in December, reaching 10.2%, despite a notable dip to 5.0% in October.
**Reduced Customer Engagement:** There is a decline in overall social media engagement, with views dropping throughout the year.While clicks and likes are low compared to views, the click-through rate stands at 15.37%, meaning that engaged users are still interacting effectively
**Customer Feedback Analysis:** Customer ratings have remained consistent, averaging around 3.7 throughout the year.Although stable, the average rating is below the target of 4.0, suggesting a need for focusedimprovements in customer satisfaction, for products below 3,5.

1.  **Decreased Conversion Rates:**

Seasonal Peaks: February, July, and especially January (18.5%) showed strong conversion rates, with Ski Boots reaching 150%.

Lowest Month: May had the lowest conversion (4.3%), indicating the need for improved marketing strategies during this period.

Insight: Seasonal demand and targeted campaigns significantly influence conversion performance.

![image alt]()

2.**Reduced Customer Engagement:**
Declining Views: Engagement dropped steadily after July, following peak activity in February and July.

Low Interaction: Clicks and likes stayed low despite high views, highlighting a gap in content effectiveness.

Content Performance: Blogs performed best—especially in April and July—while videos and social media posts saw moderate engagement.

![image alt]()

3. **Customer Feedback Analysis:**
Rating Distribution: Most reviews are positive, with the majority at 4 and 5 stars; low ratings form a small portion.

Sentiment Analysis: Positive sentiment dominates (275 reviews), though 82 negative and several mixed reviews indicate areas for improvement.

Improvement Opportunity: Addressing mixed feedback can help turn neutral experiences into positive ones, boosting overall satisfaction.

![iamge alt]()

**Goals & Actions Summary:**

**Increase Conversion Rates:** Analyzed drop-off points in the conversion funnel and recommended focusing on high-performing products (e.g., Kayaks, Ski Boots) with seasonal campaigns to boost sales.

**Enhance Customer Engagement:** Suggested revamping the content strategy with interactive formats and stronger CTAs, especially during low-engagement months (Sep–Dec).

**Improve Customer Feedback Scores:** Proposed a feedback loop to address mixed/negative reviews, with targeted follow-ups to improve satisfaction and push average ratings toward the 4.0 goal.

