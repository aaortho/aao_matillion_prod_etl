# LMS Event Session Details Historical Load Pipeline Documentation

## Pipeline Overview

**Pipeline Name:** `pl_lms_event_session_details_historical_load.orch.yaml`  
**Pipeline Type:** Orchestration (Iterator-Based Historical Processing)  
**Purpose:** Iteratively extract session details for all events and process into comprehensive silver layer table

### Business Description

This pipeline orchestrates the complete historical loading of event session details across all events in the LMS system. It uses an iterator pattern to process each event individually, calling the transformation pipeline for detailed session extraction, and consolidates all session data into a structured silver layer table with comprehensive audit logging.

### Data Flow Architecture

```
Events Bronze Table → Table Iterator → Session Transform Pipeline → Silver Layer
       │                    │                       │                    │
    (Event IDs)      (Per Event Processing)    (Session Details)      (Structured Data)
                                 ↓
                            Audit Logging
```

- **Source**: Event IDs from `PROD_BRONZE_DB.LMS.EVENTS`
- **Processing**: Iterative transformation via `pl_lms_event_session_details_transformation.orch.yaml`
- **Bronze Layer**: `PROD_BRONZE_DB.LMS.EVENT_SESSION_DETAILS` (consolidated)
- **Silver Layer**: `PROD_SILVER_DB.LMS.EVENT_SESSION_DETAILS` (structured)
- **Audit**: `PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG`

---

## Pipeline Variables

| Variable | Type | Scope | Visibility | Default Value | Description |
|----------|------|-------|------------|---------------|-------------|
| `v_event_id` | TEXT | COPIED | PUBLIC | `39d0efc1-5d3c-40f8-ad2b-85d7310998aa` | Current event ID being processed |

---

## Component Details

### 1. Start Component
**Type:** `start` - Initiates the historical processing workflow

#### Transitions
- **Unconditional:** → Truncate Event Session details table

### 2. Truncate Event Session Details Table

**Type:** `sql-executor`  
**Purpose:** Clears previous session details data for fresh load

```sql
TRUNCATE TABLE PROD_BRONZE_DB.LMS.EVENT_SESSION_DETAILS;
```

#### Transitions
- **Success:** → Table Iterator

### 3. Table Iterator (Core Processing Engine)

**Type:** `table-iterator`  
**Purpose:** Processes each event ID through the transformation pipeline

#### Iterator Configuration
- **Mode:** Advanced (custom SQL query)
- **Concurrency:** Concurrent (parallel processing)
- **Break on Failure:** No (continues processing other events)
- **Target:** Run Event Session Transform

#### Source Query
```sql
SELECT
    item.value:id::STRING AS id
FROM PROD_BRONZE_DB.LMS.EVENTS,
     LATERAL FLATTEN(input => DATA_VALUE:data) AS item
     limit 1
```

#### Column Mapping
- **ID** → **v_event_id** (passes event ID to transformation pipeline)

#### Transitions
- **Success:** → Log event session detail ingestion
- **Failure:** → Log LMS API Failure

### 4. Run Event Session Transform (Iterator Target)

**Type:** `run-orchestration`  
**Purpose:** Executes transformation pipeline for each event

#### Configuration
- **Target Pipeline:** `lms_data_ingestion/pl_lms_event_session_details_transformation.orch.yaml`
- **Variable Mapping:** Passes `v_event_id` to transformation pipeline
- **Execution:** Called once per event by the iterator

### 5. Log Event Session Detail Ingestion

**Type:** `sql-executor`  
**Purpose:** Records successful processing metrics in audit log

```sql
INSERT INTO PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG (
    TABLE_NAME,
    ROW_COUNT,
    LOAD_TIME,
    LOAD_STATUS,
    ERROR_MESSAGE
)
SELECT 
    'EVENT_SESSION_DETAILS' AS TABLE_NAME,
    COUNT(*) AS ROW_COUNT,
    CURRENT_TIMESTAMP AS LOAD_TIME,
    'SUCCESS' AS LOAD_STATUS,
    NULL AS ERROR_MESSAGE
FROM PROD_BRONZE_DB.LMS.EVENT_SESSION_DETAILS;
```

#### Transitions
- **Success:** → Data Parser Event Session

### 6. Data Parser Event Session

**Type:** `sql-executor`  
**Purpose:** Transforms consolidated session details to silver layer

#### MERGE Operation to Silver Layer

**Target Table:** `PROD_SILVER_DB.LMS.EVENT_SESSION_DETAILS`

**Data Processing:**
- Uses `LATERAL FLATTEN` on consolidated bronze data
- Extracts session, timing, and product information
- Performs MERGE operation with session ID as key

#### Data Fields Processed
- **Session Identity:** `SESSION_ID`, `EVAL_ID`
- **Timing:** `START_TIME`, `END_TIME`, `TIMEZONE`  
- **Product Association:** `PRODUCT_ID`, `PRODUCT_IDENTIFICATION`, `PRODUCT_NAME`, `PRODUCT_TYPE`
- **Session Content:** `SESSION_TITLE`

### 7. Log LMS API Failure

**Type:** `sql-executor`  
**Purpose:** Records iterator processing failures

```sql
INSERT INTO PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG (
    TABLE_NAME, ROW_COUNT, LOAD_TIME, LOAD_STATUS, ERROR_MESSAGE
)
VALUES (
    'EVENT_SESSION_DETAILS', -1, CURRENT_TIMESTAMP, 'FAILED',
    'Failed Unknown error occurred during LMS API batch processing'
);
```

---

## Pipeline Flow Diagram

```
┌───────────────────────────────────────────────────────────────────────────────────┐
│                           Start                                                  │
└───────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│                     Truncate Event Session Details Table                       │
└───────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│         Table Iterator + Run Event Session Transform (Per Event)               │
│           ┌───────────────────────────────────────────────────────┐         │
│           │  Event ID 1 → Transform Pipeline → Session Details  │         │
│           │  Event ID 2 → Transform Pipeline → Session Details  │         │
│           │  Event ID N → Transform Pipeline → Session Details  │         │
│           └───────────────────────────────────────────────────────┘         │
└───────────────────────────────────────────────────────────────────────────────────┘
               │ (success)                                    │ (failure)
               ▼                                              ▼
┌─────────────────────────────┐        ┌──────────────────────────────┐
│ Log Event Session Detail  │        │ Log LMS API Failure         │
│ Ingestion (Success)        │        │ (Error Logging)             │
└─────────────────────────────┘        └──────────────────────────────┘
               │
               ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│              Data Parser Event Session (Bronze → Silver)                     │
└───────────────────────────────────────────────────────────────────────────────────┘
```

---

## Key Features

### Iterator-Based Processing
- **Event-by-Event Processing:** Handles each event individually for detailed extraction
- **Concurrent Execution:** Processes multiple events simultaneously for efficiency
- **Fault Tolerance:** Continues processing other events if individual event fails
- **Parameterized Calls:** Passes event ID to transformation pipeline

### Advanced Data Processing
- **Two-Stage Processing:** Iterator + transformation pipeline for complex workflows
- **Data Consolidation:** Aggregates session details from all events
- **Bronze-Silver Pattern:** Maintains medallion architecture consistency
- **Comprehensive Audit Trail:** Tracks both iterator and data processing success/failures

### Modular Architecture
- **Pipeline Orchestration:** Coordinates multiple pipeline executions
- **Separation of Concerns:** Iterator logic separate from transformation logic
- **Reusable Components:** Transformation pipeline can be used independently
- **Scalable Design:** Can handle large numbers of events efficiently

---

## Data Schema (Silver Layer)

| Field | Data Type | Description |
|-------|-----------|-------------|
| SESSION_ID | STRING | Unique session identifier (primary key) |
| EVAL_ID | STRING | Evaluation system identifier |
| START_TIME | STRING | Session start timestamp |
| END_TIME | STRING | Session end timestamp |
| TIMEZONE | STRING | Session timezone offset |
| PRODUCT_ID | STRING | Associated product identifier |
| PRODUCT_IDENTIFICATION | STRING | Product identification code |
| PRODUCT_NAME | STRING | Product/course name |
| PRODUCT_TYPE | STRING | Product category/type |
| SESSION_TITLE | STRING | Session title/name |

---

## Usage Guidelines

### Historical Load Execution
- **Complete Refresh:** Truncates and reloads all session details
- **Event Dependency:** Events must be loaded before session details
- **Resource Considerations:** High API usage during concurrent processing
- **Monitoring:** Track iterator completion and individual event processing

### Performance Optimization
- **Concurrent Processing:** Leverages parallel execution for efficiency
- **API Rate Limiting:** Monitor LMS API rate limits during execution
- **Error Isolation:** Individual event failures don't stop overall processing
- **Resource Usage:** Consider compute resources for large event volumes

### Data Quality Validation
- **Session Count Verification:** Compare expected vs. actual session counts
- **Event Coverage:** Ensure all events have been processed
- **Product Association:** Validate session-product relationships
- **Timing Data:** Verify session scheduling information accuracy

---

## Business Use Cases

- **Event Management:** Complete session scheduling and organization
- **Resource Planning:** Session-based resource allocation and management
- **Analytics:** Session attendance, timing, and product analysis
- **Evaluation Integration:** Link sessions to assessment and evaluation systems
- **Product Management:** Track which products are delivered in which sessions

---

## Dependencies

### Pipeline Dependencies
- **Events Data:** `PROD_BRONZE_DB.LMS.EVENTS` must be populated
- **Transformation Pipeline:** `pl_lms_event_session_details_transformation.orch.yaml` must be available
- **Bronze/Silver Tables:** Target tables must exist with proper schemas

### Technical Requirements
- **Iterator Support:** Pipeline orchestration with table iterator functionality
- **API Access:** LMS Event_session_details endpoint availability
- **Concurrent Processing:** Support for parallel pipeline execution
- **Variable Passing:** Parameterized pipeline execution capabilities