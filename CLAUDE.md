# CLAUDE.md - AI Assistant Guide for FL Legislation ETL

**Last Updated**: 2026-01-10
**Project**: Florida Legislative Voting Database ETL Pipeline
**Primary Language**: R
**Database**: PostgreSQL

---

## Table of Contents
- [Project Overview](#project-overview)
- [Repository Structure](#repository-structure)
- [Database Architecture](#database-architecture)
- [Naming Conventions](#naming-conventions)
- [ETL Pipeline Workflow](#etl-pipeline-workflow)
- [Coding Patterns & Conventions](#coding-patterns--conventions)
- [Development Workflows](#development-workflows)
- [Key Scripts Reference](#key-scripts-reference)
- [Common Tasks](#common-tasks)
- [Data Sources & Integration](#data-sources--integration)
- [Quality Assurance](#quality-assurance)

---

## Project Overview

### Purpose
This repository contains the ETL (Extract, Transform, Load) pipeline for the Jacksonville Tributary's **Florida Legislative Voting Dashboard**. The project analyzes roll call voting patterns of Florida state legislators, focusing on:
- **Party loyalty**: Legislators' tendency to vote with/against their party (0-1 scale, 1 = most loyal)
- **Partisan lean**: Legislative electorate's partisanship (percentage point difference between D vs R voting)

### Live Application
- Dashboard: https://andrewpantazi.com/interactives/fl-legislative-compass.html
- App Repository: https://github.com/reliablerascal/fl-legislation-app-postgres

### Key Terminology
- **Party Line**: Voting with same party's majority, regardless of opposing party
- **Cross Party**: Voting against same party's majority, with opposing party (also "Maverick")
- **Against Both Parties**: Voting against both party majorities (also "Independent")

---

## Repository Structure

```
fl-legislation-etl/
├── data-app/           # CSV exports for web apps and visualizations
├── data-raw/           # Raw JSON data from LegiScan API (not in repo)
├── docs/               # Documentation, data dictionaries, diagrams
│   ├── db_architecture.md
│   ├── etl.md
│   ├── app_dev_guide.md
│   └── dev_workplan.md
├── notebooks/          # Jupyter notebooks for API exploration
├── qa/                 # Quality assurance logs and anomaly tables
│   └── qa_checks.log
├── scripts/            # All ETL scripts (see workflow below)
└── viz/                # Screenshots and visualizations
```

---

## Database Architecture

### Three-Layer Design
The PostgreSQL database (`fl_leg_votes`) uses a three-layer architecture:

| Layer | Schema | Purpose |
|-------|--------|---------|
| **Raw** | `raw_legiscan`, `raw_daves`, `raw_user_entry` | Unmodified data from sources |
| **Processed** | `proc` | Cleaned, organized, calculated fields |
| **Application** | `app` | Data prepared for specific web apps |

### Layer Principles
1. **Raw Layer**: Preserve source data integrity, no modifications
2. **Processed Layer**: All cleaning, calculations, and alignments happen here
3. **Application Layer**: Only access processed layer, prepare views for specific apps

---

## Naming Conventions

### Table/DataFrame Prefixes

| Prefix | Layer | Purpose | Example |
|--------|-------|---------|---------|
| `t_` | raw | **T**ables of raw data in original format | `t_bills`, `t_roll_calls` |
| `user_` | raw | **User**-entered data | `user_bill_categories` |
| `hist_` | proc | **Hist**orical processed data with time dimension | `hist_leg_sessions` |
| `jct_` | proc | **J**unction table for many-to-many relationships | `jct_bill_categories` |
| `p_` | proc | **P**rocessed data (current/most recent) | `p_legislators`, `p_bills` |
| `calc_` | proc/app | Intermediate **calc**ulations (not stored in DB) | `calc_elections_weighted` |
| `qry_` | app | **Query** foundational views supporting apps | `qry_legislators_incumbent` |
| `qa_` | app | **Q**uality **A**ssurance review tables | `qa_loyalty_ranks` |
| `app_` | app | **App**lication-ready data for specific apps | `app01_vote_patterns` |

### Field Naming Patterns
- `bill_*`: Bill-related fields (e.g., `bill_id`, `bill_number`, `bill_title`)
- `roll_call_*`: Roll call fields (e.g., `roll_call_id`, `roll_call_date`)
- `legislator_*` or `leg_*`: Legislator fields (e.g., `legislator_name`, `leg_party_loyalty`)
- `n_*`: Count fields (e.g., `n_total`, `n_present`)
- `pct_*`: Percentage fields (e.g., `pct_of_total`)
- `is_*`: Boolean fields (e.g., `is_incumbent_primaried`)
- `rank_*`: Ranking fields (e.g., `rank_partisan_leg_D`)

---

## ETL Pipeline Workflow

### Running the Pipeline

**Prerequisites:**
1. Docker container running PostgreSQL
2. Database password
3. LegiScan API key (for data refresh)

**Start Database:**
```bash
docker start my_postgres
docker exec -it my_postgres bash
psql -U postgres -d fl_leg_votes
```

**Run ETL:**
Execute `scripts/etl_main.R` in RStudio, which orchestrates the following sequence:

### Script Execution Order

| # | Script | Description |
|---|--------|-------------|
| 0 | `functions_database.R` | DB connection, write, verify functions |
| 1 | `01_request_api_legiscan.R` | Request data from LegiScan API (optional) |
| 2a | `02a_raw_parse_legiscan.R` | Parse LegiScan JSON into data frames |
| 2b | `02b_raw_read_csvs.R` | Read CSV files (user data, Dave's Redistricting) |
| 2z | `02z_raw_load.R` | Load raw layer into PostgreSQL |
| 3a | `03a_process.R` | Transform, clean, calculate processed layer |
| 3z | `03z_process_load.R` | Load processed layer into PostgreSQL |
| 4a | `04a_app_settings.R` | Create queries based on app settings |
| 4b | `04b_app_prep.R` | Prepare app-specific datasets |
| QA | `qa_checks.R` | Quality assurance checks → `qa/qa_checks.log` |
| 4z | `04z_app_load.R` | Load app layer to DB, export CSVs to `data-app/` |

### Data Flow Diagram
See `docs/etl-schematic.png` for visual representation.

---

## Coding Patterns & Conventions

### R Code Style

**1. Library Loading (in `etl_main.R`):**
```r
library(tidyverse)   # Data manipulation
library(jsonlite)    # JSON parsing
library(DBI)         # Database interface
library(RPostgres)   # PostgreSQL connection
library(googlesheets4) # User-entered data from Google Sheets
```

**2. Working Directory:**
```r
# Always set working directory to script location
setwd(script_dir <- dirname(rstudioapi::getActiveDocumentContext()$path))
```

**3. Data Frame Transformations:**
```r
# Pattern: Use descriptive intermediate names, prefer left_join over merge
p_roll_calls <- t_roll_calls %>%
  left_join(p_bills %>% select(bill_id, bill_title), by = "bill_id") %>%
  rename(
    roll_call_date = date,
    roll_call_desc = desc
  ) %>%
  mutate(
    pct_of_total = yea/n_total,
    final_vote = ifelse(grepl("third", roll_call_desc, ignore.case = TRUE), "Y", "N")
  ) %>%
  select(-chamber_id)
```

**4. Database Operations:**
```r
# Use custom functions from functions_database.R
con <- attempt_connection()  # Prompts for password
write_tables_in_list(con, "proc", list_tables, primary_keys)
verify_table(con, "proc", "p_legislators")
```

**5. Primary Keys:**
Always define and enforce primary keys in processed/app layers:
```r
primary_keys <- list(
  p_bills = c("bill_id"),
  p_legislator_votes = c("person_id", "roll_call_id"),
  hist_leg_sessions = c("person_id", "session_year")
)
```

**6. Filtering Patterns:**
```r
# Chain filters logically, use meaningful comments
app01_vote_patterns <- qry_leg_votes %>%
  filter(
    !is.na(party) & party != "" &           # Valid party affiliation
    !grepl("2010", session, ignore.case = TRUE) &  # Exclude old sessions
    (vote_text == "Yea" | vote_text == "Nay") &    # Only present votes
    !is.na(partisan_vote_type)              # Valid partisan classification
  )
```

**7. QA Logging:**
```r
# Redirect output to log file
qa_log <- "../qa/qa_checks.log"
fileConn <- file(qa_log, "w")
close(fileConn)
sink(qa_log, split = TRUE)
source("qa_checks.R")
sink()
```

---

## Development Workflows

### Making Changes to the Pipeline

**1. Modifying Calculations:**
- Changes to metrics (party loyalty, partisan lean) belong in `03a_process.R`
- Update corresponding comments explaining the calculation
- Review impact on downstream apps (`04b_app_prep.R`)

**2. Adding New Data Sources:**
- Add parsing logic to `02a_raw_parse_legiscan.R` or `02b_raw_read_csvs.R`
- Add raw tables to `02z_raw_load.R`
- Create processed versions in `03a_process.R`
- Update `03z_process_load.R` to load new tables

**3. Creating New App Datasets:**
- Configure settings in `04a_app_settings.R`
- Build app dataset in `04b_app_prep.R`
- Add to export list in `04z_app_load.R`
- Document in `docs/app_dev_guide.md`

**4. Updating QA Checks:**
- Add checks to `qa_checks.R`
- Review output in `qa/qa_checks.log` after pipeline run
- Create `qa_*` tables for persistent anomaly tracking

### Git Workflow

**Commit Message Style (from git log):**
- Use imperative, lowercase style
- Focus on "what" changed, be specific
- Examples:
  - `refactored election weight calculations and created new weights`
  - `update partisan_votes to 6 types, add loyalty metric "partisan_cross"`
  - `improve comments on 04b app prep`

**Branch Strategy:**
- Development branches: `claude/claude-md-mk7vdwrolra2amow-KKUJm` (feature branches)
- Main branch for production-ready code
- Push to feature branches, create PRs for review

---

## Key Scripts Reference

### Core ETL Scripts

**`etl_main.R`** - Orchestrator
- Loads all libraries
- Sources all scripts in order
- Sets working directory
- Manages QA logging

**`functions_database.R`** - Database Utilities
- `attempt_connection()`: Connect to PostgreSQL with password prompt
- `write_table()`: Write data frames with progress bar
- `write_tables_in_list()`: Batch write multiple tables
- `create_pk()`: Add primary key constraints
- `verify_table()`: Display record counts

**`03a_process.R`** - Core Transformations
- Constructs all `p_*` and `hist_*` tables
- Calculates party loyalty metrics
- Joins legislator sessions with vote data
- Computes district demographics

**`04a_app_settings.R`** - App Configuration
- Defines app settings (demographic source, partisan lean metrics)
- Creates foundational `qry_*` views
- Settings are configurable without breaking pipeline

**`04b_app_prep.R`** - App Data Preparation
- Builds `app01_vote_patterns` for Voting Patterns dashboard
- Builds `app03_district_context` for District Context app
- Creates visualization datasets (`viz_*`)

---

## Common Tasks

### 1. Add a New Calculated Field to Processed Layer

**Location:** `scripts/03a_process.R`

```r
# Example: Add a new metric to p_legislators
p_legislators <- p_legislators %>%
  mutate(
    new_metric = calculation_here,  # Add your calculation
    # ... existing code ...
  )
```

**Don't forget:**
- Add to `03z_process_load.R` if new table
- Update QA checks in `qa_checks.R`
- Document in `docs/db_architecture.md`

### 2. Export a New Dataset for Apps

**Location:** `scripts/04z_app_load.R`

```r
# Add to list of tables to export
app_tables <- c(
  "qry_bills",
  "qry_districts",
  "app01_vote_patterns",
  "app03_district_context",
  "your_new_dataset"  # Add here
)
```

### 3. Update App Settings

**Location:** `scripts/04a_app_settings.R`

```r
# Configure at top of file
setting_demographics_source <- "CVAP"  # or "ACS"
setting_partisan_lean <- "composite_2016_2022"  # Election weighting
setting_party_loyalty <- "for_against"  # Loyalty metric
```

### 4. Add a QA Check

**Location:** `scripts/qa_checks.R`

```r
# Pattern: Print clear headers, show counts, list anomalies
cat("\n=== NEW QA CHECK ===\n")
cat("Description of what you're checking\n\n")

anomalies <- data_frame %>%
  filter(condition_for_anomaly) %>%
  select(relevant_fields)

cat(nrow(anomalies), "anomalies found\n")
print(anomalies)
```

### 5. Refresh Data from LegiScan

**Location:** `scripts/etl_main.R`

```r
# Uncomment this line (WARNING: API limits apply)
source("01_request_api_legiscan.R")
```

---

## Data Sources & Integration

### External Data Sources

**1. LegiScan API**
- Bills, legislators, roll calls, individual votes
- JSON format, parsed in `02a_raw_parse_legiscan.R`
- API Manual: https://legiscan.com/misc/LegiScan_API_User_Manual.pdf

**2. Dave's Redistricting**
- District demographics (ACS 2020, CVAP 2022)
- Partisan preference (governor + presidential elections 2016-2022)
- CSV format: `t_dave_districts_house`, `t_dave_districts_senate`

**3. User-Entered Data (Google Sheets)**
- `user_bill_categories`: Bill categorization (prototype)
- `user_incumbents_challenged`: Primary challenger tracking
- Read via `googlesheets4` package

### Data Relationships

**Primary Keys:**
- `bill_id`: Unique bill identifier
- `roll_call_id`: Unique roll call identifier
- `people_id`: Unique legislator identifier
- `(people_id, roll_call_id)`: Unique vote
- `(chamber, district_number)`: Unique district

---

## Quality Assurance

### QA Process

**Automated Checks (`qa_checks.R`):**
1. Record count reconciliation between layers
2. Missing data identification
3. Duplicate primary key detection
4. Anomaly flagging (e.g., unexpected vote types)
5. Metric range validation

**Output:**
- Console output (during run)
- `qa/qa_checks.log` (permanent record)
- `qa_*.csv` tables (exported anomalies)

### Data Integrity Principles

1. **Primary Keys Must Be Unique**: ETL will error if duplicates found in processed/app layers
2. **No Null Keys**: All primary key fields must be non-null
3. **Preserve Raw Data**: Never modify `t_*` tables after initial load
4. **Document Anomalies**: Don't hide data quality issues, track them in QA
5. **Validate Calculations**: Compare metrics across different aggregations

---

## Key Principles for AI Assistants

### When Making Changes

**DO:**
- Read existing code patterns before implementing changes
- Follow established naming conventions strictly
- Update documentation when modifying data structures
- Add QA checks for new calculations
- Test changes with small datasets first
- Maintain the three-layer architecture
- Comment complex calculations explaining "why"
- Use descriptive variable names matching the data dictionary

**DON'T:**
- Mix concerns between layers (e.g., app logic in processed layer)
- Break existing primary key constraints
- Modify raw layer data
- Skip QA validation after changes
- Use generic names like `df`, `temp`, `data` for final data frames
- Hardcode values that should be settings
- Delete historical tracking (use `termination_date` patterns instead)

### Understanding Context

**Before suggesting changes:**
1. Identify which layer(s) are affected (raw/processed/app)
2. Check if changes impact downstream dependencies
3. Verify primary key implications
4. Review naming convention compliance
5. Consider QA check requirements

**When debugging:**
1. Check `qa/qa_checks.log` for data quality issues
2. Verify record counts between layers
3. Review recent git commits for context
4. Examine related tables via joins
5. Test queries against PostgreSQL directly if needed

### Code Review Checklist

- [ ] Follows naming conventions (correct prefix)
- [ ] Primary keys defined and enforced
- [ ] QA checks added for new metrics
- [ ] Documentation updated (README, data dictionaries)
- [ ] No hardcoded values (use settings)
- [ ] Comments explain complex logic
- [ ] Changes tested with full pipeline run
- [ ] CSV exports updated in `04z_app_load.R`
- [ ] Git commit message follows style

---

## Additional Resources

### Documentation
- Database Architecture: `docs/db_architecture.md`
- ETL Process: `docs/etl.md`
- App Development: `docs/app_dev_guide.md`
- Development Workplan: `docs/dev_workplan.md`

### External Links
- LegiScan Database ERD: https://api.legiscan.com/dl/Database_ERD.png
- Live Dashboard: https://andrewpantazi.com/interactives/fl-legislative-compass.html
- App Repository: https://github.com/reliablerascal/fl-legislation-app-postgres
- Original Project: https://github.com/apantazi/legislator_dashboard

### Data Dictionaries
- Located in `docs/data_dictionaries/`
- CSV format for easy reference
- Updated as schema evolves

---

## Version History

- **2026-01-10**: Initial CLAUDE.md created with comprehensive codebase analysis
- Most recent commit: `1a4cd63` - "refactored election weight calculations and created new weights"

---

*This document is maintained for AI assistants (like Claude) to understand the codebase structure, conventions, and development workflows. Keep it updated as the project evolves.*
