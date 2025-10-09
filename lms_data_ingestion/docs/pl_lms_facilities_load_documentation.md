# LMS Facilities Load Pipeline Documentation

## Pipeline Overview

**Pipeline Name:** `pl_lms_facilities_load.orch.yaml`  
**Pipeline Type:** Orchestration  
**Purpose:** Extract facility data from LMS API, load into Snowflake bronze layer, and process into structured silver layer format

### Business Description

This pipeline handles the ingestion of LMS facility/location data. It retrieves comprehensive facility information including names, addresses, contact details, and geographical information from the external LMS API, processes the raw JSON data into a structured format for analytics and reporting purposes.

### Data Flow Architecture

```
LMS API → Bronze Layer (RAW JSON) → Silver Layer (STRUCTURED)
                                        ↓ 
                                    Audit Log
```

- **Source**: LMS API Facilities endpoint
- **Bronze Layer**: `PROD_BRONZE_DB.LMS.FACILITIES` (raw JSON data)
- **Silver Layer**: `PROD_SILVER_DB.LMS.FACILITIES` (parsed and structured data)
- **Audit**: `PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG` (processing status and metrics)

---

## Component Details

### 1. Start Component
**Type:** `start` - Initiates pipeline execution

### 2. LMS FACILITIES (API Extract & Load)
**Type:** `modular-api-extract-input-v2`  
**Purpose:** Extracts facility data from LMS API and loads into bronze layer

#### API Configuration
- **Endpoint:** `Facilities`
- **Authentication:** None
- **Load Mode:** Full extract with TRUNCATE_AND_INSERT

#### Transitions
- **Success:** → Log Facilities Ingestion
- **Failure:** → Log Facilities API Failure

### 3. Log Facilities Ingestion
**Type:** `sql-executor`  
**Purpose:** Records successful ingestion metrics

```sql
INSERT INTO PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG (
    TABLE_NAME, ROW_COUNT, LOAD_TIME, LOAD_STATUS, ERROR_MESSAGE
)
SELECT 'FACILITIES', COUNT(*), CURRENT_TIMESTAMP, 'SUCCESS', NULL
FROM PROD_BRONZE_DB.LMS.FACILITIES;
```

### 4. Data Parser Facilities
**Type:** `sql-executor`  
**Purpose:** Parses JSON and upserts to silver layer

#### Data Fields Extracted
- **Core Information:** `id`, `name`, `description`
- **Address Details:** `address_1`, `address_2`, `city`, `state`, `zipcode`, `country`
- **Contact Information:** `phone`, `website`

#### JSON Parsing
- Uses `LATERAL FLATTEN` on `DATA_VALUE:data`
- MERGE operation on `PROD_SILVER_DB.LMS.FACILITIES`
- Match condition: `target.ID = source.id`

### 5. Log Facilities API Failure
**Type:** `sql-executor`  
**Purpose:** Records API failures for troubleshooting

---

## Data Schema

| Field | Data Type | Description |
|-------|-----------|-------------|
| ID | STRING | Unique facility identifier |
| NAME | STRING | Facility name |
| DESCRIPTION | STRING | Facility description |
| ADDRESS_1 | STRING | Primary address line |
| ADDRESS_2 | STRING | Secondary address line |
| CITY | STRING | City name |
| STATE | STRING | State/province |
| ZIPCODE | STRING | Postal/zip code |
| COUNTRY | STRING | Country name |
| PHONE | STRING | Contact phone number |
| WEBSITE | STRING | Facility website URL |

---

## Dependencies

- LMS API Facilities endpoint availability
- Snowflake bronze and silver layer databases
- LMS audit log table for monitoring