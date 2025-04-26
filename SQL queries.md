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

1.**Track average nitrogen fertilizer use over time by IMF country group.**

````sql
SELECT f.Year, i.IMF_Groups, AVG(f.use_per_area_cropland_kgha) AS Avg_Nitrogen_Use
FROM fertilizer_use f
JOIN "IMF_groupings.csv" i ON f.ISO = i.ISO
WHERE Item LIKE '%nitrogen%'
GROUP BY f.Year, i.IMF_Groups
ORDER BY Year;
````

<img width="500" alt="image" src="https://github.com/user-attachments/assets/492a5095-d4b3-4d5f-8294-ce43cec071f0" />

Advanced Economies consistently show the highest nitrogen use, with a slight decline after 2021. Emerging Market Economies follow a similar trend with lower values and a mid-series peak in 2020. Low-Income Developing Countries have the lowest usage, remaining flat or slightly declining. The results are in line with expectations as
•	Wealthier countries have more resources and infrastructure to invest in agricultural inputs like fertilizers. 
•	Lower-income countries may face constraints related to cost, availability, or agricultural practices.


2. **Compare average fertilizer use and wheat yield by IMF group and year.**

````sql
SELECT f.Year, i.IMF_Groups,
       AVG(f.use_per_area_cropland_kgha) AS Avg_Fertilizer_Use, 
       AVG(c.Yield_kg_per_ha) AS Avg_Wheat_Yield
FROM fertilizer_use f
JOIN crop_production c 
  ON f.ISO = c.ISO 
  AND f.Area = c.Area 
  AND f.Year = c.Year
JOIN "IMF_groupings.csv" i ON c.ISO = i.ISO
WHERE c.Item = 'Wheat' 
  AND c.Data_Status = 'Reported'
GROUP BY f.Year, i.IMF_Groups
ORDER BY f.Year;
````

![image](https://github.com/user-attachments/assets/1d6d521f-f18b-47eb-92c0-39ffaa69ec39)

There’s a clear positive correlation between fertilizer use and wheat yield across groups: Advanced economies use more fertilizer and achieve higher yields and Low-income countries lag significantly behind both in input and output. These results are aligned with global development and agricultural trends as resource-rich countries invest more in fertilizer and modern agriculture, yielding better outcomes.

3. **Get country-level fertilizer use and wheat yield data for reporting or inspection.**

````sql
SELECT f.Year, i.IMF_Groups,
       f.ISO, 
       c.Item AS Crop, 
       f.use_per_area_cropland_kgha AS Fertilizer_Use, 
       c.Yield_kg_per_ha AS Wheat_Yield
FROM fertilizer_use f
JOIN crop_production c 
  ON f.ISO = c.ISO 
  AND f.Area = c.Area 
  AND f.Year = c.Year
JOIN "IMF_groupings.csv" i ON c.ISO = i.ISO
WHERE c.Item = 'Wheat'
  AND c.Data_Status = 'Reported'
ORDER BY f.Year;
````

![image](https://github.com/user-attachments/assets/2cb7904c-a960-4527-b4e2-525b4b8fc758)


We saw in query 2 that high fertilizer use does contribute to high yield. But from above
graph, we can also see that advanced economies even with low fertilizer use can reach
high level of yields. Even if low-income countries utilize higher amount of fertilizer the
yield is not as good as the better economies. This suggests that factors beyond fertilizer
play a substantial role in yield efficiency, particularly in wealthier countries.

4. **Count how many countries in each IMF group are above global average dietary
energy supply adequacy.**

````sql
SELECT 
  imf.IMF_Groups,
  COUNT(DISTINCT f.ISO) AS num_countries_above_avg
FROM food_security_clean f
JOIN "IMF_groupings.csv" imf
  ON f.ISO = imf.ISO
WHERE f.Avg_dietary_energy_supply_adequacy > (
    SELECT AVG(Avg_dietary_energy_supply_adequacy)
    FROM food_security_clean
)
GROUP BY imf.IMF_Groups;
````

![image](https://github.com/user-attachments/assets/27351263-881e-4e76-b344-7603ec4495a2)

Emerging Market Economies Have the Most Above-Average Countries (50): Indicates
diversity within this group with many countries have improved food systems and exceed
the global average. Out of only 41 Advanced Economies, 33 of them perform above
average. Expectedly, very few countries from Low-Income Developing group exceed the
global average, highlighting the persistent food security challenge.

5. **Based on above observation of 11 Low-Income Developing Countries, compare
agricultural and economic performance across food-secure and insecure countries.**

````sql
SELECT 
  f.Year,
  CASE 
    WHEN f.Avg_dietary_energy_supply_adequacy > (
      SELECT AVG(Avg_dietary_energy_supply_adequacy)
      FROM food_security_clean
    ) THEN 'Above Global'
    ELSE 'Below or Equal Global'
  END AS food_security_status,
  ROUND(AVG(c.Yield_kg_per_ha), 2) AS avg_yield_kg_per_ha,
  ROUND(AVG(e.Unemployment_percent_total_labor), 2) AS avg_unemployment_rate
FROM food_security_clean f
JOIN "IMF_groupings.csv" imf
  ON f.ISO = imf.ISO
JOIN fertilizer_use fe 
  ON f.ISO = fe.ISO AND f.Year = fe.Year
JOIN crop_production c 
  ON fe.ISO = c.ISO AND fe.Year = c.Year
  AND c.Data_Status = 'Reported'
JOIN clean_economics_wide e 
  ON f.ISO = e.ISO AND f.Year = e.Year
WHERE imf.IMF_Groups = 'Low-Income Developing Countries'
GROUP BY f.Year, food_security_status
ORDER BY f.Year, food_security_status;
````

![image](https://github.com/user-attachments/assets/702190cf-14ad-49a0-ad4b-e8f510476682)

In all years, countries with above-average food adequacy also have higher average
yields. This matches expectations, better food supply tends to be linked with stronger
agricultural performance. Unemployment is consistently lower in the “Above Global”
group. The gap becomes more visible over time (especially post-2020), which aligns
with global COVID-19 disruptions (2020–2021). Despite that, Above Global countries
maintained better yield and food security, showing resilience in agriculture.

6. **Identify low-income country with below-average undernourishment rates.**

````sql
SELECT 
    imf.Country,
    imf.IMF_Groups,
    ROUND(AVG(fs.undernourishment_percent), 2) AS avg_undernourishment
FROM "IMF_groupings.csv" imf
JOIN food_security_clean fs 
  ON imf.ISO = fs.ISO
WHERE imf.IMF_Groups = 'Low-Income Developing Countries'
GROUP BY imf.Country, imf.IMF_Groups
HAVING AVG(fs.undernourishment_percent) < (
  SELECT AVG(undernourishment_percent)
  FROM food_security_clean
  WHERE undernourishment_percent IS NOT NULL
)
ORDER BY avg_undernourishment ASC, imf.Country
LIMIT 1;
````

![image](https://github.com/user-attachments/assets/548ce3d9-38de-41ad-b071-92c8ef05e8c4)

Moldova stands out as the best-performing low-income country when it comes to
minimizing undernourishment. With an average of just 2.5%, it performs significantly
better than many peers.

7. **From query 6, we saw that Moldova is a low-income country with below average
undernourishment so we try to understand whether they have better crop yield that
leads to better nourishment**

````sql
SELECT
  c.Item AS crop_type,
  ROUND(AVG(c.Yield_kg_per_ha), 2) AS moldova_yield,
  ROUND((
    SELECT AVG(c2.Yield_kg_per_ha)
    FROM crop_production c2
    JOIN "IMF_groupings.csv" imf2 ON c2.ISO = imf2.ISO
    WHERE c2.Item = c.Item
      AND c2.Data_Status = 'Reported'
      AND imf2.IMF_Groups = 'Low-Income Developing Countries'
  ), 2) AS low_income_avg_yield
FROM crop_production c
JOIN "IMF_groupings.csv" imf ON c.ISO = imf.ISO
WHERE c.ISO = 'MDA'
  AND c.Item IN ('Maize (corn)', 'Rice', 'Wheat')
  AND c.Data_Status = 'Reported'
GROUP BY c.Item
ORDER BY moldova_yield DESC;
````

![image](https://github.com/user-attachments/assets/076f7468-f0a3-4460-b268-d9a095abf000)

For both crops, Moldova’s yields are ~30–40% higher than the Low-Income Developing
Countries’ average. This performance reinforces what we saw in Query 6: Moldova is an
agricultural outlier within its economic group. These results are not surprising as the
country has favorable soil resources and conditions for agricultural production and
combined with affordable labor costs, especially in rural areas, favor the production of
high-yield and labor-intensive crops.

8. **Identify countries that apply nitrogen fertilizer but do not produce any of the
staple crops: rice, maize, or wheat but still maintain high dietary energy supply
adequacy, indicating strong food availability despite not growing staple crops.**

````sql
SELECT
    f.ISO,
    f.Area,
    f.Year,
    f.Item,
    fs.Avg_dietary_energy_supply_adequacy,
    ROUND(f.use_per_area_cropland_kgha, 2) AS fertilizer_use
FROM fertilizer_use AS f
LEFT JOIN crop_production AS c
    ON f.ISO = c.ISO AND f.Year = c.Year
INNER JOIN food_security_clean AS fs
    ON
        f.ISO = fs.ISO
        AND f.Year = fs.Year
WHERE
    f.Item LIKE '%nitrogen%'
    AND c.Item IS NULL
    AND fs.Avg_dietary_energy_supply_adequacy > 100
ORDER BY fs.Avg_dietary_energy_supply_adequacy DESC, fertilizer_use DESC
LIMIT 3;
````

![image](https://github.com/user-attachments/assets/07f75dd3-6c41-4e55-ab6e-90c1de066825)

Iceland has no recorded production of any of the staple yet it achieves 144% dietary
energy supply adequacy — meaning it supplies 44% more than the average minimum
daily requirement per person. This result is surprising because a country’s food security
often depends on staple crops. It has good food security because its local fishing industry 
provides food for locals and exports, so it keeps the population’s food secure.

9. **Find countries with both high inflation and high potash fertilizer use.**

````sql
SELECT ISO
FROM clean_economics_wide
WHERE Inflation_percent_change IS NOT NULL
GROUP BY ISO
HAVING AVG(Inflation_percent_change) > (
    SELECT AVG(Inflation_percent_change)
    FROM clean_economics_wide
    WHERE Inflation_percent_change IS NOT NULL
)

INTERSECT

--Countries with above-average potash fertilizer use
SELECT ISO
FROM fertilizer_use
WHERE Item LIKE '%potash%' AND use_per_area_cropland_kgha IS NOT NULL
GROUP BY ISO
HAVING AVG(use_per_area_cropland_kgha) > (
    SELECT AVG(use_per_area_cropland_kgha)
    FROM fertilizer_use
    WHERE Item LIKE '%potash%' AND use_per_area_cropland_kgha IS NOT NULL
);
````

Above query shows that there are no countries where there is a high inflation change and
simultaneously has high potash fertilizer use. This result is not surprising as rising
prices lead to higher costs for inputs like fertilizer.

10. **Measure the change in nitrogen fertilizer efficiency between 2019 and 2022.**

````sql
SELECT 
  f2019.ISO,
  f2019.Area,
  ROUND(f2019.use_per_value_ag_prod_gdollar, 2) AS efficiency_2019,
  ROUND(f2022.use_per_value_ag_prod_gdollar, 2) AS efficiency_2022,
  ROUND(f2022.use_per_value_ag_prod_gdollar - f2019.use_per_value_ag_prod_gdollar, 2) AS change_in_efficiency
FROM fertilizer_use f2019
JOIN fertilizer_use f2022
  ON f2019.ISO = f2022.ISO
  AND f2019.Area = f2022.Area
  AND f2019.Item = f2022.Item
WHERE f2019.Item LIKE '%nitrogen%'
  AND f2022.Item LIKE '%nitrogen%'
  AND f2019.Year = 2019
  AND f2022.Year = 2022
  AND f2019.use_per_value_ag_prod_gdollar IS NOT NULL
  AND f2022.use_per_value_ag_prod_gdollar IS NOT NULL
ORDER BY change_in_efficiency ASC, efficiency_2022 ASC
LIMIT 2;
````

![image](https://github.com/user-attachments/assets/571d1750-bb3e-45ed-af7f-cc4439a65700)

Belize and Mali has had a high decrease in efficiency meaning that both countries used
much more fertilizer per dollar of agricultural output in 2022 vs 2019. Both countries
have experienced high land degradation. Intensification of land use by shifting Maya
agriculturists in Belize has led to a decline in soil fertility and crop yields. In Mali, land
degradation and post-harvest losses due to poor storage and processing capacity, and
limited access to markets has contributed to low efficiency.

11. **Measure how prolonged drought periods in the U.S. correlate with nitrogen
fertilizer use and maize (corn) yield.**

````sql
WITH drought_weeks AS (
  SELECT 
    *,
    EXTRACT(YEAR FROM ValidStart) AS year,
    ROW_NUMBER() OVER (ORDER BY ValidStart) - ROW_NUMBER() OVER (PARTITION BY (D3 + D4 > 10) ORDER BY ValidStart) AS streak_group
  FROM "Drought_US.csv"
  WHERE AreaOfInterest = 'Total'
),

streaks AS (
  SELECT 
    year,
    MIN(ValidStart) AS streak_start,
    MAX(ValidEnd) AS streak_end,
    COUNT(*) AS num_weeks,
    DATEDIFF('day', MIN(ValidStart), MAX(ValidEnd)) AS streak_length_days
  FROM drought_weeks
  WHERE (D3 + D4) > 10
  GROUP BY year, streak_group
),

max_streaks AS (
  SELECT 
    year,
    MAX(streak_length_days) AS longest_drought_streak
  FROM streaks
  GROUP BY year
)

SELECT 
  ms.year,
  ms.longest_drought_streak,
  ROUND(AVG(fe.use_per_area_cropland_kgha), 2) AS avg_nitrogen_use,
  ROUND(AVG(cp.Yield_kg_per_ha), 2) AS avg_crop_yield
FROM max_streaks ms
JOIN fertilizer_use fe ON fe.ISO = 'USA' AND fe.Year = ms.year AND fe.Item LIKE '%nitrogen%'
JOIN crop_production cp ON cp.ISO = 'USA' AND cp.Year = ms.year AND cp.Data_Status = 'Reported'
WHERE cp.Item = 'Maize (corn)'
GROUP BY ms.year, ms.longest_drought_streak
ORDER BY ms.year;
````
![image](https://github.com/user-attachments/assets/41e73463-1935-4a23-89af-75307d702fc5)

US had longest drought weeks in 2021 and 2022. But maize (corn) average crop yield
increased from 2020 to 2021 even though there was a significant increase in number of
drought weeks. For corn, the national average yield was 177 bushels per acre, a record.
While the number of drought weeks remained the same from 2021, the crop yield
dropped in 2022 due to the hot and dry weather coinciding with corn’s reproductive
stage.

12. **Explore whether prolonged drought periods in the U.S. are associated with
increased undernourishment levels.**

````sql
WITH drought_weeks AS (
  SELECT 
    EXTRACT(YEAR FROM ValidStart) AS year,
    COUNT(*) AS severe_drought_weeks
  FROM "Drought_US.csv"
  WHERE AreaOfInterest = 'Total'
    AND (D3 + D4) > 10
  GROUP BY EXTRACT(YEAR FROM ValidStart)
)

SELECT 
  dw.year,
  dw.severe_drought_weeks,
  fs.undernourishment_percent
FROM drought_weeks dw
JOIN food_security_clean fs ON fs.ISO = 'USA' AND fs.Year = dw.year
GROUP BY dw.year, dw.severe_drought_weeks, fs.undernourishment_percent
ORDER BY dw.year;
````

![image](https://github.com/user-attachments/assets/fd5d24bf-45d1-4737-9970-8169d56f3d91)

The United States’ food security system is resilient enough that even prolonged national-level
drought does not directly affect overall undernourishment levels. They’re largely in line with
expectations for a high-income country like the U.S. But the fact that undernourishment didn’t
change at all might be a bit surprising, suggesting that the national food system is well-insulated
from even extreme climate stress.
