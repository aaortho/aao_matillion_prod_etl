# LMS Data Ingestion Pipelines - User Guide

## Overview

The LMS (Learning Management System) data pipelines automatically extract data from the LMS API and load it into our Snowflake data warehouse. The system works in two layers:

- **Bronze Layer**: Raw data exactly as received from the LMS API
- **Silver Layer**: Cleaned and structured data ready for analysis

The system includes 7 main data endpoints plus additional detail pipelines that run automatically.

---

## How the System Works

### Master Coordination Pipeline

The system starts with a master pipeline that runs all 7 data extraction pipelines at the same time for faster processing. This master pipeline:

1. **Starts all pipelines simultaneously** - All 7 endpoints begin extracting data at once
2. **Runs independently** - If one endpoint fails, the others continue running
3. **Triggers follow-up processes** - Some endpoints automatically start detail pipelines when they finish successfully

### General Data Flow Process

Every endpoint follows the same 4-step process:

1. **Extract Data**: Connect to the LMS API and download raw data
2. **Load to Bronze**: Store the raw data exactly as received
3. **Log Success**: Record that the extraction worked and how many records were processed
4. **Transform to Silver**: Clean and structure the data for use

If any step fails, the system logs the error and continues with other processes.

### Error Handling and Monitoring

The system maintains a detailed audit log that tracks:
- Which endpoint was processed
- How many records were loaded
- When the process ran
- Whether it succeeded or failed
- Error details if something went wrong

---

## Individual Endpoint Documentation

### 1. ACCREDITATION ENDPOINT

**What it does**: Downloads certification and training completion records

**Pipeline File**: `pl_lms_accreditation_load.orch.yaml`

#### Data Extraction Process

**API Connection Details**:
- **Endpoint**: `/Accreditation`
- **Method**: Uses date range filtering to get records between specific dates
- **Date Variables**: 
  - Start Date: Configurable (default: 2022-01-01)
  - End Date: Configurable (default: 2023-01-01)
- **Pagination**: Processes 100 records at a time

**Data Flow**:
1. **Connect to API**: The system connects to the LMS Accreditation endpoint with the specified date range
2. **Download Raw Data**: All certification records within the date range are downloaded as JSON
3. **Store in Bronze**: Raw data is stored in `PROD_BRONZE_DB.LMS.ACCREDITATION` table
4. **Log Success**: System records how many accreditation records were processed
5. **Transform Data**: The raw JSON is parsed and structured into the Silver layer
6. **Store in Silver**: Clean data is stored in `PROD_SILVER_DB.LMS.ACCREDITATION`

**What Data Gets Extracted**:
- Certificate information (ID, award date)
- User details (name, email, username)
- Test results (scores, attempts, completion date)
- Product information (course details)
- Credit information (type, amount, units)

**Data Transformation Process**:
- The system takes the complex nested JSON response
- Extracts user information from embedded user objects
- Parses test details including scores and attempts
- Processes credit information which can have multiple credits per record
- Creates structured rows with all information flattened

*[Screenshot Placeholder: Accreditation pipeline flow diagram]*
*[Screenshot Placeholder: Sample accreditation data structure]*

#### Special Features
- **Date Range Filtering**: Only gets records modified within specified dates
- **Complex Data Structure**: Handles nested objects for users, tests, and credits
- **Multiple Credits**: One accreditation can have multiple credit types

---

### 2. EVENTS ENDPOINT

**What it does**: Downloads learning event and course information

**Pipeline File**: `pl_lms_events_load.orch.yaml`

#### Data Extraction Process

**API Connection Details**:
- **Endpoint**: `/Events`
- **Method**: Gets all available events (no filtering)
- **Data Type**: Simple event catalog information

**Data Flow**:
1. **Connect to API**: System connects to the LMS Events endpoint
2. **Download All Events**: Gets complete list of available learning events
3. **Store in Bronze**: Raw event data stored in `PROD_BRONZE_DB.LMS.EVENTS`
4. **Log Success**: Records how many events were processed
5. **Transform Data**: Converts JSON to structured columns
6. **Store in Silver**: Clean data stored in `PROD_SILVER_DB.LMS.EVENTS`
7. **Trigger Detail Pipeline**: Automatically starts the Event Session Details pipeline

**What Data Gets Extracted**:
- Event ID and name
- Event identification codes
- API keys for the events
- Start and end dates

**Data Transformation Process**:
- Simple transformation from JSON to table columns
- Date fields converted to proper timestamp format
- Text fields cleaned and formatted

**Follow-up Process**:
After the main events are loaded, the system automatically:
- Takes each event ID
- Calls a separate pipeline to get detailed session information
- This creates comprehensive event session data

*[Screenshot Placeholder: Events pipeline with child pipeline connection]*
*[Screenshot Placeholder: Events data sample]*

---

### 3. FACILITIES ENDPOINT

**What it does**: Downloads training facility and location information

**Pipeline File**: `pl_lms_facilities_load.orch.yaml`

#### Data Extraction Process

**API Connection Details**:
- **Endpoint**: `/Facilities`
- **Method**: Gets all facility information (no filtering)
- **Data Type**: Location and contact information

**Data Flow**:
1. **Connect to API**: System connects to LMS Facilities endpoint
2. **Download Facilities**: Gets all facility records
3. **Store in Bronze**: Raw data stored in `PROD_BRONZE_DB.LMS.FACILITIES`
4. **Log Success**: Records processing statistics
5. **Transform Data**: Structures address and contact information
6. **Store in Silver**: Clean data in `PROD_SILVER_DB.LMS.FACILITIES`

**What Data Gets Extracted**:
- Facility ID and name
- Descriptions
- Complete address information (address lines, city, state, zip, country)
- Contact information (phone, website)

**Data Transformation Process**:
- Address fields are properly separated and cleaned
- Phone numbers formatted consistently
- Website URLs validated and stored
- Geographic information structured for location queries

*[Screenshot Placeholder: Facilities data structure]*
*[Screenshot Placeholder: Address parsing example]*

---

### 4. PRODUCTS ENDPOINT

**What it does**: Downloads course and product catalog information

**Pipeline File**: `pl_lms_products_load.orch.yaml`

#### Data Extraction Process

**API Connection Details**:
- **Endpoint**: `/Products`
- **Method**: Gets all product catalog data
- **Data Type**: Course and training product information

**Data Flow**:
1. **Connect to API**: System connects to LMS Products endpoint
2. **Download Products**: Gets complete product catalog
3. **Store in Bronze**: Raw data in `PROD_BRONZE_DB.LMS.PRODUCTS`
4. **Log Success**: Records how many products processed
5. **Transform Data**: Structures product information
6. **Store in Silver**: Clean data in `PROD_SILVER_DB.LMS.PRODUCTS`
7. **Trigger Detail Pipeline**: Starts Product Details pipeline for additional information

**What Data Gets Extracted**:
- Product ID and API keys
- Creation dates
- Product names and descriptions
- Status information (active, inactive, etc.)
- Product types and categories
- Visibility settings
- Marketing information (teasers, icons)

**Data Transformation Process**:
- Boolean fields (like visibility) properly converted
- Date fields formatted to standard timestamps
- Status codes mapped to readable values
- Product categories standardized

**Follow-up Process**:
After main products are loaded:
- System takes each product ID
- Makes individual API calls to get detailed product information
- Includes pricing, group information, and extended details
- Creates comprehensive product catalog

*[Screenshot Placeholder: Products pipeline with detail pipeline trigger]*
*[Screenshot Placeholder: Product catalog data sample]*

---

### 5. USERS ENDPOINT

**What it does**: Downloads user account and profile information

**Pipeline File**: `pl_lms_users_load.orch.yaml`

#### Data Extraction Process

**API Connection Details**:
- **Endpoint**: `/Users`
- **Method**: Gets all user account information
- **Data Type**: User profiles and account details

**Data Flow**:
1. **Connect to API**: System connects to LMS Users endpoint
2. **Download Users**: Gets all user account records
3. **Store in Bronze**: Raw data in `PROD_BRONZE_DB.LMS.USERS`
4. **Log Success**: Records user count processed
5. **Transform Data**: Structures user profile information
6. **Store in Silver**: Clean data in `PROD_SILVER_DB.LMS.USERS`

**What Data Gets Extracted**:
- User ID (primary identifier)
- API usernames
- Email addresses
- First and last names

**Data Transformation Process**:
- Email addresses validated and standardized
- Names properly formatted (title case)
- User IDs mapped consistently
- API usernames preserved exactly as received

*[Screenshot Placeholder: User data structure]*
*[Screenshot Placeholder: User profile examples]*

---

### 6. ORDERS ENDPOINT

**What it does**: Downloads purchase and transaction information

**Pipeline File**: `pl_lms_orders_load.orch.yaml`

#### Data Extraction Process

**API Connection Details**:
- **Endpoint**: `/Orders`
- **Method**: Gets all order and transaction data
- **Data Type**: Purchase history and financial transactions

**Data Flow**:
1. **Connect to API**: System connects to LMS Orders endpoint
2. **Download Orders**: Gets all order records
3. **Store in Bronze**: Raw data in `PROD_BRONZE_DB.LMS.ORDERS`
4. **Log Success**: Records order count processed
5. **Transform Data**: Processes financial and user information
6. **Store in Silver**: Clean data in `PROD_SILVER_DB.LMS.ORDERS`
7. **Trigger Detail Pipeline**: Starts Order Details pipeline

**What Data Gets Extracted**:
- Order unique identifiers
- Creation dates and timestamps
- External reference numbers
- Order status information
- User information (embedded in order)
- Financial totals and amounts

**Data Transformation Process**:
- Financial amounts converted from cents to dollars for easier analysis
- User information extracted from nested order data
- Timestamps converted to standard format
- Status codes mapped to readable descriptions
- Order totals properly formatted as currency

**Follow-up Process**:
After orders are loaded:
- System processes detailed line items for each order
- Gets product-specific information for each order
- Creates complete transaction history

*[Screenshot Placeholder: Orders data with financial processing]*
*[Screenshot Placeholder: Currency conversion example]*

---

### 7. SPEAKERS ENDPOINT

**What it does**: Downloads instructor and presenter information

**Pipeline File**: `pl_lms_speakers_load.orch.yaml`

#### Data Extraction Process

**API Connection Details**:
- **Endpoint**: `/Speakers`
- **Method**: Gets all speaker/instructor information
- **Data Type**: Presenter profiles and contact information

**Data Flow**:
1. **Connect to API**: System connects to LMS Speakers endpoint
2. **Download Speakers**: Gets all speaker records
3. **Store in Bronze**: Raw data in `PROD_BRONZE_DB.LMS.SPEAKERS`
4. **Log Success**: Records speaker count processed
5. **Transform Data**: Structures presenter information
6. **Store in Silver**: Clean data in `PROD_SILVER_DB.LMS.SPEAKERS`
7. **Trigger Detail Pipeline**: Starts Speaker Details pipeline

**What Data Gets Extracted**:
- Speaker unique identifiers
- API keys for speaker access
- Email addresses
- First and last names
- Identification codes

**Data Transformation Process**:
- Contact information validated and formatted
- Names standardized and cleaned
- Speaker identifiers mapped consistently
- Email addresses verified for format

**Follow-up Process**:
After speakers are loaded:
- System gets detailed speaker information
- Includes biographical information, specialties
- Creates complete instructor profiles

*[Screenshot Placeholder: Speaker profiles data]*
*[Screenshot Placeholder: Speaker details pipeline flow]*

---

## Additional Detail Pipelines

### Event Session Details Pipeline

**What it does**: Gets detailed information about individual training sessions within events

**Process**:
1. **Reads Event List**: Takes event IDs from the main Events table
2. **Individual Processing**: Makes separate API calls for each event
3. **Session Information**: Gets detailed session schedules, timings, and content
4. **Product Mapping**: Links sessions to specific products and courses

**Data Collected**:
- Session IDs and evaluation IDs
- Start and end times with timezone information
- Session titles and descriptions
- Product information for each session

### Product Details Pipeline

**What it does**: Gets comprehensive information about each product including pricing

**Process**:
1. **Reads Product List**: Takes product IDs from main Products table
2. **Individual API Calls**: Makes separate calls for each product ID
3. **Detailed Information**: Gets pricing, groups, and extended metadata
4. **Complex Data Processing**: Handles nested pricing arrays and group information

**Data Collected**:
- Complete product specifications
- Pricing information including minimum quantities
- Group associations and pricing tiers
- Extended product metadata

---

## System Configuration and Variables

### Date Range Controls

For the Accreditation endpoint, you can control which data gets extracted:
- **Start Date Variable**: `v_starttime` - Controls the earliest date for accreditation records
- **End Date Variable**: `v_endtime` - Controls the latest date for accreditation records

These can be adjusted based on your data needs:
- Daily runs: Set to yesterday's date
- Historical loads: Set to cover desired time period
- Full refresh: Set to cover all available data

### API Connection Settings

The system uses a custom API connector profile that includes:
- Authentication credentials (stored securely)
- API endpoint URLs
- Connection timeout settings
- Error handling preferences

### Data Storage Configuration

**Bronze Layer**:
- Stores data exactly as received from API
- Uses "truncate and insert" approach for fresh data each run
- Maintains raw JSON for troubleshooting

**Silver Layer**:
- Uses "merge" approach to handle updates and new records
- Maintains data history and changes
- Structured for easy querying and analysis

---

## Monitoring and Troubleshooting

### Checking Pipeline Status

The audit log table `PROD_BRONZE_DB.LMS.LMS_AUDIT_LOG` tracks every pipeline run:
- **Success Records**: Show how many records were processed
- **Failure Records**: Include error messages and timestamps
- **Performance Tracking**: Shows how long each process takes

### Common Issues and Solutions

**API Connection Problems**:
- Check network connectivity
- Verify API credentials are current
- Ensure API endpoints haven't changed

**Data Processing Issues**:
- Compare bronze and silver record counts
- Check for changes in API response format
- Verify date ranges for Accreditation endpoint

**Performance Issues**:
- Monitor concurrent pipeline execution
- Check Snowflake warehouse capacity
- Review data volumes and processing times

### Data Quality Monitoring

Regular checks should include:
- Comparing record counts between bronze and silver layers
- Verifying data freshness (latest load times)
- Checking for null values in critical fields
- Monitoring error rates and success percentages

---

## Maintenance and Operations

### Regular Tasks

**Daily**:
- Check audit logs for any failures
- Verify data freshness
- Monitor record counts for anomalies

**Weekly**:
- Review error trends
- Check system performance metrics
- Validate data quality

**Monthly**:
- Review and update date range variables if needed
- Check API documentation for changes
- Analyze usage patterns and performance

### When to Update Variables

**Accreditation Date Ranges**:
- For daily incremental loads: Set to previous day
- For catch-up processing: Extend date range as needed
- For historical analysis: Adjust to cover required period

**Detail Pipeline Controls**:
- Monitor processing times for iterator pipelines
- Adjust concurrency settings if needed
- Update ID ranges for large datasets

---

*Note: Screenshots and visual diagrams should be added to illustrate:*
- *Pipeline flow diagrams for each endpoint*
- *Sample data structures and transformations*
- *Error handling and audit logging examples*
- *Configuration screens and variable settings*
- *Monitoring dashboards and status views*