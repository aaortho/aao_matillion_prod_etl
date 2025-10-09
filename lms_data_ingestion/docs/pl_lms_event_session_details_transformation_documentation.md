# LMS Event Session Details Transformation Pipeline Documentation

## Pipeline Overview

**Pipeline Name:** `pl_lms_event_session_details_transformation.orch.yaml`  
**Pipeline Type:** Orchestration (Parameterized Transformation)  
**Purpose:** Extract detailed session information for a specific event ID and transform into structured format

### Business Description

This pipeline performs parameterized extraction of detailed session information for individual events. It takes an event ID as input parameter, calls the LMS Event_session_details API endpoint, and transforms the raw JSON data into a structured format containing session schedules, product associations, and evaluation details.

### Data Flow Architecture

```
Event ID Parameter → LMS API (Event_session_details) → Bronze Layer → Structured Table
```

- **Input Parameter**: `v_event_id` (specific event identifier)
- **API Endpoint**: `Event_session_details` with eventId URI parameter
- **Bronze Layer**: `PROD_BRONZE_DB.LMS.EVENT_SESSION_DETAILS_RAW`
- **Output**: `EVENT_SESSION_DETAILS` structured table

---

## Pipeline Variables

| Variable | Type | Scope | Visibility | Default Value | Description |
|----------|------|-------|------------|---------------|-------------|
| `v_event_id` | TEXT | COPIED | PUBLIC | `a` | Event ID for session details extraction |

---

## Component Details

### 1. Start Component
**Type:** `start` - Initiates pipeline execution with event ID parameter

### 2. LMS API (Event Session Details Extract)

**Type:** `modular-api-extract-input-v2`  
**Purpose:** Extracts session details for specific event ID

#### API Configuration
- **Endpoint:** `Event_session_details`
- **URI Parameters:** `eventId` using `${v_event_id}` variable
- **Authentication:** None
- **Target Table:** `EVENT_SESSION_DETAILS_RAW`
- **Load Mode:** `TRUNCATE_AND_INSERT`

#### Transitions
- **Success:** → SQL Script (Data Processing)

### 3. SQL Script (Session Details Processing)

**Type:** `sql-executor`  
**Purpose:** Transforms raw JSON into structured session details table

#### Data Processing Logic

**Creates structured table:** `EVENT_SESSION_DETAILS`

**Extracted Fields:**
- **Event Reference:** `EVENT_ID` (from variable)
- **Session Identity:** `SESSION_ID`, `EVAL_ID`
- **Timing:** `START_TIME`, `END_TIME`, `TIMEZONE`
- **Product Association:** `PRODUCT_ID`, `PRODUCT_IDENTIFICATION`, `PRODUCT_NAME`, `PRODUCT_TYPE`
- **Session Content:** `SESSION_TITLE`

#### JSON Processing Strategy
```sql
CREATE OR REPLACE TABLE EVENT_SESSION_DETAILS AS
SELECT 
    '${v_event_id}' AS EVENT_ID,
    item.value:id::STRING AS SESSION_ID,
    item.value:eval_id::STRING AS EVAL_ID,
    item.value:start_date_time::STRING AS START_TIME,
    item.value:end_date_time::STRING AS END_TIME,
    item.value:offset::STRING AS TIMEZONE,
    item.value:product:id::STRING AS PRODUCT_ID,
    item.value:product:identification::STRING AS PRODUCT_IDENTIFICATION,
    item.value:product:name::STRING AS PRODUCT_NAME,
    item.value:product:type::STRING AS PRODUCT_TYPE,
    item.value:title::STRING AS SESSION_TITLE
FROM PROD_BRONZE_DB.LMS.EVENT_SESSION_DETAILS_RAW,
     LATERAL FLATTEN(input => DATA_VALUE:data) AS item
CROSS JOIN (SELECT '${v_event_id}' AS variable_value) v;
```

---

## Pipeline Flow Diagram

```
┌─────────┐    ┌────────────────────────────┐    ┌──────────────────────────────┐
│  Start  │───▶│ LMS API                    │───▶│ SQL Script                   │
│ (Event  │    │ (Event_session_details)    │    │ (Transform to Structured)    │
│ ID Param│    │ eventId=${v_event_id}      │    │ CREATE EVENT_SESSION_DETAILS │
└─────────┘    └────────────────────────────┘    └──────────────────────────────┘
```

---

## Key Features

### Parameterized Processing
- **Event-Specific Extraction:** Uses event ID parameter for targeted data retrieval
- **Variable Integration:** Incorporates event ID into output data structure
- **Cross Join Pattern:** Links variable value with extracted session data

### Session Detail Processing
- **Nested Product Data:** Extracts product information embedded in session records
- **Timing Information:** Processes session start/end times with timezone
- **Evaluation Integration:** Links sessions to evaluation systems via eval_id

### Data Structure Creation
- **Direct Table Creation:** Uses CREATE OR REPLACE for immediate structured output
- **Event Association:** Each session record linked to source event ID
- **Product Relationship:** Maintains session-to-product associations

---

## Data Schema (Output Table)

| Field | Data Type | Description |
|-------|-----------|-------------|
| EVENT_ID | STRING | Source event identifier (from parameter) |
| SESSION_ID | STRING | Unique session identifier |
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

### Parameterized Execution
- **Set Event ID:** Provide valid event ID via `v_event_id` variable
- **Variable Scope:** Uses COPIED scope for concurrent execution safety
- **Single Event Focus:** Designed for processing one event at a time

### Integration Pattern
- **Child Pipeline:** Called by Event Session Details Historical Load pipeline
- **Iterator Target:** Used within table iterator for batch processing
- **Structured Output:** Creates consumable table for downstream processing

### Data Validation
- **Event ID Validation:** Ensure event ID exists in LMS system
- **Session Count Verification:** Validate expected number of sessions returned
- **Product Association:** Verify product linkages are complete

---

## Business Use Cases

- **Event Planning:** Detailed session scheduling and organization
- **Product Assignment:** Link sessions to specific courses/products
- **Evaluation Integration:** Connect sessions to assessment systems
- **Scheduling Analytics:** Session timing and duration analysis
- **Resource Management:** Product and instructor allocation

---

## Dependencies

### Input Requirements
- **Event ID Parameter:** Valid event identifier must be provided
- **Source Event Data:** Event must exist in LMS system
- **API Accessibility:** Event_session_details endpoint must be available

### Infrastructure
- **Bronze Layer Table:** `EVENT_SESSION_DETAILS_RAW` must exist
- **Database Permissions:** CREATE/REPLACE table permissions required
- **Variable Processing:** Support for parameterized pipeline execution