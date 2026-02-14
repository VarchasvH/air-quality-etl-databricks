# Air Quality ETL Pipeline

## Overview
This project builds an end-to-end ETL pipeline that processes real-time air quality data from 490 monitoring stations across India. Using data from data.gov.in (Bihar State Pollution Control Board), the pipeline transforms raw pollutant measurements into analytics-ready datasets for calculating AQI scores, identifying pollution hotspots, and ranking stations by air quality levels.

## Architecture

The pipeline follows the Medallion Architecture pattern:

- **Bronze Layer**: Ingests raw CSV data without any transformations, preserving the original structure and adding ingestion metadata (3,276 rows).

- **Silver Layer**: Cleans and transforms the data:
  - Converts "NA" string values to proper nulls using `try_cast`.
  - Parses string timestamps to proper timestamp type.
  - Pivots from long format (one row per pollutant) to wide format (one row per station).
  - Result: 490 rows with 15 columns (metadata + 7 pollutants).

- **Gold Layer**: *(In Progress)* Will provide business-ready analytics:
  - Calculate AQI scores for each station.
  - Rank stations and cities by pollution levels.
  - Identify most polluted areas.
  - Generate time-series trends.

## Data Quality Issues Found & Solutions

1. **"NA" string values in pollutant columns**
   - Problem: Pollutant min/max/avg contained "NA" as strings instead of proper nulls.
   - Solution: Used `try_cast` to convert strings to doubles, automatically handling "NA" → null (166 affected records).

2. **Incorrect timestamp format**
   - Problem: `last_update` stored as string in DD-MM-YYYY format instead of proper timestamp.
   - Solution: Used `to_timestamp()` with custom format pattern "dd-MM-yyyy HH:mm:ss" to parse correctly.

3. **Long format structure**
   - Problem: Data had one row per pollutant per station (3,276 rows), making analysis difficult.
   - Solution: Pivoted on `pollutant_id` to create one row per station with 7 pollutant columns (490 rows).

4. **Missing pollutant measurements**
   - Problem: Not all stations measured all 7 pollutants, resulting in nulls after pivot.
   - Solution: Kept nulls as-is in Silver layer; will handle appropriately in Gold layer analytics.

## Technologies Used
- **Databricks**: Cloud-based data engineering platform
- **PySpark**: Distributed data processing
- **Delta Lake**: ACID transactions and time travel for data lake
- **Python**: Core programming language

## Dataset
- **Source**: data.gov.in
- **Date**: February 14, 2026 snapshot
- **Pollutants Measured**: SO2, CO, NO2, OZONE, PM2.5, PM10, NH3
- **Stations**: 490 monitoring locations across Indian cities

## What I Learned

- **PySpark DataFrame operations**: Working with groupBy, pivot, and aggregations to transform data at scale.
- **Data quality handling**: Real-world data is messy - learned to identify and clean issues like NA values and incorrect types.
- **Delta Lake**: Understanding medallion architecture (Bronze→Silver→Gold) and why each layer has a specific purpose.
- **Pivot mechanics**: How to transform long format data to wide format for analytical use cases.
- **Timestamp parsing**: Converting string dates to proper timestamp types with custom format patterns.

## Project Structure
```
air-quality-etl-databricks/
├── notebooks/
│   ├── 01_bronze_layer.py       # Raw data ingestion
│   ├── 02_silver_layer.py       # Data cleaning & transformation
│   └── 03_gold_layer.py         # Analytics (in progress)
├── data/
│   └── aqi_data.csv             # Source data
└── README.md
```

## Next Steps
- [ ] Complete Gold layer with AQI calculations.
- [ ] Add data visualizations.
- [ ] Upgrade to API ingestion for real-time updates.
- [ ] Implement historical data loading.
- [ ] Add automated data quality tests.
