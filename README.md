# Fingertips Public Health Profiles – AWS Ingestion

This repository contains code to ingest selected indicators from the **Fingertips** public health profiles platform into an AWS data lake (Bronze/Silver). Fingertips, maintained by the Office for Health Improvement and Disparities (OHID), is a rich source of indicators on health, wellbeing, and wider determinants of health for local areas in England.  [oai_citation:1‡Fingertips](https://fingertips.phe.org.uk/?utm_source=chatgpt.com)

---

## Source dataset

- **Name:** Fingertips – Public Health Profiles  
- **Owner:** Office for Health Improvement and Disparities (OHID), Department of Health and Social Care  
- **Portal:** Fingertips public health profiles  [oai_citation:2‡Fingertips](https://fingertips.phe.org.uk/?utm_source=chatgpt.com)  
- **Licence:** Usually Open Government Licence (OGL v3.0) – always check the latest licence text on the source site before use.  

Fingertips provides:
- Hundreds of indicators across multiple themed “profiles” (e.g. wider determinants, healthcare, health protection).
- Data at local authority level and sometimes smaller geographies.
- Time series for many indicators.

---

## Geographic coverage & granularity

- **Coverage:** England  
- **Levels (depending on indicator):**  
  - Local Authorities / Upper & Lower Tier  
  - Regions  
  - Integrated Care Boards (ICBs) (for some newer series)  
  - England total  

---

## Time coverage & refresh

- **Historical coverage:** Varies by indicator (often 10+ years).  
- **Update frequency:** Irregular but typically annual or quarterly, depending on indicator.  
- This repo is designed so the same pipeline can be re-run when new extractions are downloaded.

---

## This project’s data model

We use a simple Lakehouse-style model on S3:

- **Bronze layer**  
  - Raw extracts from Fingertips downloads/API, as close to source as possible.  
  - Minimal transformation: add file metadata, standardise column names where safe.  

- **Silver layer**  
  - Cleaned, type-cast tables suitable for analysis and downstream dashboards.  
  - Common fields such as:
    - `indicator_id` / `indicator_name`  
    - `area_code` / `area_name` / `area_type`  
    - `time_period` (and/or `year`, `quarter`)  
    - `value`, `lower_ci`, `upper_ci`, `denominator`, `numerator` (where available)

---

## Repository contents

- **`bronzepipelinelatest.ipynb`** – Notebook to download, standardise, and upload Fingertips extracts into the Bronze S3 area.  
- **`silver_glue_file`** – Glue script / notebook for transforming Bronze data into Silver tables.  
- **`uploade_s3.ipynb`** – Utility notebook for pushing local files to S3.

(Adjust file names above if you rename things.)

---

## How to use

1. Configure AWS credentials and S3 bucket paths in the notebook parameters or environment.  
2. Run the Bronze pipeline notebook to ingest raw Fingertips CSVs into S3.  
3. Run the Silver Glue job / notebook to produce cleaned, analysis-ready tables.  
4. Point Athena/Tableau/Power BI at the Silver tables for analysis and dashboarding.

---

## Attribution

Please cite Fingertips / OHID / DHSC when publishing any analysis from this dataset, and always respect the Open Government Licence conditions shown on the source website.  [oai_citation:3‡Fingertips](https://fingertips.phe.org.uk/?utm_source=chatgpt.com)
