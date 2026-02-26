# Monitoring, Alerting & Observability

Complete guide to monitoring the LLM Performance Observatory system in production.

## Table of Contents

1. [Monitoring Architecture](#monitoring-architecture)
2. [Key Metrics](#key-metrics)
3. [Alert Configuration](#alert-configuration)
4. [Dashboards](#dashboards)
5. [Log Analysis](#log-analysis)
6. [Health Checks](#health-checks)
7. [Incident Response](#incident-response)

---

## Monitoring Architecture

### Multi-Layer Monitoring Stack

```
┌─────────────────────────────────────────────────────┐
│              Application Layer                       │
│  (Data Factory, Databricks, Azure SQL)              │
└────────────────┬────────────────────────────────────┘
                 │
    ┌────────────┼────────────┐
    ▼            ▼            ▼
┌────────────┐ ┌───────────┐ ┌──────────────┐
│ Azure      │ │ Application │ │ Log Analytics │
│ Monitor    │ │ Insights   │ │ (Azure SQL)   │
└────┬───────┘ └─────┬─────┘ └────────┬──────┘
     │               │                │
     └───────────────┼────────────────┘
                     │
         ┌───────────▼───────────┐
         │  Alert Processing     │
         │  (Severity Routing)   │
         └───────────┬───────────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
    ┌────────┐  ┌────────┐  ┌────────┐
    │  SMS   │  │ Email  │  │ Slack  │
    └────────┘  └────────┘  └────────┘
```

---

## Key Metrics

### Tier 1: Critical Metrics (Must Monitor)

#### Pipeline Health

| Metric | Target | Alert |
|--------|--------|-------|
| **Daily Job Success Rate** | 99.9% | <99% |
| **Job Execution Time** | <2 hours | >3 hours |
| **Job Failure Count** | 0 per day | >0 |
| **Automatic Recovery** | 100% | <90% |

**Monitoring Query:**
```sql
SELECT 
    DATE(execution_start_time) as execution_date,
    COUNT(*) as total_runs,
    SUM(CASE WHEN status = 'SUCCEEDED' THEN 1 ELSE 0 END) as successful_runs,
    ROUND(100.0 * SUM(CASE WHEN status = 'SUCCEEDED' THEN 1 ELSE 0 END) / 
          COUNT(*), 2) as success_rate,
    AVG(DATEDIFF(MINUTE, execution_start_time, execution_end_time)) as avg_duration_min
FROM data_factory_runs
WHERE execution_start_time >= DATEADD(DAY, -30, GETDATE())
GROUP BY DATE(execution_start_time)
ORDER BY execution_date DESC;
```

#### Data Completeness

| Metric | Target | Alert |
|--------|--------|-------|
| **Daily Record Count** | 1.2M ±5% | <1.14M or >1.26M |
| **Record Count Variance** | <2% day-over-day | >2% |
| **Missing Timestamps** | 0 hours | Any gap >1 hour |
| **Model Coverage** | All 3 models | Missing any model |

**Monitoring Dashboard Query:**
```sql
SELECT 
    DATE(load_date) as load_date,
    model_name,
    COUNT(*) as record_count,
    MIN(timestamp) as min_timestamp,
    MAX(timestamp) as max_timestamp,
    COUNT(DISTINCT DATE_TRUNC(HOUR, timestamp)) as hours_covered,
    ROUND(100.0 * COUNT(*) / LAG(COUNT(*)) OVER (PARTITION BY model_name ORDER BY DATE(load_date)), 2) as day_over_day_change_pct
FROM silver.interactions
WHERE load_date >= DATEADD(DAY, -30, CAST(GETDATE() as DATE))
GROUP BY DATE(load_date), model_name
ORDER BY load_date DESC, model_name;
```

#### Query Performance

| Metric | Target | Alert |
|--------|--------|-------|
| **Query P95 Latency** | <1 second | >2 seconds |
| **Dashboard Load Time** | <3 seconds | >5 seconds |
| **Query Success Rate** | 99.9% | <99% |
| **Concurrent Users** | <50 | >100 |

```sql
-- Track query performance
SELECT 
    DATE_TRUNC(HOUR, end_time) as hour,
    COUNT(*) as total_queries,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duration_ms) as p95_duration_ms,
    PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY duration_ms) as p99_duration_ms,
    MAX(duration_ms) as max_duration_ms,
    AVG(rows_scanned) as avg_rows_scanned,
    SUM(CASE WHEN status = 'FAILED' THEN 1 ELSE 0 END) as failed_queries
FROM query_performance_log
WHERE end_time >= DATEADD(HOUR, -24, GETDATE())
GROUP BY DATE_TRUNC(HOUR, end_time)
ORDER BY hour DESC;
```

### Tier 2: Important Metrics (Check Regularly)

#### Storage & Cost

| Metric | Target | Warning | Alert |
|--------|--------|---------|-------|
| **Bronze Layer Size** | <50GB | >40GB | >50GB |
| **Silver Layer Size** | <30GB | >25GB | >30GB |
| **Gold Layer Size** | <5GB | >4GB | >5GB |
| **Monthly Cost** | <$200 | $150-200 | >$200 |

```sql
-- Monitor storage usage
SELECT 
    container_name,
    SUM(blob_size_bytes) / (1024*1024*1024) as total_size_gb,
    COUNT(*) as blob_count,
    AVG(last_modified) as last_modified_avg
FROM adls_blob_inventory
WHERE snapshot_date = CAST(GETDATE() as DATE)
GROUP BY container_name;
```

#### Data Quality Scores

| Metric | Target | Alert |
|--------|--------|-------|
| **Completeness Score** | >99.8% | <99.5% |
| **Null Rate (per column)** | <0.1% | >1% |
| **Duplicate Rate** | 0% | >0.01% |
| **Schema Validation** | 100% | <99.9% |

```python
# Data quality monitoring
def calculate_quality_score(df):
    """Calculate overall data quality"""
    
    metrics = {}
    
    # Completeness
    completeness = (len(df) - df.isnull().sum().sum()) / (len(df) * len(df.columns))
    metrics['completeness'] = completeness
    
    # Duplicates
    duplicate_rate = len(df[df.duplicated()]) / len(df)
    metrics['duplicate_rate'] = duplicate_rate
    
    # Null rates per column
    metrics['null_rates'] = df.isnull().sum() / len(df)
    
    # Overall score
    quality_score = (completeness * 0.5) + ((1 - duplicate_rate) * 0.3) + \
                   ((1 - metrics['null_rates'].mean()) * 0.2)
    
    return quality_score, metrics
```

---

## Alert Configuration

### Alert Severity Levels

#### Severity 1: CRITICAL (Immediate Action Required)

**Conditions:**
- Pipeline job failed (status = FAILED)
- Data completeness < 98%
- Database connection failed
- Schema validation failed

**Notification Method:**
- SMS to on-call engineer
- PagerDuty alert
- Team email
- Slack #critical-alerts

**Response Time:** <15 minutes

**Alert Rule Example:**
```yaml
alert_name: pipeline_failed
severity: critical
condition: 
  data_factory_runs.status = 'FAILED'
  AND DATE(execution_start_time) = CURRENT_DATE
notification:
  sms: "+1-XXX-XXX-XXXX"
  email: "oncall@company.com"
  slack: "#critical-alerts"
  pagerduty: true
action:
  investigate: pipeline logs
  if_issue_found:
    - Stop dependent jobs
    - Notify stakeholders
    - Page on-call manager
```

#### Severity 2: WARNING (Needs Attention)

**Conditions:**
- Job running >3 hours (slow)
- Data variance >5% from baseline
- Query latency P95 > 2 seconds
- Storage cost trending high

**Notification Method:**
- Slack notification
- Email to data team
- Dashboard alert

**Response Time:** <1 hour

#### Severity 3: INFO (Log for Later Review)

**Conditions:**
- Job completed with minor warnings
- Isolated failed records
- Unusual query patterns
- Data anomalies detected

**Notification Method:**
- Dashboard log entry
- Weekly report

---

## Dashboards

### Dashboard 1: Pipeline Health

**Purpose**: Real-time view of data ingestion pipeline

```sql
-- Pipeline Status Dashboard
SELECT 
    CAST(execution_start_time as DATE) as execution_date,
    COUNT(*) as total_runs,
    SUM(CASE WHEN status = 'SUCCEEDED' THEN 1 ELSE 0 END) as successful_runs,
    SUM(CASE WHEN status = 'FAILED' THEN 1 ELSE 0 END) as failed_runs,
    ROUND(100.0 * SUM(CASE WHEN status = 'SUCCEEDED' THEN 1 ELSE 0 END) / 
          COUNT(*), 2) as success_rate,
    AVG(DATEDIFF(MINUTE, execution_start_time, execution_end_time)) as avg_duration_min,
    MAX(DATEDIFF(MINUTE, execution_start_time, execution_end_time)) as max_duration_min
FROM data_factory_runs
WHERE execution_start_time >= DATEADD(DAY, -30, GETDATE())
GROUP BY CAST(execution_start_time as DATE)
ORDER BY execution_date DESC;
```

**Key Visualizations:**
- Success rate trend (last 30 days)
- Average execution time (last 30 days)
- Failed runs count
- Last run status & time

### Dashboard 2: Data Quality

**Purpose**: Monitor data completeness and quality

```sql
-- Data Quality Dashboard
WITH daily_stats AS (
    SELECT 
        DATE(load_date) as load_date,
        model_name,
        COUNT(*) as record_count,
        COUNT(DISTINCT DATE_TRUNC(HOUR, timestamp)) as hours_covered,
        COUNT(DISTINCT error_type) as error_types,
        ROUND(100.0 * SUM(CASE WHEN error_flag THEN 1 ELSE 0 END) / COUNT(*), 2) as error_rate
    FROM silver.interactions
    WHERE load_date >= DATEADD(DAY, -30, CAST(GETDATE() as DATE))
    GROUP BY DATE(load_date), model_name
)
SELECT 
    load_date,
    model_name,
    record_count,
    hours_covered,
    CASE 
        WHEN record_count > 1300000 THEN 'HIGH'
        WHEN record_count > 1100000 THEN 'NORMAL'
        ELSE 'LOW'
    END as volume_status,
    error_rate,
    CASE 
        WHEN error_rate > 5 THEN 'HIGH'
        WHEN error_rate > 2 THEN 'MEDIUM'
        ELSE 'LOW'
    END as error_status
FROM daily_stats
ORDER BY load_date DESC, model_name;
```

**Key Visualizations:**
- Daily record count by model
- Error rate trending
- Hours with data (should be 24)
- Completeness percentage

### Dashboard 3: Query Performance

**Purpose**: Monitor analytics layer performance

```sql
-- Query Performance Dashboard
SELECT 
    DATE_TRUNC(HOUR, query_end_time) as hour,
    COUNT(*) as query_count,
    PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY duration_ms) as p50_latency_ms,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duration_ms) as p95_latency_ms,
    PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY duration_ms) as p99_latency_ms,
    ROUND(100.0 * SUM(CASE WHEN status = 'SUCCESS' THEN 1 ELSE 0 END) / 
          COUNT(*), 2) as success_rate
FROM query_logs
WHERE query_end_time >= DATEADD(DAY, -7, GETDATE())
GROUP BY DATE_TRUNC(HOUR, query_end_time)
ORDER BY hour DESC;
```

**Key Visualizations:**
- Query latency percentiles (P50, P95, P99)
- Query success rate
- Concurrent user count
- Most common queries

---

## Log Analysis

### Log Sources

#### 1. Azure Data Factory Logs

```
Location: Azure Portal > Data Factory > Monitor > Pipeline Runs
Details: 
- Start time, end time
- Status (succeeded, failed, in progress)
- Duration
- Activities run
- Errors (if failed)
```

**Query Azure Monitor for ADF logs:**
```kusto
AzureDiagnostics
| where Category == "PipelineRuns"
| where TimeGenerated > ago(24h)
| summarize
    total_runs = dcount(CorrelationId),
    succeeded = dcountif(CorrelationId, Status == "Succeeded"),
    failed = dcountif(CorrelationId, Status == "Failed")
    by bin(TimeGenerated, 1h)
| project TimeGenerated, total_runs, succeeded, failed,
    success_rate = round(100.0 * succeeded / total_runs, 2)
```

#### 2. Application Insights Logs

```
Location: Azure Portal > Application Insights > Logs
Details:
- Custom events (job start, job end, validation results)
- Performance metrics
- Dependencies (database calls)
- Exceptions
```

**Query Application Insights:**
```kusto
customEvents
| where name == "DataLoadCompleted"
| where timestamp > ago(7d)
| extend 
    duration_minutes = todecimal(tostring(customDimensions.duration_minutes)),
    rows_loaded = todecimal(tostring(customDimensions.rows_loaded)),
    completeness = todecimal(tostring(customDimensions.completeness))
| summarize 
    avg_duration = avg(duration_minutes),
    avg_rows = avg(rows_loaded),
    avg_completeness = avg(completeness)
    by bin(timestamp, 1d)
```

#### 3. Azure SQL Audit Logs

```
Location: Azure Portal > SQL Database > Auditing
Details:
- Database access (who, when, what)
- Schema changes
- Data modifications
```

```sql
SELECT 
    server_principal_name,
    database_name,
    object_name,
    action_name,
    event_time,
    statement
FROM audit_log
WHERE event_time >= DATEADD(DAY, -7, GETDATE())
AND (action_name LIKE '%INSERT%' OR action_name LIKE '%DELETE%')
ORDER BY event_time DESC;
```

#### 4. Databricks Logs

```
Location: Databricks workspace > Clusters > Event logs
Details:
- Job execution status
- Cluster autoscaling events
- Performance metrics
```

---

## Health Checks

### Automated Health Check Script

```python
import logging
from datetime import datetime, timedelta
from azure.storage.blob import BlobServiceClient
from azure.data.tables import TableServiceClient
import requests

class SystemHealthCheck:
    def __init__(self):
        self.checks = []
        self.issues = []
        self.logger = self._setup_logging()
    
    def _setup_logging(self):
        logging.basicConfig(level=logging.INFO)
        return logging.getLogger(__name__)
    
    def check_pipeline_status(self):
        """Check if last pipeline run succeeded"""
        try:
            # Query metadata table
            table_client = TableServiceClient.from_connection_string(
                "connection_string"
            )
            
            last_run = table_client.get_entity(
                partition_key="pipeline",
                row_key="last_run"
            )
            
            status = last_run['status']
            run_time = last_run['timestamp']
            
            # Check: ran in last 24 hours
            if datetime.now() - run_time > timedelta(hours=24):
                self.issues.append({
                    'severity': 'CRITICAL',
                    'check': 'pipeline_status',
                    'message': f'Pipeline not run in 24 hours (last: {run_time})'
                })
                return False
            
            # Check: status was successful
            if status != 'SUCCEEDED':
                self.issues.append({
                    'severity': 'CRITICAL',
                    'check': 'pipeline_status',
                    'message': f'Last pipeline run failed with status: {status}'
                })
                return False
            
            self.logger.info("✓ Pipeline status: OK")
            return True
            
        except Exception as e:
            self.issues.append({
                'severity': 'CRITICAL',
                'check': 'pipeline_status',
                'message': f'Error checking pipeline: {str(e)}'
            })
            return False
    
    def check_data_completeness(self):
        """Check if yesterday's data is complete"""
        try:
            yesterday = (datetime.now() - timedelta(days=1)).date()
            
            # Query record count
            from sqlalchemy import create_engine, text
            engine = create_engine("mssql+pyodbc://...")
            
            with engine.connect() as conn:
                query = text("""
                SELECT COUNT(*) as row_count
                FROM silver.interactions
                WHERE DATE(load_date) = :date
                """)
                
                result = conn.execute(query, {"date": yesterday}).fetchone()
                row_count = result[0]
            
            # Expected: ~1.2 million rows (±10%)
            expected = 1_200_000
            tolerance = 0.10
            
            if row_count < expected * (1 - tolerance):
                self.issues.append({
                    'severity': 'WARNING',
                    'check': 'data_completeness',
                    'message': f'Low row count: {row_count:,} (expected ~{expected:,})'
                })
                return False
            
            self.logger.info(f"✓ Data completeness: {row_count:,} rows (OK)")
            return True
            
        except Exception as e:
            self.issues.append({
                'severity': 'WARNING',
                'check': 'data_completeness',
                'message': f'Error checking completeness: {str(e)}'
            })
            return False
    
    def check_storage_health(self):
        """Check if ADLS is accessible"""
        try:
            blob_client = BlobServiceClient.from_connection_string(
                "connection_string"
            )
            
            # Test by listing containers
            containers = blob_client.list_containers()
            container_list = [c['name'] for c in containers]
            
            required_containers = ['bronze', 'silver', 'gold']
            
            for container in required_containers:
                if container not in container_list:
                    self.issues.append({
                        'severity': 'CRITICAL',
                        'check': 'storage_health',
                        'message': f'Missing container: {container}'
                    })
                    return False
            
            self.logger.info("✓ Storage health: OK")
            return True
            
        except Exception as e:
            self.issues.append({
                'severity': 'CRITICAL',
                'check': 'storage_health',
                'message': f'Storage access error: {str(e)}'
            })
            return False
    
    def check_database_connection(self):
        """Check if databases are accessible"""
        try:
            from sqlalchemy import create_engine, text
            
            # Test source database
            engine_source = create_engine("mssql+pyodbc://...")
            with engine_source.connect() as conn:
                result = conn.execute(text("SELECT COUNT(*) FROM llm_interactions")).fetchone()
                source_count = result[0]
            
            self.logger.info(f"✓ Source database: OK ({source_count:,} rows)")
            
            # Test Azure SQL
            engine_azure = create_engine("postgresql+psycopg2://...")
            with engine_azure.connect() as conn:
                result = conn.execute(text("SELECT COUNT(*) FROM silver.interactions")).fetchone()
                azure_count = result[0]
            
            self.logger.info(f"✓ Azure SQL: OK ({azure_count:,} rows)")
            return True
            
        except Exception as e:
            self.issues.append({
                'severity': 'CRITICAL',
                'check': 'database_connection',
                'message': f'Database connection error: {str(e)}'
            })
            return False
    
    def run_all_checks(self):
        """Run all health checks"""
        print("\n" + "="*50)
        print("System Health Check Started")
        print("="*50)
        
        all_passed = True
        
        all_passed &= self.check_pipeline_status()
        all_passed &= self.check_data_completeness()
        all_passed &= self.check_storage_health()
        all_passed &= self.check_database_connection()
        
        print("\n" + "="*50)
        print("Health Check Results")
        print("="*50)
        
        if all_passed:
            print("✓ All checks passed - System is healthy")
        else:
            print(f"✗ {len(self.issues)} issues found:")
            for issue in self.issues:
                print(f"\n  [{issue['severity']}] {issue['check']}")
                print(f"  {issue['message']}")
        
        return all_passed, self.issues

# Run health checks daily
if __name__ == "__main__":
    health_check = SystemHealthCheck()
    passed, issues = health_check.run_all_checks()
    
    if not passed:
        # Send alert
        send_alert_to_slack(issues)
        send_alert_to_email(issues)
```

---

## Incident Response

### Incident Response Procedure

#### Step 1: Alert Received

```
Alert triggers → On-call engineer receives notification
Time: Immediate
```

#### Step 2: Triage (5 minutes)

```
1. Check dashboard for status
2. Determine severity level
3. Check if issue is self-healing
4. Decide if immediate action needed
```

**Triage Script:**
```python
def triage_incident(alert):
    severity = alert['severity']
    
    if severity == 'CRITICAL':
        # Page on-call manager
        escalate_to_manager(alert)
        
        # Check if self-healing
        if not is_self_healing(alert):
            start_incident_response(alert)
    
    elif severity == 'WARNING':
        # Log and monitor
        log_to_dashboard(alert)
        check_again_in_30_minutes(alert)
```

#### Step 3: Investigation (15 minutes)

**For Pipeline Failures:**
```
1. Check pipeline logs in Azure Data Factory
2. Identify which activity failed
3. Check error message
4. Determine root cause
```

**For Data Quality Issues:**
```
1. Query recent data
2. Identify which model has issues
3. Check for schema changes in source
4. Verify network connectivity
```

#### Step 4: Resolution

**Quick Fixes:**
- Retry failed job
- Restart connection
- Clear cache

**Complex Fixes:**
- Update transformation code
- Fix schema mismatch
- Adjust resource allocation

#### Step 5: Verification (5 minutes)

```
1. Confirm issue is resolved
2. Run health check again
3. Verify data completeness
4. Update team in Slack
```

### Common Issues & Resolutions

**Issue 1: Pipeline Job Timed Out**

```
Symptoms: Job running >3 hours
Cause: Data volume higher than normal
Solution:
  1. Increase Databricks worker count
  2. Optimize Spark query
  3. Re-run job
Expected Resolution Time: 30 minutes
```

**Issue 2: Source Database Connection Failed**

```
Symptoms: "Connection refused" error
Cause: Network issue or source DB down
Solution:
  1. Test connectivity to source DB
  2. Check firewall rules
  3. Restart connection
  4. If persists, use cached data
Expected Resolution Time: 15 minutes
```

**Issue 3: Data Completeness Below 99%**

```
Symptoms: Row count < 1.2M
Cause: Source data delayed, network issue, or transformation error
Solution:
  1. Check if source has data
  2. Check Azure import logs
  3. Re-run job with full extract
  4. Investigate transformation
Expected Resolution Time: 1 hour
```

---

## Summary

The monitoring system provides:

✓ **Real-time alerts** for critical issues  
✓ **Automated health checks** daily  
✓ **Comprehensive dashboards** for visibility  
✓ **Detailed logging** for investigation  
✓ **Clear incident response** procedures  

**Check dashboards daily, respond to alerts within SLA**
