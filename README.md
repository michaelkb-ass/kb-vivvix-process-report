# kb-vivvix-process-report
A Python utility to automate the cleaning, ingestion, and generation of quarterly Vivvix-code submission files for the US PRO's (ASCAP, BMI, SESAC)

# Ad Claim Report Processor

A Python utility designed to streamline the processing of North American music advertisement usage reports. This script automates the ingestion of raw data, cleans and formats it for a centralized database, and generates quarterly submission-ready spreadsheets tailored for specific Performing Rights Organizations (PROs) like ASCAP, BMI, and SESAC.

## Overview

This codebase addresses the workflow of processing music usage reports for royalty claims. It involves a two-step process:

1.  **Ingestion (`NA_Report` class):** Takes raw report files (Excel or CSV), validates columns, cleans the data (removes commas, formats dates), and appends it to a master ingestion file. This master file acts as a staging area for a data warehouse (e.g., AWS Athena).
2.  **Submission Generation (`Report` class):** Reads a processed, enriched report (generated from the ingested data via an external query, e.g., Athena), filters for new and controlled works, and transforms the data into the specific formats required by ASCAP, BMI, and SESAC, saving them as separate Excel files.

## Features

-   **Handles Multiple Formats:** Ingests both `.xlsx` and `.csv` raw report files.
-   **Data Cleaning & Formatting:** Automatically formats datetime columns to be compatible with AWS Athena and removes disruptive characters like commas from text fields.
-   **Automated Submission File Generation:** Creates tailored `.xlsx` files for ASCAP, BMI, and SESAC with the correct column names and order.
-   **De-duplication:** Checks against a master list of previous submissions to ensure only new ad usages are processed for the current quarter.
-   **AWS Integration:** Designed to work with an AWS Athena backend for querying and enriching the ingested data (`tandbi_aws_utils`).

## Workflow

1.  **Place Raw Report:** A new raw report (e.g., from BMAT) is placed in the `MASTER_RAW` directory.
2.  **Run Ingestion Script:** An instance of the `NA_Report` class is created, and the `ingest_to_master()` method is called. This cleans the raw data and appends it to the `bmat_na_reports - Copy.csv` master file.
3.  **Data Processing (External):** The data from the master ingestion file is loaded into a database/data lake. The `reg_file_generate()` method (which uses `aws.athena_get_ad_registrations`) is likely used to query this data, join it with internal registration information, and produce a "processed" CSV file in the `MASTER_REG_DIR`.
4.  **Generate Submission Files:** An instance of the `Report` class is created using the name of the processed file. The `process_quarter()` method is called for the desired year and quarter.
5.  **Output:** The script generates the final, society-specific `.xlsx` files in the `OUTPUT_DIR`, organized into subfolders for each PRO.

## Prerequisites

-   Python 3.7+
-   `pandas` library
-   `openpyxl` (for `.xlsx` support with pandas)
-   An internal AWS utility library (`tandbi_aws_utils`) with dependencies (e.g., `boto3`, `awswrangler`).
-   Properly configured AWS credentials with access to the required Athena databases and S3 buckets.
-   Access to the network drive (`Z:\...`) where the data files are stored.

## Installation

1.  **Clone the repository:**
    ```bash
    git clone <kb-vivvix-process-report>
    cd <your-repository-name>
    ```

2.  **Install the required Python libraries:**
    ```bash
    pip install pandas openpyxl
    ```

3.  **Install internal dependencies:**
    Ensure the `tandbi_aws_utils` library is installed and configured in your Python environment.

## Usage

The script is structured into two main classes. The following shows a typical execution flow.

### Step 1: Ingest a New Raw Report

To process a raw report file (e.g., `NA_Report_Q1_2025.xlsx`), use the `NA_Report` class.

```python
from report_processor import NA_Report # Assuming the code is in a file named report_processor.py

# The file name should be without the extension
report_file_name = 'NA_Report_Q1_2025'

# Initialize the report object
ingestion_report = NA_Report(file_name=report_file_name)

# Ingest the data into the master file
ingestion_report.ingest_to_master()

# Optionally, print some stats from the raw file
ingestion_report.read_raw()
```

### Step 2: Generate Quarterly Submission Files

After the ingested data has been processed by the external Athena query and a processed file (e.g., `processed_registrations_Q1_2025.csv`) is ready, use the `Report` class.

```python
from report_processor import Report # Assuming the code is in a file named report_processor.py

# The file name of the processed report, without extension
processed_file_name = 'processed_registrations_Q1_2025'

# Initialize the submission report object
submission_generator = Report(file_name=processed_file_name)

# Generate submission files for Q1 2025
submission_generator.process_quarter(year=2025, quarter='Q1')
```

## Configuration

All file paths and directories are hardcoded as class variables at the top of the script. Before running, you may need to update these paths to match your local environment or network configuration.

**Key paths to configure:**

-   `NA_Report.MASTER_RAW`
-   `NA_Report.MASTER_INGEST`
-   `Report.MASTER_SUBMISSIONS_FILE`
-   `Report.MASTER_REG_DIR`
-   `Report.OUTPUT_DIR`

## Athena Tables

- 'tracking_and_bi - bmat_na_reports: kobalt-athena-user-data-prod -----> raw files directly from BMAT'
- 'tracking_and_bi - bmat_na_reports_workids -----> raw files directly from BMAT appended with internal SAX Work Ids'
- 'tracking_and_bi - bmat_na_reports_work_registrations -----> bmat_na_reports_workids appended with registrations information from SAX'
- 'tracking_and_bi - bmat_controlled_works -----> bmat_na_reports_work_registrations ----> a filtered view of work_registrations to identify controlled and non-controlled works'
- 'rdl_authorised - royalty_payment_detail_bmat -----> royalty_payment_detail filtered for TV Ad-Revenue from the United States: powers the data extracts in Tableau'

### Future Improvements

-   Move hardcoded file paths to a `config.ini` or `.env` file for easier management.
-   Implement a logging framework (e.g., `logging`) instead of using `print()` statements for better monitoring and debugging.
-   Add more comprehensive error handling for file I/O and data processing steps.
-   Write unit tests to validate the data transformation logic for each PRO.
