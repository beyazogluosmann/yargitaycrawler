# yargitaycrawler
OtomaticCrawler


# GIB Decisions Workflow (n8n)

## Overview

This n8n workflow automates the process of fetching, processing, and storing tax rulings (özelge) from the Turkish Revenue Administration (Gelir İdaresi Başkanlığı - GIB). The workflow retrieves decision data from the GIB API, checks for existing records in MongoDB, processes new entries, generates PDFs, and publishes them to a Kafka queue.

## Workflow Architecture

### Main Components

1. **Data Fetching**: Retrieves ruling metadata from GIB API
2. **Deduplication**: Checks MongoDB for existing records
3. **Detail Processing**: Fetches full details for new rulings
4. **PDF Generation**: Converts ruling data to PDF format
5. **Message Publishing**: Sends processed data to Kafka

## Node Breakdown

### 1. Manual Trigger
- **Node**: `When clicking 'Execute workflow'`
- **Purpose**: Initiates the workflow manually

### 2. GIB API - Meta Information
- **Node**: `GİB API - Meta Bilgiler`
- **Type**: HTTP Request (POST)
- **Endpoint**: `https://gib.gov.tr/api/gibportal/mevzuat/ozelge/list`
- **Parameters**:
  - `page`: 0
  - `size`: 5
  - `sortFieldName`: priority
  - `sortType`: ASC
- **Purpose**: Fetches the initial list of tax rulings with metadata

### 3. Extract IDs
- **Node**: `ID'leri Çıkar`
- **Type**: Code (JavaScript)
- **Purpose**: Extracts ruling IDs from the API response and creates individual items for each ruling
- **Output**: Array of ruling objects with stats

### 4. Batch Query Preparation
- **Node**: `Toplu Sorgu Hazırlığı`
- **Type**: Code (JavaScript)
- **Purpose**: Prepares a list of ruling IDs for batch MongoDB query
- **Output**: Single item containing array of all IDs

### 5. MongoDB Batch Query
- **Node**: `Find documents (Toplu Sorgu)`
- **Type**: MongoDB
- **Collection**: `jurisprudence`
- **Query**: `{"gibId": {"$in": [array of IDs]}}`
- **Purpose**: Checks which rulings already exist in the database

### 6. Detect Missing Data
- **Node**: `Eksik Dataları Tespit Et`
- **Type**: Code (JavaScript)
- **Purpose**: Compares original IDs with MongoDB results to identify missing records
- **Output**: Three types of items:
  - `existing_data`: Records found in MongoDB
  - `missing_data`: Records not found in MongoDB
  - `summary`: Statistics summary

### 7. Data Processing Hub
- **Node**: `Veri İşleme Merkezi`
- **Type**: Code (JavaScript)
- **Purpose**: Routes data based on type (existing vs. missing)
- **Status Flags**:
  - `existing_in_mongodb`: For existing records
  - `missing_from_mongodb`: For new records

### 8. MongoDB Existence Check
- **Node**: `IF - MongoDB'de Var mı?`
- **Type**: IF (Conditional)
- **Condition**: `status === 'existing_in_mongodb'`
- **Routes**:
  - **True**: → Already Exists
  - **False**: → False Item Collector

### 9. Already Exists Handler
- **Node**: `Already Exists`
- **Type**: Code (JavaScript)
- **Purpose**: Logs existing records and returns them to the loop

### 10. False Item Collector
- **Node**: `False Item Toplayıcı`
- **Type**: Code (JavaScript)
- **Purpose**: Collects items missing from MongoDB and prepares them for detail fetching

### 11. Fetch Detail Information
- **Node**: `Detay Bilgisi Çek`
- **Type**: HTTP Request (GET)
- **Endpoint**: `https://gib.gov.tr/api/gibportal/mevzuat/ozelge/findById`
- **Parameter**: `id` (from validId)
- **Purpose**: Retrieves complete ruling details including description

### 12. Data Manipulation
- **Node**: `Detaylı Datayı Manipule İşlemleri`
- **Type**: Set (JSON Mode)
- **Purpose**: Structures the data for final output with fields:
  - `siteLink`, `index`, `type`, `DocType`
  - `lawNumber`, `lawName`, `filePath`
  - `gibId`, `decisionDate`, `decisionNo`
  - `description` (escaped)
  - Unix timestamps for dates

### 13. Kafka Data Preparation
- **Node**: `Producer Edilecek Kafka Datası`
- **Type**: Set (JSON Mode)
- **Purpose**: Adds `RDate` (current timestamp) to the data structure

### 14. Kafka Producer
- **Node**: `Kafka`
- **Type**: Kafka
- **Topic**: `deneme`
- **Key**: `multipage_{{ DocType }}_{{ gibId }}`
- **Purpose**: Publishes processed ruling data to Kafka queue

### 15. Wait After PDF
- **Node**: `PDF İşlemi Sonrası Bekleme (10-30 saniye)`
- **Type**: Wait
- **Duration**: Random 10-30 seconds
- **Purpose**: Rate limiting for PDF generation API

### 16. PDF Generation
- **Node**: `HTML Template Oluştur ve PDF'e Çevir`
- **Type**: HTTP Request (POST)
- **Endpoint**: `http://host.docker.internal:3001/api/v1/n8n/html-template-to-pdf`
- **Parameters**:
  - `description`: Ruling content
  - `title`: CommitDesc
  - `fileName`: Generated filename
- **Purpose**: Converts ruling data to PDF format

### 17. Loop Controller
- **Node**: `Loop Over Items`
- **Type**: Split In Batches
- **Purpose**: Processes items iteratively and controls workflow completion

## Data Flow

```
Manual Trigger
    ↓
GIB API (fetch metadata)
    ↓
Extract IDs (parse response)
    ↓
Batch Query Prep (prepare ID list)
    ↓
MongoDB Query (check existing)
    ↓
Detect Missing Data (compare lists)
    ↓
Data Processing Hub (route by status)
    ↓
    ├─→ IF Check
    │   ├─→ TRUE → Already Exists → Loop
    │   └─→ FALSE → False Item Collector
    │                    ↓
    │               Fetch Details
    │                    ↓
    │               Manipulate Data
    │                    ↓
    │               ├─→ Kafka Prep → Kafka → Loop
    │               └─→ Wait → PDF Generation → Loop
```

## Output Structure

### Kafka Message Format
```json
{
  "siteLink": "https://...",
  "index": "gsi_gib_alias",
  "type": "_doc",
  "DocType": "gib",
  "User": "scrapy",
  "language": "turkce",
  "Status": -1,
  "Searchable": true,
  "id": "multipage_gib_{id}",
  "lawNumber": 123,
  "lawName": "...",
  "filePath": "gib/pdf/gib-{id}.pdf",
  "CommitDesc": "...",
  "DocName": "multipage_gib_{id}",
  "fileName": "gib-{id}.pdf",
  "Category": "Gelir İdaresi Başkanlığı Özelge Kararları",
  "decisionSubject": "...",
  "acType": "ozelge",
  "gibId": 12345,
  "decisionDate": 1234567890,
  "decisionNo": "...",
  "Date": 1234567890,
  "RDate": "2025-02-10T12:00:00.000Z"
}
```

## Configuration Requirements

### Credentials Needed
1. **MongoDB**: Connection to `jurisprudence` collection
2. **Kafka**: Producer access to `deneme` topic

### Environment Variables
- MongoDB connection string
- Kafka broker addresses
- PDF generation service endpoint

## Rate Limiting

The workflow includes built-in rate limiting:
- Random 10-30 second delays between PDF operations
- Prevents API throttling from GIB servers
- Optional wait after API calls (currently disabled)

## Error Handling

Each processing node includes:
- Try-catch blocks for error recovery
- Validation checks for required fields (especially `ozelgeId`)
- Logging for debugging
- Fallback paths for missing data

## Monitoring Points

Key logging outputs:
- Total items fetched from API
- Existing vs. missing records count
- Success rate percentage
- Individual item processing status
- Error messages with timestamps

## Usage

1. Ensure all credentials are configured
2. Click "Execute workflow" to start
3. Monitor execution in n8n interface
4. Check Kafka topic for published messages
5. Verify PDFs are generated in the file path

## Notes

- The workflow processes 5 items per execution (configurable via `size` parameter)
- All dates are converted to Unix timestamps
- Descriptions are escaped for JSON compatibility
- The workflow uses Docker internal networking for PDF service

## Tags

- GİB API Test (1. Sayfa)
- GİB API
- GİB Workflow
- GİB API (Sayfalama)
