# ⚙️ Project Overview

This is a real-life **business intelligence** project which I worked on at my company in Q2 2025.

The aim of the project was to deliver an **end-to-end analytical solution** 📈 for a department that had been still heavily dependent on Excel and manual reporting. I explored the database, accessed and structured the data I was interested in, and finally built a dashboard:

![Fleet_Monitoring_Dashboard](/Pictures/Dashboard%20demo.gif)

As a result, the executives and other analysts can now get an up-to-date report in just a minute 🐎 instead of waiting one month or even one quarter until someone performs the super-time-consuming reporting activity and produces another set of Excel files. 🐢

If you are interested in how I achieved it, you are in the right place! 🫡

# 💡 Background

First of all, let me explain why there was a need for this project.

## 🗨️ The story behind

I work at a car leasing company. 🚗 We buy cars from the manufacturers, lease them to our clients and after the leasing period, when the vehicle comes back to us, we usually sell it. 💰 All of this happens in a system which has its own database. Now, the problem is that the people responsible for the reporting have always used the pre-defined CSV reports from the system. These files are quite incomplete so that the analysts needed to merge multiple files together (using even some sort of "Excel database") in an very old-school Excel approach - million VLOOKUPs, tons of sheets and workbooks (Sales_012025.xlsx, Sales_022025.xlsx...) etc. 😫

🛑 I stopped and thought: "Wait, it doesn't make sense to have a database but not use it to its full potential". **Why should people waste so much time on manual reporting if a major part of it can be automated?**

**At that point the idea to make this project came to my mind.** 💡

It wasn't easy at the beginning. I had to start over multiple times because I got lost in the business process (which isn't as easy as it seems to be!). 🤷‍♂️ Finally, after almost two months of work (in the meantime I had also many other tasks) I was able to present my product to my supervisors. Now, I'm presenting some part of it to you.

I think that working on this project can be split into **three major sections**:
- ***data wrangling*** (talking to the business, getting familiar with the business process, finding the data in the database, writing SQL queries)
- ***data cleaning*** (cleaning and grouping the data in Power Query)
- ***data visualization*** (builing the dashboard, writing DAX calculations)

On the contrary to what is commonly said, data cleaning wasn't the most time consuming part...

![Time_split](/Pictures/Time%20split.png)

❕ *One more disclaimer before we dive into the project: as it is a professional project and the data I worked on was highly sensitive, I can't share with you all details. I removed or anonymized all such sensitive information. As a result, the dashboard you see has some parts anonymized - just for demonstration purposes.*

## 🛠️ Tools I used

- **SQL** - helped me to extract from the system the data I wanted
- **Power BI** - where I visualized the data
- **Power Query** - I used it to clean and group the data in Power BI
- **DAX** - I used it for the calculations in Power BI
- **Oracle Analytics** - where I got the data from and wrote SQL queries
- **VS Code** - as writing SQL in Oracle Analytics is quite uncomfortable (lack of formatting and comments), I used VS Code to write and keep track of my SQL queries
- **Git** - I used it for the version control of the queries written in VS Code

# 🔍 Data Wrangling

The first step was to thoroughly **understand the business process** behind the data. For this, I talked to multiple people at the company - from salesmen to pricing analysts. I crosschecked each information given to me by one department with what others told me and with the data in the system. Then, once I had enough knowledge, I started to write the first SQL queries to extract some part of the data. In the beginning it was a case-by-case approach: I compared a sample data row by row. Fortunately, my logic and understanding of the business turned out to be correct. Finally, I aggregated the data month by month to see if the numbers match with what was reported. Apart from individual cases that could be easily explained, the matching was very satisfactory. ✅

But! What was **really challenging** at this stage was to specify the date when a contract should be reported as *order* and when as *production*. As you can imagine, the database contained dozens of date columns. However, I couldn't just use one of them. 🙅‍♂️

To do it properly, I created a couple of separate tables (separate SELECT statements). My access rights to Oracle Analytics don't allow to use SQL to its full potential (instead, I can only use Logical SQL which is somehow limited), so I wasn't able to create views and join the tables in one go. I used the UI to create a sort of data model composed of 5 tables. This allowed me to finally merge these tables into one big flat table which became my main data source. I download it as a CSV file to feed the Power BI dashboard. 📊

Here are some examples of my SQL queries that I wrote during this project...

- the main query being the core of my fact table:

``` sql
SELECT
    -- general info
    "DIM_CONTRACT"."CONTRACTNR" AS contract_nr,
    "DIM_CONTRACT"."SERNR" AS contract_serial_nr,
    TRIM("DIM_COUTRY"."COUNTRY") ||
        CAST("DIM_CONTRACT"."CONTRACTNR" AS CHAR) || '-' ||
        TRIM(
            CAST("DIM_CONTRACT"."SERNR" AS CHAR)
        )
    AS full_nr, -- composite surrogate key to work as primary key
    "DIM_COUTRY"."COUNTRY" AS country,

    -- vehicle info
    "DIM_CONTRACT"."NR_REGISTRATION" AS registration_nr,
    "DIM_CLIENT"."NAME" AS client_name,
    "DIM_CLIENT"."NR_CLIENT" AS client_nr,
    "DIM_VEHICLE"."MAKE_NAME" AS make_name,
    "DIM_VEHICLE"."MODEL_NAME" AS model_name,
    "DIM_CONTRACT"."FUEL" AS fuel_type,

    -- contract info
    "DIM_CONTRACT"."YEARLYKM" AS yearly_mileage,
    "DIM_CONTRACT"."DURATION" AS contract_duration,
    "DIM_DATE_DEHIRE"."DATE" AS date_dehire,
    "DIM_DATE_COM"."DATE" AS date_commencement,
    "DIM_DATE_ORDER"."DATE" AS date_order,
    "FACT_CONTRACT"."VEH_PRICE" AS vehicle_price,

    -- and many other fields that I am not showcasing here (all field names have been changed for demonstration purposes and are not the actual ones)

FROM "CONT"
WHERE
  country IN ('AA', 'BB', 'CC', 'DD') -- here I input the country codes
  AND make_name <> 'Others' -- exclude other makes
  AND sort_contract <> 'B' -- exclude specific type of contracts
  AND client_nr <> 9999999 -- used to indicate unusual contracts that should not be analyzed
```
- I used the following nested function to the number of clients per each contract but excluding situations where the client remains the same although the company name changes slightly (e.g. a change from "Example UK" to "Example Ltd" should not be considered as a change). I used LOCATE and LEFT to identify only the first word and see if it changed:

``` sql
COUNT(DISTINCT
    CASE
    WHEN "DIM_CLIENT"."CLIENT_NR" <> 9999999 THEN -- exclude a specific type of contracts
        LEFT(
            "DIM_CLIENT"."NAME",
            LOCATE(
                ' ',
                UPPER(
                    TRIM(
                        LEADING FROM "DIM_CLIENT"."NAME"
                    )
                )
            )
        )
    ELSE NULL
    END
) AS client_count
```

# 🧹 Data Cleaning

The next step was to import the data to Power BI and clean it using the Power Query M language.

⭐ I went for the solution that works best - a classic **star schema**. The central table of my model was *Fact_Contract*, surrounded by the dimension tables: *Dim_Date* (I used a function that I had written earlier to quickly create and reuse a date table in Power Query - you can find it [here](https://github.com/MarcinCzerkas/Power-Query/blob/main/Calendar.txt)) and *Dim_Countries*, related additioinally to two budget dimensions: *BGT_Orders* and *BGT_Production*.

![DATA_MODEL](/Pictures/data%20model.png)

As my fact table contained multiple date columns, I created multiple inactive relationships to *Dim_Date* and activated them directly in DAX measures like in this one:

``` dax
Orders Count =
CALCULATE(
    COUNTROWS(Fact_Contract),
    Fact_Contract[orders_reporting_date] <> BLANK(),
    USERELATIONSHIP(Fact_Contract[orders_reporting_date], Dim_Date[Date])
)
```

Apart from the tables loaded to the model, I used additional three tables as **lookup tables** for *Fact_Contract*. I merged them with the fact table using Table.Buffer to buffer them in the memory and speed up the refresh process.

![M_Code](/Pictures/PQ%20all.png)
*The whole query of the fact table is quite long (500+ rows) but well structured*

Then, I had to take my time to very carefully **clean and group** the data. I created additional columns with cleaned data (like *fuel_cleaned*, where I specify the actual fuel type based on values from different columns) and assigned the cars to age and mileage bins (to be able to easily build histograms later on). I used nested *let* statements to make my M Code more readable:

``` m
// GROUPING DATA INTO BINS

// Clean and group mileage
Mileage =
let
    TotalMileage = Table.AddColumn(ReplacedValues, "contractual_mileage",
        each Number.From([yearly_mileage]) / 12 * Number.From([contract_duration]),
        type number
    ),

    MileageBins = Table.AddColumn(
        TotalMileage, "mileage_bins",
        each if [contractual_mileage] <= 50000 then "<50k"
        else if [contractual_mileage] <= 100000 then "50k-100k"
        else if [contractual_mileage] <= 150000 then "100k-150k"
        else if [contractual_mileage] <= 200000 then "150k-200k"
        else if [contractual_mileage] > 200000 then ">200k"
        else "empty",
        type text
    ),

    MileageBinsSort = Table.AddColumn(
        MileageBins, "mileage_bins_sort",
        each if [mileage_bins] = "<50k" then 1
        else if [mileage_bins] = "50k-100k" then 2
        else if [mileage_bins] = "100k-150k" then 3
        else if [mileage_bins] = "150k-200k" then 4
        else if [mileage_bins] = ">200k" then 5
        else 0,
        Int64.Type
    ),

    YearlyMileageBins = Table.AddColumn(
        MileageBinsSort, "yearly_mileage_bins",
        each if Number.From([yearly_mileage]) <= 10000 then "<10k"
        else if Number.From([yearly_mileage]) <= 30000 then "10k-30k"
        else if Number.From([yearly_mileage]) <= 50000 then "30k-50k"
        else if Number.From([yearly_mileage]) <= 70000 then "50k-70k"
        else if Number.From([yearly_mileage]) > 70000 then ">70k"
        else "empty",
        type text
    ),

    YearlyMileageBinsSort = Table.AddColumn(
        YearlyMileageBins, "yearly_mileage_bins_sort",
        each if [yearly_mileage_bins] = "<10k" then 1
        else if [yearly_mileage_bins] = "10k-30k" then 2
        else if [yearly_mileage_bins] = "30k-50k" then 3
        else if [yearly_mileage_bins] = "50k-70k" then 4
        else if [yearly_mileage_bins] = ">70k" then 5
        else 0,
        Int64.Type
    )
in
    YearlyMileageBinsSort
```

📁 Finally, I really don't like mess. That's why I used a standardized nomenclature in my semantic model and created a clear folder hierarchy for the tables and measures:

![folders_measures](/Pictures/folders%20measures.png)
![folders_facts](/Pictures/folders%20facts.png)
![folders_date](/Pictures/folders%20date.png)

# 📊 Data Visualization

And here is the final product - my dashboard. As of now it is composed of 3 pages: one being the fleet monitoring overview for the executives and the other two - a risk control panel for the more in-detail and analytical approach.

![DASHBOARD](/Pictures/Demo%201.png)
*The main page of the dashboard*

💡 I tried to focus on what is important and not to overload the user with too much information. However, I gave them the possibility to deep dive into the particular topics using **tooltips**, **swapping visuals** and the **slicer panel**:

![TOOLTIP](/Pictures/Demo%202.png)
*Tooltip showing the top 5 brands/models*

🤷‍♀️ Should the user still feel overwhelmed and not know where to start from, they can use the **help button** to get familiarized with the dashboard page they're looking at:

![HELP](/Pictures/Demo%203.png)
*Help overlay displayed after clicking on the help button*

Finally, I had to use some **DAX** (of course!) for my calculations. My semantic model contains 63 measures:

![Semantic_model](/Pictures/semantic%20model.png)

Here are some examples of my DAX code...

a measure to switch between orders and production:

``` dax
Orders/Production Count Switched =
SWITCH(
    SELECTEDVALUE('Orders/Production_Switch'[Reporting]),
    "Orders", [Orders Count],
    "Production", [Production Count]
)
```
a measure for the budget vs YTD volumes matrix visual:

``` dax
Orders Target =
VAR SelectedFuel = SELECTEDVALUE(Fact_Contract[fuel_type_cleaned_emv_short])

// The logic to calculate specific fuel types
VAR Result =
    SWITCH(
        SelectedFuel,
        "DI",  CALCULATE(SUM(BGT_Orders[Diesel]),   USERELATIONSHIP(BGT_Orders[Country], Dim_Country[Country])),
        "PET", CALCULATE(SUM(BGT_Orders[Petrol]),   USERELATIONSHIP(BGT_Orders[Country], Dim_Country[Country])),
        "HEV", CALCULATE(SUM(BGT_Orders[Hybrid]),   USERELATIONSHIP(BGT_Orders[Country], Dim_Country[Country])),
        "BEV", CALCULATE(SUM(BGT_Orders[Electric]), USERELATIONSHIP(BGT_Orders[Country], Dim_Country[Country])),
        BLANK()
    )

RETURN
IF(
    ISINSCOPE(Fact_Contract[fuel_type_cleaned_emv_short]),
    Result,
    
    // Total row logic
    CALCULATE(
        SUM(BGT_Orders[Diesel]) +
        SUM(BGT_Orders[Petrol]) +
        SUM(BGT_Orders[Hybrid]) +
        SUM(BGT_Orders[Electric]),
        USERELATIONSHIP(BGT_Orders[Country], Dim_Country[Country])
    )
)
```

# 💡 Conclusion

My supervisors told me this project is a game changer. 🥇

I transformed a manual process that took a lot of time and was prone to errors into an automated dashboard that needs just a couple of clicks to be refreshed. ✅

I learned a lot while working on this project. I think the biggest lesson I've learned was how important and how hard it is to understand the business before I even start to collect the data. 🎓

### What can be improved?

This project is still to be continued. My aim is to add additional dashboard pages (e.g. to monitor the maintenance costs) based on more data from the system.

Additionally, in an ideal world, my Power BI dashboard should be connected directly to the database instead of relying on CSV exports from the Oracle BI tool. However, I consider it as barely achievable since it would require the IT department to grant access which is very unlikely to happen. 🤷‍♂️

I am writing this paragraph in October 2025, more than 4 months after delivering the first version of the dashboard: I kept working on this project and added new measures and KPIs. I created new report pages to deep dive into another details answering questions like: *How long does a contract last in reality in comparison to the contractual duration? Do we overestimate or underestimate the mileage on contract vs in reality?* I also integrated **Python** in the process to recalculate the residual value of the vehicles based on statistical data and measured on different age & mileage combinations.

## Appendix: Who am I? 🤹‍♂️

My name is Marcin Czerkas. I am a data-lover and user-oriented Power BI specialist.

I work with data since 2023. At that time I changed job and started helping my new team by automating the process and building dashboards. At the time I am writing this file (June 2025), I work in the asset risk department and help my company not to lose money by spotting dangerous trends in the data early on. ⚡

When I transitioned into the world of data, I started a learning journey that lasts until now and is probably going to last for a long time. Projects like this one help me test my analytical skills in practice and showcase them.

### 🔗 **Visit [my LinkedIn profile](https://www.linkedin.com/in/marcin-czerkas-95150727a/) to learn more.**