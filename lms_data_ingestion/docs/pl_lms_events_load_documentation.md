# LMS Events Load Pipeline Documentation

## Pipeline Overview

**Pipeline Name:** `pl_lms_events_load.orch.yaml`  
**Pipeline Type:** Orchestration  
**Purpose:** Extract event data from LMS API, load into Snowflake bronze layer, process into silver layer, and trigger downstream event session details pipeline

### Business Description

This pipeline manages the complete lifecycle of LMS event data ingestion. It retrieves event information from the external LMS API (including event names, dates, and identifications), loads raw JSON data into the bronze layer, transforms it into a structured format in the silver layer, and executes the event session details pipeline for deeper event information processing.

### Data Flow Architecture

```
LMS API → Bronze Layer (RAW JSON) → Silver Layer (STRUCTURED) → Event Session Details Pipeline
                                                    ↓ 
                                                Audit Log
```

- **Source**: LMS API Events endpoint
- **Bronze Layer**: `PROD_BRONZE_DB.LMS.EVENTS` (raw JSON data)
- **Silver Layer**: `PROD_SILVER_DB.LMS.EVENTS` (parsed and structured data)
- **Downstream**: Event Session Details Historical Load Pipeline
- **Audit**: `PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG` (processing status and metrics)

---

## Component Details

### 1. Start Component

**Type:** `start`  
**Purpose:** Initiates the pipeline execution

- **Flow:** Unconditionally transitions to LMS EVENTS component
- **Configuration:** Standard start component

---

### 2. LMS EVENTS (API Extract & Load)

**Type:** `modular-api-extract-input-v2`  
**Purpose:** Extracts event data from LMS API and loads into Snowflake bronze layer

#### API Configuration
- **Profile:** `custom-b95fee52-8db0-4fe4-94d1-623e1ca69a28`
- **Endpoint:** `Events`
- **Authentication:** None (authType: NONE)
- **Log Level:** ERROR
- **Load Mode:** Full extract (no pagination limit specified)

#### Snowflake Output Configuration
- **Database:** `[Environment Default]` (PROD_BRONZE_DB)
- **Schema:** `LMS`
- **Table:** `EVENTS`
- **Load Mode:** `TRUNCATE_AND_INSERT` (replaces existing data)
- **Stage Platform:** Snowflake internal staging

#### Transitions
- **Success:** → Log Events Ingestion
- **Failure:** → Log Events API Failure

---

### 3. Log Events Ingestion

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
    'EVENTS' AS TABLE_NAME,
    COUNT(*) AS ROW_COUNT,
    CURRENT_TIMESTAMP AS LOAD_TIME,
    'SUCCESS' AS LOAD_STATUS,
    NULL AS ERROR_MESSAGE
FROM PROD_BRONZE_DB.LMS.EVENTS;
```

#### Transitions
- **Success:** → Data Parser Events

---

### 4. Data Parser Events

**Type:** `sql-executor`  
**Purpose:** Parses JSON data from bronze layer and upserts into structured silver layer table

#### Data Extraction Fields

**Event Information:**
- `id`: Unique event identifier
- `name`: Event name/title
- `identification`: Event identification code
- `api_key`: Event API key
- `start_date`: Event start timestamp
- `end_date`: Event end timestamp

#### JSON Parsing Strategy
- Uses `LATERAL FLATTEN` on `DATA_VALUE:data` to parse event records
- Extracts core event attributes with proper data type casting
- Handles timestamp conversion for date fields

#### MERGE Operation
- **Target Table:** `PROD_SILVER_DB.LMS.EVENTS`
- **Match Condition:** `target.id = source.id`
- **When Matched:** Updates all event fields with latest values
- **When Not Matched:** Inserts new event record

#### Transitions
- **Success:** → Event Session Details Pipeline

---

### 5. Log Events API Failure

**Type:** `sql-executor`  
**Purpose:** Records API extraction failures in audit log

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
    'EVENTS',
    -1,  -- Negative count indicates failure
    CURRENT_TIMESTAMP,
    'FAILED',
    'Failed: Unknown error occurred during LMS Events API processing'
);
```

#### Functionality
- Terminal failure state with comprehensive error logging
- Uses negative row count (-1) to indicate processing failure

---

### 6. Event Session Details Pipeline

**Type:** `run-orchestration`  
**Purpose:** Executes downstream pipeline for detailed event session data processing

#### Configuration
- **Orchestration Job:** `lms_data_ingestion/pl_lms_event_session_details_historical_load.orch.yaml`
- **Execution:** Triggers after successful event data processing
- **Purpose:** Processes detailed session information related to events

---

## Pipeline Flow Diagram

```
┌─────────┐    ┌─────────────┐    ┌─────────────────────┐    ┌──────────────────┐    ┌─────────────────────────┐
│  Start  │───▶│ LMS EVENTS  │───▶│ Log Events          │───▶│ Data Parser      │───▶│ Event Session Details   │
└─────────┘    │ (API Load)  │    │ Ingestion (Success) │    │ Events           │    │ Pipeline                │
               └─────────────┘    └─────────────────────┘    │ (JSON→Structured)│    │ (Downstream)            │
                      │                                      └──────────────────┘    └─────────────────────────┘
                      │ (on failure)
                      ▼
                ┌─────────────────────┐
                │ Log Events API      │
                │ Failure (Error Log) │
                └─────────────────────┘
```

---

## Data Quality & Monitoring

### Audit Trail
- All processing attempts logged in `LMS_AUDIT_LOG`
- Success logs include actual row counts processed
- Failure logs with error details and timestamps
- Pipeline orchestration status tracking

### Error Handling
- API failures captured without stopping downstream processing
- Comprehensive error logging for troubleshooting
- Separate failure path prevents data loss visibility

### Data Freshness
- Full refresh strategy ensures complete event data
- TRUNCATE_AND_INSERT provides clean bronze layer data
- MERGE operation handles updates and new events in silver layer

---

## Usage Guidelines

### Running the Pipeline
1. Execute pipeline through orchestration scheduler or manual trigger
2. Monitor audit logs for processing success/failure status
3. Verify data quality in silver layer table
4. Check downstream event session details pipeline execution

### Monitoring Points
- Check `LMS_AUDIT_LOG` for current processing status
- Validate row counts between bronze and silver layers
- Monitor API response times and data completeness
- Verify downstream pipeline execution status

---

## Dependencies

### External Systems
- **LMS API**: Must be accessible and responsive
- **Snowflake Environment**: Bronze and Silver databases must exist
- **API Profile**: Custom profile configuration must be valid

### Database Objects
- `PROD_BRONZE_DB.LMS.EVENTS` (bronze layer table)
- `PROD_SILVER_DB.LMS.EVENTS` (silver layer table)
- `PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG` (audit logging table)

### Downstream Dependencies
- Event Session Details Historical Load Pipeline must be available
- Proper permissions for pipeline orchestration execution