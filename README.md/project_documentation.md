# ✈️ Travel Agency | Airline Delay & Operational Performance Analysis

An end-to-end business intelligence and data engineering project analyzing 20,000 commercial flights across 12 domestic carriers. This project audits dirty operational data, enforces FAA/BTS aviation reporting standards using Python, models a Star Schema, and delivers an interactive Power BI executive dashboard with actionable recommendations for travel agency operations.

## 📌 Executive Summary

Flight delays and cancellations impose major financial and logistical burdens on travel agencies, corporate travel managers, and individual passengers. When flights run late, agencies face elevated customer support costs, missed connection re-bookings, and hotel voucher claims.

The objective of this project is to analyze operational airline data to answer three critical business questions:

1. **When are delays most likely to occur?** (Temporal & hourly network bottlenecks)

2. **Which airlines perform best and worst on operational punctuality?** (Carrier benchmarking)

3. **What actionable booking policies can a travel agency implement to minimize itinerary disruption?**

## 🛠️ Tech Stack & Architecture

| Layer | Tools / Technologies | Key Activities | 
| ----- | ----- | ----- | 
| **Data Cleaning & Auditing** | Python (`pandas`, `numpy`) | Anomaly detection, domain range auditing, cross-field integrity checks, regex cleaning | 
| **Data Modeling** | Star Schema, DAX | Relationship design (1-to-many), role-playing dimensions, key performance measures | 
| **Business Intelligence** | Microsoft Power BI | Executive KPI cards, trend line charts, carrier ranking bar charts, hourly delay distribution | 
| **Stakeholder Presentation** | Microsoft PowerPoint (`.pptx`) | Business translation, risk mitigation strategy, client communication framework | 
| **Version Control** | Git & GitHub | Modular folder structure, code and report versioning | 

## 📁 Repository Structure

```
├── Dataset/
│   └── data_dictionary.md       # Schema definitions, table relationships, and column data types
├── cleaning and eda.ipynb       # Jupyter Notebook: Data auditing, anomaly detection, cleaning pipeline
├── flight delay analysis.pbix   # Power BI interactive report (Data Model + Visualizations)
├── presentation.pptx            # Executive slide deck for business stakeholders and leadership
└── README.md                    # Project documentation and interview briefing

```

## 🧹 Data Quality Audit & Cleaning Pipeline

Real-world operational telemetry rarely arrives clean. During the initial diagnostic phase in `cleaning and eda.ipynb`, systematic profiling revealed five critical data quality issues that would have severely skewed executive reporting if left unaddressed:

### 1. Extreme Outlier Glitches (`9999` Minutes)

* **Detection:** Running `.describe()` showed standard deviation exploding to over $143$ minutes on arrival delays, with maximum values recorded as $9,999$ minutes (\~7 days).

* **Root Cause:** Upstream sensor timeouts and dummy filler codes placed into numeric fields.

* **Resolution:** Replaced corrupted values with `NaN` (Null). This prevented the mean from being artificially inflated while preserving total flight record counts.

### 2. Physical Domain Violations (Negative Distances)

* **Detection:** Profiling minimum boundaries on `DistanceMiles` identified negative entries (e.g., `-1250`).

* **Root Cause:** Sign-inversion bugs during legacy ETL ingestion. Distance is a scalar physical quantity and must satisfy $\text{Distance} > 0$.

* **Resolution:** Corrected using vectorized `.abs()` transformation after verifying that the absolute magnitude matched valid flights on the identical origin-destination airport pair.

### 3. Inconsistent Identifier Formatting (`FlightNumber`)

* **Detection:** `FlightNumber` was parsed as `object` (string) rather than integer. A regex character scan revealed sporadic strings prefixed with `'FL-'` (e.g., `'FL-1042'`).

* **Root Cause:** Merging data streams from disparate regional partners with custom text prefixes.

* **Resolution:** Stripped leading/trailing whitespace, removed the `'FL-'` substring via `.str.replace('FL-', '', regex=False)`, and cast the column to standard `int64`.

### 4. Cross-Field Business Logic Violations (Cancelled Flight Metrics)

* **Detection:** Filtered for flights where `Cancelled == 1` and evaluated delay metric presence. Found records where cancelled flights still recorded arrival and departure delay minutes.

* **Root Cause:** Legacy flight dispatch software populating expected arrival delay estimates prior to logging final flight cancellation.

* **Resolution:** In strict compliance with US Department of Transportation (DOT) and Federal Aviation Administration (FAA) reporting guidelines, delay metrics for cancelled flights were set to `NaN` using `.loc[Factflight['Cancelled'] == 1, ['DepartureDelayMinutes', 'ArrivalDelayMinutes']] = np.nan`.

### 5. Table Deduplication & Whitespace Trimming

* **Resolution:** Removed duplicate rows based on primary transaction keys using `.drop_duplicates(subset=['FlightKey'], keep='first')` to maintain a strict one-to-many primary key relationship for Power BI modeling. All dimension text attributes (City, AirlineName) were stripped of leading/trailing whitespace.

## 📊 Data Model (Star Schema)

The analytical data model was designed as a Kimball-style Star Schema inside Power BI to guarantee optimal query performance and measure filtering:

```
                  +-------------------+
                  |      DimDate      |
                  +-------------------+
                            | (1)
                            | 
                            | (*)
+-------------------+     +-------------------------+     +-------------------+
|    DimAirline     |-(1)-|       FactFlight        |-(*)-|    DimAirport     |
+-------------------+ (*) +-------------------------+ (1) +-------------------+
                                                              (Role-Playing:
                                                              OriginAirportKey &
                                                              DestAirportKey)

```

* **`FactFlight` (Fact Table):** Contains transactional flight metrics: `FlightKey`, `ScheduledDeparture`, `DistanceMiles`, `DepartureDelayMinutes`, `ArrivalDelayMinutes`, `Cancelled`, `Diverted`.

* **`DimDate` (Dimension):** Bridges calendar hierarchies (`DateKey`, `Year`, `MonthName`, `MonthNumber`, `DayOfWeek`).

* **`DimAirline` (Dimension):** Contains carrier metadata (`AirlineKey`, `AirlineName`, `AirlineType`).

* **`DimAirport` (Dimension):** Contains airport geographical attributes (`AirportKey`, `AirportCode`, `AirportName`, `City`, `State`). Implemented as role-playing dimensions for Origin and Destination.

## 📈 Core Business KPIs & DAX Measures

The following key operational metrics were built to assess network health:

| Metric | Formula / Logic | Project Value | Business Interpretation | 
| ----- | ----- | ----- | ----- | 
| **Total Flights** | `COUNTROWS(FactFlight)` | **20,000** | Total scheduled capacity analyzed across the network. | 
| **On-Time Arrival Rate (A14)** | $\frac{\text{Flights with Arrival Delay} \le 14}{\text{Total Completed Flights}} \times 100$ | **72.88%** | Industry gold standard: flights reaching the gate within 14 minutes of schedule. | 
| **Delayed Flight Rate** | $\frac{\text{Flights with Arrival Delay} > 14}{\text{Total Completed Flights}} \times 100$ | **27.12%** | Proportion of completed operations requiring customer intervention. | 
| **Average Arrival Delay** | `AVERAGE(FactFlight[ArrivalDelayMinutes])` | **8.81 min** | Network-wide average delay across all completed flights. | 
| **Cancellation Rate** | $\frac{\text{Cancelled Flights}}{\text{Total Flights}} \times 100$ | **2.18%** | Baseline disruption rate due to severe weather, mechanical issues, or crew shortages. | 

## 🔍 Key Findings & Operational Insights

### 1. The Afternoon Delay Cascade (Time-of-Day Risk)

* **Morning Operations (05:00 – 11:00):** Flights departing before midday demonstrate exceptional operational resilience, with average delays hovering consistently low between **4.1 and 5.4 minutes**.

* **The Afternoon Surge (14:00 – 22:00):** Beginning at 14:00 (2:00 PM), average delays jump abruptly to **14.0 minutes** and peak at **15.1 minutes** around 17:00 (5:00 PM), remaining elevated above 13.5 minutes through the night.

* **Driver:** Aircraft rotations and knock-on propagation. A minor delay in an early leg compounds across subsequent flight segments due to gate hold-ups, crew duty-hour limits, and congested airspace.

### 2. Airline Reliability Ranking

* **Top Performers (Lowest Average Delay):**

  1. Frontier Airlines: **8.1 min**

  2. Spirit Airlines: **8.3 min**

  3. Hawaiian Airlines: **8.4 min**

  4. United Airlines: **8.5 min**

* **Lagging Performers (Highest Delay Burden):**

  1. Endeavor Air: **10.1 min**

  2. Alaska Airlines: **9.8 min**

  3. Allegiant Air: **9.6 min**

  4. Southwest Airlines: **9.2 min**

### 3. Monthly Seasonality & Disruption Patterns

* **Delay Variance:** Peak average arrival delays occurred in **February (9.5 minutes)** and **March (9.4 minutes)**, while **April** achieved the lowest average delay of the year (**8.1 minutes**).

* **Cancellation Shocks:** Cancellations peaked sharply in **January (2.81%)** and **March (2.60%)**, driven primarily by severe winter storms and airport ground stop programs. Late summer (July–August) saw cancellation rates drop to their annual lows (\~1.64% – 1.65%).

## 💡 Strategic Business Recommendations (Travel Agency Action Plan)

1. **Implement "Morning Preferred" Policy for High-Stakes Travel:**

   * Travel agents should configure corporate travel booking engines to prioritize flight departures before 11:00 AM. Clients booking morning flights experience an average delay of \~4.5 minutes, compared to \~14.5 minutes for afternoon/evening flights.

2. **Buffer Management for Connection Windows:**

   * For itineraries booked after 14:00, enforce an automatic minimum connection time (MCT) buffer of at least **75–90 minutes** (up from standard 45-minute windows) to prevent missed connections caused by the afternoon delay cascade.

3. **Carrier-Specific Risk Assessment:**

   * When pricing package tours or corporate travel contracts, flag regional feeder carriers (e.g., Endeavor Air at 10.1 min avg delay) with higher delay probability warnings compared to mainline carriers.

4. **Proactive Winter Disruption Protocols:**

   * During January and February, automatically prompt clients with comprehensive trip-protection insurance options and automated re-booking alert services to combat the elevated 2.8% cancellation rate.

## 🎯 How to Run & Reproduce This Project

### 1. Prerequisites

* Python 3.9+ with `pandas`, `numpy`, and `jupyter`

* Microsoft Power BI Desktop (Free version)

* Microsoft PowerPoint (to view `presentation.pptx`)

### 2. Execution Steps

1. **Clone the repository:**

   ```
   git clone https://github.com/DasAnurag64/Airline-Delay-Analysis.git
   cd Airline-Delay-Analysis
   
   ```

2. **Run Data Cleaning & Pipeline:**
   Open and execute all cells in `cleaning and eda.ipynb` to inspect data profiles, anomaly detection logic, and generated clean exports.

3. **Open Power BI Dashboard:**
   Launch `flight delay analysis.pbix` in Power BI Desktop to interact with the visualizations, slicers, and data models.

4. **Review Stakeholder Deck:**
   Open `presentation.pptx` for the business summary slide deck.