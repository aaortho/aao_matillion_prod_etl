# LMS Accreditation Load Pipeline Documentation

## Pipeline Overview

**Pipeline Name:** `pl_lms_accreditation_load.orch.yaml`  
**Pipeline Type:** Orchestration  
**Purpose:** Extract accreditation data from LMS API, load into Snowflake bronze layer, and process into silver layer with comprehensive audit logging

### Business Description

This pipeline orchestrates the end-to-end ingestion and processing of LMS (Learning Management System) accreditation data. It connects to an external LMS API to extract accreditation records, loads the raw data into a bronze layer table, then parses and transforms the JSON data into a structured format in the silver layer. The pipeline includes robust error handling and audit logging to track data lineage and processing status.

### Data Flow Architecture

```
LMS API → Bronze Layer (RAW JSON) → Silver Layer (STRUCTURED) → Audit Log
```

- **Source**: LMS API Accreditation endpoint
- **Bronze Layer**: `PROD_BRONZE_DB.LMS.ACCREDITATION` (raw JSON data)
- **Silver Layer**: `PROD_SILVER_DB.LMS.ACCREDITATION` (parsed and structured data)
- **Audit**: `PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG` (processing status and metrics)

---

## Pipeline Variables

| Variable | Type | Scope | Visibility | Default Value | Description |
|----------|------|-------|------------|---------------|-------------|
| `v_starttime` | TEXT | SHARED | PUBLIC | `2022-01-01T00:00:00Z` | Start timestamp for API data extraction window |
| `v_endtime` | TEXT | COPIED | PUBLIC | `2023-01-01T00:00:00Z` | End timestamp for API data extraction window |

---

## Component Details

### 1. Start Component

**Type:** `start`  
**Purpose:** Initiates the pipeline execution

- **Flow:** Unconditionally transitions to LMS ACCREDITATION component
- **Configuration:** Standard start component with no special parameters

---

### 2. LMS ACCREDITATION (API Extract & Load)

**Type:** `modular-api-extract-input-v2`  
**Purpose:** Extracts accreditation data from LMS API and loads into Snowflake bronze layer

#### API Configuration
- **Profile:** `custom-b95fee52-8db0-4fe4-94d1-623e1ca69a28`
- **Endpoint:** `Accreditation`
- **Authentication:** None (authType: NONE)
- **Page Limit:** 100 records per request
- **Log Level:** ERROR

#### Query Parameters
- `start_date`: Uses `${v_starttime}` variable to define extraction window start
- `end_date`: Uses `${v_endtime}` variable to define extraction window end

#### Snowflake Output Configuration
- **Database:** `[Environment Default]` (PROD_BRONZE_DB)
- **Schema:** `LMS`
- **Table:** `ACCREDITATION`
- **Load Mode:** `TRUNCATE_AND_INSERT` (replaces existing data)
- **Stage Platform:** Snowflake internal staging
- **Clean Staged Files:** Yes

#### Transitions
- **Success:** → Log Accreditation Ingestion
- **Failure:** → Log Accreditation API Failure

---

### 3. Log Accreditation Ingestion

**Type:** `sql-executor`  
**Purpose:** Records successful data ingestion metrics in audit log

#### SQL Operation
```sql
INSERT INTO PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG (
    TABLE_NAME,
    ROW_COUNT,
    LOAD_TIME,
    LOAD_STATUS,
    ERROR_MESSAGE
)
SELECT 
    'ACCREDITATION' AS TABLE_NAME,
    COUNT(*) AS ROW_COUNT,
    CURRENT_TIMESTAMP AS LOAD_TIME,
    'SUCCESS' AS LOAD_STATUS,
    NULL AS ERROR_MESSAGE
FROM PROD_BRONZE_DB.LMS.ACCREDITATION;
```

#### Functionality
- Counts records successfully loaded into bronze layer
- Records timestamp of successful completion
- Logs "SUCCESS" status for audit trail

#### Transitions
- **Success:** → Data Parser Accreditation

---

### 4. Data Parser Accreditation

**Type:** `sql-executor`  
**Purpose:** Parses JSON data from bronze layer and upserts into structured silver layer table

#### Key Processing Logic

##### JSON Parsing Strategy
- Uses `LATERAL FLATTEN` to parse nested JSON structures
- Extracts accreditation records and credit information
- Handles complex nested objects (user, related_test, product, credits)

##### Data Extraction Fields

**Accreditation Fields:**
- `id`, `awarded_on`, `certificate_id`

**User Information:**
- `user_id`, `user_email`, `api_username`, `first_name`, `last_name`

**Test Information:**
- `test_id`, `test_type`, `score_percentage`, `attempts`, `completed_on`

**Product Information:**
- `product_id`, `product_name`, `product_type`, `product_identification`, `product_api_key`

**Credit Information:**
- `credit_type_id`, `credit_name`, `credit_unit`, `credit_amount`

##### MERGE Operation
- **Match Condition:** `target.id = source.id`
- **When Matched:** Updates all fields with latest values
- **When Not Matched:** Inserts new accreditation record
- **Target Table:** `PROD_SILVER_DB.LMS.ACCREDITATION`

---

### 5. Log Accreditation API Failure

**Type:** `sql-executor`  
**Purpose:** Records API extraction failures in audit log for troubleshooting

#### SQL Operation
```sql
INSERT INTO PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG (
    TABLE_NAME,
    ROW_COUNT,
    LOAD_TIME,
    LOAD_STATUS,
    ERROR_MESSAGE
)
VALUES (
    'ACCREDITATION',
    -1,  -- Negative count indicates failure
    CURRENT_TIMESTAMP,
    'FAILED',
    'Failed: Unknown error occurred during LMS Accreditation API processing'
);
```

#### Functionality
- Logs failure status with timestamp
- Uses negative row count (-1) to indicate failure
- Provides error context for debugging
- No downstream transitions (terminal failure state)

---

## Pipeline Flow Diagram

```
┌─────────┐    ┌──────────────────┐    ┌─────────────────────────┐    ┌────────────────────────┐
│  Start  │───▶│ LMS ACCREDITATION│───▶│ Log Accreditation       │───▶│ Data Parser            │
└─────────┘    │ (API Extract)    │    │ Ingestion (Success)     │    │ Accreditation          │
               └──────────────────┘    └─────────────────────────┘    │ (JSON → Structured)    │
                        │                                              └────────────────────────┘
                        │ (on failure)
                        ▼
               ┌─────────────────────────┐
               │ Log Accreditation API   │
               │ Failure (Error Log)     │
               └─────────────────────────┘
```

---

## Data Quality & Monitoring

### Audit Trail
- All processing attempts are logged in `LMS_AUDIT_LOG`
- Success logs include actual row counts processed
- Failure logs include error messages and timestamps
- Load status tracking enables monitoring and alerting

### Error Handling
- API failures are captured and logged without stopping the pipeline
- Separate error logging path prevents data loss visibility
- Audit log serves as single source of truth for processing status

### Data Freshness
- Uses configurable date range variables for incremental loading
- TRUNCATE_AND_INSERT ensures clean bronze layer data
- MERGE operation in silver layer handles updates and new records

---

## Usage Guidelines

### Running the Pipeline
1. Set appropriate `v_starttime` and `v_endtime` variables for desired date range
2. Execute pipeline through orchestration scheduler or manual trigger
3. Monitor audit logs for processing success/failure status
4. Verify data quality in silver layer table post-processing

### Variable Configuration
- **Production runs**: Set variables to appropriate date ranges for incremental processing
- **Historical loads**: Use wider date ranges but consider API rate limits
- **Testing**: Use small date windows to validate pipeline functionality

### Monitoring Points
- Check `LMS_AUDIT_LOG` for processing status
- Validate row counts between bronze and silver layers
- Monitor API response times and potential timeouts
- Verify silver layer data quality and completeness

---

## Dependencies

### External Systems
- **LMS API**: Must be accessible and responsive
- **Snowflake Environment**: Bronze and Silver databases must exist
- **API Profile**: Custom profile configuration must be valid

### Database Objects
- `PROD_BRONZE_DB.LMS.ACCREDITATION` (bronze layer table)
- `PROD_SILVER_DB.LMS.ACCREDITATION` (silver layer table)
- `PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG` (audit logging table)

### Pipeline Variables
- `v_starttime` and `v_endtime` must be set with valid ISO timestamp format
- Variables can be overridden at runtime for different extraction windows