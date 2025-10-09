# LMS Product Details Historical Load Pipeline Documentation

## Pipeline Overview

**Pipeline Name:** `pl_lms_product_details_historical_load.orch.yaml`  
**Pipeline Type:** Orchestration (Iterator-Based Detail Processing)  
**Purpose:** Extract comprehensive product details for all products using iterator pattern and process into detailed silver layer table

### Business Description

This pipeline performs detailed product information extraction by iterating through all products in the LMS system. It calls the Product_details API endpoint for each product individually to gather comprehensive product specifications, pricing information, and group associations, then consolidates this data into a structured silver layer table with detailed audit logging.

### Data Flow Architecture

```
Products Bronze Table → Table Iterator → Product Details API → Silver Layer
       │                    │                     │                  │
    (Product IDs)      (Per Product API)     (Detailed Data)    (Structured)
                                 ↓
                            Audit Logging
```

- **Source**: Product IDs from `PROD_BRONZE_DB.LMS.PRODUCTS`
- **Processing**: Iterative API calls to `Product_details` endpoint
- **Bronze Layer**: `PROD_BRONZE_DB.LMS.PRODUCT_DETAILS`
- **Silver Layer**: `PROD_SILVER_DB.LMS.PRODUCT_DETAILS`
- **Audit**: `PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG`

---

## Pipeline Variables

| Variable | Type | Scope | Visibility | Default Value | Description |
|----------|------|-------|------------|---------------|-------------|
| `v_product_id` | TEXT | COPIED | PUBLIC | `39d0efc1-5d3c-40f8-ad2b-85d7310998aa` | Current product ID being processed |

---

## Component Details

### 1. Start Component
**Type:** `start` - Initiates the product details processing workflow

#### Transitions
- **Unconditional:** → Truncate product details table

### 2. Truncate Product Details Table

**Type:** `sql-executor`  
**Purpose:** Clears previous product details data for fresh load

```sql
TRUNCATE TABLE PROD_BRONZE_DB.LMS.PRODUCT_DETAILS;
```

#### Transitions
- **Success:** → Table Iterator

### 3. Table Iterator (Core Processing Engine)

**Type:** `table-iterator`  
**Purpose:** Processes each product ID through the API extraction

#### Iterator Configuration
- **Mode:** Advanced (custom SQL query)
- **Concurrency:** Concurrent (parallel processing)
- **Break on Failure:** No (continues processing other products)
- **Target:** LMS API

#### Source Query
```sql
SELECT
    item.value:id::STRING AS id
FROM PROD_BRONZE_DB.LMS.PRODUCTS,
     LATERAL FLATTEN(input => DATA_VALUE:data) AS item
```

#### Column Mapping
- **ID** → **v_product_id** (passes product ID to API component)

#### Transitions
- **Success:** → Log product detail ingestion
- **Failure:** → Log LMS API Failure

### 4. LMS API (Iterator Target)

**Type:** `modular-api-extract-input-v2`  
**Purpose:** Extracts detailed product information for each product

#### API Configuration
- **Endpoint:** `Product_details`
- **URI Parameters:** `productId` using `${v_product_id}` variable
- **Authentication:** None
- **Target Table:** `PRODUCT_DETAILS`
- **Load Mode:** `APPEND` (accumulates data from all products)

### 5. Log Product Detail Ingestion

**Type:** `sql-executor`  
**Purpose:** Records successful processing metrics

```sql
INSERT INTO PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG (
    TABLE_NAME, ROW_COUNT, LOAD_TIME, LOAD_STATUS, ERROR_MESSAGE
)
SELECT 
    'PRODUCT_DETAILS' AS TABLE_NAME,
    COUNT(*) AS ROW_COUNT,
    CURRENT_TIMESTAMP AS LOAD_TIME,
    'SUCCESS' AS LOAD_STATUS,
    NULL AS ERROR_MESSAGE
FROM PROD_BRONZE_DB.LMS.PRODUCT_DETAILS;
```

#### Transitions
- **Success:** → Data Parser Product Details

### 6. Data Parser Product Details

**Type:** `sql-executor`  
**Purpose:** Transforms consolidated product details to silver layer

#### Complex Data Processing

**Target Table:** `PROD_SILVER_DB.LMS.PRODUCT_DETAILS`

#### Extracted Fields

**Core Product Information:**
- `api_key`, `created_at`, `description`, `discoverable`
- `icon_url`, `id`, `identification`, `name`
- `status`, `teaser`, `type`

**Pricing Information (First Price Element):**
- `price_api_key` - API key for pricing
- `price_min_quantity` - Minimum quantity for price
- `price_amount` - Price amount
- `price_type` - Type of pricing

**Price Group Association:**
- `price_group_id` - Group identifier
- `price_group_name` - Group name

#### Advanced JSON Processing
```sql
-- Prices array: first element
data_value:"prices"[0]:"api_key"::STRING AS price_api_key,
data_value:"prices"[0]:"min_quantity"::NUMBER AS price_min_quantity,
data_value:"prices"[0]:"price"::NUMBER AS price_amount,
data_value:"prices"[0]:"type"::STRING AS price_type,

-- Nested group object inside prices[0]
data_value:"prices"[0]:"group":"id"::STRING AS price_group_id,
data_value:"prices"[0]:"group":"name"::STRING AS price_group_name
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
│                       Truncate Product Details Table                          │
└───────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│           Table Iterator + LMS API (Per Product Processing)                    │
│           ┌───────────────────────────────────────────────────────┐         │
│           │ Product ID 1 → API Call → Product Details        │         │
│           │ Product ID 2 → API Call → Product Details        │         │
│           │ Product ID N → API Call → Product Details        │         │
│           └───────────────────────────────────────────────────────┘         │
└───────────────────────────────────────────────────────────────────────────────────┘
               │ (success)                                    │ (failure)
               ▼                                              ▼
┌─────────────────────────────┐        ┌──────────────────────────────┐
│ Log Product Detail         │        │ Log LMS API Failure         │
│ Ingestion (Success)        │        │ (Error Logging)             │
└─────────────────────────────┘        └──────────────────────────────┘
               │
               ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│               Data Parser Product Details (Bronze → Silver)                   │
└───────────────────────────────────────────────────────────────────────────────────┘
```

---

## Key Features

### Advanced Pricing Data Processing
- **Array Element Extraction:** Processes first element of prices array
- **Nested Object Handling:** Extracts nested group information from pricing
- **Complex Data Structures:** Handles multi-level JSON nesting
- **Price Group Association:** Links pricing to organizational groups

### Comprehensive Product Information
- **Product Metadata:** Complete product specifications and configuration
- **Pricing Intelligence:** Detailed pricing rules and group associations
- **Product Lifecycle:** Creation timestamps and status tracking
- **Discovery Settings:** Product visibility and availability configuration

### Iterator-Based Architecture
- **Product-by-Product Processing:** Individual API calls for complete detail extraction
- **Concurrent Processing:** Parallel processing for efficiency
- **Fault Tolerance:** Individual product failures don't stop overall processing
- **Append Strategy:** Accumulates data from all product API calls

---

## Data Schema (Silver Layer)

| Field | Data Type | Description |
|-------|-----------|-------------|
| **Product Core** | | |
| api_key | STRING | Product API access key |
| created_at | TIMESTAMP_TZ | Product creation timestamp |
| description | STRING | Product description |
| discoverable | BOOLEAN | Product visibility setting |
| icon_url | STRING | Product icon URL |
| id | STRING | Unique product identifier |
| identification | STRING | Product identification code |
| name | STRING | Product name |
| status | STRING | Product status |
| teaser | STRING | Product teaser text |
| type | STRING | Product type/category |
| **Pricing Information** | | |
| price_api_key | STRING | Price-specific API key |
| price_min_quantity | NUMBER | Minimum quantity for price |
| price_amount | NUMBER | Price amount |
| price_type | STRING | Pricing type/category |
| **Price Group** | | |
| price_group_id | STRING | Associated price group ID |
| price_group_name | STRING | Price group name |

---

## Usage Guidelines

### Historical Processing
- **Product Dependency:** Products must be loaded before product details
- **Complete Refresh:** Truncates and reloads all product details
- **API Intensive:** High number of individual API calls
- **Resource Monitoring:** Track compute and network usage

### Pricing Analysis
- **Group-Based Pricing:** Understand pricing variations by group
- **Quantity Thresholds:** Minimum quantity requirements
- **Price Type Analysis:** Different pricing models and strategies
- **Group Assignment:** Product-to-group relationship mapping

### Data Quality Validation
- **Product Coverage:** Ensure all products have detail records
- **Pricing Completeness:** Validate pricing information extraction
- **Group Association:** Verify price group relationships
- **Nested Data Integrity:** Check complex JSON processing accuracy

---

## Business Use Cases

- **Product Management:** Comprehensive product catalog management
- **Pricing Strategy:** Detailed pricing analysis and optimization
- **Group Administration:** Price group management and assignment
- **Product Analytics:** Usage, pricing, and performance analysis
- **Catalog Integration:** Product information for external systems

---

## Dependencies

### Data Dependencies
- **Products Data:** `PROD_BRONZE_DB.LMS.PRODUCTS` must be populated
- **Bronze/Silver Tables:** Target tables must exist with proper schemas
- **API Access:** LMS Product_details endpoint availability

### Technical Requirements
- **Iterator Support:** Table iterator functionality for concurrent processing
- **Complex JSON Processing:** Support for nested array and object extraction
- **Concurrent API Calls:** Ability to handle multiple simultaneous API requests
- **Append Processing:** Bronze layer table must support APPEND mode