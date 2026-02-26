# Repository Structure & Project Organization

Complete guide to the LLM Performance Observatory GitHub repository.

## Directory Structure

```
llm-performance-observatory/
│
├── 📋 README.md                          # Project overview & quick start
├── LICENSE                               # MIT License
├── .gitignore                            # Git ignore rules
├── CONTRIBUTING.md                       # Contribution guidelines
│
├── 📁 docs/                              # Documentation (everything else)
│   ├── README.md                         # Overview (you're reading this)
│   ├── ARCHITECTURE.md                   # System design & components
│   ├── MIGRATION.md                      # Data migration strategy
│   ├── DEPLOYMENT.md                     # Setup & deployment
│   ├── MONITORING.md                     # Alerting & observability
│   ├── PERFORMANCE.md                    # Scaling & optimization
│   ├── DATA_QUALITY.md                   # Validation & quality metrics
│   ├── TROUBLESHOOTING.md                # Common issues & fixes
│   ├── API.md                            # Query documentation
│   └── REPOSITORY_STRUCTURE.md           # This file
│
├── 📁 azure/                             # Azure infrastructure
│   ├── 📁 data-factory/                  # Data Factory pipelines
│   │   ├── pipeline.json                 # Main extraction pipeline
│   │   ├── linked_services.json          # Database connections
│   │   ├── datasets.json                 # Data references
│   │   └── triggers.json                 # Scheduling rules
│   │
│   ├── 📁 storage/                       # ADLS configuration
│   │   ├── lifecycle_policy.json         # Data tiering rules
│   │   ├── access_control.json           # RBAC configuration
│   │   └── container_config.json         # Container setup
│   │
│   ├── 📁 sql/                           # Azure SQL schemas
│   │   ├── schema.sql                    # Database schema
│   │   ├── tables.sql                    # Table definitions
│   │   ├── stored_procedures.sql         # SP definitions
│   │   └── indexes.sql                   # Index optimization
│   │
│   └── 📁 databricks/                    # Databricks notebooks
│       ├── transformation.scala          # ETL transformations
│       ├── aggregation.py                # Aggregation logic
│       └── quality_checks.py             # Data validation
│
├── 📁 src/                               # Python source code
│   ├── 📁 pipeline/                      # Data pipeline
│   │   ├── __init__.py
│   │   ├── extractor.py                  # Extract from source
│   │   ├── transformer.py                # Transform data
│   │   ├── loader.py                     # Load to ADLS
│   │   └── orchestrator.py               # Pipeline orchestration
│   │
│   ├── 📁 validation/                    # Data quality
│   │   ├── __init__.py
│   │   ├── schema_validator.py           # Schema checks
│   │   ├── completeness_checker.py       # Completeness validation
│   │   ├── anomaly_detector.py           # Anomaly detection
│   │   └── quality_scorer.py             # Quality metrics
│   │
│   ├── 📁 analysis/                      # Analytics engines
│   │   ├── __init__.py
│   │   ├── performance_analyzer.py       # Model comparisons
│   │   ├── cost_optimizer.py             # Cost analysis
│   │   └── error_analyzer.py             # Error pattern detection
│   │
│   └── 📁 utils/                         # Utilities
│       ├── __init__.py
│       ├── azure_client.py               # Azure SDK wrappers
│       ├── database.py                   # Database connections
│       ├── logging.py                    # Logging configuration
│       └── config.py                     # Config management
│
├── 📁 dashboard/                         # Frontend dashboards
│   ├── package.json                      # Node dependencies
│   ├── tsconfig.json                     # TypeScript config
│   │
│   ├── 📁 pages/                         # Next.js pages
│   │   ├── index.tsx                     # Dashboard home
│   │   ├── performance.tsx               # Performance metrics
│   │   ├── cost.tsx                      # Cost analysis
│   │   ├── errors.tsx                    # Error analysis
│   │   └── api/                          # API routes
│   │       ├── metrics.ts                # Metrics endpoint
│   │       ├── comparisons.ts            # Comparison endpoint
│   │       └── insights.ts               # Insights endpoint
│   │
│   ├── 📁 components/                    # React components
│   │   ├── MetricsCard.tsx               # Metric display
│   │   ├── ComparisonChart.tsx           # Comparison visualization
│   │   ├── CostBreakdown.tsx             # Cost visualization
│   │   └── ErrorDistribution.tsx         # Error visualization
│   │
│   ├── 📁 lib/                           # Utilities
│   │   ├── api_client.ts                 # API client
│   │   ├── data_formatter.ts             # Data formatting
│   │   └── constants.ts                  # Constants
│   │
│   └── 📁 styles/                        # Styling
│       ├── globals.css                   # Global styles
│       └── theme.css                     # Theme configuration
│
├── 📁 scripts/                           # Utility scripts
│   ├── deploy.py                         # Deployment script
│   ├── test-pipeline.py                  # Pipeline testing
│   ├── validate-data.py                  # Data validation runner
│   ├── generate-reports.py               # Report generation
│   ├── health-check.py                   # System health check
│   └── migrate-data.py                   # Data migration script
│
├── 📁 tests/                             # Test suite
│   ├── 📁 unit/                          # Unit tests
│   │   ├── test_extractor.py
│   │   ├── test_transformer.py
│   │   ├── test_loader.py
│   │   └── test_validators.py
│   │
│   ├── 📁 integration/                   # Integration tests
│   │   ├── test_pipeline_e2e.py          # End-to-end tests
│   │   ├── test_azure_connection.py      # Azure connectivity
│   │   └── test_database_schema.py       # Schema validation
│   │
│   └── 📁 fixtures/                      # Test data
│       ├── sample_interactions.json      # Sample records
│       ├── test_queries.sql              # Test queries
│       └── expected_output.json          # Expected results
│
├── 📁 .github/                           # GitHub configuration
│   ├── 📁 workflows/                     # CI/CD workflows
│   │   ├── tests.yml                     # Test workflow
│   │   ├── deploy.yml                    # Deploy workflow
│   │   └── validate-data.yml             # Data validation workflow
│   │
│   └── 📁 ISSUE_TEMPLATE/                # Issue templates
│       ├── bug_report.md
│       └── feature_request.md
│
├── requirements.txt                      # Python dependencies
├── package.json                          # Node dependencies
├── tsconfig.json                         # TypeScript config
├── pytest.ini                            # Pytest config
│
└── 📁 config/                            # Configuration files
    ├── dev.env.example                   # Dev environment example
    ├── prod.env.example                  # Prod environment example
    └── README.md                         # Configuration guide
```

---

## File Descriptions

### Documentation Files

| File | Purpose | Audience |
|------|---------|----------|
| **README.md** | Project overview, quick start, architecture summary | Everyone |
| **ARCHITECTURE.md** | Detailed system design, components, data flow | Engineers, Architects |
| **MIGRATION.md** | Data migration strategy and execution | Data Engineers |
| **DEPLOYMENT.md** | Setup, configuration, deployment procedures | DevOps, Operators |
| **MONITORING.md** | Alerting, observability, incident response | SREs, Ops, Engineers |
| **PERFORMANCE.md** | Scaling strategy, benchmarks, optimization | Architects, Engineers |
| **DATA_QUALITY.md** | Validation rules, quality metrics, remediation | Data Engineers, QA |
| **TROUBLESHOOTING.md** | Common issues and solutions | Support, Engineers |
| **API.md** | Query documentation, endpoint references | Data Scientists, Analysts |
| **REPOSITORY_STRUCTURE.md** | This file - directory organization | All developers |

### Azure Infrastructure Files

```
azure/data-factory/
├── pipeline.json         # ADF pipeline definition
├── linked_services.json  # Database connection configs
├── datasets.json         # Data source definitions
└── triggers.json         # Scheduling configuration

azure/storage/
├── lifecycle_policy.json # Data tiering (hot→cool→archive)
├── access_control.json   # Role-based access control
└── container_config.json # Container settings

azure/sql/
├── schema.sql            # Database structure
├── tables.sql            # Table and column definitions
├── stored_procedures.sql # Reusable database logic
└── indexes.sql           # Performance indexes
```

### Python Source Code

```
src/pipeline/
├── extractor.py          # Extract data from on-prem database
├── transformer.py        # Clean and transform data
├── loader.py             # Load to Azure Data Lake
└── orchestrator.py       # Orchestrate the pipeline

src/validation/
├── schema_validator.py   # Check schema compliance
├── completeness_checker.py # Verify data completeness
├── anomaly_detector.py   # Detect unusual patterns
└── quality_scorer.py     # Calculate quality metrics

src/analysis/
├── performance_analyzer.py # Model performance analysis
├── cost_optimizer.py      # Cost optimization insights
└── error_analyzer.py      # Error pattern detection

src/utils/
├── azure_client.py        # Azure SDK wrappers
├── database.py            # Database connection pooling
├── logging.py             # Structured logging
└── config.py              # Configuration management
```

### Dashboard Files

```
dashboard/
├── pages/                 # Next.js page routes
│   ├── index.tsx         # Homepage/dashboard
│   ├── performance.tsx    # Performance metrics page
│   ├── cost.tsx          # Cost analysis page
│   └── api/              # Backend API routes
│
├── components/            # React components
│   ├── MetricsCard.tsx   # Reusable metric cards
│   ├── ComparisonChart.tsx # Comparison charts
│   └── CostBreakdown.tsx # Cost visualization
│
└── lib/                   # Utility functions
    ├── api_client.ts     # HTTP client for API
    └── data_formatter.ts # Data transformation utilities
```

### Test Files

```
tests/unit/
├── test_extractor.py      # Test data extraction
├── test_transformer.py    # Test transformations
├── test_loader.py         # Test data loading
└── test_validators.py     # Test validation logic

tests/integration/
├── test_pipeline_e2e.py          # End-to-end pipeline test
├── test_azure_connection.py      # Azure connectivity test
└── test_database_schema.py       # Schema validation test

tests/fixtures/
├── sample_interactions.json      # Sample input data
├── expected_output.json          # Expected transformation results
└── test_queries.sql              # Test SQL queries
```

---

## Key Configuration Files

### requirements.txt
```
Python dependencies for data pipeline:
- azure-storage-blob=12.x
- azure-data-factory=x.x
- pandas=2.x
- pyspark=3.x
- sqlalchemy=2.x
- pytest=7.x
```

### package.json
```
Node dependencies for dashboard:
- next@14
- react@18
- typescript@5
- tailwindcss@3
```

### .github/workflows/
Automated CI/CD:
- **tests.yml** - Run test suite on every PR
- **deploy.yml** - Deploy to production on merge
- **validate-data.yml** - Run data validation daily

---

## Development Workflow

### 1. Clone Repository
```bash
git clone https://github.com/your-org/llm-performance-observatory.git
cd llm-performance-observatory
```

### 2. Set Up Development Environment
```bash
# Python setup
python -m venv venv
source venv/bin/activate  # or `venv\Scripts\activate` on Windows
pip install -r requirements.txt

# Node setup (for dashboard)
cd dashboard
npm install
cd ..
```

### 3. Configure Environment
```bash
cp config/dev.env.example .env.local
# Edit .env.local with your Azure credentials
```

### 4. Run Tests
```bash
# Unit tests
pytest tests/unit/

# Integration tests
pytest tests/integration/

# Full test suite
pytest
```

### 5. Run Pipeline Locally
```bash
python scripts/test-pipeline.py --mode dev
```

### 6. Run Dashboard
```bash
cd dashboard
npm run dev
# Visit http://localhost:3000
```

---

## Contributing

### Code Style
- Python: PEP 8 (use `black` for formatting)
- TypeScript: ESLint + Prettier
- SQL: Standard SQL formatting with comments

### Commit Guidelines
```
Format: <type>: <description>

Types:
- feat: New feature
- fix: Bug fix
- docs: Documentation
- test: Tests
- refactor: Code refactor
- perf: Performance improvement
- chore: Maintenance

Example:
feat: add anomaly detection for error spikes
```

### Pull Request Process
1. Create feature branch from `main`
2. Make changes with clear commit messages
3. Run all tests locally
4. Push to GitHub and create PR
5. Address code review feedback
6. Merge when approved

---

## Important Patterns

### Azure Connection Pattern
```python
# Correct: Use environment variables
from src.utils import get_azure_client
client = get_azure_client()

# Incorrect: Hardcode credentials
blob_client = BlobServiceClient.from_connection_string("...")
```

### Database Access Pattern
```python
# Correct: Use connection pooling
from src.utils.database import get_connection
with get_connection() as conn:
    cursor = conn.cursor()
    cursor.execute(query)

# Incorrect: Create new connection per query
conn = pyodbc.connect(...)  # Expensive!
```

### Error Handling Pattern
```python
# Correct: Log and raise
try:
    process_data()
except Exception as e:
    logger.error(f"Failed to process: {str(e)}", exc_info=True)
    raise

# Incorrect: Silently ignore
try:
    process_data()
except:
    pass  # Bad!
```

---

## Quick Reference

### Common Commands

```bash
# Run data pipeline
python scripts/test-pipeline.py

# Run health checks
python scripts/health-check.py

# Validate data quality
python scripts/validate-data.py

# Generate reports
python scripts/generate-reports.py

# Deploy to Azure
python scripts/deploy.py --environment prod

# Start dashboard
cd dashboard && npm run dev

# Run all tests
pytest --cov

# Check code style
black src/ tests/
flake8 src/ tests/
```

### File Locations for Common Tasks

| Task | File |
|------|------|
| Add new data validation rule | `src/validation/*.py` |
| Add new Azure resource | `azure/*/` + `scripts/deploy.py` |
| Add new dashboard page | `dashboard/pages/*.tsx` |
| Add new API endpoint | `dashboard/pages/api/*.ts` |
| Update data model | `azure/sql/tables.sql` + migration |
| Add new test | `tests/*/test_*.py` |
| Update documentation | `docs/*.md` |

---

## Deployment Architecture

```
Development
├─ Local: pytest + docker-compose
├─ Dev Environment: Separate Azure resources
└─ Feature branches: Automated testing

Staging
├─ Staging Environment: Realistic Azure setup
├─ Full data pipeline: On sample data
└─ Dashboard: Fully functional

Production
├─ Production Environment: Monitored Azure resources
├─ Real data pipeline: Full scale
├─ Dashboards: Live for stakeholders
└─ Monitoring: Alerting enabled
```

---

## Accessing the System

### As a Data Engineer
1. Clone repo: `git clone ...`
2. Install dependencies: `pip install -r requirements.txt`
3. Review: `docs/ARCHITECTURE.md` and `docs/MIGRATION.md`
4. Modify pipeline: `src/pipeline/*.py`
5. Deploy: `python scripts/deploy.py`

### As a DevOps Engineer
1. Review: `docs/DEPLOYMENT.md` and `docs/MONITORING.md`
2. Configure infrastructure: `azure/*` files
3. Set up monitoring: `docs/MONITORING.md`
4. Manage secrets: `.env` files in production

### As a Data Scientist
1. Review: `docs/ARCHITECTURE.md` and `docs/API.md`
2. Query data: Use SQL from `docs/API.md`
3. Access dashboard: `dashboard/pages/*.tsx`
4. Analyze results: Power BI or SQL queries

### As a DevOps/SRE
1. Review: `docs/DEPLOYMENT.md` and `docs/MONITORING.md`
2. Set up alerts: `docs/MONITORING.md`
3. Configure health checks: `scripts/health-check.py`
4. Handle incidents: `docs/TROUBLESHOOTING.md`

---

## Repository Statistics

```
Languages:
├── Python: ~5,000 lines (pipeline, validation)
├── SQL: ~2,000 lines (schemas, queries)
├── TypeScript/React: ~3,000 lines (dashboard)
├── Markdown: ~10,000 lines (documentation)
└── Configuration: ~1,000 lines (JSON, YAML)

Files:
├── Documentation: 10+ guides
├── Source Code: 20+ modules
├── Tests: 50+ test cases
├── Infrastructure: 15+ configuration files
└── Total: 100+ files

Commit Activity:
└── Active development with regular updates
```

---

## Support & Resources

- **Documentation**: See `docs/` folder
- **Issues**: GitHub Issues (with templates)
- **Discussions**: GitHub Discussions
- **Email**: support@your-org.com
- **Slack**: #llm-observatory channel

---

For getting started, see [README.md](./README.md)  
For questions, create an issue on GitHub
