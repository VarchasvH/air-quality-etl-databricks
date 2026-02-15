# Air Quality ETL Pipeline

## Overview
I wanted to get comfortable with the Databricks platform and practice building ETL pipelines with real, messy data from India. Air quality data felt relevant given how bad the air has been lately.

This pipeline processes data from 490 monitoring stations across India, transforming raw CSV files from data.gov.in through Bronze-Silver-Gold layers. The end result is clean, analysis-ready data for calculating AQI scores and ranking stations by pollution levels.

This is my first Databricks/PySpark project. I'm starting with batch processing to nail the fundamentals - cleaning messy data, handling nulls, and building proper data architecture - before moving to streaming pipelines.

## AQI Calculation Methodology

**Following CPCB (Central Pollution Control Board) standards:**

The pipeline calculates Air Quality Index using the official Indian AQI formula:
- Defines breakpoint ranges for all 7 pollutants (PM2.5, PM10, SO2, NO2, CO, O3, NH3)
- Calculates sub-index for each pollutant using linear interpolation formula
- Overall AQI = Maximum of all sub-indices (worst pollutant determines air quality)
- Assigns categories: Good (0-50), Satisfactory (51-100), Moderate (101-200), Poor (201-300), Very Poor (301-400), Severe (401-500)

**Implementation:** Pure PySpark expressions (no UDFs) for optimal performance and scalability.

## Architecture

The pipeline follows the Medallion Architecture pattern:

- **Bronze Layer**: Ingests raw CSV data without any transformations, preserving the original structure and adding ingestion metadata (3,276 rows).

- **Silver Layer**: Cleans and transforms the data:
  - Converts "NA" string values to proper nulls using `try_cast`
  - Parses string timestamps to proper timestamp type using custom format patterns
  - Pivots from long format (one row per pollutant) to wide format (one row per station)
  - Result: 490 rows with 15 columns (metadata + 7 pollutants)

- **Gold Layer**: Provides business-ready analytics:
  - Calculates AQI scores using CPCB methodology for all 490 stations
  - Ranks stations by pollution levels with risk categories
  - Identifies dominant pollutant for each station (which pollutant causes highest AQI)
  - City-level aggregations showing average AQI and station counts
  - Result: Clean analytical tables ready for dashboards and reporting

## Challenges I Faced

1. **Pivot confusion** - Kept losing my metadata columns until I realized I needed to add them ALL to groupBy, not just station.
2. **Cast errors with "NA"** - First tried `when()` + `cast()` but it threw errors. Had to research and find `try_cast` instead.
3. **Timestamp parsing** - Dates kept coming up null because I used MM-dd-yyyy instead of dd-MM-yyyy (Indian format).
4. **Understanding why nulls are okay** - At first wanted to drop all nulls, then realized that's throwing away valid data.
5. **CPCB formula implementation** - Converting the official AQI calculation (with multiple breakpoint ranges per pollutant) into efficient PySpark expressions required careful design to avoid performance issues.

## Data Quality Issues Found & Solutions

1. **"NA" string values in pollutant columns**
   - Problem: Pollutant min/max/avg contained "NA" as strings instead of proper nulls
   - Solution: Used `try_cast` to convert strings to doubles, automatically handling "NA" → null (166 affected records)

2. **Incorrect timestamp format**
   - Problem: `last_update` stored as string in DD-MM-YYYY format instead of proper timestamp
   - Solution: Used `to_timestamp()` with custom format pattern "dd-MM-yyyy HH:mm:ss" to parse correctly

3. **Long format structure**
   - Problem: Data had one row per pollutant per station (3,276 rows), making analysis difficult
   - Solution: Pivoted on `pollutant_id` to create one row per station with 7 pollutant columns (490 rows)

4. **Missing pollutant measurements**
   - Problem: Not all stations measured all 7 pollutants, resulting in nulls after pivot
   - Solution: Kept nulls in Silver layer; filtered out incomplete records in Gold layer for business users

## Design Decisions

**Scalability:**
- Pure PySpark expressions (no UDFs) - scales horizontally to millions of rows without code changes
- Modular helper functions with breakpoints defined as data structures

**Data Quality:**
- Nulls preserved in Silver to maintain data integrity
- Stations with insufficient measurements excluded from Gold layer analytical tables
- Clear separation between raw (Bronze), cleaned (Silver), and analytical (Gold) data

**Optimization Choices:**
- No partitioning: Dataset size (490 rows) too small; partitioning would create small file overhead
- Views for city aggregations: Always current data, no storage duplication
- Z-ordering deferred: Would only add value at scale (millions of rows with frequent queries)
- In production with larger datasets, would partition by state and date, Z-order by overall_aqi

## Technologies Used
- **Databricks Free Edition**: Cloud-based data engineering platform
- **PySpark**: Distributed data processing framework
- **Delta Lake**: ACID transactions, time travel, and efficient storage
- **Unity Catalog**: Data governance and metadata management
- **Python**: Core programming language

## Dataset
- **Source**: data.gov.in
- **Date**: February 14, 2026 (single-day snapshot)
- **Pollutants Measured**: SO2, CO, NO2, OZONE, PM2.5, PM10, NH3
- **Stations**: 490 monitoring locations across Indian cities
- **Coverage**: Multiple states including Bihar, Maharashtra, Delhi, Karnataka, and more

## What I Learned

- **PySpark DataFrame operations**: Working with groupBy, pivot, window functions, and aggregations to transform data at scale
- **Data quality handling**: Real-world data is messy - learned to identify and clean issues like NA values, incorrect types, and missing data
- **Delta Lake**: Understanding medallion architecture (Bronze→Silver→Gold) and why each layer has a specific purpose
- **Pivot mechanics**: How to transform long format data to wide format while preserving all necessary columns
- **Timestamp parsing**: Converting string dates to proper timestamp types with custom format patterns for non-standard formats
- **Production optimization**: Using pure PySpark expressions instead of UDFs for optimal performance and scalability
- **Business logic implementation**: Translating complex CPCB AQI formula (with 35+ breakpoint ranges) into efficient, maintainable code
- **Architectural decisions**: When to use views vs tables, how to handle nulls for business users, when optimizations actually matter

## Project Structure
```
air-quality-etl-databricks/
├── notebooks/
│   ├── 01_bronze_layer.py       # Raw data ingestion with metadata
│   ├── 02_silver_layer.py       # Data cleaning & transformation
│   └── 03_gold_layer.py         # AQI calculations & business analytics
├── data/
│   └── aqi_data.csv             # Source data (single-day snapshot)
└── README.md
```

## Tables Created

**Bronze Layer:**
- `aqi_bronze` - Raw data with ingestion timestamp (3,276 rows)

**Silver Layer:**
- `aqi_silver` - Cleaned, pivoted data ready for analytics (490 rows)

**Gold Layer:**
- `gold_station_aqi` - Station-level AQI with rankings, categories, and dominant pollutants
- `gold_city_aqi` - City-level aggregations (view)

## Future Enhancements (v2.0)

If expanding this project in the future, could add:
- Historical data (multi-day analysis)
- Real-time API ingestion with scheduling
- Delta Live Tables implementation
- Interactive dashboards (Tableau/Power BI)
- Predictive modeling for AQI forecasting

**Current Status:** Production-ready batch ETL pipeline ✓
