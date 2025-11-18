# Data Cleaning and Transformation Overview (Power BI)

This document outlines the main steps used to clean, transform, and enrich the airline dataset using Power Query 
(M language). The transformations were designed to integrate multiple sources (routes, flights, capacities, bookings, 
aircraft, and airports), derive analytical fields, and create standardized structures for downstream modeling 
and visualization.

## 1. Initial Data Cleaning and Standardization

Before combining and transforming data across the airline datasets, each source table underwent a structured initial 
cleaning process in Power Query to ensure consistency, accuracy, and compatibility across joins. All datasets originate
from our dataset that we made publicly available in a GitHub repository (publiusTacitus/airline_data), which provides 
simulated but realistic airline operations data for analytical and educational use.  

For example, the `airports` table was imported from CSV, where header rows were promoted, inconsistent airport names 
were standardized (e.g., simplifying “São Paulo/Guarulhos–Governador André Franco Montoro International Airport” to 
“São Paulo International Airport”), and the coordinate string was split into separate `latitude` and `longitude` fields 
with proper numeric data types. The `flights` table was similarly imported and cleaned: columns were typed explicitly 
(dates, datetimes, logicals, integers), a `Year` field was extracted from `flight_date`, and the dataset was filtered to 
include only flights from the **2023** operating period.  

The `weather` data underwent column type normalization to ensure numeric treatment of temperature, wind speed, and 
visibility values while maintaining text fields for seasonal and condition descriptions. Finally, the `bookings` 
table – read from a local or remote CSV – had headers promoted and each column cast to its appropriate type, 
including `Currency.Type` for prices and logical values for check-in and refund indicators. Together, these 
preprocessing steps established a clean, type-safe foundation for subsequent joins, aggregations, and calculations 
across the broader airline data model.  
The rest of the initial data cleaning was done in a similar fashion.  

Following the initial cleaning and standardization, the process transitioned into the data transformation phase, which 
focused on joining related tables, calculating derived metrics such as profitability and occupancy rates, and 
structuring the data for analytical modeling.

## 2. Routes Table Transformation

### Objective

Enhance the raw `routes` dataset by adding descriptive fields, categorizing routes by distance, and merging airport 
metadata.

### Main Steps

1. **Create route name**  
Added a new column route_name combining departure and arrival airport codes in a human-readable format:

    ````   
        [departure_airport_code] & " → " & [arrival_airport_code]
    ````    
   
2. **Categorize flight distances**   
Derived a new column `distance_category` to classify routes:

   - `(1) Short-haul` → distance < 1500 km
   - `(2) Medium-haul` → 1500–4999 km
   - `(3) Long-haul` → ≥ 5000 km

3. **Merge airport metadata (departure & arrival)**  
Joined with `airport_names_clean` twice to attach airport details: 

   - `departure_airport_code` → fetches departure info
   - `arrival_airport_code` → fetches arrival info  

   Expanded relevant fields such as:
   - Airport name
   - City, country, coordinates
   - Climate region and hub status

### Result

An enriched `routes` table containing readable route labels and both departure and arrival airport metadata.

## 3. Flights Table Transformation

### Objective

Combine operational flight data with cabin configurations, passenger totals, and on-time performance indicators.

### Main Steps

1. **Join with cabin configurations**  
Merged on `flight_number` to add `cabin_configuration`.
2. **Join with class capacity and bookings**  
Merged with `flight_capacity_by_class`, then aggregated the total bookings per flight: 

   ````
       Table.AggregateTableColumn("flight_capacity_by_class", {{"class_bookings", List.Sum, "bookings_total"}})
   ````

3. **Add on-time performance indicator**  
Derived a numeric flag `on_time_numeric`:  

   - `1` → Flight arrived ≤15 minutes late
   - `0` → Flight arrived >15 minutes late
   - `null` → Missing actual arrival time

### Result

A structured `flights` table linking configuration, capacity, and performance metrics.

## 4. Cabin Configuration

### Objective

Clean and summarize class-level flight capacity data for the year 2023.

### Main Steps

1. Filter data: Exclude rows with zero bookings.
2. Group by flight number: Nested all related class records per flight into a single table column (`Classes`).
3. Create class summary text: Added a `cabin_configuration` column listing all unique cabin classes for each flight:

   - Flights with a single class: `Economy Only`
   - Multiple classes: `Economy, Business`, `Economy, Business, First`
   
4. Removed intermediate columns to keep only summarized results.

### Result

A tidy dataset summarizing class structures per flight, labeled for easy joining with flight-level data.

## 5. Bookings and Financial Performance Transformation

### Objective

Aggregate bookings data and compute flight-level financial and performance metrics.

### Main Steps

1. Aggregate revenue and booking counts: 

   - `revenue_` = sum of `price_paid` per flight/date
   - `bookings` = total number of bookings
   
2. Merge flight operational and cost data:  

   - Joined with `flights_booked_passengers` for capacity/passenger metrics.
   - Joined with `costs_per_flight` to incorporate total cost values.

3. Handle cancelled flights: Nullified revenue and cost values for cancelled flights to avoid bias.
4. Join with aircraft data: Added `seat_capacity` to enable utilization calculations.
5. Compute performance metrics: 

   - Booked rate:
        ````
            booked_rate = bookings_total / seat_capacity
        ````
   - Average booked rate per line number: Aggregated across flights for each line.
   - Total revenue and cost sums: Calculated cumulative figures, followed by profit and margin.
   
6. Add tier classifications: 

   - Booked Performance Tier:
   
     - (A) Top Performance → > 85%
     - (B) Within Target → 76–85%
     - (C) Sufficient → 70–75%
     - (D) Underperforming → < 70%
     - (E) Unsustainable → < 60%
     
   - Profitability Tier:  
     - (A) ≥ 25%
     - (B) 20–24%
     - (C) 15–19%
     - (D) 5–14%
     - (E) < 5%

### Result

A comprehensive performance summary at the flight-line level, integrating revenue, cost, utilization, and profitability 
insights.

## 6. Flight-Level Revenue and Cost Integration

### Objective

Construct a unified flight-level performance table that combines booking revenue, passenger counts, operational status,
and cost breakdowns – the foundation for all higher-level profitability analyses.

### Main Steps

1. Aggregate booking data per flight and date:  

   - Grouped the `bookings` table by `{flight_number, flight_date}`.
   - Computed:
   
     - `revenue_ = SUM(price_paid)`
     - `bookings = COUNT(rows)`
     
2. Merge with operational flight data:  
Joined to `flights_booked_passengers` to bring in: 

   - `line_number`, `aircraft_id`, `bookings_total`, `passengers_total`
   - `cancelled`, `delay_reason_dep`, `delay_reason_arr`, `cabin_configuration`, `on_time_numeric`
   
3. Attach cost components:  

   - Left-joined `costs_per_flight` on `{flight_number, flight_date}`.
   - Expanded cost fields:  
   `flight_cost_total_`, `fuel_cost_`, `crew_cost_`, `maintenance_cost_`, `landing_fees_`, `catering_cost_` 
   , `other_costs_`
   
4. Adjust financial values for cancellations:  
For cancelled flights, all financial metrics were set to `null`:

    ````
        revenue = if cancelled then null else revenue_
        flight_cost_total = if cancelled then null else flight_cost_total_
        fuel_cost = if cancelled then null else fuel_cost_
        ...
    ````
   This ensures no cancelled flights inflate revenue or cost aggregates.
5. Clean intermediate columns:  
Removed temporary columns (the underscored originals like `revenue_`, `flight_cost_total_`, etc.) to keep 
the dataset tidy.
6. Add aircraft capacity:  
Joined with the `aircraft` table to retrieve `seat_capacity`, stored as `nominal_seat_capacity`.

### Result

A clean, per-flight-level table integrating:

- Revenue and cost structure
- Bookings and passengers
- Cancellation and delay indicators
- Aircraft capacity

This dataset serves as the core input for class-level profitability, route-level KPIs, and operational summaries.

## 7. Class-Level Revenue and Profit Analysis

### Objective:

Break down profitability and utilization performance by cabin class within each route, integrating cost share 
allocations and capacity data.

### Main Steps:

1. Aggregate revenue by flight and class:  
Grouped `bookings` by `{flight_number, class_name}`, summing `price_paid` into `revenue_`.
2. Merge with flight-level performance data:  
Joined with the flight-level integration table from Section 5 to access flight-level cost, cancellation, 
and route information.
3. Distribute total flight costs by class:  

   - Merged with `flight_class_cost_shares` to obtain `cost_share` (proportion of total flight cost 
   attributed to each class).
   - Used cost allocation formula:  
   
    ````
       if not cancelled then flight_cost_total_ * cost_share else null
    ````

4. Combine with class-level capacity:  
Joined with `flight_capacity_by_class` to retrieve `class_capacity` and total bookings.
5. Aggregate by route line and class:  
Summarized data by `{line_number, class_name}`, summing revenue, cost, capacity, and bookings.
6. Derive class-level KPIs:  

   - `avg_booked_rate = total_class_bookings / total_class_capacity`
   - `total_profit = total_revenue - total_flight_cost_overall`
   - `avg_profit_margin = total_profit / total_revenue`
   
7. Assign performance tiers:  

   - Booked performance: tiered (A-E) by utilization rate.
   - Profitability: tiered (A-E) by profit margin.

### Result

A structured dataset showing profitability, utilization, and cost efficiency per cabin class, forming the analytical
layer beneath route-level summaries.

## 8. Weather Hazard Aggregation

### Objective

Integrate airport metadata with weather observations and summarize the frequency of extreme conditions per climate 
region and season.

### Main Steps

1. Join with airport metadata:  
Linked weather data to `airports` to get `airport_name` and `climate_region`.
2. Engineer hazard flags:  
Added binary fields for key weather risks:

   - Fog, Blizzard, Sandstorm
   - Extreme Cold (`< -25°C`), Extreme Heat (`> 45°C`)
   - Strong Wind (`> 70 km/h`), Low Visibility (`< 1 km`)
   
3. Aggregate hazards by region and season:  
Summed total occurrences across `{climate_region, airport, season}`.
4. Unpivot hazard categories:  
Converted wide-format hazard indicators into a tidy `{hazard_type, hazard_count}` structure.

### Result

A climate-level hazard frequency table is useful for risk visualization and regional performance correlation.

## 9. Flight-Level Operational Summary

### Objective

Generate daily and seasonal operational summaries, combining the integrated flight table with delay and cost details.

### Main Steps

1. Add operational indicators:  
Added flags:  

   - `cancelled_numeric`
   - `delayed_departure`
   - `delayed_arrival`
   
2. Extract schedule features:  
Derived `day_of_week` and `travel_season` based on `flight_date`.
3. Adjust for cancellations:  
Set `actual_seat_capacity = 0` when `cancelled = true`.
4. Aggregate by route and configuration:  
Grouped by `{line_number, cabin_configuration, day_of_week, travel_season}` to compute totals for:  

   - Flights, capacities, bookings, passengers, and revenue
   - Costs, cancellations, and delay counts
   
5. Derive KPIs:  
Profit, profit margin, booked and occupancy rates, check-in gaps.
6. Add route-level tier classifications:   
Joined with `route_tiers` for contextual benchmarking.

### Result

A multi-dimensional performance table spanning revenue, cost, delay, and route utilization across temporal 
and configuration dimensions.

## 10. Aircraft Utilization Summary

### Objective

Summarize aircraft deployment and usage across time and seasonality.

### Main Steps

1. Filter active flights:  
Excluded cancelled flights.
2. Extract scheduling fields:  
Added `day_of_week` and `travel_season`.
3. Join with aircraft info:  
Added aircraft `model` metadata.
4. Aggregate usage:  
Counted flights per `{line_number, model, day_of_week, travel_season}`.

### Result

Provides aircraft activity patterns for fleet-level utilization analysis.

## 11. Delay Analysis by Cause and Season

### Objective:

Analyze delay frequency and magnitude by cause, weekday, and season.

### Main Steps:

1. Filter operational flights:  
Removed cancelled entries.
2. Compute delay duration:  

    ````
    delay_mins = Duration.TotalMinutes(actual_arrival - scheduled_arrival)
    ````
   Filtered out ≤ 15-minute differences as on-time.
3. Extract delay context:  
Kept `delay_reason_dep`, `delay_reason_arr`, and schedule features.
4. Combine and group:    
Merged departure and arrival delay causes, aggregated by:
`{line_number, delay_reason_dep, delay_reason_arr, day_of_week, travel_season}`
5. Summarize:   

   - flight_count
   - avg_delay_minutes
   - max_delay_minutes

### Result:

A delay-focused dataset for root-cause and seasonal performance diagnostics.