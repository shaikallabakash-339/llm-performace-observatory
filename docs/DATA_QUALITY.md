# Data Quality & Validation

Comprehensive guide to ensuring data quality in the LLM Performance Observatory.

## Table of Contents

1. [Data Quality Framework](#data-quality-framework)
2. [Validation Rules](#validation-rules)
3. [Quality Metrics](#quality-metrics)
4. [Anomaly Detection](#anomaly-detection)
5. [Data Lineage](#data-lineage)
6. [Quality Remediation](#quality-remediation)

---

## Data Quality Framework

### Multi-Layer Validation

```
┌─────────────────────────────────────┐
│ Layer 1: Ingestion Validation       │
│ (Source Data Check)                 │
│ - Row count                         │
│ - Schema validation                 │
│ - Data type checks                  │
└──────────────┬──────────────────────┘
               │
        ┌──────▼──────┐
        │  Extract    │
        └──────┬──────┘
               │
┌──────────────▼──────────────────────┐
│ Layer 2: Transformation Validation  │
│ - Null percentage checks            │
│ - Range validation                  │
│ - Duplicate detection               │
│ - Consistency checks                │
└──────────────┬──────────────────────┘
               │
        ┌──────▼──────┐
        │  Transform  │
        └──────┬──────┘
               │
┌──────────────▼──────────────────────┐
│ Layer 3: Output Validation          │
│ - Record count verification         │
│ - Aggregation accuracy              │
│ - Time series continuity            │
│ - Model completeness                │
└──────────────┬──────────────────────┘
               │
        ┌──────▼──────┐
        │   Load      │
        └──────┬──────┘
               │
┌──────────────▼──────────────────────┐
│ Layer 4: Analytics Validation       │
│ - Dashboard data accuracy           │
│ - Report consistency                │
│ - Trend analysis                    │
│ - Anomaly detection                 │
└─────────────────────────────────────┘
```

---

## Validation Rules

### Rule Set 1: Schema & Structure

#### Rule 1.1: Required Fields Present

```sql
-- Check all required columns exist
SELECT 
    COLUMN_NAME,
    DATA_TYPE,
    IS_NULLABLE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'llm_interactions'
    AND COLUMN_NAME IN (
        'interaction_id',
        'model_name',
        'timestamp',
        'request_tokens',
        'response_tokens',
        'latency_ms',
        'error_flag'
    )
ORDER BY ORDINAL_POSITION;

-- Validation: All columns present with correct type
ASSERT: 7 columns returned
ASSERT: interaction_id is UNIQUEIDENTIFIER (UUID)
ASSERT: timestamp is DATETIME2
ASSERT: latency_ms is INT
```

**Failure Action**: Pipeline stops, alert sent

#### Rule 1.2: Data Type Compliance

```python
# Validate data types in Parquet files
def validate_schema(df):
    schema_rules = {
        'interaction_id': 'string',
        'model_name': 'string',
        'timestamp': 'timestamp',
        'request_tokens': 'integer',
        'response_tokens': 'integer',
        'total_tokens': 'integer',
        'latency_ms': 'integer',
        'error_flag': 'boolean',
        'error_type': 'string'
    }
    
    violations = []
    for column, expected_type in schema_rules.items():
        actual_type = str(df[column].dtype)
        
        if expected_type == 'timestamp':
            if not isinstance(df[column].iloc[0], pd.Timestamp):
                violations.append(f"{column}: expected timestamp, got {actual_type}")
        elif expected_type == 'integer':
            if not pd.api.types.is_integer_dtype(df[column]):
                violations.append(f"{column}: expected integer, got {actual_type}")
    
    if violations:
        raise ValueError(f"Schema violations: {violations}")
    
    return True

# Test
validate_schema(df)  # Raises error if any type mismatch
```

**Failure Action**: Flag records, investigate source

### Rule Set 2: Data Completeness

#### Rule 2.1: Mandatory Field Non-Null

```sql
-- Check for nulls in required fields
SELECT 
    COUNT(*) as total_records,
    SUM(CASE WHEN interaction_id IS NULL THEN 1 ELSE 0 END) as null_id,
    SUM(CASE WHEN model_name IS NULL THEN 1 ELSE 0 END) as null_model,
    SUM(CASE WHEN timestamp IS NULL THEN 1 ELSE 0 END) as null_timestamp,
    SUM(CASE WHEN latency_ms IS NULL THEN 1 ELSE 0 END) as null_latency
FROM silver.interactions
WHERE load_date = CAST(GETDATE() as DATE);

-- Validation Rules
ASSERT: null_id = 0 (no missing IDs)
ASSERT: null_model = 0 (all records have model)
ASSERT: null_timestamp = 0 (all have timestamp)
ASSERT: null_latency = 0 (all have latency)

-- Warning threshold
WARN IF: Any null_* > 1000 (0.1% of 1.2M)
```

**Failure Action**: Alert data team, investigate cause

#### Rule 2.2: Expected Record Count

```sql
-- Verify daily record count is within expected range
DECLARE @yesterday DATE = CAST(GETDATE() - 1 as DATE);
DECLARE @expected_count INT = 1200000;  -- baseline
DECLARE @tolerance FLOAT = 0.05;         -- ±5%

SELECT 
    model_name,
    COUNT(*) as record_count,
    ROUND(100.0 * COUNT(*) / @expected_count, 2) as pct_of_baseline
FROM silver.interactions
WHERE load_date = @yesterday
GROUP BY model_name;

-- Validation
DECLARE @actual INT = (SELECT COUNT(*) FROM silver.interactions WHERE load_date = @yesterday);

IF @actual < @expected_count * (1 - @tolerance)
    RAISERROR('Below acceptable count', 16, 1);

IF @actual > @expected_count * (1 + @tolerance)
    RAISERROR('Above expected count (check for duplicates)', 16, 1);
```

**Expected Counts:**
- Ollama: ~400K records/day
- Gemini: ~450K records/day
- Grok: ~350K records/day
- Total: ~1.2M records/day

### Rule Set 3: Value Range Validation

#### Rule 3.1: Latency Bounds

```sql
-- Verify latency values are reasonable
SELECT 
    COUNT(*) as total_records,
    SUM(CASE WHEN latency_ms < 0 THEN 1 ELSE 0 END) as negative_latency,
    SUM(CASE WHEN latency_ms > 120000 THEN 1 ELSE 0 END) as extreme_latency,
    MIN(latency_ms) as min_latency,
    MAX(latency_ms) as max_latency,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY latency_ms) as p95_latency
FROM silver.interactions
WHERE load_date = CAST(GETDATE() - 1 as DATE);

-- Validation Rules
ASSERT: negative_latency = 0 (no negative values)
ASSERT: extreme_latency < 100 (less than 0.01% >2 minutes)
ASSERT: min_latency >= 1 (at least 1ms)
ASSERT: p95_latency < 10000 (95th percentile <10 seconds)
```

**Valid Ranges:**
- Minimum: 1ms (faster than physical limit)
- Maximum: 120,000ms (2 minutes timeout)
- P95 target: <5,000ms

#### Rule 3.2: Token Count Bounds

```sql
-- Verify token counts are reasonable
SELECT 
    model_name,
    COUNT(*) as records,
    SUM(CASE WHEN request_tokens < 1 THEN 1 ELSE 0 END) as zero_requests,
    SUM(CASE WHEN request_tokens > 100000 THEN 1 ELSE 0 END) as huge_requests,
    SUM(CASE WHEN response_tokens > 100000 THEN 1 ELSE 0 END) as huge_responses,
    AVG(request_tokens) as avg_req_tokens,
    AVG(response_tokens) as avg_resp_tokens
FROM silver.interactions
WHERE load_date = CAST(GETDATE() - 1 as DATE)
GROUP BY model_name;

-- Validation Rules
ASSERT: zero_requests = 0 (every request has tokens)
ASSERT: huge_requests < 1000 (rare edge cases)
ASSERT: avg_req_tokens between 50-500
ASSERT: avg_resp_tokens between 100-1000
```

**Expected Ranges:**
- Request tokens: 10 - 8,000
- Response tokens: 50 - 8,000
- Combined: 60 - 16,000

### Rule Set 4: Uniqueness Constraints

#### Rule 4.1: Primary Key Uniqueness

```sql
-- Check for duplicate interaction IDs
SELECT 
    interaction_id,
    COUNT(*) as occurrences
FROM silver.interactions
WHERE load_date = CAST(GETDATE() - 1 as DATE)
GROUP BY interaction_id
HAVING COUNT(*) > 1;

-- Validation
ASSERT: Query returns 0 rows (no duplicates)

-- If duplicates found:
-- 1. Log them
-- 2. Remove newer copies (keep first)
-- 3. Investigate source
```

**Failure Action**: Alert team, remove duplicates manually

#### Rule 4.2: Model Values

```sql
-- Ensure only expected model names exist
SELECT DISTINCT model_name
FROM silver.interactions
WHERE load_date = CAST(GETDATE() - 1 as DATE);

-- Validation
ASSERT: Model names are only: 'ollama', 'gemini', 'grok'
ASSERT: No misspellings or unexpected values

-- Flag any unexpected models:
SELECT *
FROM silver.interactions
WHERE load_date = CAST(GETDATE() - 1 as DATE)
    AND model_name NOT IN ('ollama', 'gemini', 'grok');
```

---

## Quality Metrics

### Metric 1: Completeness Score

```python
def calculate_completeness(df):
    """
    Completeness = (Non-null fields / Total possible fields) × 100
    """
    total_cells = len(df) * len(df.columns)
    non_null_cells = df.count().sum()
    completeness = (non_null_cells / total_cells) * 100
    
    return completeness

# Example
df = pd.read_parquet('silver/interactions/date=2024-12-01')
completeness = calculate_completeness(df)
print(f"Completeness: {completeness:.2f}%")  # Output: 99.82%

# Target: >99.8%
# Acceptable: >99.5%
# Alert if: <99.0%
```

### Metric 2: Accuracy Score

```python
def calculate_accuracy(df, rules):
    """
    Accuracy = (Records passing all rules / Total records) × 100
    """
    validation_results = []
    
    for rule_name, rule_func in rules.items():
        try:
            rule_func(df)
            validation_results.append(True)
        except:
            validation_results.append(False)
    
    passing_records = sum(validation_results)
    total_records = len(df)
    accuracy = (passing_records / total_records) * 100
    
    return accuracy

# Validation rules
rules = {
    'schema_valid': lambda df: validate_schema(df),
    'no_nulls': lambda df: validate_nulls(df),
    'latency_in_range': lambda df: validate_latency_range(df),
    'tokens_valid': lambda df: validate_tokens(df),
    'no_duplicates': lambda df: validate_no_duplicates(df)
}

accuracy = calculate_accuracy(df, rules)
print(f"Accuracy: {accuracy:.2f}%")  # Output: 99.95%

# Target: >99.9%
# Alert if: <99.0%
```

### Metric 3: Consistency Score

```python
def calculate_consistency(df_new, df_previous):
    """
    Consistency = Similarity between today's and yesterday's data distribution
    Measured using Kolmogorov-Smirnov test (0-1 scale)
    """
    from scipy.stats import ks_2samp
    
    # Compare latency distributions
    stat_latency, pvalue_latency = ks_2samp(df_new['latency_ms'], df_previous['latency_ms'])
    
    # Compare token distributions
    stat_tokens, pvalue_tokens = ks_2samp(df_new['total_tokens'], df_previous['total_tokens'])
    
    # Compare error rates
    new_error_rate = (df_new['error_flag'].sum() / len(df_new)) * 100
    prev_error_rate = (df_previous['error_flag'].sum() / len(df_previous)) * 100
    error_variance = abs(new_error_rate - prev_error_rate) / prev_error_rate * 100
    
    consistency = (
        (1 - stat_latency) * 0.4 +
        (1 - stat_tokens) * 0.4 +
        (1 - min(error_variance / 10, 1)) * 0.2
    ) * 100
    
    return consistency, {
        'latency_similarity': 1 - stat_latency,
        'token_similarity': 1 - stat_tokens,
        'error_rate_variance': error_variance
    }

consistency, details = calculate_consistency(today_data, yesterday_data)
print(f"Consistency: {consistency:.2f}%")  # Output: 97.5%

# Target: >95%
# Alert if: <80% (indicates anomaly)
```

### Daily Quality Report

```python
def generate_quality_report(load_date):
    """Generate daily data quality report"""
    
    df = pd.read_parquet(f's3://bronze/interactions/date={load_date}')
    
    report = {
        'load_date': load_date,
        'timestamp': datetime.now(),
        'metrics': {
            'record_count': len(df),
            'completeness': calculate_completeness(df),
            'accuracy': calculate_accuracy(df, validation_rules),
            'consistency': calculate_consistency(df, previous_day_df),
        },
        'by_model': {}
    }
    
    for model in ['ollama', 'gemini', 'grok']:
        model_df = df[df['model_name'] == model]
        report['by_model'][model] = {
            'record_count': len(model_df),
            'avg_latency': model_df['latency_ms'].mean(),
            'error_rate': (model_df['error_flag'].sum() / len(model_df)) * 100,
            'completeness': calculate_completeness(model_df)
        }
    
    # Overall score (weighted)
    overall_score = (
        report['metrics']['completeness'] * 0.4 +
        report['metrics']['accuracy'] * 0.3 +
        report['metrics']['consistency'] * 0.3
    )
    
    report['overall_quality_score'] = overall_score
    
    return report

# Generate and store report
report = generate_quality_report('2024-12-01')
save_to_database(report)

# Output example
#{
#  'overall_quality_score': 99.2,
#  'metrics': {
#    'record_count': 1195432,
#    'completeness': 99.82,
#    'accuracy': 99.95,
#    'consistency': 97.5
#  },
#  'by_model': {
#    'ollama': {'record_count': 398234, 'error_rate': 0.8, ...},
#    'gemini': {'record_count': 447123, 'error_rate': 1.2, ...},
#    'grok': {'record_count': 350075, 'error_rate': 2.1, ...}
#  }
#}
```

---

## Anomaly Detection

### Anomaly 1: Unexpected Volume Changes

```python
import numpy as np
from scipy import stats

def detect_volume_anomaly(daily_counts, threshold_sigma=2):
    """
    Detect unusually high or low daily record counts
    Uses z-score method (deviation from mean)
    """
    
    # Calculate z-score for each day
    mean_count = np.mean(daily_counts)
    std_dev = np.std(daily_counts)
    z_scores = [(count - mean_count) / std_dev for count in daily_counts]
    
    # Flag anomalies >2 standard deviations
    anomalies = []
    for i, z in enumerate(z_scores):
        if abs(z) > threshold_sigma:
            anomalies.append({
                'date': dates[i],
                'count': daily_counts[i],
                'expected': mean_count,
                'deviation': z,
                'severity': 'HIGH' if abs(z) > 3 else 'MEDIUM'
            })
    
    return anomalies

# Example data (last 30 days)
daily_counts = [
    1220000, 1205000, 1198000, 1225000, 1215000,
    1208000, 1222000, 1210000, 1200000, 1230000,
    # ... 20 more days
    500000,  # <-- ANOMALY: Only 500K (unusual drop)
]

anomalies = detect_volume_anomaly(daily_counts)

# Output:
# [
#   {
#     'date': '2024-12-15',
#     'count': 500000,
#     'expected': 1215000,
#     'deviation': -3.5,
#     'severity': 'HIGH'
#   }
# ]
```

**When Detected:**
- Alert: "Daily record count significantly below expected"
- Possible Causes: Source system down, network issue, source schema change
- Response: Check source database, verify network connectivity

### Anomaly 2: Latency Spikes

```python
def detect_latency_anomaly(df, threshold_percentile=95):
    """
    Detect unusual latency patterns
    """
    
    baseline_p95 = df['latency_ms'].quantile(0.95)
    baseline_p99 = df['latency_ms'].quantile(0.99)
    
    # Check by model
    anomalies = []
    
    for model in df['model_name'].unique():
        model_df = df[df['model_name'] == model]
        model_p95 = model_df['latency_ms'].quantile(0.95)
        
        # If model P95 is 2x higher than baseline
        if model_p95 > baseline_p95 * 2:
            anomalies.append({
                'model': model,
                'p95_latency': model_p95,
                'baseline_p95': baseline_p95,
                'increase_pct': ((model_p95 - baseline_p95) / baseline_p95) * 100,
                'affected_records': len(model_df[model_df['latency_ms'] > baseline_p95 * 1.5])
            })
    
    return anomalies

# Usage
anomalies = detect_latency_anomaly(df_today)

# Output:
# [
#   {
#     'model': 'grok',
#     'p95_latency': 12000,  # 12 seconds
#     'baseline_p95': 5000,  # normally 5 seconds
#     'increase_pct': 140,
#     'affected_records': 3425
#   }
# ]
```

**When Detected:**
- Alert: "Latency spike detected for model X"
- Possible Causes: API overload, model degradation, network issues
- Response: Check model API status, investigate query patterns

### Anomaly 3: Error Rate Increase

```python
def detect_error_spike(df_today, df_baseline, threshold_increase=1.5):
    """
    Detect unexpected increase in error rates
    """
    
    baseline_error_rate = (df_baseline['error_flag'].sum() / len(df_baseline)) * 100
    today_error_rate = (df_today['error_flag'].sum() / len(df_today)) * 100
    
    anomalies = []
    
    # Check if error rate increased significantly
    if today_error_rate > baseline_error_rate * threshold_increase:
        anomalies.append({
            'type': 'error_rate_increase',
            'baseline_rate': baseline_error_rate,
            'current_rate': today_error_rate,
            'increase_factor': today_error_rate / baseline_error_rate,
            'additional_errors': df_today['error_flag'].sum() - (len(df_today) * baseline_error_rate / 100)
        })
    
    # Check by error type
    error_distribution = df_today[df_today['error_flag']]['error_type'].value_counts()
    
    for error_type, count in error_distribution.items():
        error_pct = (count / len(df_today)) * 100
        
        if error_pct > 1:  # More than 1% of requests
            anomalies.append({
                'type': 'error_type_high',
                'error_type': error_type,
                'count': count,
                'percentage': error_pct
            })
    
    return anomalies
```

---

## Data Lineage

### Tracking Data Lineage

```python
class DataLineageTracker:
    """Track data origin and transformations"""
    
    def __init__(self, db_connection):
        self.db = db_connection
    
    def track_record(self, interaction_id):
        """Trace a record from source to analytics"""
        
        lineage = {
            'interaction_id': interaction_id,
            'timestamp': datetime.now(),
            'stages': []
        }
        
        # Stage 1: Source Database
        source_record = self.db.query("""
            SELECT * FROM llm_interactions_source
            WHERE interaction_id = %s
        """, (interaction_id,))
        
        lineage['stages'].append({
            'stage': 'Source',
            'system': 'On-Premise Database',
            'record_exists': source_record is not None,
            'timestamp': source_record['timestamp'] if source_record else None,
            'data_hash': hash(source_record) if source_record else None
        })
        
        # Stage 2: ADLS Bronze
        bronze_record = self.db.query("""
            SELECT * FROM bronze_interactions
            WHERE interaction_id = %s
        """, (interaction_id,))
        
        lineage['stages'].append({
            'stage': 'ADLS Bronze',
            'system': 'Azure Data Lake Storage',
            'record_exists': bronze_record is not None,
            'blob_path': f"bronze/interactions/{bronze_record['date']}/" if bronze_record else None
        })
        
        # Stage 3: ADLS Silver
        silver_record = self.db.query("""
            SELECT * FROM silver_interactions
            WHERE interaction_id = %s
        """, (interaction_id,))
        
        lineage['stages'].append({
            'stage': 'ADLS Silver',
            'system': 'Processed Data',
            'record_exists': silver_record is not None,
            'transformations_applied': ['deduplication', 'type_conversion', 'validation']
        })
        
        # Stage 4: ADLS Gold
        gold_record = self.db.query("""
            SELECT * FROM gold_daily_metrics
            WHERE metric_date = %s
        """, (silver_record['load_date'],))
        
        lineage['stages'].append({
            'stage': 'ADLS Gold',
            'system': 'Analytics Ready',
            'record_exists': gold_record is not None,
            'aggregation_applied': True
        })
        
        # Stage 5: Dashboard
        dashboard_visible = gold_record is not None
        
        lineage['stages'].append({
            'stage': 'Dashboard',
            'system': 'Power BI',
            'visible': dashboard_visible
        })
        
        return lineage

# Usage
tracker = DataLineageTracker(db)
lineage = tracker.track_record('550e8400-e29b-41d4-a716-446655440000')

# Output example:
#{
#  'interaction_id': '550e8400-e29b-41d4-a716-446655440000',
#  'stages': [
#    {
#      'stage': 'Source',
#      'system': 'On-Premise Database',
#      'record_exists': True,
#      'timestamp': '2024-12-01 14:25:33'
#    },
#    {
#      'stage': 'ADLS Bronze',
#      'system': 'Azure Data Lake Storage',
#      'record_exists': True
#    },
#    ...
#  ]
#}
```

---

## Quality Remediation

### Issue: Missing Data

```python
def remediate_missing_data(load_date, model_name):
    """
    Recover missing data from source
    """
    
    # Step 1: Identify missing time windows
    missing_hours = query_missing_hours(load_date, model_name)
    
    if not missing_hours:
        return {'status': 'no_issues', 'message': 'All data present'}
    
    print(f"Found {len(missing_hours)} missing hours: {missing_hours}")
    
    # Step 2: Re-extract from source
    for hour in missing_hours:
        print(f"Re-extracting {hour}...")
        
        extracted = extract_from_source(
            start_time=hour,
            end_time=hour + timedelta(hours=1),
            model=model_name
        )
        
        # Step 3: Upload to ADLS
        upload_to_adls(
            data=extracted,
            path=f"bronze/{model_name}/{load_date.strftime('%Y-%m-%d')}/{hour:02d}00.json.gz"
        )
    
    # Step 4: Re-transform and reload
    transform_and_load(load_date, model_name)
    
    # Step 5: Verify completeness
    completeness = verify_completeness(load_date, model_name)
    
    return {
        'status': 'remediated',
        'missing_hours_fixed': len(missing_hours),
        'final_completeness': completeness
    }
```

### Issue: Duplicate Records

```python
def remediate_duplicates(load_date):
    """
    Remove duplicate records
    """
    
    # Step 1: Identify duplicates
    duplicates = query("""
    SELECT interaction_id, COUNT(*) as count
    FROM silver.interactions
    WHERE load_date = %s
    GROUP BY interaction_id
    HAVING COUNT(*) > 1
    """, (load_date,))
    
    print(f"Found {len(duplicates)} duplicate IDs")
    
    # Step 2: Keep earliest, remove rest
    for dup_id, count in duplicates:
        remaining = query("""
        SELECT * FROM silver.interactions
        WHERE interaction_id = %s
        ORDER BY load_date, etl_timestamp
        LIMIT 1
        """, (dup_id,))
        
        delete("""
        DELETE FROM silver.interactions
        WHERE interaction_id = %s
        AND etl_timestamp > %s
        """, (dup_id, remaining['etl_timestamp']))
    
    # Step 3: Verify
    remaining_duplicates = len(query("""
    SELECT interaction_id
    FROM silver.interactions
    WHERE load_date = %s
    GROUP BY interaction_id
    HAVING COUNT(*) > 1
    """, (load_date,)))
    
    return {
        'status': 'remediated',
        'duplicates_removed': len(duplicates) * (count - 1),
        'remaining_duplicates': remaining_duplicates
    }
```

---

## Quality Checklist

- [ ] Daily quality score >99%
- [ ] No schema violations
- [ ] Completeness >99.8%
- [ ] No duplicate records
- [ ] All model names valid
- [ ] Latency values in range
- [ ] No unexplained volume changes
- [ ] Error rates within baseline
- [ ] All time periods covered
- [ ] Documentation updated

---

For monitoring implementation, see [MONITORING.md](./MONITORING.md)
