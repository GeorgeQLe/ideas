# DataPipe — No-Code ETL for Small Teams

## Executive Summary

DataPipe is a visual ETL pipeline builder that lets small data teams connect sources, transform data with a spreadsheet-like UI, and load it into their warehouse — no code required. It's Fivetran simplicity at a fraction of the cost, built for startups and small businesses that need to move data without hiring a data engineer.

---

## Problem Statement

**The pain:**
- Small companies have data in 10+ SaaS tools (Stripe, HubSpot, Shopify, GA4) but can't get it into one place for analysis
- Fivetran and Airbyte are expensive ($1+/MAR) or require DevOps to self-host
- Custom scripts break constantly, have no monitoring, and create technical debt
- Analysts spend 60%+ of their time wrangling data instead of analyzing it
- Most tools are either "connectors only" (no transformation) or "full platform" (too complex)

**Market gap:**
- Fivetran: excellent but expensive, connectors only, no transforms
- Airbyte: open-source but requires infrastructure management
- Stitch: limited connectors, acquired by Talend (enterprise focus)
- dbt: transforms only, requires SQL knowledge, no extraction
- No tool combines easy extraction + visual transforms + affordable pricing for small teams

---

## Target Users

### Primary Personas

**1. Jake — Data Analyst at a Startup**
- Runs analytics for a 30-person company
- Knows SQL but not Python/infrastructure
- Currently exports CSVs from SaaS tools and manually combines in Google Sheets
- Needs: automated data pipelines without DevOps

**2. Lauren — Operations Manager**
- Needs to combine Shopify sales data with Google Ads and inventory data
- Not technical but comfortable with spreadsheets
- Needs: visual interface, no-code transforms, Google Sheets output

**3. Sam — Solo Data Engineer**
- Only data person at a 50-person company
- Maintains a spaghetti of cron jobs and Python scripts
- Needs: reliable managed pipelines, monitoring, alerting

---

## Core Features

### F1: Source Connectors (50+)
- **Payment/Finance**: Stripe, PayPal, QuickBooks, Xero, Brex
- **CRM/Sales**: HubSpot, Salesforce, Pipedrive, Close
- **Marketing**: Google Ads, Facebook Ads, LinkedIn Ads, Mailchimp, Klaviyo
- **Analytics**: Google Analytics 4, Mixpanel, Amplitude, Segment
- **E-commerce**: Shopify, WooCommerce, BigCommerce, Amazon Seller
- **Support**: Intercom, Zendesk, Freshdesk
- **Product**: Jira, Linear, GitHub, Notion
- **Databases**: PostgreSQL, MySQL, MongoDB, SQL Server
- **Files**: S3, Google Sheets, CSV/Excel upload, SFTP
- **APIs**: Generic REST API connector, GraphQL connector, Webhook receiver
- Each connector: OAuth or API key auth, table selection, field mapping
- Incremental sync (only new/changed data) and full refresh modes

### F2: Visual Transform Builder
- **Spreadsheet-like data preview**: see actual data at each step
- **Transform nodes** (drag-and-drop):
  - **Filter**: include/exclude rows based on conditions
  - **Map/Rename**: rename columns, change types
  - **Computed columns**: formulas (like spreadsheet formulas)
  - **Aggregate**: group by + sum, count, avg, min, max
  - **Join**: combine two data sources (inner, left, right, full)
  - **Union**: stack data from similar sources
  - **Pivot/Unpivot**: reshape data
  - **Deduplicate**: remove duplicate rows
  - **Sort**: order rows
  - **Limit**: take top/bottom N rows
  - **Split column**: split text by delimiter
  - **Date functions**: extract year, month, day, parse formats
  - **Text functions**: trim, lower, upper, replace, regex extract
- **SQL editor**: write custom SQL for advanced transforms
- **Data preview at each step**: see input/output sample data
- **Schema inference**: auto-detect column types

### F3: Destination Support
- **Warehouses**: BigQuery, Snowflake, Redshift, ClickHouse
- **Databases**: PostgreSQL, MySQL
- **Files**: S3, Google Cloud Storage
- **Spreadsheets**: Google Sheets (great for non-technical users)
- **APIs**: Webhook (POST data to any endpoint)
- Load modes: append, upsert (merge), replace
- Schema management: auto-create tables, handle schema changes

### F4: Pipeline Scheduling & Orchestration
- Schedule: every 15 minutes, hourly, daily, weekly, custom cron
- Manual trigger: run now button
- Dependency chains: Pipeline B runs after Pipeline A completes
- Retry on failure: configurable retry count and backoff
- Timeout settings per pipeline
- Concurrent execution limits
- Pause/resume pipelines

### F5: Monitoring & Alerting
- Pipeline run history with status (success, failed, running)
- Detailed logs per run: rows extracted, transformed, loaded
- Error details with stack trace and failing row data
- Duration tracking and trend charts
- Alert channels: email, Slack, PagerDuty, webhook
- Alert on: failure, long-running pipelines, zero rows synced, schema change
- Dashboard with overall pipeline health

### F6: Data Preview & Exploration
- Preview source data before building pipeline
- See row counts and schema for each table
- Sample data viewer at any pipeline step
- Query builder for filtering source data at extraction
- Column-level statistics (nulls, unique values, min/max)

### F7: Version Control & Collaboration
- Pipeline version history
- Diff view between versions
- Rollback to previous version
- Team collaboration: shared pipelines, comments
- Access control: viewer, editor, admin roles
- Pipeline documentation (inline notes)

### F8: Templates & Recipes
- Pre-built pipeline templates:
  - "Stripe → BigQuery revenue reporting"
  - "Shopify + Google Ads → warehouse for ROAS"
  - "HubSpot → PostgreSQL CRM sync"
  - "GA4 → Google Sheets weekly report"
- Community-shared recipes
- Clone and customize any template

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 14+, Tailwind CSS, React Flow (pipeline editor), AG Grid (data preview) |
| Backend | Next.js API Routes, tRPC |
| Database | PostgreSQL (Neon) — metadata and pipeline configs |
| Orchestration | Temporal.io (durable workflow execution) |
| Transform Engine | DuckDB (in-process SQL for transforms) |
| Extraction Workers | Python (Singer/Airbyte protocol compatible connectors) |
| Queue | Redis (Upstash) |
| Auth | Clerk |
| Hosting | Vercel (app), AWS ECS (workers) |

### Pipeline Execution Flow

```
Schedule/Manual Trigger
        │
        ▼
  Temporal Workflow
        │
        ├── Step 1: Extract (connector worker)
        │   └── Fetch data from source API → write to staging (S3 parquet)
        │
        ├── Step 2: Transform (DuckDB worker)
        │   └── Load staging data → apply transform DAG → write to staging
        │
        ├── Step 3: Load (loader worker)
        │   └── Read transformed data → write to destination
        │
        └── Step 4: Cleanup & notify
            └── Update run status, send alerts if needed
```

### Data Model

```
Organization
├── id, name, slug, plan
│
├── Connection[]
│   ├── id, org_id, name, type (source/destination)
│   ├── connector_type (stripe/bigquery/postgres/...)
│   ├── credentials (encrypted JSON)
│   ├── schema_cache (JSON: tables, columns, types)
│   ├── status (connected/error)
│   └── last_tested_at
│
├── Pipeline[]
│   ├── id, org_id, name, description
│   ├── source_connection_id, destination_connection_id
│   ├── source_config (JSON: tables, filters, incremental keys)
│   ├── transform_config (JSON: DAG of transform nodes)
│   ├── destination_config (JSON: target table, load mode)
│   ├── schedule (cron expression)
│   ├── status (active/paused/error)
│   ├── version, created_at, updated_at
│   │
│   └── PipelineRun[]
│       ├── id, pipeline_id, version
│       ├── status (running/success/failed/cancelled)
│       ├── started_at, completed_at, duration_ms
│       ├── rows_extracted, rows_transformed, rows_loaded
│       ├── error_message, error_details (JSON)
│       ├── trigger (schedule/manual/dependency)
│       └── temporal_run_id
│
├── AlertConfig[]
│   ├── id, org_id, pipeline_id (nullable for global)
│   ├── type (failure/slow/zero_rows/schema_change)
│   ├── channel (email/slack/pagerduty/webhook)
│   └── config (JSON: threshold, recipients)
│
└── PipelineTemplate[]
    ├── id, name, description
    ├── source_connector_type, destination_connector_type
    ├── transform_config, tags[]
    └── is_system (boolean)
```

---

## UI/UX — Key Screens

### 1. Pipeline Builder
- Left panel: source tables (expandable with columns)
- Center canvas: visual DAG of transform nodes
- Right panel: selected node configuration
- Bottom panel: data preview table (updates live as you build)
- Top bar: pipeline name, save, run, schedule dropdown

### 2. Transform Node Config
- Each node opens a config panel
- Filter: condition builder (column, operator, value)
- Aggregate: drag columns to "group by" and "measures" zones
- Join: select second source, define join keys, choose join type
- Computed column: formula editor with autocomplete
- SQL: full SQL editor with syntax highlighting

### 3. Pipeline List
- Table: name, source → destination icons, schedule, last run status, next run
- Status indicators: green (healthy), yellow (warning), red (failing)
- Quick actions: run, pause, edit, delete
- Filter by status, source, destination

### 4. Run Detail
- Timeline: extract → transform → load with duration per step
- Row counts at each stage
- Error details (if failed) with affected rows
- Log viewer with search
- Re-run button

### 5. Connection Manager
- List of all connected sources and destinations
- Test connection button
- Schema browser: explore tables and columns
- Credential rotation

### 6. Monitoring Dashboard
- Pipeline health overview: success rate, average duration
- Failed pipelines requiring attention
- Data volume trends (rows synced over time)
- Next scheduled runs timeline
- Alert history

---

## Monetization

### Free Tier
- 3 pipelines
- Daily sync frequency
- 10,000 rows per sync
- 5 source connectors
- Google Sheets + PostgreSQL destinations

### Pro — $49/month
- 20 pipelines
- Hourly sync
- 500,000 rows per sync
- All connectors
- All destinations
- Visual transforms
- Email/Slack alerts
- 2 users

### Team — $149/month
- Unlimited pipelines
- 15-minute sync
- 5,000,000 rows per sync
- SQL transforms
- Pipeline dependencies
- Version control
- 10 users
- Priority support

### Enterprise — Custom
- Unlimited everything
- Custom connectors
- SSO
- SLA
- VPC deployment option
- Dedicated support

---

## Go-to-Market Strategy

- Position as "Fivetran for startups" — affordable, simple, includes transforms
- Content: "Sync Stripe to BigQuery in 5 minutes" (tutorial)
- SEO: "ETL tool", "data pipeline builder", "sync Stripe to BigQuery"
- Template gallery as discovery mechanism
- Free tier with enough value to hook data analysts
- Partner with dbt (complementary: DataPipe extracts + dbt transforms)
- Data community: dbt Slack, Locally Optimistic, Data Engineering Weekly

---

## Success Metrics

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Organizations | 200 | 800 |
| Active pipelines | 1,500 | 8,000 |
| Rows synced/month | 50M | 500M |
| Paying customers | 60 | 250 |
| MRR | $5,000 | $25,000 |
| Pipeline success rate | 95% | 98% |

---

## MVP Scope (v1.0)

### In Scope
- 10 source connectors (Stripe, Shopify, HubSpot, Google Ads, GA4, PostgreSQL, MySQL, Google Sheets, CSV upload, REST API)
- 3 destinations (BigQuery, PostgreSQL, Google Sheets)
- Visual pipeline builder with 5 transform types (filter, rename, computed column, aggregate, deduplicate)
- Daily/hourly scheduling
- Basic monitoring (run status, row counts)
- Email alerts on failure

### Out of Scope (v1.1+)
- 40+ additional connectors
- Join and union transforms
- SQL transform editor
- Pipeline dependencies
- Version control
- Templates
- Snowflake/Redshift destinations
- Sub-hourly scheduling

### MVP Timeline: 8-10 weeks
- Week 1-2: Connection framework, first 5 connectors (Stripe, Shopify, HubSpot, PostgreSQL, CSV)
- Week 3-4: Transform engine (DuckDB), visual builder UI
- Week 5-6: Destination loaders (BigQuery, PostgreSQL, Google Sheets)
- Week 7: Scheduling, monitoring, alerting
- Week 8-9: Remaining 5 connectors, dashboard, pipeline management
- Week 10: Billing, docs, polish, launch
