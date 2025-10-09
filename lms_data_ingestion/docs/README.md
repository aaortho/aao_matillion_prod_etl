# LMS Data Ingestion Pipeline Documentation

## Overview

This documentation covers the complete Learning Management System (LMS) data ingestion suite, consisting of orchestration pipelines that extract, load, and transform data from the LMS API into Snowflake data warehouse layers following the medallion architecture pattern.

## Pipeline Architecture

### Data Flow Pattern
```
LMS API → Bronze Layer (Raw JSON) → Silver Layer (Structured) → Analytics/Reporting
                                           ↓
                                      Audit Logging
```

### Pipeline Categories

#### 1. Master Orchestration Pipeline
- **Purpose:** Coordinates execution of all entity pipelines
- **Execution:** Parallel processing for maximum efficiency

| Pipeline | Documentation |
|----------|---------------|
| Master Historical Data Load | [pl_lms_master_historical_data_load_documentation.md](./pl_lms_master_historical_data_load_documentation.md) |

#### 2. Core Entity Load Pipelines
- **Purpose:** Extract and process individual LMS entities
- **Pattern:** API Extract → Bronze Load → Silver Transform → Audit Log

| Entity | Pipeline | Documentation | Downstream Processing |
|--------|----------|---------------|----------------------|
| **Accreditation** | pl_lms_accreditation_load.orch.yaml | [Documentation](./pl_lms_accreditation_load_documentation.md) | None |
| **Events** | pl_lms_events_load.orch.yaml | [Documentation](./pl_lms_events_load_documentation.md) | Event Session Details |
| **Facilities** | pl_lms_facilities_load.orch.yaml | [Documentation](./pl_lms_facilities_load_documentation.md) | None |
| **Orders** | pl_lms_orders_load.orch.yaml | [Documentation](./pl_lms_orders_load_documentation.md) | Order Details |
| **Products** | pl_lms_products_load.orch.yaml | [Documentation](./pl_lms_products_load_documentation.md) | Product Details |
| **Speakers** | pl_lms_speakers_load.orch.yaml | [Documentation](./pl_lms_speakers_load_documentation.md) | Speaker Details |
| **Users** | pl_lms_users_load.orch.yaml | [Documentation](./pl_lms_users_load_documentation.md) | None |

#### 3. Historical Detail Pipelines
- **Purpose:** Process complex nested data and historical information
- **Trigger:** Executed by core entity pipelines

| Pipeline | Purpose | Triggered By |
|----------|---------|-------------|
| pl_lms_event_session_details_historical_load.orch.yaml | Event session details and attendance | Events Pipeline |
| pl_lms_order_details_historical_load.orch.yaml | Order line items and transaction details | Orders Pipeline |
| pl_lms_product_details_historical_load.orch.yaml | Product specifications and metadata | Products Pipeline |
| pl_lms_speaker_details_historical_load.orch.yaml | Speaker qualifications and assignments | Speakers Pipeline |

---

## Common Pipeline Components

### Standard Component Flow
1. **Start** - Pipeline initiation
2. **API Extract & Load** - `modular-api-extract-input-v2` component
3. **Success Audit Logging** - Records successful ingestion metrics
4. **Data Parser** - JSON parsing and silver layer MERGE operations
5. **Failure Audit Logging** - Records API failures
6. **Downstream Processing** (optional) - Triggers detail pipelines

### Shared Configuration
- **API Profile:** `custom-b95fee52-8db0-4fe4-94d1-623e1ca69a28`
- **Authentication:** None (authType: NONE)
- **Database:** `[Environment Default]` (PROD_BRONZE_DB)
- **Schema:** `LMS`
- **Load Mode:** `TRUNCATE_AND_INSERT`
- **Audit Table:** `PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG`

---

## Data Schema Overview

### Bronze Layer Tables
- **Purpose:** Raw JSON storage from LMS API
- **Location:** `PROD_BRONZE_DB.LMS.*`
- **Structure:** Contains `DATA_VALUE` column with JSON payload

### Silver Layer Tables
- **Purpose:** Parsed and structured data for analytics
- **Location:** `PROD_SILVER_DB.LMS.*`
- **Structure:** Normalized columns with appropriate data types

### Audit Logging
- **Table:** `PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG`
- **Purpose:** Processing status, row counts, and error tracking

---

## Key Features

### Data Quality & Monitoring
- **Comprehensive Audit Trail:** Every pipeline execution logged
- **Error Handling:** Graceful API failure management
- **Data Validation:** Type casting and format validation
- **Row Count Tracking:** Bronze to silver layer verification

### Performance Optimization
- **Parallel Processing:** Master pipeline executes all entities simultaneously
- **Efficient Loading:** TRUNCATE_AND_INSERT for clean data refresh
- **MERGE Operations:** Upsert functionality in silver layer
- **Modular Design:** Individual pipelines can run independently

### JSON Processing Patterns
- **LATERAL FLATTEN:** Standard pattern for parsing nested JSON
- **Type Casting:** Proper data type conversion (STRING, TIMESTAMP, NUMBER, BOOLEAN)
- **Nested Object Extraction:** Handles complex nested structures (user objects, pricing, credits)
- **Financial Data Handling:** Currency conversion (cents to dollars)

---

## Usage Guidelines

### For Complete Data Refresh
1. Execute the **Master Historical Data Load Pipeline**
2. Monitor all 7 entity pipeline executions
3. Verify audit logs for successful completion
4. Validate data quality in silver layer tables

### For Individual Entity Updates
1. Execute specific entity load pipeline
2. Check audit logs for processing status
3. Verify downstream detail pipeline execution (if applicable)
4. Validate data consistency

### Monitoring Best Practices
- Review `LMS_AUDIT_LOG` for processing status
- Validate row counts between bronze and silver layers
- Monitor API response times and potential timeouts
- Check for data quality issues and missing records
- Verify downstream pipeline execution success

---

## Dependencies

### External Systems
- **LMS API:** All endpoints must be accessible and responsive
- **Snowflake Environment:** Bronze and silver databases must exist
- **API Profile:** Custom profile configuration must be valid

### Infrastructure Requirements
- **Network Connectivity:** Stable connection to LMS API
- **Compute Resources:** Sufficient for parallel processing
- **Storage Capacity:** Adequate for complete historical data
- **Monitoring Tools:** Pipeline execution and error tracking

---

## Troubleshooting

### Common Issues
- **API Connectivity:** Check network and authentication
- **Resource Constraints:** Monitor compute and memory usage
- **Data Quality:** Validate source data format and completeness
- **Downstream Failures:** Review detail pipeline configurations

### Error Resolution
1. Check `LMS_AUDIT_LOG` for specific error messages
2. Review individual pipeline execution logs
3. Validate API endpoint availability
4. Verify Snowflake database permissions and capacity
5. Re-execute failed pipelines after issue resolution

---

## Contact & Support

For questions about these pipelines or to report issues:
- Review the individual pipeline documentation for detailed component information
- Check audit logs for processing status and error details
- Monitor pipeline execution through Matillion orchestration interface