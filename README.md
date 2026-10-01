# Airline Flight Booking Data Warehouse

A data warehousing project that transforms airline booking data into a dimensional model for SQL analysis using Python and SQLite.

The project demonstrates staging, surrogate key lookups, incremental fact loading, and Slowly Changing Dimension (SCD) Type 2 processing for passenger information.

## Project Overview

Operational databases support everyday booking transactions. This project creates a separate analytical database to explore booking amounts, flight performance, destinations, routes, and passenger loyalty tiers.

The ETL pipeline extracts records from the source database, refreshes staging tables, loads dimensions, and inserts new booking facts.

| Detail | Description |
|---|---|
| Module | CCS3307 — Data Warehousing |
| Group | 20 |
| Domain | Airline and travel |
| Business process | Flight ticket booking |
| Technologies | Python, SQLite, SQL |
| Warehouse design | One fact table and four dimension tables |
| Historical tracking | SCD Type 2 for passenger attributes |

## Key Features

- Operational source database with passenger, airport, flight, and booking tables.
- Staging tables separating extraction from warehouse loading.
- Dimensional model for analytical queries.
- Surrogate keys linking facts to dimensions.
- Passenger history tracking using SCD Type 2.
- New booking insertion with duplicate checks.
- Date dimension supporting booking-date and flight-date analysis.
- SQL reports for flight, destination, route, and loyalty-tier analysis.

## Data Pipeline

1. Read passenger, airport, flight, and booking records from the source database.
2. Clear and reload the corresponding staging tables.
3. Insert new dimension records.
4. Detect changes in passenger attributes and create historical versions.
5. Look up dimension keys for each new booking.
6. Calculate the total booking amount.
7. Insert bookings that are not already present in the fact table.

**Loading approach:** Source extraction and staging refresh are full loads.
Booking facts are loaded incrementally by checking `booking_id`.

## Warehouse Design

### Fact Table

**Table:** `Fact_Flight_Booking`

**Grain:** One row per booking record, identified by `booking_id`, for one passenger and one flight.

| Measure | Meaning |
|---|---|
| `ticket_fare` | Base ticket fare |
| `taxes_and_fees` | Additional charges |
| `baggage_weight_kg` | Recorded baggage weight |
| `distance_km` | Flight distance associated with the booking |
| `total_amount` | Ticket fare plus taxes and fees |

```text
total_amount = ticket_fare + taxes_and_fees
```

Flight distance is repeated for each booking. Summing it across bookings
does not represent the total distance flown by the airline's aircraft.

### Dimension Tables

| Dimension | Purpose |
|---|---|
| `Dim_Passenger` | Passenger details and historical attribute versions |
| `Dim_Airport` | Airport name, city, and country |
| `Dim_Flight` | Flight number, aircraft type, and distance |
| `Dim_Date` | Calendar attributes for time-based analysis |

`Dim_Airport` is used twice: for departure and arrival airports.

`Dim_Date` is used twice: for booking date and flight date.

These are role-playing dimensions: the same dimension supports different
relationships with the fact table.

## Passenger History — SCD Type 2

The ETL checks changes to:

- Passenger name
- Gender
- Nationality
- Loyalty tier

When an attribute changes:

1. The existing current record is marked as historical.
2. Its `effective_end_date` is set to the ETL run date.
3. A new record is inserted with a new surrogate key.
4. The new record receives `is_current = 1`.

### Example

Passenger `P001` changes loyalty tier from Silver to Gold.

| Passenger ID | Loyalty Tier | Current Status |
|---|---|---|
| P001 | Silver | Historical — `is_current = 0` |
| P001 | Gold | Current — `is_current = 1` |

Previously loaded booking facts retain their existing passenger keys.
New booking facts use the passenger version marked as current when loaded.

**Implementation detail:** Effective dates reflect ETL processing dates.
The pipeline does not perform booking-date-based historical version lookup.

## Repository Guide

| Path | Contents |
|---|---|
| `db/` | Source and warehouse SQLite databases |
| `sql/create_source.sql` | Operational database schema |
| `sql/insert_source_data.sql` | Initial demonstration records |
| `sql/create_warehouse.sql` | Dimensions, fact table, and staging tables |
| `sql/run2_source_changes.sql` | Changes for the second demonstration run |
| `sql/analytical_queries.sql` | Analytical SQL reports |
| `etl/etl_pipeline.py` | Python ETL implementation |
| `diagrams/` | Schema and architecture diagrams |
| `screenshots/` | Demonstration screenshots |

## Requirements

- Python 3 with the standard-library `sqlite3` module
- Git, or a downloaded copy of the repository
- A SQLite client for executing SQL scripts and inspecting results

The ETL script uses Python's standard library and does not require
third-party Python packages.

## Getting Started

### 1. Clone the Repository

```bash
git clone https://github.com/balakrishnanpremnath/airline-data-warehouse.git
cd airline-data-warehouse
```

Run the following commands from the repository root because the ETL
uses relative database paths.

### 2. Choose the Database Starting Point

To inspect or run the existing database files:

```bash
python etl/etl_pipeline.py
```

The existing databases may already contain demonstration results.

For a clean, reproducible demonstration, move any existing
`db/airline_source.db` and `db/airline_warehouse.db` files into a backup
folder first. Keep the `db/` directory.

Then use the SQLite command-line client to initialize new databases.

Create the source database:

```bash
sqlite3 db/airline_source.db
```

At the SQLite prompt:

```sql
.read sql/create_source.sql
.read sql/insert_source_data.sql
.quit
```

Create the warehouse database:

```bash
sqlite3 db/airline_warehouse.db
```

At the SQLite prompt:

```sql
.read sql/create_warehouse.sql
.quit
```

The `sqlite3` command-line client is separate from Python's `sqlite3`
module. Alternatively, use a SQLite GUI to create the database files
and execute the same SQL scripts.

Run schema and initial-data scripts only against fresh databases.

### 3. Run the Initial ETL Load

```bash
python etl/etl_pipeline.py
```

The initial sample contains:

| Source Entity | Records |
|---|---:|
| Passengers | 10 |
| Airports | 6 |
| Flights | 8 |
| Bookings | 20 |

Starting from an empty warehouse, the expected results include
20 booking facts and 10 passenger dimension records.

### 4. Apply the Second Demonstration Change

Open the source database:

```bash
sqlite3 db/airline_source.db
```

Execute:

```sql
.read sql/run2_source_changes.sql
.quit
```

This script:

- Changes passenger `P001` from Silver to Gold.
- Adds passenger `P011`.
- Adds booking `B021`.

Apply this change script once per clean demonstration. Its insert
statements are not designed for repeated execution.

### 5. Run the ETL Again

```bash
python etl/etl_pipeline.py
```

Expected results after the second load:

| Check | Expected Result |
|---|---:|
| Booking fact records | 21 |
| Distinct passengers | 11 |
| Passenger dimension records, including history | 12 |
| Dimension versions for P001 | 2 |
| Current passenger records | 11 |

### 6. Check Repeat-Run Behaviour

Run the ETL once more without modifying the source:

```bash
python etl/etl_pipeline.py
```

Booking and passenger-version counts should remain unchanged.

## Validation Queries

Run these queries against `db/airline_warehouse.db`.

### Count Booking Facts

```sql
SELECT COUNT(*) AS booking_count
FROM Fact_Flight_Booking;
```

### Inspect Passenger History

```sql
SELECT
    passenger_sk,
    passenger_id,
    loyalty_tier,
    effective_start_date,
    effective_end_date,
    is_current
FROM Dim_Passenger
WHERE passenger_id = 'P001'
ORDER BY passenger_sk;
```

### Check for Duplicate Bookings

```sql
SELECT
    booking_id,
    COUNT(*) AS record_count
FROM Fact_Flight_Booking
GROUP BY booking_id
HAVING COUNT(*) > 1;
```

Expected result: no rows.

### Check Current Passenger Versions

```sql
SELECT
    passenger_id,
    SUM(CASE WHEN is_current = 1 THEN 1 ELSE 0 END) AS current_versions
FROM Dim_Passenger
GROUP BY passenger_id
HAVING SUM(CASE WHEN is_current = 1 THEN 1 ELSE 0 END) <> 1;
```

Expected result: no rows.

## Analytical Reporting

The project includes queries for:

- Total booking amount by flight
- Total booking amount by passenger loyalty tier
- Total booking amount by destination
- Daily booking amounts
- Total booking amount by route

Here, booking amounts include ticket fares and taxes/fees. They should
not automatically be interpreted as accounting revenue or profit.

### Example: Booking Amount by Flight

```sql
SELECT
    f.flight_number,
    COUNT(*) AS total_bookings,
    ROUND(SUM(b.total_amount), 2) AS total_booking_amount
FROM Fact_Flight_Booking AS b
JOIN Dim_Flight AS f
    ON b.flight_sk = f.flight_sk
GROUP BY f.flight_number
ORDER BY total_booking_amount DESC;
```

Additional queries are available in
[`sql/analytical_queries.sql`](sql/analytical_queries.sql).

## Current Scope and Limitations

- This is an academic demonstration using a small sample dataset.
- Each run extracts all source records and refreshes staging.
- Existing booking facts are skipped; changes or deletions to those
  bookings are not synchronized.
- Airport and flight dimensions insert new records but do not update
  existing records when their attributes change.
- Passenger history uses load dates rather than source event timestamps.
- Late-arriving bookings are linked to the current passenger version,
  rather than a version selected by the booking date.
- The ETL commits in stages rather than as one atomic transaction.

## Future Improvements

- Add automated validation tests for duplicate prevention and SCD history.
- Add structured logging and error handling.
- Validate missing dimension references before fact insertion.
- Add transaction management for safer recovery from failed loads.
- Support historical passenger lookups for late-arriving bookings.
- Introduce change-based extraction for larger datasets.
- Connect the warehouse to a BI dashboard.

## Skills Demonstrated

- Relational database design
- Dimensional modelling
- SQL querying and aggregation
- Python ETL development
- Staging-table management
- Surrogate key lookups
- SCD Type 2 processing
- Incremental fact loading
- Data validation

## My Contribution

This project was submitted as a Group 20 academic assignment.
I independently completed the full technical implementation, including:

- Source database and data warehouse schema design
- Python ETL pipeline development
- SCD Type 2 passenger history tracking
- Incremental booking loads and duplicate prevention
- Analytical SQL queries
- Validation, diagrams, and project documentation

## Academic Credits

Developed for **CCS3307 — Data Warehousing**, Group 20.

| Member | Student ID |
|---|---|
| Premnath | CIT-24-01-0241 |
| Afrith | CIT-24-01-0297 |
| Aasim | CIT-24-01-0298 |
| Himas | CIT-24-01-0302 |

## Contact

**Balakrishnan Premnath**  
BSc (Hons) in Data Science — Sri Lanka Technology Campus

[GitHub](https://github.com/balakrishnanpremnath) |
[LinkedIn](https://www.linkedin.com/in/balakrishnan-premnath)
