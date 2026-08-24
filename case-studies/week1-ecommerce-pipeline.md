# E-Commerce Analytics Pipeline

## Problem
Every data engineering job posting asks for the same core skill: turning raw, messy data into clean, tested, analytics-ready tables. This project builds that pipeline end to end — from raw CSVs to dashboard-ready marts — using the two most commonly requested tools in DE postings: dbt for transformation and Airflow for orchestration.

## Architecture
Nine raw CSVs (Brazilian e-commerce data, ~100K orders) load into a PostgreSQL `raw` schema via a Python ingestion script. dbt builds a staging layer on top (renamed, typed, cleaned — no business logic), then a marts layer (`fct_orders`, `dim_customers`) with real business logic and 11 automated dbt tests. Apache Airflow orchestrates the full sequence — ingestion → dbt run → dbt test — as a single DAG, with the entire stack reproducible via Docker Compose.

## Tech Stack
PostgreSQL 15 (storage) · Python/pandas/SQLAlchemy (ingestion) · dbt-core (transformation, testing, documentation) · Apache Airflow 2.9 (orchestration) · Docker Compose (reproducibility)

## Key Decisions
1. **Medallion architecture (staging → marts)** — thin staging views isolate cleaning from business logic, keeping transformations modular and independently testable.
2. **dbt tests as the data-quality layer** — primary-key uniqueness, null constraints, and business-rule checks (e.g., spend can't be negative) run automatically on every `dbt test`, rather than relying on manual spot-checks.
3. **Docker Compose for full local reproducibility** — the entire stack (Postgres, Airflow, dbt) starts with one command, so the project is trivially demoable without environment setup friction.

## Results
A fully working, one-command pipeline: 9 raw files landed, transformed through 2 dbt layers, validated by 11 automated tests, and orchestrated by a 3-task Airflow DAG (`load_raw_data` → `dbt_run` → `dbt_test`).

## What I Learned
Real debugging, not just following steps: resolved a dependency conflict between dbt-core and Airflow's pinned SQLAlchemy version by aligning packages against Airflow's official constraints file; fixed Postgres `DependentObjectsStillExist` errors by explicitly cascading drops before re-ingestion, keeping the pipeline idempotent; debugged Docker bind-mount permission issues by redirecting dbt's writable output to container-local paths; configured a shared Airflow secret key across webserver and scheduler containers to enable cross-container log retrieval.

## GitHub
https://github.com/shreya-t-data/ecommerce-analytics-pipeline
