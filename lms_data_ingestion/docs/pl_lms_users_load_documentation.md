# LMS Users Load Pipeline Documentation

## Pipeline Overview

**Pipeline Name:** `pl_lms_users_load.orch.yaml`  
**Pipeline Type:** Orchestration  
**Purpose:** Extract user data from LMS API, load into bronze layer, and process into structured silver layer format

### Business Description

This pipeline handles the ingestion of LMS user/learner data. It retrieves user profile information including personal details, login credentials, and identification data from the external LMS API, processing it from raw JSON format into structured tables for analytics and reporting.

### Data Flow Architecture

```
LMS API → Bronze Layer (RAW JSON) → Silver Layer (STRUCTURED)
                                        ↓ 
                                    Audit Log
```

- **Source**: LMS API Users endpoint
- **Bronze Layer**: `PROD_BRONZE_DB.LMS.USERS` (raw JSON data)
- **Silver Layer**: `PROD_SILVER_DB.LMS.USERS` (parsed and structured data)
- **Audit**: `PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG`

---

## Component Details

### 1. Start Component
**Type:** `start` - Initiates pipeline execution

### 2. LMS USERS (API Extract & Load)
**Type:** `modular-api-extract-input-v2`  
**Purpose:** Extracts user data from LMS API and loads into bronze layer

#### API Configuration
- **Endpoint:** `Users`
- **Authentication:** None
- **Load Strategy:** Full refresh with TRUNCATE_AND_INSERT
- **Target Table:** `PROD_BRONZE_DB.LMS.USERS`

#### Transitions
- **Success:** → Log Users Ingestion
- **Failure:** → Log Users API Failure

### 3. Log Users Ingestion
**Type:** `sql-executor`  
**Purpose:** Records successful data ingestion metrics

```sql
INSERT INTO PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG (
    TABLE_NAME, ROW_COUNT, LOAD_TIME, LOAD_STATUS, ERROR_MESSAGE
)
SELECT 'USERS', COUNT(*), CURRENT_TIMESTAMP, 'SUCCESS', NULL
FROM PROD_BRONZE_DB.LMS.USERS;
```

### 4. Data Parser Users
**Type:** `sql-executor`  
**Purpose:** Parses JSON data and performs MERGE operation to silver layer

#### Data Fields Extracted
- **User ID:** `id` (mapped to `user_id` in silver layer)
- **Authentication:** `api_username`, `email`
- **Personal Info:** `first_name`, `last_name`

#### JSON Processing Logic
- Uses `LATERAL FLATTEN` on `DATA_VALUE:data` to parse user records
- Extracts user profile information with string casting
- MERGE operation ensures upsert functionality

#### MERGE Operation Details
- **Target Table:** `PROD_SILVER_DB.LMS.USERS`
- **Match Condition:** `target.user_id = source.user_id`
- **When Matched:** Updates all user profile fields
- **When Not Matched:** Inserts new user record

### 5. Log Users API Failure
**Type:** `sql-executor`  
**Purpose:** Records API extraction failures

```sql
INSERT INTO PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG (
    TABLE_NAME, ROW_COUNT, LOAD_TIME, LOAD_STATUS, ERROR_MESSAGE
)
VALUES (
    'USERS', -1, CURRENT_TIMESTAMP, 'FAILED',
    'Failed: Unknown error occurred during LMS Users API processing'
);
```

---

## Data Schema (Silver Layer)

| Field | Data Type | Description |
|-------|-----------|-------------|
| user_id | STRING | Unique user identifier (primary key) |
| api_username | STRING | User's API access username |
| email | STRING | User's email address |
| first_name | STRING | User's first name |
| last_name | STRING | User's last name |

---

## Pipeline Flow N

```
┌─────────┐    ┌───────────┐    ┌───────────────────┐    ┌──────────────────┐
│  Start  │───▶│ LMS USERS │───▶│ Log Users          │───▶│ Data Parser      │
└─────────┘    │(API Load)│    │ Ingestion (Success)│    │ Users            │
               └───────────┘    └───────────────────┘    └──────────────────┘
                     │ (on failure)
                     ▼
               ┌────────────────────┐
               │ Log Users API      │
               │ Failure (Error)    │
               └────────────────────┘
```

---

## Key Features

- **Simple User Profile Management:** Focus on core user identification and contact information
- **Full Refresh Strategy:** Ensures complete user data synchronization
- **Audit Logging:** Comprehensive processing status tracking
- **Error Handling:** Graceful API failure management
- **Data Quality:** Proper field mapping and type casting

---

## Usage Guidelines

### Monitoring
- Check `LMS_AUDIT_LOG` for processing status
- Validate user count between bronze and silver layers
- Monitor for duplicate user records
- Verify email and username data quality

### Dependencies
- LMS API Users endpoint accessibility
- Snowflake bronze and silver database availability
- LMS audit log table structure