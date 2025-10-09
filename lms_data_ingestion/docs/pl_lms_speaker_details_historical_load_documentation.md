# LMS Speaker Details Historical Load Pipeline Documentation

## Pipeline Overview

**Pipeline Name:** `pl_lms_speaker_details_historical_load.orch.yaml`  
**Pipeline Type:** Orchestration (Iterator-Based Comprehensive Speaker Processing)  
**Purpose:** Extract comprehensive speaker profile details for all speakers using iterator pattern and process into detailed silver layer table

### Business Description

This pipeline performs exhaustive speaker profile extraction by iterating through all speakers in the LMS system. It calls the Speaker_details API endpoint for each speaker individually to gather comprehensive biographical information, contact details, professional credentials, and address information, then consolidates this rich dataset into a structured silver layer table with detailed audit logging.

### Data Flow Architecture

```
Speakers Bronze Table → Table Iterator → Speaker Details API → Silver Layer
       │                    │                      │                  │
   (Speaker UUIDs)      (Per Speaker API)      (Detailed Profiles)   (Structured)
                                 ↓
                            Audit Logging
```

- **Source**: Speaker UUIDs from `PROD_BRONZE_DB.LMS.SPEAKERS`
- **Processing**: Iterative API calls to `Speaker_details` endpoint
- **Bronze Layer**: `PROD_BRONZE_DB.LMS.SPEAKER_DETAILS`
- **Silver Layer**: `PROD_SILVER_DB.LMS.SPEAKER_DETAILS`
- **Audit**: `PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG`

---

## Pipeline Variables

| Variable | Type | Scope | Visibility | Default Value | Description |
|----------|------|-------|------------|---------------|-------------|
| `v_speaker_id` | TEXT | COPIED | PUBLIC | `39d0efc1-5d3c-40f8-ad2b-85d7310998aa` | Current speaker UUID being processed |

---

## Component Details

### 1. Start Component
**Type:** `start` - Initiates the speaker details processing workflow

#### Transitions
- **Unconditional:** → Truncate Speaker Details Table

### 2. Truncate Speaker Details Table

**Type:** `sql-executor`  
**Purpose:** Clears previous speaker details data for fresh load

```sql
TRUNCATE TABLE PROD_BRONZE_DB.LMS.SPEAKER_DETAILS;
```

#### Transitions
- **Success:** → Table Iterator

### 3. Table Iterator (Core Processing Engine)

**Type:** `table-iterator`  
**Purpose:** Processes each speaker UUID through the API extraction

#### Iterator Configuration
- **Mode:** Advanced (custom SQL query)
- **Concurrency:** Concurrent (parallel processing)
- **Break on Failure:** No (continues processing other speakers)
- **Target:** LMS API

#### Source Query
```sql
SELECT
    item.value:uuid::STRING AS uuid
FROM PROD_BRONZE_DB.LMS.SPEAKERS,
     LATERAL FLATTEN(input => DATA_VALUE:data) AS item
```

#### Column Mapping
- **UUID** → **v_speaker_id** (passes speaker UUID to API component)

#### Transitions
- **Success:** → Log speaker detail ingestion
- **Failure:** → Log LMS API Failure

### 4. LMS API (Iterator Target)

**Type:** `modular-api-extract-input-v2`  
**Purpose:** Extracts comprehensive speaker information for each speaker

#### API Configuration
- **Endpoint:** `Speaker_details`
- **URI Parameters:** `speakerId` using `${v_speaker_id}` variable
- **Authentication:** None
- **Target Table:** `SPEAKER_DETAILS`
- **Load Mode:** `APPEND` (accumulates data from all speakers)

### 5. Log Speaker Detail Ingestion

**Type:** `sql-executor`  
**Purpose:** Records successful processing metrics

```sql
INSERT INTO PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG (
    TABLE_NAME, ROW_COUNT, LOAD_TIME, LOAD_STATUS, ERROR_MESSAGE
)
SELECT 
    'SPEAKER_DETAILS' AS TABLE_NAME,
    COUNT(*) AS ROW_COUNT,
    CURRENT_TIMESTAMP AS LOAD_TIME,
    'SUCCESS' AS LOAD_STATUS,
    NULL AS ERROR_MESSAGE
FROM PROD_BRONZE_DB.LMS.SPEAKER_DETAILS;
```

#### Transitions
- **Success:** → Data Speaker Detail

### 6. Data Speaker Detail

**Type:** `sql-executor`  
**Purpose:** Transforms consolidated speaker details to silver layer

#### Comprehensive Data Processing

**Target Table:** `PROD_SILVER_DB.LMS.SPEAKER_DETAILS`

#### Extracted Fields (Comprehensive Speaker Profile)

**Address Information:**
- `address1`, `address2`, `city`, `state`, `country`, `zip_code`

**Professional Information:**
- `api_key`, `company`, `credentials`, `title`, `description`

**Personal Information:**
- `first_name`, `middle_name`, `last_name`, `salutation`, `suffix`, `informal`

**Contact Information:**
- `email`, `phone`, `mobile`

**Social Media:**
- `linkedin`, `twitter`, `url`

**System Information:**
- `id`, `identification`, `icon`, `created_at`, `updated_at`

#### Direct JSON Field Extraction
```sql
DATA_VALUE:address1::STRING           AS address1,
DATA_VALUE:address2::STRING           AS address2,
DATA_VALUE:api_key::STRING            AS api_key,
DATA_VALUE:city::STRING               AS city,
DATA_VALUE:company::STRING            AS company,
DATA_VALUE:country::STRING            AS country,
DATA_VALUE:created_at::TIMESTAMP_TZ   AS created_at,
DATA_VALUE:credentials::STRING        AS credentials,
-- ... (continues for all 27 fields)
```

### 7. Log LMS API Failure

**Type:** `sql-executor`  
**Purpose:** Records iterator processing failures

---

## Pipeline Flow Diagram

```
┌───────────────────────────────────────────────────────────────────────────────────┐
│                           Start                                                  │
└───────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│                      Truncate Speaker Details Table                           │
└───────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│          Table Iterator + LMS API (Per Speaker Processing)                     │
│           ┌───────────────────────────────────────────────────────┐         │
│           │ Speaker UUID 1 → API Call → Speaker Profile    │         │
│           │ Speaker UUID 2 → API Call → Speaker Profile    │         │
│           │ Speaker UUID N → API Call → Speaker Profile    │         │
│           └───────────────────────────────────────────────────────┘         │
└───────────────────────────────────────────────────────────────────────────────────┘
               │ (success)                                    │ (failure)
               ▼                                              ▼
┌─────────────────────────────┐        ┌──────────────────────────────┐
│ Log Speaker Detail         │        │ Log LMS API Failure         │
│ Ingestion (Success)        │        │ (Error Logging)             │
└─────────────────────────────┘        └──────────────────────────────┘
               │
               ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│               Data Speaker Detail (Bronze → Silver)                         │
└───────────────────────────────────────────────────────────────────────────────────┘
```

---

## Key Features

### Comprehensive Speaker Profiling
- **Complete Biographical Data:** Full personal and professional profiles
- **Multi-Channel Contact Information:** Phone, mobile, email, and social media
- **Professional Credentials:** Title, company, credentials, and descriptions
- **Geographic Information:** Complete address details with international support
- **Social Media Integration:** LinkedIn, Twitter, and website links

### Rich Data Model
- **27 Data Fields:** Most comprehensive speaker information in the LMS suite
- **Timestamp Tracking:** Created and updated timestamps for data governance
- **Flexible Name Handling:** Salutation, first, middle, last names, suffix, and informal names
- **Professional Context:** Company affiliation and professional credentials

### Advanced Processing Architecture
- **UUID-Based Processing:** Uses speaker UUID for precise identification
- **Direct Field Mapping:** Straightforward JSON-to-SQL field extraction
- **Timestamp Conversion:** Proper handling of timezone-aware timestamps
- **Comprehensive Coverage:** Processes all available speaker profile fields

---

## Data Schema (Silver Layer)

| Field Category | Field | Data Type | Description |
|----------------|-------|-----------|-------------|
| **Address** | address1 | STRING | Primary address line |
| | address2 | STRING | Secondary address line |
| | city | STRING | City name |
| | state | STRING | State/province |
| | country | STRING | Country name |
| | zip_code | STRING | Postal/zip code |
| **Professional** | api_key | STRING | Speaker API access key |
| | company | STRING | Company/organization |
| | credentials | STRING | Professional credentials |
| | title | STRING | Professional title |
| | description | STRING | Speaker biography/description |
| **Personal** | first_name | STRING | First name |
| | middle_name | STRING | Middle name |
| | last_name | STRING | Last name |
| | salutation | STRING | Title/salutation |
| | suffix | STRING | Name suffix |
| | informal | STRING | Informal/preferred name |
| **Contact** | email | STRING | Email address |
| | phone | STRING | Primary phone number |
| | mobile | STRING | Mobile phone number |
| **Social Media** | linkedin | STRING | LinkedIn profile URL |
| | twitter | STRING | Twitter handle/URL |
| | url | STRING | Personal/professional website |
| **System** | id | STRING | Unique speaker identifier |
| | identification | STRING | Speaker identification code |
| | icon | STRING | Speaker profile icon URL |
| | created_at | TIMESTAMP_TZ | Profile creation timestamp |
| | updated_at | TIMESTAMP_TZ | Profile last update timestamp |

---

## Usage Guidelines

### Historical Processing
- **Speaker Dependency:** Speakers must be loaded before speaker details
- **Complete Refresh:** Truncates and reloads all speaker profiles
- **UUID Mapping:** Ensures proper speaker identification across systems
- **Rich Profile Data:** Comprehensive speaker information for all use cases

### Data Management
- **Privacy Considerations:** Handle personal information according to data governance policies
- **Contact Information Validation:** Verify phone numbers, emails, and social media links
- **Professional Credential Tracking:** Maintain up-to-date professional information
- **Address Normalization:** Consider standardizing address formats

### Integration Use Cases
- **Speaker Directory:** Complete speaker profiles for web portals and catalogs
- **Contact Management:** Comprehensive contact information for event coordination
- **Professional Networking:** LinkedIn and social media integration
- **Event Planning:** Speaker qualification and availability management
- **Communication:** Multi-channel contact options for speaker outreach

---

## Business Use Cases

- **Speaker Management:** Comprehensive speaker directory and profile management
- **Event Coordination:** Complete speaker contact and scheduling information
- **Professional Development:** Track speaker credentials and qualifications
- **Marketing Integration:** Speaker profiles for promotional materials
- **Communication Strategy:** Multi-channel speaker contact and engagement
- **Analytics:** Speaker performance and engagement analysis

---

## Dependencies

### Data Dependencies
- **Speakers Data:** `PROD_BRONZE_DB.LMS.SPEAKERS` must be populated with UUID data
- **Bronze/Silver Tables:** Target tables must exist with comprehensive schema
- **API Access:** LMS Speaker_details endpoint availability

### Technical Requirements
- **Iterator Support:** Table iterator functionality for UUID-based processing
- **Timestamp Processing:** Support for timezone-aware timestamp conversion
- **Concurrent API Processing:** Ability to handle multiple simultaneous API requests
- **Comprehensive Field Mapping:** Support for 27-field data structure