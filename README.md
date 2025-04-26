# Food-Systems-Analysis

# Dataset Choice:
The following 6 datasets were chosen to explore global relationship between agricultural inputs
(fertilizer use), agricultural outputs (production and yield), economic development, and food
security outcomes. Each table was chosen to represent a different but complementary aspect of
this complex system:

## FOAUN Fertilizer Use (Input)
Represents the agricultural input intensity of 3 different fertilizers (nitrogen, phosphate,
potash). It gives the standard measure of input intensity (Use per area of cropland) but also
gives fertilizer efficiency: how much input used per dollar of output (Use per value of
agricultural production)
**Table Name:** Fertilizer Use
**Column Names:** ISO3, Area, Element_Code, Element, Item_Code, Year_Code, Year, Unit,
Value

## FOAUN Agricultural Production (Output)
Captures how much food is produced and how efficiently, using indicators like crop yield and
production quantity. This dataset helps quantify productivity and output across countries.
**Table Name:** Crop Production
**Column Names:** ISO3, Area, Element_Code, Element, Item_Code, Item, Year_Code, Year,
Unit, Value

## IMF World Economic Outlook (Context)
Offers context for economic conditions, including GDP per capita, unemployment and inflation.
These factors heavily influence both agricultural performance and a population’s ability to
access food.

**Table Name:** Economic
**Column Names:** WEO_Country_Code, ISO, Country, Subject_Descriptor, Units, Scale, 2019,
2020, 2021, 2022

## FOAUN Food Security Indicators (Outcome)
Includes crucial metrics like national calorie access and undernourishment. This table directly
reflects the nutritional outcomes and access to food: the goal of agricultural systems.
**Table Name:** Food Security
**Column Names:** ISO, Area, Element_Code, Item_Code, Item, Year_Code, Year, Unit, Value

## IMF Country Groups
Data was created from the metadata that grouped countries into Advanced Economies,
Emerging Economies and Low-Income Developing Economies and ISO number was added.
**Table Name:** IMF Groupings
**Column Names:** ISO, Country, IMG_Groups

## US Drought Monitor Categories
The U.S. Drought Monitor data provides weekly, national-level measures of
drought severity across the country. It categorizes the percentage of land affected by
different drought intensities: None (No Drought) D0 (Abnormally Dry), D1 (Moderate
Drought), D2 (Severe Drought), D3 (Extreme Drought) and D4 (Exceptional Drought).
**Table Name:** Drought_US
**Column Names:** ISO, MapDate, Area of Interest, None, D0, D1, D2, D3, D4, Valid
Start, Valid End

Why is dataset choice interesting?
This combination of dataset is interesting because it enables me to explore different perspectives
of the agricultural system: from the inputs that go into farming to the final outcomes of food
security, in conjunction with the economic context in which these processes occur. The data for
US drought can further help me understand the food system from input to impact, while
accounting for climate stress that significantly can affect the outcome.
By including IMF country groups, I can further analyze how these dynamics differ between low-
income, emerging market, and advanced economies. This makes it possible to uncover
structural inequalities. In this changing global climate, this dataset gives us a small insight in the current food systems.
This analysis is essential for steering food systems toward greater sustainability and
inclusiveness and this approach aligns with United Nations’ Sustainable Development Goals. [6]
