# Hospitality Revenue Engine: MotherDuck to Power BI

## Overview
This project is an end-to-end data pipeline and business intelligence dashboard designed for hotel operations. It tracks the "Total Guest Wallet"—moving beyond standard room reservations to integrate Food & Beverage (F&B), Spa, and Event revenues into a single, unified daily pacing model. 

The architecture bridges a cloud-based **MotherDuck** data warehouse with **Power BI**, providing visibility into property performance and departmental revenue mix.

## Tech
* **Database / Data Warehouse:** MotherDuck (Cloud DuckDB)
* **Data Transformation:** SQL (Views and Aggregations)
* **Business Intelligence:** Power BI (Semantic Modeling, DAX, Interactive Visualization)

## Data Architecture & Modeling Challenge
When dealing with multiple transactional systems in hospitality (PMS, PoS, Catering), BI tools often struggle with errors or circular dependencies when linking multiple daily performance tables to a single calendar. 

**The Solution:** Rather than forcing complex cross relationships in Power BI that consume server memory, the data model was flattened at the warehouse level. A "Super Table" view (`vw_total_property_revenue`) was created in MotherDuck using `FULL OUTER JOIN` logic. This completely bypasses Power BI's relationship engine limits, ensuring perfect alignment between Rooms Occupancy and ancillary revenues (F&B, Events) without dotted-line errors.

## SQL Pipeline (MotherDuck)
Below is the core SQL script executed in MotherDuck to consolidate the isolated departmental data into the master unified view used by Power BI.

```sql
-- Create a unified daily performance view
CREATE OR REPLACE VIEW vw_total_property_revenue AS
SELECT 
    COALESCE(r.performance_date, f.date, e.date) AS date,
    COALESCE(r.property_id, f.property_id, e.property_id) AS property_id,
    
    -- Rooms
    MAX(r.occupancy_rate) AS occupancy_rate,
    MAX(r.total_room_revenue) AS rooms_revenue,
    
    -- Inc Revenue Streams (F&B, Spa, Events)
    SUM(f.fnb_revenue) AS fnb_revenue,
    SUM(f.spa_revenue) AS spa_revenue,
    SUM(e.total_event_revenue) AS event_revenue,
    
    -- Total Guest Wallet
    (MAX(r.total_room_revenue) + SUM(f.fnb_revenue) + SUM(f.spa_revenue) + SUM(e.total_event_revenue)) AS total_operating_revenue

FROM vw_property_daily_performance r
FULL OUTER JOIN vw_fnb_daily_performance f 
    ON r.performance_date = f.date AND r.property_id = f.property_id
FULL OUTER JOIN vw_events_daily_performance e 
    ON (COALESCE(r.performance_date, f.date) = e.date) AND (COALESCE(r.property_id, f.property_id) = e.property_id)
GROUP BY 
    date, 
    property_id;
```

## Power BI Dashboard Structure
The dashboard is navigated via an interactive button menu (Page Navigator) and is divided into distinct operational views:

### 1. Executive Summary (Total Property Revenue)
* **Objective:** A pulse on the property's financial health.
* **Key Visuals:** 
  * Master Pacing Chart overlaying Average Occupancy Rate against a stacked column of all revenue streams.
  * Guest Wallet Mix (Donut Chart) visualizing the percentage split of Rooms vs. F&B vs. Events.
  * Property Leaderboard Matrix with conditional color scaling for cross-portfolio comparison.

### 2. Rooms & Loyalty
* **Objective:** Deep dive into inventory yield and loyalty program contribution.
* **Key Visuals:** 
  * Hierarchical Date Slicer (Year > Quarter > Month).
  * Dual-Axis Line/Column chart mapping RevPAR against Occupancy limits.
  * Premium vs. Standard inventory matrix to track suite upgrade efficiency and transient vs. loyalty member revenue split.

### 3. F&B & Events Scorecard
* **Objective:** Track the financial impact of group blocks and outlet traffic.
* **Key Visuals:** 
  * Event Revenue Matrix tracking performance by venue and event type (Corporate vs. Social).
  * KPI Scorecards for Total Event Revenue, Distinct Event Count, and Average Revenue per Event.

## Key Takeaways
By pushing the heavy lifting of the data transformation back to **MotherDuck** via SQL, the **Power BI** semantic model remains incredibly lightweight. This ensures the dashboard loads instantly in the cloud environment, requires zero complex DAX routing for date filtering, and maintains absolute data integrity across all hospitality departments.
