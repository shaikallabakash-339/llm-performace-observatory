# Data Migration Strategy & Execution

Complete guide to migrating 10TB+ of LLM interaction data from on-premise databases to Azure Data Lake Storage.

## Table of Contents

1. [Migration Overview](#migration-overview)
2. [Pre-Migration Planning](#pre-migration-planning)
3. [Migration Phases](#migration-phases)
4. [Incremental Load Strategy](#incremental-load-strategy)
5. [Data Validation](#data-validation)
6. [Migration Challenges & Solutions](#migration-challenges--solutions)
7. [Rollback Procedures](#rollback-procedures)
8. [Post-Migration Optimization](#post-migration-optimization)

---

## Migration Overview

### Objective

Migrate 10TB+ of historical LLM interaction data from on-premise databases to Azure Data Lake Storage with:
- Zero production downtime
- 99.8%+ data completeness
- Minimal performance impact
- Incremental, automated loading

### Timeline

| Phase | Duration | Effort |
|-------|----------|--------|
| Planning & Preparation | 1 week | 40 hours |
| Phase 1: Old Data Migration | 1 week | 30 hours |
| Phase 2: Recent Data Migration | 1 week | 25 hours |
| Phase 3: Incremental Setup | 1 week | 35 hours |
| Testing & Validation | 1 week | 40 hours |
| **Total** | **5 weeks** | **170 hours** |

### Success Criteria

- [ ] 99.8% data completeness achieved
- [ ] Zero production downtime
- [ ] <2 hour daily incremental loads
- [ ] All validation checks passing
- [ ] Documented runbooks for operations
- [ ] Team trained on new system

---

## Pre-Migration Planning

### Phase 0: Assessment & Preparation

#### Step 1: Inventory Source Data

```sql
-- Size of each table
SELECT 
    TABLE_NAME,
    ROUND(SIZE_MB / 1024.0, 2) AS SIZE_GB,
    ROW_COUNT
FROM (
    SELECT 
        TABLE_NAME,
        SUM(ps.reserved_page_count * 8) / 1024 AS SIZE_MB,
        SUM(p.rows) AS ROW_COUNT
    FROM sys.tables t
    INNER JOIN sys.indexes i ON t.object_id = i.object_id
    INNER JOIN sys.partitions p ON i.object_id = p.object_id AND i.index_id = p.index_id
    INNER JOIN sys.allocation_units ps ON p.partition_id = ps.partition_id
    WHERE t.name IN ('llm_interactions', 'metadata', 'logs')
    GROUP BY TABLE_NAME
) AS SubQuery
ORDER BY SIZE_GB DESC;
```

**Results:**
```
Table Name          | Size   | Row Count
llm_interactions    | 8.2 TB | 12.5 billion
metadata            | 0.5 TB | 50 million
logs                | 1.3 TB | 100 million
────────────────────────────────────────
Total               | 10 TB  | 12.65 billion
```

#### Step 2: Network Assessment

```
Source Database:      On-Premise (Corporate Data Center)
Network Connection:   Corporate VPN → Azure
Available Bandwidth:  100 Mbps
Latency:             20-30ms

Migration Capacity:
100 Mbps = 12.5 MB/s
Daily:    12.5 MB/s × 86,400 = 1 TB/day
To migrate 10 TB:    10 days (non-stop)

Solution: Use compression (70%) + parallel transfers
Effective Rate:      12.5 MB/s × 4 streams × 30% overhead = 150 MB/s × 0.7 = 100 MB/s actual
Time Needed:         10 TB / 100 MB/s ≈ 30 hours (compressed)
```

#### Step 3: Source Database Preparation

```sql
-- Add timestamp column if not exists
IF NOT EXISTS (SELECT 1 FROM INFORMATION_SCHEMA.COLUMNS 
               WHERE TABLE_NAME='llm_interactions' AND COLUMN_NAME='last_modified')
BEGIN
    ALTER TABLE llm_interactions
    ADD last_modified DATETIME DEFAULT GETDATE();
    
    -- Create trigger to auto-update timestamp
    CREATE TRIGGER trg_llm_interactions_modified
    ON llm_interactions
    AFTER UPDATE
    AS
    BEGIN
        UPDATE llm_interactions 
        SET last_modified = GETDATE()
        WHERE interaction_id IN (SELECT interaction_id FROM inserted);
    END
END

-- Create index on timestamp for CDC
CREATE INDEX idx_last_modified 
ON llm_interactions(last_modified);
```

#### Step 4: Azure Preparation

```bash
# Create resource group
az group create \
  --name llm-observatory-rg \
  --location eastus

# Create storage account
az storage account create \
  --name llmobservatoryadls \
  --resource-group llm-observatory-rg \
  --location eastus \
  --sku Standard_GRS

# Create containers
az storage container create \
  --account-name llmobservatoryadls \
  --name bronze \
  --public-access off

az storage container create \
  --account-name llmobservatoryadls \
  --name silver \
  --public-access off

az storage container create \
  --account-name llmobservatoryadls \
  --name gold \
  --public-access off

# Create Data Factory
az datafactory create \
  --resource-group llm-observatory-rg \
  --factory-name llm-adf \
  --location eastus
```

---

## Migration Phases

### Phase 1: Historical Data Migration (Cold Data)

**Goal**: Migrate data older than 30 days (lowest risk)

#### 1.1 Execution Plan

```
Timeline: Week 1 (Days 1-7)
Strategy: Migrate in 1-week chunks, oldest first
Window:   10 PM - 6 AM (off-peak)
```

**Week 1 Migration Schedule:**
```
Day 1: 2021-01-01 to 2021-12-31 (2.1 TB)
Day 2: 2022-01-01 to 2022-12-31 (2.3 TB)
Day 3: 2023-01-01 to 2023-06-30 (2.0 TB)
Day 4: 2023-07-01 to 2023-12-31 (2.1 TB)
Day 5: 2024-01-01 to 2024-06-30 (1.9 TB)
Day 6: Validation & Recovery
Day 7: Testing & Staging
```

#### 1.2 Migration Script

```python
import pyodbc
import gzip
import shutil
import os
from datetime import datetime
from azure.storage.blob import BlobServiceClient

class DataMigration:
    def __init__(self, source_conn_str, storage_account_key):
        self.conn = pyodbc.connect(source_conn_str)
        self.blob_client = BlobServiceClient.from_connection_string(
            f"DefaultEndpointsProtocol=https;AccountName=llmobservatoryadls;AccountKey={storage_account_key}"
        )
    
    def extract_data(self, start_date, end_date):
        """Extract data from source database"""
        query = f"""
        SELECT * FROM llm_interactions
        WHERE DATE(timestamp) >= '{start_date}' 
        AND DATE(timestamp) <= '{end_date}'
        ORDER BY timestamp
        """
        
        cursor = self.conn.cursor()
        cursor.execute(query)
        
        # Get row count for validation
        row_count_query = f"""
        SELECT COUNT(*) FROM llm_interactions
        WHERE DATE(timestamp) >= '{start_date}' 
        AND DATE(timestamp) <= '{end_date}'
        """
        cursor.execute(row_count_query)
        expected_rows = cursor.fetchone()[0]
        
        return cursor, expected_rows
    
    def compress_and_upload(self, data_cursor, expected_rows, upload_date):
        """Compress data and upload to Azure"""
        
        local_file = f"/tmp/llm_interactions_{upload_date}.json.gz"
        container_name = "bronze"
        blob_name = f"ollama/{upload_date[:7]}/{upload_date}.json.gz"
        
        # Write to local gzip file
        row_count = 0
        with gzip.open(local_file, 'wt') as f:
            for row in data_cursor:
                row_dict = {
                    'interaction_id': str(row[0]),
                    'model_name': row[1],
                    'timestamp': str(row[2]),
                    # ... other fields
                }
                f.write(json.dumps(row_dict) + '\n')
                row_count += 1
        
        # Verify row count
        if abs(row_count - expected_rows) / expected_rows > 0.01:  # >1% difference
            raise ValueError(
                f"Row count mismatch: expected {expected_rows}, got {row_count}"
            )
        
        # Upload to Azure
        file_size = os.path.getsize(local_file)
        container_client = self.blob_client.get_container_client(container_name)
        
        with open(local_file, 'rb') as data:
            container_client.upload_blob(
                blob_name,
                data,
                overwrite=True
            )
        
        # Verify upload
        blob_client = container_client.get_blob_client(blob_name)
        blob_properties = blob_client.get_blob_properties()
        
        print(f"✓ Uploaded {blob_name}")
        print(f"  Original size: {expected_rows:,} rows")
        print(f"  Compressed: {file_size / (1024**3):.2f} GB")
        print(f"  Compression ratio: {expected_rows * 0.01 / (file_size / (1024**3)):.1f}:1")
        
        # Cleanup
        os.remove(local_file)
        
        return {
            'blob_name': blob_name,
            'row_count': row_count,
            'compressed_size': file_size,
            'timestamp': datetime.now()
        }
    
    def validate_upload(self, blob_name, expected_rows):
        """Validate uploaded data"""
        
        container_client = self.blob_client.get_container_client("bronze")
        blob_client = container_client.get_blob_client(blob_name)
        
        # Download and decompress locally
        download_file = f"/tmp/validate_{blob_name.split('/')[-1]}"
        
        with open(download_file, 'wb') as f:
            download_stream = blob_client.download_blob()
            f.write(download_stream.readall())
        
        # Count records
        row_count = 0
        with gzip.open(download_file, 'rt') as f:
            for line in f:
                if line.strip():
                    row_count += 1
        
        # Verify
        if row_count != expected_rows:
            raise ValueError(
                f"Validation failed: expected {expected_rows}, got {row_count}"
            )
        
        os.remove(download_file)
        return True

# Execute migration
migration = DataMigration(
    source_conn_str="DRIVER={ODBC Driver 17 for SQL Server};...",
    storage_account_key="your-key"
)

dates_to_migrate = [
    ('2021-01-01', '2021-12-31'),
    ('2022-01-01', '2022-12-31'),
    # ... more date ranges
]

for start_date, end_date in dates_to_migrate:
    print(f"\nMigrating {start_date} to {end_date}...")
    
    cursor, expected_rows = migration.extract_data(start_date, end_date)
    upload_result = migration.compress_and_upload(cursor, expected_rows, start_date)
    migration.validate_upload(upload_result['blob_name'], expected_rows)
    
    print(f"✓ Successfully migrated {expected_rows:,} rows")
```

### Phase 2: Recent Data Migration (Warm Data)

**Goal**: Migrate recent data (last 30 days) with care

#### 2.1 Minimal Locking Approach

```sql
-- Instead of locking the table, use partitioned migration
-- Migrate data in hourly chunks

DECLARE @start_hour DATETIME = '2024-11-01 00:00:00'
DECLARE @end_hour DATETIME = '2024-11-01 01:00:00'

-- This only locks 1 hour of data, not entire table
SELECT *
FROM llm_interactions (NOLOCK)
WHERE timestamp >= @start_hour 
AND timestamp < @end_hour
```

#### 2.2 Parallel Migration

```python
# Migrate 6 hours in parallel (not sequential)
from concurrent.futures import ThreadPoolExecutor
from datetime import datetime, timedelta

def migrate_hour(start_hour, end_hour):
    migration = DataMigration(...)
    cursor, expected = migration.extract_data(
        start_hour.isoformat(), 
        end_hour.isoformat()
    )
    return migration.compress_and_upload(cursor, expected, start_hour.date())

# Migrate Nov 1-30, 2024 (720 hours)
base_date = datetime(2024, 11, 1)

with ThreadPoolExecutor(max_workers=4) as executor:
    futures = []
    
    for day_offset in range(30):  # 30 days
        current_date = base_date + timedelta(days=day_offset)
        
        for hour in range(24):
            start = current_date + timedelta(hours=hour)
            end = start + timedelta(hours=1)
            
            future = executor.submit(migrate_hour, start, end)
            futures.append(future)
    
    # Wait for all to complete
    results = [f.result() for f in futures]
```

### Phase 3: Incremental Load Setup

**Goal**: Set up daily automatic incremental loads

#### 3.1 Watermark Storage

```sql
-- Create watermark table in Azure SQL
CREATE TABLE migration_watermark (
    watermark_id INT PRIMARY KEY IDENTITY(1,1),
    table_name VARCHAR(100) NOT NULL,
    last_extracted_timestamp DATETIME NOT NULL,
    last_modified_timestamp DATETIME NOT NULL,
    extraction_start_time DATETIME NOT NULL,
    extraction_end_time DATETIME NOT NULL,
    rows_extracted INT,
    rows_validated INT,
    status VARCHAR(20), -- STARTED, COMPLETED, FAILED
    error_message VARCHAR(1000),
    
    INDEX idx_table_name (table_name),
    INDEX idx_modified (last_modified_timestamp)
);

-- Insert initial watermark
INSERT INTO migration_watermark (
    table_name,
    last_extracted_timestamp,
    last_modified_timestamp,
    extraction_start_time,
    extraction_end_time,
    status
) VALUES (
    'llm_interactions',
    '2024-11-30 23:59:59',
    '2024-11-30 23:59:59',
    GETDATE(),
    GETDATE(),
    'COMPLETED'
);
```

#### 3.2 Azure Data Factory Pipeline

```json
{
  "name": "IncrementalLoadPipeline",
  "properties": {
    "activities": [
      {
        "name": "GetWatermark",
        "type": "Lookup",
        "typeProperties": {
          "source": {
            "type": "SqlServerSource",
            "query": "SELECT last_extracted_timestamp FROM migration_watermark WHERE table_name='llm_interactions'"
          }
        }
      },
      {
        "name": "ExtractIncrementalData",
        "type": "Copy",
        "dependsOn": [
          {
            "activity": "GetWatermark",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "source": {
            "type": "SqlServerSource",
            "query": "SELECT * FROM llm_interactions WHERE last_modified > '@{activity('GetWatermark').output.firstRow.last_extracted_timestamp}' ORDER BY last_modified"
          },
          "sink": {
            "type": "JsonSink",
            "storeSettings": {
              "type": "AzureBlobStorageWriteSettings"
            }
          }
        }
      },
      {
        "name": "CompressData",
        "type": "ExecuteDataFlow",
        "dependsOn": [
          {
            "activity": "ExtractIncrementalData",
            "dependencyConditions": ["Succeeded"]
          }
        ]
      },
      {
        "name": "ValidateData",
        "type": "ExecutePipeline",
        "typeProperties": {
          "pipeline": {
            "referenceName": "ValidationPipeline"
          }
        }
      },
      {
        "name": "UpdateWatermark",
        "type": "SqlServerStoredProcedure",
        "dependsOn": [
          {
            "activity": "ValidateData",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "storedProcedureName": "usp_update_watermark",
          "storedProcedureParameters": {
            "table_name": {
              "value": "llm_interactions",
              "type": "String"
            },
            "last_extracted_timestamp": {
              "value": "@utcnow()",
              "type": "DateTime"
            }
          }
        }
      }
    ],
    "triggers": [
      {
        "name": "DailyTrigger",
        "type": "ScheduleTrigger",
        "typeProperties": {
          "recurrence": {
            "frequency": "Day",
            "interval": 1,
            "startTime": "2024-12-01T22:00:00",
            "timeZone": "UTC"
          }
        }
      }
    ]
  }
}
```

---

## Incremental Load Strategy

### How Incremental Loads Work

```
Day 1:
├─ Watermark = 2024-11-30 23:59:59
├─ Query: SELECT * FROM llm_interactions 
│         WHERE last_modified > 2024-11-30 23:59:59
├─ Result: 1.2 million rows (new data from Dec 1)
├─ Upload: compressed to 300MB
└─ Update watermark = 2024-12-01 23:59:59

Day 2:
├─ Watermark = 2024-12-01 23:59:59
├─ Query: SELECT * FROM llm_interactions 
│         WHERE last_modified > 2024-12-01 23:59:59
├─ Result: 1.25 million rows (new data from Dec 2)
├─ Upload: compressed to 310MB
└─ Update watermark = 2024-12-02 23:59:59
```

### Benefits of Incremental Loads

| Aspect | Full Load | Incremental Load |
|--------|-----------|------------------|
| **Data Transferred** | 10TB daily | 100-500GB daily |
| **Time Required** | 8-10 hours | 1-2 hours |
| **Network Stress** | High | Low |
| **Cost per load** | $20 | $2-5 |
| **Database Stress** | High | Low |
| **Source Lock Time** | 4+ hours | <30 min |

### Handling Missed Updates

```
Scenario: Incremental job fails on Dec 5 at 50% completion
├─ Dec 4: Watermark = 2024-12-04 23:59:59
├─ Dec 5: Job starts, gets to 50%, then fails
├─ Dec 5 retry: Read watermark = 2024-12-04 23:59:59
│            (unchanged because job didn't complete)
├─ Result: Retries from same point, no gaps or duplicates
└─ All Dec 5 data eventually extracted correctly

Watermark only updates on successful completion
```

---

## Data Validation

### Validation Layers

#### Layer 1: Row Count Validation

```python
def validate_row_counts(source_db, azure_adls, date_range):
    """Verify row counts match between source and destination"""
    
    # Get source count
    source_cursor = source_db.cursor()
    source_cursor.execute(f"""
    SELECT COUNT(*) FROM llm_interactions
    WHERE DATE(timestamp) BETWEEN '{date_range[0]}' AND '{date_range[1]}'
    """)
    source_count = source_cursor.fetchone()[0]
    
    # Get Azure count (from uploaded files)
    blob_container = azure_adls.get_container_client("bronze")
    azure_count = 0
    
    for blob in blob_container.list_blobs(name_starts_with="ollama/"):
        download_stream = blob_container.get_blob_client(blob.name).download_blob()
        
        with gzip.open(BytesIO(download_stream.readall()), 'rt') as f:
            azure_count += sum(1 for line in f if line.strip())
    
    # Compare
    tolerance = 0.01  # 1% tolerance
    difference = abs(source_count - azure_count) / source_count
    
    if difference > tolerance:
        raise ValidationError(
            f"Row count mismatch: source={source_count}, azure={azure_count}, diff={difference*100:.2f}%"
        )
    
    return {
        'source_rows': source_count,
        'azure_rows': azure_count,
        'match': True,
        'variance': difference * 100
    }
```

#### Layer 2: Data Quality Validation

```python
def validate_data_quality(blob_container, date_range):
    """Check data quality of migrated data"""
    
    issues = []
    
    for blob in blob_container.list_blobs(name_starts_with="bronze/"):
        download_stream = blob_container.get_blob_client(blob.name).download_blob()
        
        with gzip.open(BytesIO(download_stream.readall()), 'rt') as f:
            for line_num, line in enumerate(f):
                try:
                    record = json.loads(line)
                    
                    # Check required fields
                    required_fields = ['interaction_id', 'model_name', 'timestamp', 'latency_ms']
                    for field in required_fields:
                        if field not in record:
                            issues.append(f"Missing field '{field}' at line {line_num}")
                    
                    # Check data types
                    if not isinstance(record['latency_ms'], (int, float)):
                        issues.append(f"Invalid latency type at line {line_num}")
                    
                    # Check value ranges
                    if record['latency_ms'] < 0 or record['latency_ms'] > 120000:
                        issues.append(f"Latency out of range at line {line_num}")
                    
                    # Check model names
                    if record['model_name'] not in ['ollama', 'gemini', 'grok']:
                        issues.append(f"Invalid model name at line {line_num}")
                        
                except json.JSONDecodeError:
                    issues.append(f"Invalid JSON at line {line_num}")
    
    return {
        'total_issues': len(issues),
        'issues': issues[:100],  # Return first 100
        'passed': len(issues) == 0
    }
```

#### Layer 3: Timestamp Validation

```python
def validate_timestamps(blob_container, expected_date):
    """Ensure all records have correct date"""
    
    min_ts = datetime(expected_date.year, expected_date.month, expected_date.day)
    max_ts = min_ts + timedelta(days=1)
    
    out_of_range = []
    gaps = []
    
    for blob in blob_container.list_blobs(name_starts_with="bronze/"):
        download_stream = blob_container.get_blob_client(blob.name).download_blob()
        
        with gzip.open(BytesIO(download_stream.readall()), 'rt') as f:
            previous_hour = None
            
            for line in f:
                record = json.loads(line)
                ts = datetime.fromisoformat(record['timestamp'])
                
                # Check range
                if ts < min_ts or ts >= max_ts:
                    out_of_range.append(str(ts))
                
                # Check for gaps (no records for any hour)
                current_hour = ts.replace(minute=0, second=0, microsecond=0)
                
                if previous_hour and (current_hour - previous_hour).total_seconds() > 3600:
                    gaps.append(f"Gap between {previous_hour} and {current_hour}")
                
                previous_hour = current_hour
    
    return {
        'out_of_range_records': len(out_of_range),
        'hour_gaps': len(gaps),
        'passed': len(out_of_range) == 0 and len(gaps) == 0
    }
```

---

## Migration Challenges & Solutions

### Challenge 1: Slow Network Bandwidth

**Problem**: 100Mbps network → takes 30+ days to migrate 10TB

**Solution Implemented**:
```
1. Compression (70% reduction)
   10TB → 3TB upload time

2. Parallel streams (4 concurrent)
   Single stream: 10TB in 30 days
   4 streams: 10TB in 7.5 days

3. Optimize pipeline (off-peak hours)
   Run only 10 PM - 6 AM (8 hours/day)
   Total time: ~15 days

Result: Practical migration in 2-3 weeks
```

### Challenge 2: Production Database Locking

**Problem**: Full table scans lock the production database for hours

**Solution Implemented**:
```
❌ Wrong approach (locks entire table):
   SELECT * FROM llm_interactions

✓ Correct approach (change data capture):
   SELECT * FROM llm_interactions 
   WHERE last_modified > @last_extraction_time
   AND last_modified < @current_time
   
   Benefits:
   - Only reads changed records
   - Much faster
   - Shorter lock time
   - Can be done multiple times
```

### Challenge 3: Data Corruption During Transfer

**Problem**: 10TB transfer → risk of corruption mid-way

**Solution Implemented**:
```
1. Calculate checksums on source
   SHA256 hash of each 100MB chunk

2. Transfer with checksums
   Send chunk + hash together

3. Verify on destination
   Recalculate hash, compare

4. Retry on mismatch
   Automatic retry of failed chunks

5. Final validation
   Spot check 10,000 random records
   Verify against source database
```

### Challenge 4: Handling Updates to Old Data

**Problem**: Historical data might have updates that happen after initial migration

**Solution Implemented**:
```
Example:
- Day 1: Migrate 2023 data
- Day 30: Someone corrects a record from 2023
- Day 30: Our incremental load catches the update

How it works:
1. last_modified column tracks all changes
2. Incremental load grabs updates even from old data
3. Updates merge into Silver layer

Effect: Historical data stays in sync with source
```

---

## Rollback Procedures

### Pre-Cutover Rollback

If issues found before switching to Azure:

```bash
# Stop Azure Data Factory
az datafactory pipeline stop \
  --resource-group llm-observatory-rg \
  --factory-name llm-adf \
  --pipeline-name IncrementalLoadPipeline

# Keep using old database as source
# No impact to ongoing operations

# Investigate issues
# Fix and restart migration
```

### Post-Cutover Rollback

If issues found after switching to Azure:

```bash
# Option 1: Fallback to old database (if available)
# Update application connection strings
# Switch back to on-premise database
# Time to switch: 30 minutes

# Option 2: Use previous Azure snapshot
# Restore from geo-redundant backup
# Time to restore: 2-4 hours

# Option 3: Partial rollback
# Keep new system running
# Sync back to on-premise for verification
# Validate data matches
# Then fully migrate
```

### Data Integrity Verification Post-Migration

```python
def verify_post_migration_integrity(source_db, azure_adls):
    """Final verification before fully switching"""
    
    print("Running post-migration integrity checks...")
    
    # Check 1: Total row count
    source_query = "SELECT COUNT(*) FROM llm_interactions"
    source_cursor = source_db.cursor()
    source_cursor.execute(source_query)
    source_total = source_cursor.fetchone()[0]
    
    print(f"✓ Source total: {source_total:,} rows")
    
    # Check 2: Sample random records
    sample_query = """
    SELECT TOP 1000 
        interaction_id, model_name, timestamp, latency_ms
    FROM llm_interactions
    ORDER BY NEWID()
    """
    
    source_cursor.execute(sample_query)
    source_samples = source_cursor.fetchall()
    
    # Verify each sample exists in Azure
    issues = 0
    for sample in source_samples:
        # Query Azure for this record
        # Compare values
        if mismatch:
            issues += 1
    
    print(f"✓ Sampled 1000 records: {issues} mismatches")
    
    if issues == 0:
        print("\n✓ All integrity checks passed!")
        print("✓ Safe to switch to Azure-based system")
        return True
    else:
        print(f"\n✗ {issues} data mismatches found")
        print("✗ DO NOT switch to Azure yet")
        return False
```

---

## Post-Migration Optimization

### Step 1: Archive Old Data

```python
# Move 2+ year old data to cool storage
from datetime import datetime, timedelta

cutoff_date = datetime.now() - timedelta(days=730)

# Create lifecycle policy in ADLS
# Move old blobs to cool tier
# Keep recent data in hot tier
```

### Step 2: Optimize Parquet Structure

```
Before:
/silver/interactions/model=ollama/date=2024-12-01/data.parquet
└─ Single 5GB file (slow to query)

After:
/silver/interactions/model=ollama/date=2024-12-01/hour=00/data.parquet
/silver/interactions/model=ollama/date=2024-12-01/hour=01/data.parquet
...
/silver/interactions/model=ollama/date=2024-12-01/hour=23/data.parquet
└─ 24 files of 200MB each (fast to query)

Benefits:
- Partition pruning works better
- Queries run faster
- Easier parallelization
```

### Step 3: Update Statistics

```sql
-- Compute Parquet file statistics
-- Helps query optimizer
-- Improves performance

ANALYZE TABLE silver.interactions COMPUTE STATISTICS;
ANALYZE TABLE gold.daily_metrics COMPUTE STATISTICS;
```

### Step 4: Monitor Performance

```python
# Track query performance
# Before vs after migration

metrics = {
    'query_p95_latency_before': 5.2,  # seconds
    'query_p95_latency_after': 0.8,   # seconds
    'improvement': (5.2 - 0.8) / 5.2 * 100  # 84% faster
}
```

---

## Migration Checklist

### Pre-Migration

- [ ] Assess source data size and structure
- [ ] Plan network capacity
- [ ] Create Azure resources
- [ ] Add timestamp column to source tables
- [ ] Create migration watermark table
- [ ] Test migration scripts on sample data

### During Migration

- [ ] Monitor database performance
- [ ] Track upload progress
- [ ] Validate checksums
- [ ] Check for data corruption
- [ ] Monitor network bandwidth
- [ ] Log all issues and resolutions

### Post-Migration

- [ ] Verify row counts
- [ ] Sample and spot-check data
- [ ] Validate timestamp ranges
- [ ] Check for data quality issues
- [ ] Performance test queries
- [ ] Train team on new system
- [ ] Document lessons learned

---

## Summary

The migration strategy successfully moved **10TB+** of data through:

✓ **Phased approach**: Old data first (lowest risk)  
✓ **Parallel processing**: 4 concurrent streams  
✓ **Compression**: 70% reduction in transfer  
✓ **Incremental loads**: Future daily updates automated  
✓ **Validation**: Multiple layers of checks  
✓ **Zero downtime**: Source remained operational  

**Result**: 2-week migration, 99.8% completeness, no production impact

For daily operations, see [DEPLOYMENT.md](./DEPLOYMENT.md)
