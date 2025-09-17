# ETA-ETL — Taxi Trip ETL & Analytics (PySpark notebook)

### Note: public Kaggle dataset used in this project `yellow_tripdata_2015-01.csv`, to be placed in data folder after downloading from: 

https://www.kaggle.com/datasets/elemento/nyc-yellow-taxi-trip-data?resource=download. 

**Short description:** A compact end-to-end ETL and analytics notebook that demonstrates ingesting NYC yellow taxi CSV, cleaning & schema enforcement, deriving features (trip time, speed, tip %), writing partitioned Parquet, running windowed aggregations (top drivers per day), showing broadcast join patterns, caching performance effects, and producing a small set of business-ready artifacts & visuals.

---

## Project goals

- Show a clean, reproducible Spark ETL pipeline for trip-level telemetry.
- Enforce schema, defensive parsing of timestamps and invalid rows.
- Derive analytics-ready features (trip duration, avg speed, tip %).
- Demonstrate Big Data patterns: partitioned Parquet, predicate pushdown, broadcast joins, window functions, caching & explain plan.
- Produce concise outputs: top drivers per day CSV, per-day trip counts plot, dataset summary CSV.

---

## What’s included

- `etl_analytics.ipynb` — the Jupyter/Colab notebook with all steps (Spark session, ingestion, cleaning, feature derivation, writes, aggregations, caching tests, visualization).
- `data/output/dataset_summary.csv` — dataset-level summary (num_trips, avg_trip_distance, avg_fare_amount, min/max timestamps).
- This `README.md`.

---

## Technologies & libs used

- Apache Spark (PySpark)
- Python 3.x
- pandas, matplotlib (for smaller visualizations)
- python-dateutil (defensive timestamp parsing)
- Parquet (snappy compression)
- (Optional) Google Colab or a similar notebook environment

---

## Prerequisites

- Java 8/11 (for Spark)
- Python 3.8+
- PySpark installed (matching local Spark version), e.g. `pip install pyspark`.
- pandas, matplotlib, python-dateutil: `pip install pandas matplotlib python-dateutil`

If you use Google Colab, PySpark can be installed in a cell; code in notebook assumes local paths `/content/...` but can be adapted.

---

## How to run (quick)

1. Clone repo:
   ```bash
   git clone https://github.com/your-username/eta-etl-spark
   cd eta-etl-spark
   ```

2. Put your `yellow_tripdata_2015-01.csv` file under `data/` or update the path in the notebook.

3. Create a Python env and install dependencies:
   ```bash
   python -m venv venv
   source venv/bin/activate
   pip install pyspark pandas matplotlib python-dateutil
   ```

4. Launch Jupyter Notebook and open `etl_analytics.ipynb`:
   ```bash
   jupyter notebook
   ```

5. Run notebook cells sequentially. (Notebook includes helpful code and comments.)

---

## Notebook sections & explanations

1. ### Spark session setup
   - Creates a `SparkSession` with tuned options:
     - `spark.sql.shuffle.partitions = 8` (small demo setting — tune for cluster size)
     - Arrow enabled for some conversions
   - Explanation: configure partitions appropriately for the dataset / environment.

2. ### Read CSV into DataFrame
   - Read CSV with explicit `schema` to avoid expensive schema inference and string-typed surprises.
   - Defensive parsing: timestamps are string initially; convert via `to_timestamp` with correct format or use a UDF only if needed.
   - Explanation: reading with schema prevents Spark from doing extra scans and yields correct dtypes.

3. ### Cleaning + schema enforcement
   - Normalize boolean flags, cast timestamps to `TimestampType`, compute `pickup_date`.
   - Filter out bad rows: `trip_distance > 0`, dropoff > pickup, `fare_amount >= 0`, `passenger_count > 0`.
   - Explanation: ensures only meaningful trips keep flowing through the pipeline.

4. ### Feature derivation (trip_time, speed, tip_pct)
   - `trip_minutes` and `trip_hours` computed using timestamp cast to long.
   - `avg_speed_mph` = `trip_distance / trip_hours` (rounded).
   - `tip_pct` computed as `tip_amount / fare_amount`.
   - Explanation: features often used for downstream analytics / anomaly detection.

5. ### Partitioned Parquet write (by date / driver_mod)
   - Simulate `driver_mod` using `crc32` of pickup lat/lon modulo N (for demo partitioning).
   - Write Parquet partitioned by `pickup_date` and `driver_mod` using Snappy compression.
   - Explanation: partitioned Parquet enables fast predicate pushdown and partition pruning on reads.

6. ### Windowed aggregation (top drivers per day)
   - Group by `pickup_date` & `driver_mod`, `sum(total_amount)`, then window and `row_number()` to get top 5 per date.
   - Explanation: window functions are common for ranking and leaderboards.

7. ### Broadcast join example (small driver metadata)
   - Create a tiny `driver_meta` DF and broadcast join against top drivers to demonstrate small-table join patterns.
   - Explanation: broadcasting small reference tables avoids large shuffle.

8. ### Caching + explain() plan + timing
   - Demonstrates effect of `cache()`:
     - Example timings observed: **Without cache ≈ 4.92s**, **With cache ≈ 0.71s** (your numbers will vary).
   - Use `q.explain(True)` to inspect logical/physical plans, and demonstrate how filtering/operations flow into plan.

9. ### Read-back & partition pruning test
   - Read Parquet and filter by `pickup_date` to show partition pruning improves read time (example: `Read date 2015-01-11: 28612 rows in 3.90s`).
   - Explanation: partition pruning avoids scanning irrelevant files.

10. ### Visualization & outputs
   - Convert aggregated per-day trips to pandas and plot (simple line chart).
   - Save `top_drivers_by_day_csv` and dataset summary CSV.

---

## Key results (example numbers from notebook run)

- Cleaned rows count: **411,746** trips
- Caching speed-up (sample): `4.92s → 0.71s`
- Partition read example: read 28,612 rows in ~3.90s using partition pruning
- Top drivers per day: exported to CSV for downstream analysis

---

## Conclusions & business insights

- Partitioning by `pickup_date` (and a driver hash) provides strong read performance for date-scoped queries.
- Caching intermediate cleaned DataFrame can significantly speed up repeated analyses.
- Windowed aggregation reveals consistent top-performing driver partitions per day — useful for resource allocation and incentive analysis.
- Feature engineering (e.g., `avg_speed_mph`, `tip_pct`) uncovers suspicious low/high speed trips and tip-behavior patterns that could inform fraud detection or driver training programs.

---

## Improvements & next steps

1. **Avoid Python UDFs** where possible — prefer built-in Spark functions for better performance and serialization.
2. **Use Delta Lake** (or Hive/Glue tables) to enable ACID writes, time travel, and efficient compaction.
3. **Autoscaling & Cluster Tuning** — set `spark.sql.shuffle.partitions` proportional to cluster cores; tune executor memory and cores.
4. **More robust partitioning scheme** — consider `year/month/day` partitions for larger datasets; use `driver_id` if real.
5. **Add unit tests & CI pipelines** — create small sample datasets and test transforms using `pytest`.
6. **Monitoring & Observability** — enable Spark UI, logs, and metrics export; track job durations and shuffle sizes.
7. **Production orchestration** — orchestrate notebook as DAG (Airflow / Prefect) with clear staging to S3/GCS.
8. **Data quality checks** — implement Deequ / Great Expectations checks to validate schema and data ranges.
9. **Model & anomaly detection** — attach a streaming pipeline or batch ML model for anomaly detection on trip times and speeds.
10. **Security & privacy** — if using real data, ensure PII is masked and access is controlled.

---

## Reproducibility notes

- All numerical values are sample-run dependent (hardware, Spark configs, and dataset size matter).
- The notebook uses small `spark.sql.shuffle.partitions` for demo; increase in real deployments.
- Use the same Spark and PySpark versions in production or within a container/conda environment for consistency.

---

## License & contact

- **License:** Apache
- **Maintainer:** Krish — [github.com/KrishT97](https://github.com/KrishT97)  
