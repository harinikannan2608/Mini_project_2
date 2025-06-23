# streamlit_app.py

import streamlit as st
import pandas as pd
import sqlite3

# Connect to the SQLite DB
conn = sqlite3.connect("nutrition_paradox.db")

# Function to run queries
def run_query(query):
    return pd.read_sql_query(query, conn)

# Categorized query dictionary
obesity_queries = {
    "Top 5 Regions with Highest Obesity (2022)": """
        SELECT Region, AVG(Mean_Estimate) as Avg_Obesity
        FROM obesity
        WHERE Year = 2022
        GROUP BY Region
        ORDER BY Avg_Obesity DESC
        LIMIT 5;
    """,
    "Top 5 Countries with Highest Obesity": """
        SELECT Country, AVG(Mean_Estimate) as Avg_Obesity
        FROM obesity
        GROUP BY Country
        ORDER BY Avg_Obesity DESC
        LIMIT 5;
    """,
    "Obesity Trend in India": """
        SELECT Year, AVG(Mean_Estimate) as Obesity_Trend
        FROM obesity
        WHERE Country = 'India'
        GROUP BY Year
        ORDER BY Year;
    """,
    "Average Obesity by Gender": """
        SELECT Gender, AVG(Mean_Estimate) as Avg_Obesity
        FROM obesity
        GROUP BY Gender;
    """,
    "Obesity Count by Level and Age Group": """
        SELECT Obesity_Level, Age_Group, COUNT(*) as Count
        FROM obesity
        GROUP BY Obesity_Level, Age_Group;
    """,
    "Top 5 Least and Most Reliable Countries by CI_Width": """
        WITH
        LeastReliable AS (
            SELECT Country, AVG(CI_Width) AS Avg_CI
            FROM obesity
            GROUP BY Country
            ORDER BY Avg_CI DESC
            LIMIT 5
        ),
        MostConsistent AS (
            SELECT Country, AVG(CI_Width) AS Avg_CI
            FROM obesity
            GROUP BY Country
            ORDER BY Avg_CI ASC
            LIMIT 5
        )
        SELECT
            lr.Country AS Least_Reliable_Country,
            lr.Avg_CI AS Least_Reliable_CI,
            mc.Country AS Most_Consistent_Country,
            mc.Avg_CI AS Most_Consistent_CI
        FROM
            (SELECT ROW_NUMBER() OVER () AS rn, * FROM LeastReliable) lr
        JOIN
            (SELECT ROW_NUMBER() OVER () AS rn, * FROM MostConsistent) mc
        ON lr.rn = mc.rn;
    """,
    "Average Obesity by Age Group": """
        SELECT Age_Group, AVG(Mean_Estimate) as Avg_Obesity
        FROM obesity
        GROUP BY Age_Group;
    """,
    "Top 10 Consistent Low Obesity Countries": """
        SELECT Country, AVG(Mean_Estimate) as Avg_Obesity, AVG(CI_Width) as Avg_CI
        FROM obesity
        GROUP BY Country
        HAVING Avg_Obesity < 25 AND Avg_CI < 5
        ORDER BY Avg_Obesity ASC
        LIMIT 10;
    """,
    "Countries where Female Obesity Exceeds Male": """
        SELECT o1.Year, o1.Country, o1.Mean_Estimate as Female, o2.Mean_Estimate as Male,
               (o1.Mean_Estimate - o2.Mean_Estimate) as Difference
        FROM obesity o1
        JOIN obesity o2 ON o1.Country = o2.Country AND o1.Year = o2.Year
        WHERE o1.Gender = 'Female' AND o2.Gender = 'Male' AND (o1.Mean_Estimate - o2.Mean_Estimate) > 5
        ORDER BY Difference DESC;
    """,
    "Global Obesity Trend by Year": """
        SELECT Year, AVG(Mean_Estimate) as Global_Avg_Obesity
        FROM obesity
        GROUP BY Year
        ORDER BY Year;
    """
}

malnutrition_queries = {
    "Average Malnutrition by Age Group": """
        SELECT Age_Group, AVG(Mean_Estimate) as Avg_Malnutrition
        FROM malnutrition
        GROUP BY Age_Group;
    """,
    "Top 5 Countries with Highest Malnutrition": """
        SELECT Country, AVG(Mean_Estimate) as Avg_Malnutrition
        FROM malnutrition
        GROUP BY Country
        ORDER BY Avg_Malnutrition DESC
        LIMIT 5;
    """,
    "Malnutrition Trend in Africa": """
        SELECT Year, AVG(Mean_Estimate) as Malnutrition_Trend
        FROM malnutrition
        WHERE Region = 'Africa'
        GROUP BY Year
        ORDER BY Year;
    """,
    "Average Malnutrition by Gender": """
        SELECT Gender, AVG(Mean_Estimate) as Avg_Malnutrition
        FROM malnutrition
        GROUP BY Gender;
    """,
    "Malnutrition Level-wise Avg CI_Width by Age Group": """
        SELECT Malnutrition_Level, Age_Group, AVG(CI_Width) as Avg_CI
        FROM malnutrition
        GROUP BY Malnutrition_Level, Age_Group;
    """,
    "Malnutrition Change in India, Nigeria, Brazil": """
        SELECT Country, Year, AVG(Mean_Estimate) as Avg
        FROM malnutrition
        WHERE Country IN ('India', 'Nigeria', 'Brazil')
        GROUP BY Country, Year
        ORDER BY Country, Year;
    """,
    "Regions with Lowest Malnutrition": """
        SELECT Region, AVG(Mean_Estimate) as Avg_Malnutrition
        FROM malnutrition
        GROUP BY Region
        ORDER BY Avg_Malnutrition ASC
        LIMIT 5;
    """,
    "Countries with Increasing Malnutrition": """
        SELECT Country, MAX(Mean_Estimate) - MIN(Mean_Estimate) AS Change
        FROM malnutrition
        GROUP BY Country
        HAVING Change > 0
        ORDER BY Change DESC;
    """,
    "Min/Max Malnutrition by Year": """
        SELECT Year, MIN(Mean_Estimate) as Min_Malnutrition, MAX(Mean_Estimate) as Max_Malnutrition
        FROM malnutrition
        GROUP BY Year
        ORDER BY Year;
    """,
    "High CI_Width Flags (CI > 5)": """
        SELECT Country, Year, CI_Width
        FROM malnutrition
        WHERE CI_Width > 5
        ORDER BY CI_Width DESC;
    """
}

combined_queries = {
    "Obesity vs Malnutrition (5 Countries)": """
        SELECT o.Country, AVG(o.Mean_Estimate) as Obesity, AVG(m.Mean_Estimate) as Malnutrition
        FROM obesity o
        JOIN malnutrition m ON o.Country = m.Country
        GROUP BY o.Country
        LIMIT 5;
    """,
    "Gender Disparity in Obesity & Malnutrition": """
        SELECT o.Gender, AVG(o.Mean_Estimate) as Obesity, AVG(m.Mean_Estimate) as Malnutrition
        FROM obesity o
        JOIN malnutrition m ON o.Gender = m.Gender
        GROUP BY o.Gender;
    """,
    "Region-wise Comparison: Africa vs Americas": """
        SELECT o.Region, AVG(o.Mean_Estimate) as Avg_Obesity, AVG(m.Mean_Estimate) as Avg_Malnutrition
        FROM obesity o
        JOIN malnutrition m ON o.Region = m.Region
        WHERE o.Region IN ('Africa', 'Americas')
        GROUP BY o.Region;
    """,
    "Countries with High Obesity and Low Malnutrition": """
        SELECT o.Country
        FROM obesity o
        JOIN malnutrition m ON o.Country = m.Country
        GROUP BY o.Country
        HAVING AVG(o.Mean_Estimate) > 25 AND AVG(m.Mean_Estimate) < 10;
    """,
    "Age-wise Trend in Obesity & Malnutrition": """
        SELECT o.Age_Group, AVG(o.Mean_Estimate) as Avg_Obesity, AVG(m.Mean_Estimate) as Avg_Malnutrition
        FROM obesity o
        JOIN malnutrition m ON o.Age_Group = m.Age_Group
        GROUP BY o.Age_Group;
    """
}

# Title
st.title("Nutrition Paradox Dashboard")

# Run and show result based on user selection
st.markdown("---")
st.subheader("Choose Category")
category = st.radio("Select Category", ["Obesity", "Malnutrition", "Combined"])

if category == "Obesity":
    query_key = st.selectbox("Select an Obesity Query", list(obesity_queries.keys()))
    result = run_query(obesity_queries[query_key])
elif category == "Malnutrition":
    query_key = st.selectbox("Select a Malnutrition Query", list(malnutrition_queries.keys()))
    result = run_query(malnutrition_queries[query_key])
else:
    query_key = st.selectbox("Select a Combined Query", list(combined_queries.keys()))
    result = run_query(combined_queries[query_key])

# Display results
st.markdown("---")
st.dataframe(result)


conn.close()
