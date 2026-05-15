# SQL Analytics: Costa Rica Crime Data (2024-2026)

## Overview
This document shows advanced SQL techniques used to transform raw incident reports from the Organismo de Investigación Judicial into actionable business intelligence. 

The queries below demonstrate proficiency in **Window Functions**, **Common Table Expressions (CTEs)**, and **Data Aggregation** to solve complex, real-world business questions regarding public safety resource allocation.

---

### 1. Identifying Risk Leaders (Ranking)
**The Business Question:** Law enforcement leadership needs to identify the top 3 most dangerous cantons in *each* province to prioritize budget allocation for homicide prevention.

**Technical Approach:** Utilized a CTE to aggregate total volume, followed by `DENSE_RANK()` partitioned by province to ensure accurate, gapless ranking even in the event of tied crime volumes.

```sql
WITH RankedHomicides AS (
    SELECT 
        provincia,
        canton,
        COUNT(*) AS volume,
        DENSE_RANK() OVER (PARTITION BY provincia ORDER BY COUNT(*) DESC) AS rank_c
    FROM 
        crime_stats
    WHERE 
        delito = 'HOMICIDIO'
    GROUP BY 
        1, 2
)
SELECT provincia, canton, volume, rank_c
FROM RankedHomicides
WHERE rank_c <= 3;
```
### 2. Month-to-Date (MTD) Incident Tracking (Running Totals)

**The Business Question:** Operations managers need to track the daily cumulative growth of vehicle thefts in the capital (San José) to see if early-month interventions are slowing the overall trend.

**Technical Approach:** Applied the SUM() aggregate as a window function ordered by date to calculate a dynamic running total without collapsing the daily rows. Filtered cleanly using BETWEEN to isolate Q1 data.


```sql
SELECT 
    fecha AS date_of_incident,
    COUNT(*) AS daily_incidents,
    SUM(COUNT(*)) OVER (ORDER BY fecha ASC) AS running_total
FROM 
    crime_stats
WHERE 
    delito = 'ROBO DE VEHICULO' 
    AND provincia = 'SAN JOSE'
    AND fecha BETWEEN '2026-01-01' AND '2026-01-31'
GROUP BY 
    1;
```
### 3. Day over Day

**The Business Question:** Did nationwide assaults increase or decrease today compared to yesterday? We need to calculate the daily variance for automated reporting.

**Technical Approach:** Leveraged the LAG() offset function within a secondary CTE to pull the previous day's aggregate into the current row, enabling simple mathematical variance calculation without complex self-joins.

```sql
WITH cantidad_asaltos AS (
    SELECT fecha, COUNT(*) AS asaltos_hoy
    FROM crime_stats
    WHERE delito = 'ASALTO'
    GROUP BY 1
),
asaltos_ayer AS (
    SELECT 
        fecha, 
        asaltos_hoy,
        LAG(asaltos_hoy, 1) OVER (ORDER BY fecha) AS asaltos_ayer
    FROM cantidad_asaltos 
)
SELECT 
    fecha, 
    asaltos_hoy, 
    asaltos_ayer,
    (asaltos_hoy - asaltos_ayer) AS diferencia_diaria
FROM asaltos_ayer;
```

### 4. Automated Risk Classification

**The Business Question:** To simplify dashboard filters for non-technical users, categorize all cantons nationally into 4 equal-sized "Risk Tiers" (Critical to Low) based on historical incident volume.

**Technical Approach:** Implemented the NTILE(4) window function to automatically distribute pre-aggregated geographical data into quartiles, creating a new categorical dimension for BI tools.

```SQL
WITH INCIDENTS AS (
    SELECT canton, COUNT(*) AS incidents
    FROM crime_stats
    GROUP BY 1 
)
SELECT 
    canton,
    incidents,
    NTILE(4) OVER(ORDER BY incidents DESC) AS risk_tier
FROM INCIDENTS;
```
### 5. Tactical Growth Report: Spatiotemporal Isolation

**The Business Question:** The Intelligence Director requires a Q1 2026 MoM (Month-over-Month) assault report for San José. It must show the monthly volume, the previous month's volume, and the quarterly running total, isolated perfectly for each specific canton.

**Technical Approach:** Combined multiple window functions (LAG and SUM() OVER) operating simultaneously. Critically applied PARTITION BY canton paired with ORDER BY month inside the OVER() clause to ensure calculations reset at geographic boundaries, preventing data cross-contamination.

```SQL
WITH total_asaltos_mes AS (
    SELECT  
        canton,
        EXTRACT(MONTH FROM fecha) AS mes_del_incidente,
        COUNT(*) AS asaltos_del_mes
    FROM 
        crime_stats
    WHERE 
        delito = 'ASALTO' 
        AND provincia = 'SAN JOSE'
        AND fecha BETWEEN '2026-01-01' AND '2026-03-31'
    GROUP BY 
        1, 2
)
SELECT 
    canton,
    mes_del_incidente,
    asaltos_del_mes,
    LAG(asaltos_del_mes) OVER(PARTITION BY canton ORDER BY mes_del_incidente) AS asaltos_mes_anterior,
    SUM(asaltos_del_mes) OVER(PARTITION BY canton ORDER BY mes_del_incidente) AS acumulado_trimestral
FROM 
    total_asaltos_mes;
```
