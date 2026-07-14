# Synthea-patient-journey-lakehouse
# Patient Journey Analytics — SQL Lakehouse

An end-to-end data lakehouse built on Databricks over synthetic EHR data
(Synthea, ~10,000 patients), implementing the medallion architecture
(Bronze → Silver → Gold) with a focus on advanced SQL: joins, CTEs, and
window functions.

## Architecture
_Bronze (raw ingestion) → Silver (cleaned & conformed) → Gold (star schema + marts)_

<!-- architecture diagram to be added: docs/architecture.png -->

## Data
Synthetic EHR data generated with [Synthea](https://github.com/synthetichealth/synthea).
No real patient data is used. 12+ interrelated tables covering patients,
encounters, conditions, medications, procedures, observations, and claims.

## Showcase Analytics
- 30-day readmission analysis
- Comorbidity pair mining
- Medication persistence
- Care cascade (hypertension funnel)
- CKD cohort vs. control cost comparison

## Tech Stack
Databricks · Unity Catalog · Delta Lake · Spark SQL · Databricks Workflows

## Repository Structure
- `notebooks/` — pipeline and analytics notebooks (Bronze → Gold → analytics)
- `docs/` — architecture and data model diagrams
- `data/sample/` — small runnable sample dataset

## Status
🚧 In progress
