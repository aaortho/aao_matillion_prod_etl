# LMS Speakers Load Pipeline Documentation

## Pipeline Overview

**Pipeline Name:** `pl_lms_speakers_load.orch.yaml`  
**Pipeline Type:** Orchestration  
**Purpose:** Extract speaker/instructor data from LMS API, process through bronze to silver layers, and trigger detailed speaker information processing

### Business Description

This pipeline manages the ingestion of LMS speaker/instructor data. It retrieves speaker profile information, contact details, and identification data from the LMS API, processes the data from raw JSON format into structured tables, and executes downstream processing for comprehensive speaker details.

### Data Flow Architecture

```
LMS API → Bronze Layer (RAW JSON) → Silver Layer (STRUCTURED) → Speaker Details Pipeline
                                                    ↓ 
                                                Audit Log
```

- **Source**: LMS API Speakers endpoint
- **Bronze Layer**: `PROD_BRONZE_DB.LMS.SPEAKERS` (raw JSON data)
- **Silver Layer**: `PROD_SILVER_DB.LMS.SPEAKERS` (parsed and structured data)
- **Downstream**: Speaker Details Historical Load Pipeline
- **Audit**: `PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG`

---

## Component Details

### 1. Start Component
**Type:** `start` - Initiates pipeline execution

### 2. LMS SPEAKERS (API Extract & Load)
**Type:** `modular-api-extract-input-v2`  
**Purpose:** Extracts speaker data from LMS API and loads into bronze layer

#### API Configuration
- **Endpoint:** `Speakers`
- **Authentication:** None
- **Load Strategy:** Full refresh with TRUNCATE_AND_INSERT
- **Target Table:** `PROD_BRONZE_DB.LMS.SPEAKERS`

#### Transitions
- **Success:** → Log Speakers Ingestion
- **Failure:** → Log Speakers API Failure

### 3. Log Speakers Ingestion
**Type:** `sql-executor`  
**Purpose:** Records successful data ingestion metrics

```sql
INSERT INTO PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG (
    TABLE_NAME, ROW_COUNT, LOAD_TIME, LOAD_STATUS, ERROR_MESSAGE
)
SELECT 'SPEAKERS', COUNT(*), CURRENT_TIMESTAMP, 'SUCCESS', NULL
FROM PROD_BRONZE_DB.LMS.SPEAKERS;
```

### 4. Data Parser Speakers
**Type:** `sql-executor`  
**Purpose:** Parses JSON data and performs MERGE operation to silver layer

#### Data Fields Extracted
- **Unique Identifier:** `uuid` (primary key)
- **Technical Access:** `api_key` (speaker API access key)
- **Contact Information:** `email` (speaker email address)
- **Personal Details:** `first_name`, `last_name` (speaker name)
- **System Reference:** `identification` (speaker identification code)

#### JSON Processing Logic
- Uses `LATERAL FLATTEN` on `DATA_VALUE:data` to parse speaker records
- Extracts speaker profile information with string type casting
- Handles UUID-based identification system

#### MERGE Operation Details
- **Target Table:** `PROD_SILVER_DB.LMS.SPEAKERS`
- **Match Condition:** `target.uuid = source.uuid`
- **When Matched:** Updates all speaker profile fields
- **When Not Matched:** Inserts new speaker record

#### Transitions
- **Success:** → Speaker Details Pipeline

### 5. Log Speakers API Failure
**Type:** `sql-executor`  
**Purpose:** Records API extraction failures for troubleshooting

### 6. Speaker Details Pipeline
**Type:** `run-orchestration`  
**Purpose:** Executes downstream detailed speaker processing
- **Target:** `lms_data_ingestion/pl_lms_speaker_details_historical_load.orch.yaml`

---

## Data Schema (Silver Layer)

| Field | Data Type | Description |
|-------|-----------|-------------|
| uuid | STRING | Unique speaker identifier (primary key) |
| api_key | STRING | Speaker's API access key |
| email | STRING | Speaker's email address |
| first_name | STRING | Speaker's first name |
| last_name | STRING | Speaker's last name |
| identification | STRING | Speaker identification/reference code |

---

## Pipeline Flow Diagram

```
┌─────────┐    ┌─────────────┐    ┌────────────────────┐    ┌───────────────────┐    ┌───────────────────────┐
│  Start  │───▶│ LMS SPEAKERS│───▶│ Log Speakers       │───▶│ Data Parser       │───▶│ Speaker Details       │
└─────────┘    │ (API Load) │    │ Ingestion (Success)│    │ Speakers          │    │ Pipeline              │
               └────────────┘    └────────────────────┘    │ (JSON→Structured) │    │ (Downstream)          │
                      │ (on failure)              └───────────────────┘    └───────────────────────┘
                      ▼
               ┌─────────────────────┐
               │ Log Speakers API     │
               │ Failure (Error)      │
               └─────────────────────┘
```

---

## Key Features

### Speaker Profile Management
- **UUID-Based Identification:** Uses universally unique identifiers for speakers
- **API Integration:** Individual API keys for speaker system integration
- **Contact Management:** Complete email and name information
- **System Reference:** Identification codes for cross-system linking

### Data Processing
- **Clean Data Structure:** Simple, focused on core speaker attributes
- **Full Refresh Strategy:** Ensures complete speaker data synchronization
- **Downstream Processing:** Triggers detailed speaker information analysis

### Quality Assurance
- **Comprehensive Audit Trail:** Complete processing status tracking
- **Error Handling:** Graceful API failure management
- **Validation:** Proper UUID and email format handling

---

## Business Use Cases

- **Speaker Directory Management:** Maintain complete speaker profiles
- **Event Planning:** Link speakers to events and sessions
- **Communication:** Speaker contact information for coordination
- **Analytics:** Speaker performance and engagement tracking
- **Integration:** API-based speaker data sharing with other systems

---

## Usage Guidelines

### Data Quality Monitoring
- Verify UUID uniqueness and format
- Validate email addresses for communication purposes
- Check for complete name information (first and last)
- Monitor API key assignments and validity

### Integration Considerations
- Speaker data feeds into event session details
- Links with accreditation and certification systems
- Supports speaker-led training and course delivery
- Enables speaker performance analytics

---

## Dependencies

- LMS API Speakers endpoint availability
- Speaker Details Historical Load Pipeline
- Event and session data for speaker assignment
- Email and communication systems integration
- UUID generation and management systems