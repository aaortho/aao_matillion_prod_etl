# LMS Products Load Pipeline Documentation

## Pipeline Overview

**Pipeline Name:** `pl_lms_products_load.orch.yaml`  
**Pipeline Type:** Orchestration  
**Purpose:** Extract product data from LMS API, load into bronze layer, process into silver layer, and trigger product details historical pipeline

### Business Description

This pipeline manages the complete ingestion of LMS product/course data. It retrieves product information including metadata, status, descriptions, and identifications from the LMS API, processes the data through bronze to silver layers, and executes downstream processing for detailed product information.

### Data Flow Architecture

```
LMS API → Bronze Layer (RAW JSON) → Silver Layer (STRUCTURED) → Product Details Pipeline
                                                    ↓ 
                                                Audit Log
```

---

## Component Details

### 1. Start Component
**Type:** `start` - Initiates pipeline execution

### 2. LMS PRODUCTS (API Extract & Load)
**Type:** `modular-api-extract-input-v2`  
**Purpose:** Extracts product data from LMS API

#### Transitions
- **Success:** → Log Products Ingestion
- **Failure:** → Log Products API Failure

### 3. Log Products Ingestion
**Type:** `sql-executor`  
**Purpose:** Records successful ingestion audit trail

### 4. Data Parser Products
**Type:** `sql-executor`  
**Purpose:** Transforms JSON data to structured format

#### Data Fields Extracted
- **Core Product Info:** `id`, `name`, `description`, `type`
- **Technical Details:** `api_key`, `identification`, `status`
- **Metadata:** `created_at`, `icon_url`, `teaser`
- **Configuration:** `discoverable` (boolean flag)

#### JSON Processing
- Uses `LATERAL FLATTEN` on `DATA_VALUE:data`
- Handles boolean conversion for `discoverable` field
- Timestamp parsing for `created_at` field
- MERGE operation to `PROD_SILVER_DB.LMS.PRODUCTS`

#### Transitions
- **Success:** → Product Details Pipeline

### 5. Log Products API Failure
**Type:** `sql-executor`  
**Purpose:** Error logging for failed API calls

### 6. Product Details Pipeline
**Type:** `run-orchestration`  
**Purpose:** Executes downstream detailed product processing
- **Target:** `lms_data_ingestion/pl_lms_product_details_historical_load.orch.yaml`

---

## Data Schema

| Field | Data Type | Description |
|-------|-----------|-------------|
| id | STRING | Unique product identifier |
| api_key | STRING | Product API access key |
| created_at | TIMESTAMP | Product creation timestamp |
| description | STRING | Product description text |
| discoverable | BOOLEAN | Product visibility flag |
| icon_url | STRING | Product icon URL |
| identification | STRING | Product identification code |
| name | STRING | Product name/title |
| status | STRING | Product status (active/inactive) |
| teaser | STRING | Product teaser text |
| type | STRING | Product type/category |

---

## Pipeline Features

- **Full Data Refresh:** TRUNCATE_AND_INSERT strategy
- **Downstream Processing:** Triggers detailed product analysis
- **Error Handling:** Comprehensive API failure logging
- **Data Quality:** Proper type casting and validation
- **Audit Trail:** Complete processing status tracking

---

## Dependencies

- LMS API Products endpoint
- Product Details Historical Load Pipeline
- Snowflake bronze/silver databases
- LMS audit logging infrastructure