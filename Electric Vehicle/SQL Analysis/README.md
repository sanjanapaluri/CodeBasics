# Market Study for AtliQ Motors' EV/Hybrid Expansion in India

## SQL Analysis

#### Question 1: List the top 3 and bottom 3 makers for the fiscal years 2023 and 2024 innterms of the number of 2-wheelers sold?

```sql
WITH RankedSales AS (
    SELECT maker, 
           SUM(electric_vehicles_sold) AS total_sold,
           ROW_NUMBER() OVER (ORDER BY SUM(electric_vehicles_sold) DESC) AS rank_desc,
           ROW_NUMBER() OVER (ORDER BY SUM(electric_vehicles_sold) ASC) AS rank_asc
    FROM electric_vehicle.electric_vehicle_sales_by_makers
    GROUP BY maker
)
SELECT maker, total_sold
FROM RankedSales
WHERE rank_desc <= 3
   OR rank_asc <= 3
ORDER BY total_sold DESC, rank_desc;
```

### Question 2: Identify the top 5 states with the highest penetration rate in 2-wheeler and 4-wheeler EV sales in FY 2024?

```sql
 WITH PenetrationRateByState AS (
    SELECT
        es.State,
        es.vehicle_category,
        SUM(es.electric_vehicles_sold) AS TotalElectricVehiclesSold,
        SUM(es.total_vehicles_sold) AS TotalVehiclesSold,
        (SUM(es.electric_vehicles_sold) * 100.0 / SUM(es.total_vehicles_sold)) AS PenetrationRate
    FROM
        electric_vehicle.electric_vehicle_sales_by_state es
    JOIN
        electric_vehicle.dim_date dd ON es.date = dd.date
    WHERE
        dd.fiscal_year = 2024
    GROUP BY
        es.State, es.vehicle_category
),
RankedPenetrationRates AS (
    SELECT
        State,
        vehicle_category,
        PenetrationRate,
        RANK() OVER (PARTITION BY vehicle_category ORDER BY PenetrationRate DESC) AS R
    FROM
        PenetrationRateByState
)
SELECT
    State,
    vehicle_category,
    PenetrationRate
FROM
    RankedPenetrationRates
WHERE
    R <= 5
ORDER BY
    vehicle_category, R;
```

### Question 3 : List the states with negative penetration (decline) in EV sales from 2022 to 2024?

```sql
 WITH PenetrationRateByState AS (
    SELECT
        es.State,
        es.vehicle_category,
        SUM(es.electric_vehicles_sold) AS TotalElectricVehiclesSold,
        SUM(es.total_vehicles_sold) AS TotalVehiclesSold,
        (SUM(es.electric_vehicles_sold) * 100.0 / SUM(es.total_vehicles_sold)) AS PenetrationRate
    FROM
        electric_vehicle.electric_vehicle_sales_by_state es
    JOIN
        electric_vehicle.dim_date dd ON es.date = dd.date
    WHERE
        dd.fiscal_year = 2024
    GROUP BY
        es.State, es.vehicle_category
),
RankedPenetrationRates AS (
    SELECT
        State,
        vehicle_category,
        PenetrationRate,
        RANK() OVER (PARTITION BY vehicle_category ORDER BY PenetrationRate DESC) AS R
    FROM
        PenetrationRateByState
)
SELECT
    State,
    vehicle_category,
    PenetrationRate
FROM
    RankedPenetrationRates
WHERE
    R <= 5
ORDER BY
    vehicle_category, R;

;
```

### Question 4 : What are the quarterly trends based on sales volume for the top 5 EV makers (4-wheelers) from 2022 to 2024?

```sql
WITH QuarterlySales AS (
    SELECT
        esbm.maker,
        dd.fiscal_year,
        dd.quarter,
        SUM(esbm.electric_vehicles_sold) AS total_sales
    FROM
        electric_vehicle.electric_vehicle_sales_by_makers esbm
    JOIN
        electric_vehicle.dim_date dd
    ON
        esbm.date = dd.date
    WHERE
        dd.fiscal_year BETWEEN 2022 AND 2024
    GROUP BY
        esbm.maker,
        dd.fiscal_year,
        dd.quarter
),
MakerTotalSales AS (
    SELECT
        maker,
        fiscal_year,
        SUM(total_sales) AS annual_total_sales
    FROM
        QuarterlySales
    GROUP BY
        maker,
        fiscal_year
),
Top5Makers AS (
    SELECT
        maker
    FROM
        MakerTotalSales
    GROUP BY
        maker
    ORDER BY
        SUM(annual_total_sales) DESC
    LIMIT 5
)
SELECT
    qs.maker,
    qs.fiscal_year,
    qs.quarter,
    qs.total_sales
FROM
    QuarterlySales qs
JOIN
    Top5Makers t5m
ON
    qs.maker = t5m.maker
ORDER BY
    qs.maker,
    qs.fiscal_year,
    qs.quarter;

```

### Question 5 : How do the EV sales and penetration rates in Delhi compare to Karnataka for 2024?

```sql
WITH PenetrationRateByState AS (
    SELECT
        es.state,
        (SUM(es.electric_vehicles_sold) * 100.0 / SUM(es.total_vehicles_sold)) AS PenetrationRate
FROM electric_vehicle.electric_vehicle_sales_by_state es
JOIN electric_vehicle.dim_date dd
ON es.date = dd.date
WHERE dd.fiscal_year = 2024 AND es.state IN ('Delhi','Karnataka')
GROUP BY es.state
)
SELECT state,
       PenetrationRate
FROM PenetrationRateByState
ORDER BY state
```

### Question 6 : List down the compounded annual growth rate (CAGR) in 4-wheeler units for the top 5 makers from 2022 to 2024?

```sql
WITH SalesData AS (
    SELECT
        evsbm.maker,
        dd.fiscal_year,
        SUM(evsbm.electric_vehicles_sold) AS total_sales
    FROM
        electric_vehicle.electric_vehicle_sales_by_makers evsbm
	JOIN electric_vehicle.dim_date dd
    ON evsbm.date = dd.date
    WHERE
        dd.fiscal_year IN (2022, 2024)
	GROUP BY evsbm.maker,
        dd.fiscal_year
),
YearlySales AS (
    SELECT
        maker,
        MAX(CASE WHEN fiscal_year = 2022 THEN total_sales END) AS units_2022,
        MAX(CASE WHEN fiscal_year = 2024 THEN total_sales END) AS units_2024
    FROM
        SalesData
    GROUP BY
        maker
),
CAGR_Calculation AS (
    SELECT
        maker,
        units_2022,
        units_2024,
        CASE
            WHEN units_2022 = 0 THEN NULL
            ELSE (POW((units_2024 / units_2022), (1.0 / (2024 - 2022))) - 1) * 100
        END AS CAGR
    FROM
        YearlySales
	GROUP BY maker, units_2022,
        units_2024
)
SELECT
    maker,
    CAGR
FROM
    CAGR_Calculation
GROUP BY maker, CAGR
ORDER BY
    CAGR DESC
LIMIT 5;
```

### Question 7 : List down the top 10 states that had the highest compounded annual growth rate (CAGR) from 2022 to 2024 in total vehicles sold.

```sql
WITH SalesData AS (
    SELECT
        es.state,
        dd.fiscal_year,
        SUM(es.total_vehicles_sold) total_sold
    FROM
        electric_vehicle.electric_vehicle_sales_by_state es
	JOIN electric_vehicle.dim_date dd
    ON es.date = dd.date
    WHERE
       dd.fiscal_year  IN (2022, 2024)
	GROUP BY es.state,dd.fiscal_year
),
YearlySales AS (
    SELECT
        state,
        MAX(CASE WHEN fiscal_year = 2022 THEN total_sold END) AS total_2022,
        MAX(CASE WHEN fiscal_year = 2024 THEN total_sold END) AS total_2024
    FROM
        SalesData
    GROUP BY
        state
),
CAGR_Calculation AS (
    SELECT
        state,
        total_2022,
        total_2024,
        CASE
            WHEN total_2022 = 0 THEN NULL
            ELSE (POW((total_2024 / total_2022), (1.0 / (2024 - 2022))) - 1) * 100
        END AS CAGR
    FROM
        YearlySales
 GROUP BY state,
        total_2022,
        total_2024
)
SELECT
    state,
    CAGR
FROM
    CAGR_Calculation
GROUP BY state,CAGR
ORDER BY
    CAGR DESC
LIMIT 10;
```

### Question 8 : What are the peak and low season months for EV sales based on the data from 2022 to 2024?

```sql
SELECT
    dd.fiscal_year,
    MONTH(dd.date) AS month,
    SUM(evsbm.electric_vehicles_sold) AS total_units_sold
FROM
    electric_vehicle.electric_vehicle_sales_by_state evsbm
JOIN electric_vehicle.dim_date dd
ON evsbm.date = dd.date
WHERE
    dd.fiscal_year BETWEEN 2022 AND 2024
GROUP BY
    dd.fiscal_year,
    MONTH(dd.date)
ORDER BY
    fiscal_year,
    month;
```

### Question 9 : What is the projected number of EV sales (including 2-wheelers and 4-wheelers) for the top 10 states by penetration rate in 2030, based on the compounded annual growth rate (CAGR) from previous years?

```sql
WITH PenetrationRateByState AS (
    SELECT
        state,
        (SUM(electric_vehicles_sold) * 100.0 / SUM(total_vehicles_sold)) AS PenetrationRate
    FROM 
        electric_vehicle.electric_vehicle_sales_by_state 
    GROUP BY 
        state
    ORDER BY 
        PenetrationRate DESC
    LIMIT 10
),
SalesData AS (
    SELECT
        evsbm.state,
        dd.fiscal_year,
        SUM(evsbm.electric_vehicles_sold) AS total_sales
    FROM
        electric_vehicle.electric_vehicle_sales_by_state evsbm
    JOIN 
        electric_vehicle.dim_date dd ON evsbm.date = dd.date
    WHERE
        dd.fiscal_year IN (2022, 2024)
        AND evsbm.state IN (SELECT state FROM PenetrationRateByState)
    GROUP BY 
        evsbm.state, dd.fiscal_year
),
YearlySales AS (
    SELECT
        state,
        SUM(CASE WHEN fiscal_year = 2022 THEN total_sales END) AS units_2022,
        SUM(CASE WHEN fiscal_year = 2024 THEN total_sales END) AS units_2024
    FROM
        SalesData
    GROUP BY
        state
),
CAGR_Calculation AS (
    SELECT
        state,
        units_2022,
        units_2024,
        CASE
            WHEN units_2022 = 0 THEN NULL
            ELSE (POWER((units_2024 / units_2022), (1.0 / (2024 - 2022))) - 1) * 100
        END AS CAGR
    FROM
        YearlySales
),
Projected_EV_Sales_2030 AS (
    SELECT
        state,
        ROUND(units_2024 * POWER(1 + (CAGR / 100), 2030 - 2024)) AS Proj_EV_sales_2030
    FROM
        CAGR_Calculation
    WHERE 
        CAGR IS NOT NULL
)
SELECT 
    p.state,
    p.Proj_EV_sales_2030,
    pr.PenetrationRate
FROM 
    Projected_EV_Sales_2030 p
JOIN 
    PenetrationRateByState pr ON p.state = pr.state
ORDER BY 
    p.Proj_EV_sales_2030 DESC;
```

### Question 10 : Estimate the revenue growth rate of 4-wheeler and 2-wheelers EVs in India for 2022 vs 2024 and 2023 vs 2024, assuming an average unit price.

![image](https://github.com/user-attachments/assets/0f74c25c-37fc-4097-9c95-27166989d354)

```sql
 ---First, calculate the total revenue for each state and year
WITH RevenueByStateYear AS (
    SELECT
        d.fiscal_year,
        e.state,
        SUM(CASE 
                WHEN e.vehicle_category = '2-Wheelers' THEN e.electric_vehicles_sold * 85000
                WHEN e.vehicle_category = '4-Wheelers' THEN e.electric_vehicles_sold * 1500000
            END) AS total_revenue
    FROM
        electric_vehicle.electric_vehicle_sales_by_state e
    JOIN
        electric_vehicle.dim_date d ON e.date = d.date
    WHERE
        e.vehicle_category IN ('2-Wheelers', '4-Wheelers')
    GROUP BY
        d.fiscal_year, e.state
),

-- Calculate the total revenue for 2-Wheelers by year and state
Revenue2WByYearState AS (
    SELECT
        fiscal_year,
        state,
        SUM(CASE 
                WHEN vehicle_category = '2-Wheelers' THEN electric_vehicles_sold * 85000
            END) AS revenue_2w
    FROM
        electric_vehicle.electric_vehicle_sales_by_state e
    JOIN
        electric_vehicle.dim_date d ON e.date = d.date
    GROUP BY
        d.fiscal_year, e.state
),

-- Calculate the growth rates
GrowthRates AS (
    SELECT
        state,
        MAX(CASE WHEN fiscal_year = 2024 THEN revenue_2w END) AS revenue_2024,
        MAX(CASE WHEN fiscal_year = 2023 THEN revenue_2w END) AS revenue_2023,
        MAX(CASE WHEN fiscal_year = 2022 THEN revenue_2w END) AS revenue_2022
    FROM
        Revenue2WByYearState
    GROUP BY
        state
)

-- Compute the growth rates for each state
SELECT
    state,
    CASE 
        WHEN revenue_2023 IS NOT NULL AND revenue_2023 > 0 THEN
            (revenue_2024 - revenue_2023) / revenue_2023 * 100
        ELSE
            NULL
    END AS growthrate_2w_24vs23,
    
    CASE 
        WHEN revenue_2022 IS NOT NULL AND revenue_2022 > 0 THEN
            (revenue_2024 - revenue_2022) / revenue_2022 * 100
        ELSE
            NULL
    END AS growthrate_2w_24vs22
FROM
    GrowthRates;
```
