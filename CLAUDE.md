# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Companion code for the "Data Engineering for Beginners" course. Everything runs inside a single Docker container (`airflow-spark`) that bundles Spark 4.0.1 + Iceberg, Airflow 3.1.3 (standalone), dbt-spark, DuckDB and Jupyter Lab. There is no local (host) Python environment and no test suite beyond `dbt test`.

## Commands

All commands except the compose ones run **inside the container** (`docker exec -it airflow-spark bash`). Tools live in the uv venv at `/home/airflow/.venv`; either `source /home/airflow/.venv/bin/activate` or prefix with `uv run` from `/home/airflow` (as the DAG does).

```bash
# Host
docker compose up -d --build      # build + start (Jupyter :8888, Airflow :8080, Spark UI :4040, dbt docs :8081)
docker compose down -v            # stop

# Container: create source data (what notebooks/starter-notebook.ipynb does)
cd /home/airflow/notebooks && python ./generate_data.py && python ./run_ddl.py

# Container: dbt (profiles.yml lives inside the project dir)
cd /home/airflow
uv run dbt run  --profiles-dir tpch_analytics --project-dir tpch_analytics
uv run dbt test --profiles-dir tpch_analytics --project-dir tpch_analytics
uv run dbt run  --profiles-dir tpch_analytics --project-dir tpch_analytics --select customer_outreach_metrics   # single model
uv run dbt test --profiles-dir tpch_analytics --project-dir tpch_analytics --select dim_customer                # single model's tests

# Lint (ruff is installed in the image; no config file)
uv run ruff check . && uv run ruff format --check .
```

## Architecture

**Data flow:** `generate_data.py` (DuckDB `dbgen`, TPC-H sf=0.1 → CSVs) → `run_ddl.py` (drops/recreates Iceberg tables in `prod_db` and loads the CSVs) → dbt models → `dashboard.py` (Plotly HTML). The Airflow DAG `generate_customer_marketing_metrics` runs only the last part: `dbt run >> dbt test >> dbt docs generate >> dashboard.py`. It does **not** generate or load source data — that must be done first from a notebook or the shell.

**Path coupling:** `generate_data.py` writes to `./data` relative to the CWD, and `run_ddl.py` hard-codes `/home/airflow/notebooks/data`. So `generate_data.py` must be run from `/home/airflow/notebooks`. `*.csv` is gitignored.

**Container paths vs repo paths** (bind mounts in `docker-compose.yml`): `airflow/dags` → `/home/airflow/dags`, `airflow/tpch_analytics` → `/home/airflow/tpch_analytics`, `notebooks` → `/home/airflow/notebooks`. `AIRFLOW_HOME` is `/home/airflow`. Code in the repo refers to the container paths.

**Spark / catalog setup** is global, not per-script: `spark_defaults.conf` is copied to `$SPARK_HOME/conf/spark-defaults.conf` and makes the Iceberg Hadoop catalog `local` (warehouse `file:///home/airflow/warehouse`) the default catalog. Every Spark session — notebooks, `run_ddl.py`, dbt (`method: session`, schema `prod_db`), `dashboard.py` — therefore reads and writes the same Iceberg tables. The warehouse is **not** a bind mount, so data is lost when the container is recreated. Changing `spark_defaults.conf`, the `Dockerfile`, or `ipython_scripts/` requires `docker compose up -d --build`; changes to mounted dirs are live.

**Notebooks:** `ipython_scripts/startup/` is copied into the IPython profile: `01-spark-init.py` creates a global `spark` session, `02-sql-magics.py` registers the `%%sql` cell magic (runs `spark.sql`, auto-appends a 1000-row limit unless the query contains "limit", displays via pandas). Notebooks assume both exist. `notebooks/SQL.qmd` / `SQL.html` / `SQL_files/` are Quarto renderings of `SQL.ipynb`; `notebooks/old/` holds earlier course material.

**dbt project** (`airflow/tpch_analytics`): sources are `prod_db.{customer,nation,region,orders,lineitem}` (`models/staging/src.yml`). Layers: `staging/stg_*` (renames TPC-H `c_`/`o_`/`l_` columns to readable names) → `marts/core` (`dim_customer`, `fct_orders`, `fct_lineitem`, then `wide_orders`/`wide_lineitem` joins) → `marts/sales` (`order_lineitem_metrics`, `customer_outreach_metrics`). All models materialize as Iceberg-backed tables (`+materialized: table`). Tests are declared in the `*.yml` files next to the models.
