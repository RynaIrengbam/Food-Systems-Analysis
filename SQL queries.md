## Data Cleaning

#### Fertilizer
The table was pivot using Element so that ‘Use per area of cropland’ and ‘Use per value
of agricultural production’ are columns for easier querying and charting.

````sql
CREATE TABLE IF NOT EXISTS fertilizer_use AS
SELECT
    ISO3 AS ISO,
    Area,
    Year,
    Item,
    MAX(CASE WHEN Element = 'Use per area of cropland' THEN Value END) AS "use_per_area_cropland_kgha",
    MAX(CASE WHEN Element = 'Use per value of agricultural production' THEN Value END) AS "use_per_value_ag_prod_gdollar"
FROM "FertilizerUse.csv"
GROUP BY ISO3, Area, Year, Item;
````

#### Crop Production
The table was pivot on Elements column so we could get yield and production as
columns which would make querying easier. Adding a column ‘data status’ such that for later querying we know which ones are
missing, reported or zero production.

````sql
ALTER TABLE crop_production ADD COLUMN Data_Status TEXT;

UPDATE crop_production
SET Data_Status = CASE
  WHEN Production_t IS NULL AND Yield_kg_per_ha IS NULL THEN 'Missing'
  WHEN Production_t = 0 THEN 'Zero Production'
  ELSE 'Reported'
END;
````

Assuming that countries with 0 production had 0 yield in general and imputing it in the
table.

````sql
UPDATE crop_production
SET Yield_kg_per_ha = 0
WHERE Production_t = 0 AND Yield_kg_per_ha IS NULL;
````

#### Economic Data

Making year columns to a single column with all the years to ensure consistency with
other tables.

````sql
CREATE TABLE IF NOT EXISTS economics AS
SELECT WEO_Country_Code, ISO, Country, Subject_Descriptor, Units, '2019' AS Year, "2019" AS Value FROM "Economic.csv"
UNION ALL
SELECT WEO_Country_Code, ISO, Country, Subject_Descriptor, Units, '2020' AS Year, "2020" AS Value FROM "Economic.csv"
UNION ALL
SELECT WEO_Country_Code, ISO, Country, Subject_Descriptor, Units, '2021' AS Year, "2021" AS Value FROM "Economic.csv"
UNION ALL
SELECT WEO_Country_Code, ISO, Country, Subject_Descriptor, Units, '2022' AS Year, "2022" AS Value FROM "Economic.csv";
````
Unemployment group has missing values for countries. Since we have IMF groupings for
each of the countries, it makes sense to impute these missing values with group average
unemployment rate.

````sql
UPDATE economic_cleaned AS e
SET Value = i.Value
FROM (
    WITH economics_with_group AS (
        SELECT
            e.*,
            g.IMF_Groups
        FROM economics AS e
        LEFT JOIN "IMF_groupings.csv" AS g ON e.ISO = g.ISO
    ),

    unemployment_group_avg AS (
        SELECT
            IMF_Groups,
            Year,
            AVG(CAST(NULLIF(Value, 'n/a') AS DOUBLE)) AS avg_unemployment
        FROM economics_with_group
        WHERE
            Subject_Descriptor = 'Unemployment rate'
            AND Value IS NOT NULL
        GROUP BY IMF_Groups, Year
    )

    SELECT
        e.ISO,
        e.Year,
        e.Subject_Descriptor,
        CASE 
            WHEN e.Value = 'n/a' OR e.Value IS NULL THEN u.avg_unemployment
            ELSE CAST(e.Value AS DOUBLE)
END AS Value
    FROM economics_with_group AS e
    LEFT JOIN unemployment_group_avg AS u
        ON e.IMF_Groups = u.IMF_Groups AND e.Year = u.Year
    WHERE e.Subject_Descriptor = 'Unemployment rate'
) AS i
WHERE
    e.ISO = i.ISO
    AND e.Year = i.Year
    AND e.Subject_Descriptor = i.Subject_Descriptor;
````

#### Food Security

Since this table has 3-year averages, years are represented like 2018-2020 which is not
consistent with other tables and would make join difficult. Changing each year to their
midpoint year in the same FOA considers in their modeling.

````sql
CREATE TABLE IF NOT EXISTS food_security AS
SELECT 
  ISO,
  Area,
  Element_Code,
  Item_Code,
  Item,
  Year_Code,
  CASE 
    WHEN Year = '2018-2020' THEN 2019
    WHEN Year = '2019-2021' THEN 2020
    WHEN Year = '2020-2022' THEN 2021
    WHEN Year = '2021-2023' THEN 2022
    ELSE CAST(Year AS INTEGER)
  END AS Year,
  Unit,
  Value
FROM "Food Security.csv";
````

### Data Analysis

1. Track average nitrogen fertilizer use over time by IMF country group.

````sql
SELECT f.Year, i.IMF_Groups, AVG(f.use_per_area_cropland_kgha) AS Avg_Nitrogen_Use
FROM fertilizer_use f
JOIN "IMF_groupings.csv" i ON f.ISO = i.ISO
WHERE Item LIKE '%nitrogen%'
GROUP BY f.Year, i.IMF_Groups
ORDER BY Year;
````

<img width="437" alt="image" src="https://github.com/user-attachments/assets/492a5095-d4b3-4d5f-8294-ce43cec071f0" />

2. 


