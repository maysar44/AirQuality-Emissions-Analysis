# Air Quality and Emissions Analysis

## Description
Conducted analysis in SQL to surface insights on the relationship between air quality and pollutant emissions using datasets detailing the Air Quality Index (AQI) by city and emissions of air pollutants by country. Worked using RegEx and Excel to extract and clean data. Analyzed data in SQL and built comprehensive queries to identify the most harmful pollutants and investigate correlations between emissions and air quality. Surface insights and recommendations are geared towards environmental sustainability and public health policies.

## Datasets
- [World Air Quality Index by City and Coordinates](https://www.kaggle.com/datasets/adityaramachandran27/world-air-quality-index-by-city-and-coordinates): Contains data on air quality index (AQI) values by city, including specific pollutant AQI values.
- [Emissions of Air Pollutants](https://stats.oecd.org/Index.aspx?DataSetCode=AIR_EMISSIONS): Provides national emissions data for various air pollutants.

## Data Audit and Wrangling
### Data Source 1
The first dataset details the AQI values by city and coordinates. It contains 16,695 rows with columns such as country, city, AQI value, pollutant-specific AQI values, and geographic coordinates. Key steps in the data wrangling process included:
- **Inspection**: Checked for missing values and inconsistencies.
- **Handling Missing Values**: Excluded rows with missing country values after unsuccessful attempts to match using latitude and longitude.
- **Foreign Key Addition**: Added a foreign key column for country codes using VLOOKUP.

### Data Source 2
The second dataset provides emissions data by country and pollutant. It contains 5,440 entries with columns such as country code, pollutant, year, and emission values. Key steps included:
- **Format Conversion**: Converted the dataset from XML to a structured format using regular expressions and Excel functions.
- **Data Extraction**: Extracted relevant data attributes and year values into new columns.
- **Handling Missing Values**: Replaced NULL values with 0 for consistency.
- **Unit Consistency**: Used only data with units in tonnes.

### Database
The cleaned datasets were imported into an SQLite database named `matk663_DB.db` and joined on the common attribute of country codes. The join was successful and ready for further analysis.

## Key Findings
- **Most Harmful Pollutants**: Carbon Monoxide (CO) is the most harmful pollutant, contributing 66% to worsening air quality. Particulate Matter (PM10) and Non-Methane Volatile Organic Compounds (NMVOC) follow at 22% and 18% respectively.
- **Emission and Air Quality Relationship**: There are expected correlations between high emissions and poor air quality in countries like the USA. However, outliers like Russia and Canada show differing patterns.

![image](https://github.com/maysar44/AirQuality-Emissions-Analysis/assets/173674412/8bcf5d2d-b8c1-49b4-b70d-75bc46c8c876)

## Analysis
### SQL Queries
1. **Top Three Most Harmful Pollutants**:
   ```sql
   SELECT
       POL,
       (AVG(CASE WHEN POL = 'SOX' THEN "2021" ELSE NULL END) * 1.0 / AVG(s1.AQIValue)) AS SOXImpact,
       (AVG(CASE WHEN POL = 'NOX' THEN "2021" ELSE NULL END) * 1.0 / AVG(s1.AQIValue)) AS NOXImpact,
       (AVG(CASE WHEN POL = 'PM10' THEN "2021" ELSE NULL END) * 1.0 / AVG(s1.AQIValue)) AS PM10Impact,
       (AVG(CASE WHEN POL = 'PM2-5' THEN "2021" ELSE NULL END) * 1.0 / AVG(s1.AQIValue)) AS PM2_5Impact,
       (AVG(CASE WHEN POL = 'CO' THEN "2021" ELSE NULL END) * 1.0 / AVG(s1.AQIValue)) AS COImpact,
       (AVG(CASE WHEN POL = 'NMVOC' THEN "2021" ELSE NULL END) * 1.0 / AVG(s1.AQIValue)) AS VOCImpact
   FROM
       "Source_1" AS s1
   JOIN
       "Source_2" AS s2
   ON
       s1.CountryCode = s2.COU
   WHERE
       s2.UNIT = 'TONNE'
   GROUP BY
       POL;
   ```
Result: CO, PM10, and NMVOC are the most harmful pollutants.

2. **Countries with Hazardous AQI and Emissions**:
```sql
SELECT
    s1.Country,
    SUM(CASE WHEN s2.POL = 'CO' AND s2.UNIT = 'TONNE' THEN s2."2021" ELSE 0 END) AS TotalCO,
    SUM(CASE WHEN s2.POL = 'PM10' AND s2.UNIT = 'TONNE' THEN s2."2021" ELSE 0 END) AS TotalPM10,
    SUM(CASE WHEN s2.POL = 'NMVOC' AND s2.UNIT = 'TONNE' THEN s2."2021" ELSE 0 END) AS TotalNMVOC,
    AVG(AQIValue)
FROM
    "Source_1" AS s1
JOIN
    "Source_2" AS s2
ON
    s1.CountryCode = s2.COU
WHERE
    s1.AQIValue >= 300
GROUP BY
    s1.Country
HAVING
    SUM(CASE WHEN s2.POL = 'CO' AND s2.UNIT = 'TONNE' THEN s2."2021" ELSE 0 END) > 0 OR
    SUM(CASE WHEN s2.POL = 'PM10' AND s2.UNIT = 'TONNE' THEN s2."2021" ELSE 0 END) > 0 OR
    SUM(CASE WHEN s2.POL = 'NMVOC' AND s2.UNIT = 'TONNE' THEN s2."2021" ELSE 0 END) > 0
ORDER BY AVG(s1.AQIValue) DESC;
```
Result: Countries like Chile and the USA show a strong correlation between emissions and hazardous AQI values.

3. **Top 10 Emitters and Their Maximum AQI**:

```sql
WITH TotalEmissions AS (
    SELECT
        s2.COU AS CountryCode,
        SUM(CASE WHEN s2.POL = 'CO' AND s2.UNIT = 'TONNE' THEN s2."2021" ELSE 0 END) AS TotalCO,
        SUM(CASE WHEN s2.POL = 'PM10' AND s2.UNIT = 'TONNE' THEN s2."2021" ELSE 0 END) AS TotalPM10,
        SUM(CASE WHEN s2.POL = 'NMVOC' AND s2.UNIT = 'TONNE' THEN s2."2021" ELSE 0 END) AS TotalNMVOC,
        SUM(CASE WHEN s2.POL IN ('CO', 'PM10', 'NMVOC') AND s2.UNIT = 'TONNE' THEN s2."2021" ELSE 0 END) AS TotalEmissions
    FROM
        "Source_2" AS s2
    GROUP BY
        s2.COU
)
SELECT
    s1.Country AS CountryName,
    te.TotalCO,
    te.TotalPM10,
    te.TotalNMVOC,
    te.TotalEmissions,
    MAX(s1.AQIValue) AS MaxAQI
FROM
    TotalEmissions AS te
JOIN
    "Source_1" AS s1
ON
    s1.CountryCode = te.CountryCode
GROUP BY
    s1.Country, te.TotalCO, te.TotalPM10, te.TotalNMVOC, te.TotalEmissions
ORDER BY
    te.TotalEmissions DESC
LIMIT 10;
```
Result: Six out of the ten top emitters had maximum AQI values under 150, indicating a moderate to unhealthy level for sensitive groups.

