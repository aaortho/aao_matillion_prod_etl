# LMS Orders Load Pipeline Documentation

## Pipeline Overview

**Pipeline Name:** `pl_lms_orders_load.orch.yaml`  
**Pipeline Type:** Orchestration  
**Purpose:** Extract order/transaction data from LMS API, process through bronze to silver layers, and trigger detailed order processing pipeline

### Business Description

This pipeline manages the complete lifecycle of LMS order data ingestion. It retrieves order transactions, customer information, and financial data from the LMS API, processes the complex JSON structure containing nested user and pricing information, and executes downstream processing for detailed order analysis.

### Data Flow Architecture

```
LMS API → Bronze Layer (RAW JSON) → Silver Layer (STRUCTURED) → Order Details Pipeline
                                                    ↓ 
                                                Audit Log
```

- **Source**: LMS API Orders endpoint
- **Bronze Layer**: `PROD_BRONZE_DB.LMS.ORDERS` (raw JSON data)
- **Silver Layer**: `PROD_SILVER_DB.LMS.ORDERS` (parsed and structured data)
- **Downstream**: Order Details Historical Load Pipeline
- **Audit**: `PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG`

---

## Component Details

### 1. Start Component
**Type:** `start` - Initiates pipeline execution

### 2. LMS ORDERS (API Extract & Load)
**Type:** `modular-api-extract-input-v2`  
**Purpose:** Extracts order data from LMS API and loads into bronze layer

#### API Configuration
- **Endpoint:** `Orders`
- **Authentication:** None
- **Load Strategy:** Full refresh with TRUNCATE_AND_INSERT
- **Target Table:** `PROD_BRONZE_DB.LMS.ORDERS`

#### Transitions
- **Success:** → Log Orders Ingestion
- **Failure:** → Log Orders API Failure

### 3. Log Orders Ingestion
**Type:** `sql-executor`  
**Purpose:** Records successful data ingestion metrics in audit log

### 4. Data Parser Orders
**Type:** `sql-executor`  
**Purpose:** Parses complex JSON structure and upserts to silver layer

#### Complex Data Extraction

**Order Information:**
- `uuid`: Unique order identifier
- `created_at`: Order creation timestamp
- `external_reference`: External system reference
- `status`: Order processing status

**Customer Information (Nested):**
- `user_id`: User identifier from nested user object
- `api_username`: API username from user object
- `user_email`: Email address from user object
- `first_name`: Customer first name from user object
- `last_name`: Customer last name from user object

**Financial Information (Nested):**
- `current_total`: Order total amount (converted from cents to dollars)

#### JSON Processing Logic
- Uses `LATERAL FLATTEN` on `DATA_VALUE:data` to parse order records
- Extracts nested user information using dot notation (`item.value:user.id`)
- Handles financial data conversion (divides by 100 for cent-to-dollar conversion)
- Processes nested `current_totals.total` for pricing information

#### MERGE Operation
- **Target Table:** `PROD_SILVER_DB.LMS.ORDERS`
- **Match Condition:** `target.uuid = source.uuid`
- **When Matched:** Updates all order fields including customer and financial data
- **When Not Matched:** Inserts new order record

#### Transitions
- **Success:** → Order Details Pipeline

### 5. Log Orders API Failure
**Type:** `sql-executor`  
**Purpose:** Records API extraction failures for troubleshooting

### 6. Order Details Pipeline
**Type:** `run-orchestration`  
**Purpose:** Executes downstream detailed order analysis
- **Target:** `lms_data_ingestion/pl_lms_order_details_historical_load.orch.yaml`

---

## Data Schema (Silver Layer)

| Field | Data Type | Description |
|-------|-----------|-------------|
| uuid | STRING | Unique order identifier (primary key) |
| created_at | TIMESTAMP | Order creation date and time |
| external_reference | STRING | External system reference number |
| status | STRING | Current order status |
| user_id | STRING | Customer user ID (from nested user object) |
| api_username | STRING | Customer API username |
| user_email | STRING | Customer email address |
| first_name | STRING | Customer first name |
| last_name | STRING | Customer last name |
| current_total | NUMBER | Order total amount in dollars |

---

## Pipeline Flow Diagram

```
┌─────────┐    ┌────────────┐    ┌───────────────────┐    ┌──────────────────┐    ┌─────────────────────┐
│  Start  │───▶│ LMS ORDERS │───▶│ Log Orders         │───▶│ Data Parser      │───▶│ Order Details       │
└─────────┘    │ (API Load) │    │ Ingestion (Success)│    │ Orders           │    │ Pipeline            │
               └────────────┘    └───────────────────┘    │ (JSON→Structured) │    │ (Downstream)        │
                      │ (on failure)              └──────────────────┘    └─────────────────────┘
                      ▼
               ┌────────────────────┐
               │ Log Orders API     │
               │ Failure (Error)    │
               └────────────────────┘
```

---

## Key Features

### Complex JSON Processing
- **Nested Object Extraction:** Handles nested user and pricing objects
- **Data Type Conversion:** Converts financial amounts from cents to dollars
- **Comprehensive Mapping:** Extracts both order and customer information

### Financial Data Handling
- **Currency Conversion:** Automatically converts cents to dollars (`/100`)
- **Pricing Structure:** Processes nested `current_totals.total` field
- **Accurate Financial Reporting:** Maintains precision in monetary calculations

### Customer Information Integration
- **User Profile Extraction:** Pulls customer data from nested user object
- **Complete Customer View:** Includes identification and contact information
- **Relationship Mapping:** Links orders to customer profiles

---

## Usage Guidelines

### Data Quality Monitoring
- Verify financial calculations (cent-to-dollar conversion)
- Validate customer information completeness
- Monitor order status distribution
- Check for orphaned orders (missing user data)

### Business Intelligence Applications
- Order volume and trend analysis
- Customer purchase behavior tracking
- Revenue reporting and forecasting
- Order status workflow monitoring

---

## Dependencies

- LMS API Orders endpoint with nested data structure
- Order Details Historical Load Pipeline
- Customer/user data consistency
- Financial data accuracy requirements