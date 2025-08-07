# Fact Episode ETL Documentation

## Overview

The `fact_episode.txt` file contains a comprehensive ETL (Extract, Transform, Load) process that populates the `[Staging].[fact_episode]` table in a mental health data warehouse. This script is designed to consolidate episode data from multiple source systems and dimension tables into a single fact table for analytical purposes.

## Purpose

This ETL process serves to:
- Extract episode data from the working (`WRK`) schema
- Enrich the data by joining with organizational, patient, diagnosis, and leave dimension tables
- Transform and standardize the data for analytical consumption
- Load the processed data into the staging fact table

## Script Structure

### 1. Data Truncation
```sql
TRUNCATE TABLE [Staging].[fact_episode];
```
**Purpose**: Clears the target table before loading new data to ensure data freshness.

### 2. Common Table Expressions (CTEs)

#### A. episode_data CTE
**Purpose**: Extracts base episode data from the source system
**Source Table**: `[WRK].[episode]`
**Key Transformations**:
- Casts `episode_number` to varchar
- Maps coded values to descriptive names:
  - `MH_legal_status_mapped_value` → `mental_health_legal_status`
  - `MH_care_referral_mapped_value` → `referral`
  - `indigenous_status_mapped_value` → `indigenous_status`
  - `marital_status_mapped_value` → `marital_status`
  - `sex_mapped_value` → `sex`

#### B. Org_table_join CTE
**Purpose**: Enriches episode data with organizational hierarchy information
**Join Logic**: 
- Links episodes to organizations via `team_id`
- Applies temporal logic using episode start date within organization's active period
**Source Table**: `staging.ref_mental_health_organisational_hierarchy`

#### C. person_join CTE
**Purpose**: Adds patient/person dimension information
**Join Logic**: Links via `final_mental_health_person_group_id`
**Source Table**: `staging.dim_person`

#### D. diagnosis_join CTE
**Purpose**: Incorporates diagnosis information
**Join Logic**: Links via `episode_id`
**Source Table**: `WRK.flattened_diagnosis`

#### E. leave_join CTE
**Purpose**: Adds leave/absence day information
**Join Logic**: 
- Links via `episode_number` to `source_number`
- Applies temporal constraint ensuring leave dates fall within episode period
**Source Table**: `[Staging].[Dim_leave_days_agg]`

### 3. Final Insert Statement
**Target Table**: `[Staging].[fact_episode]`
**Data Source**: `leave_join` CTE (final enriched dataset)

## Data Model

### Key Fields

| Field Name | Data Type | Source | Description |
|------------|-----------|---------|-------------|
| `episode_key` | Integer | WRK.episode | Unique identifier for episode |
| `episode_id` | String | WRK.episode | Business identifier for episode |
| `episode_number` | Varchar | WRK.episode | Episode number (cast from numeric) |
| `patient_key` | Integer | staging.dim_person | Foreign key to person dimension |
| `organisation_key` | Integer | staging.ref_mental_health_organisational_hierarchy | Foreign key to organization |
| `diagnosis_key` | Integer | WRK.flattened_diagnosis | Foreign key to diagnosis dimension |
| `leave_key` | Integer | Staging.Dim_leave_days_agg | Foreign key to leave days dimension |

### Episode Details

| Field Name | Description |
|------------|-------------|
| `episode_type` | Type/category of the mental health episode |
| `establishment_id` | Identifier for the healthcare establishment |
| `patient_id` | Patient identifier |
| `episode_start_date` | When the episode began |
| `episode_end_date` | When the episode ended |
| `episode_start_mode` | How the episode was initiated |
| `episode_end_mode` | How the episode was concluded |
| `episode_leave_days` | Number of leave days during episode |

### Clinical Information

| Field Name | Description |
|------------|-------------|
| `mental_health_legal_status` | Legal status under mental health legislation |
| `referral` | Referral source/type |
| `primary_diagnosis` | Primary clinical diagnosis |
| `additional_diagnosis` | Secondary diagnoses |

### Demographics

| Field Name | Description |
|------------|-------------|
| `indigenous_status` | Indigenous status classification |
| `marital_status` | Marital status |
| `sex` | Biological sex |
| `gender` | Gender identity (placeholder - not yet submitted) |
| `area_of_usual_residence_sa2` | Statistical area of residence |

### System Fields

| Field Name | Description |
|------------|-------------|
| `team_id` | Healthcare team identifier |
| `agency_mapping` | Agency mapping information |
| `combined_agency_patient_id` | Combined patient identifier across agencies |
| `final_mental_health_person_group_id` | Final person grouping identifier |
| `source_system_id` | Originating system identifier |
| `created_datetime` | Record creation timestamp |
| `last_updated_datetime` | Last update timestamp |

## Data Flow Diagram

```
[WRK.episode] 
    ↓ (episode_data CTE)
[Base Episode Data]
    ↓ (LEFT JOIN with staging.ref_mental_health_organisational_hierarchy)
[Episode + Organization Data]
    ↓ (LEFT JOIN with staging.dim_person)
[Episode + Organization + Person Data]
    ↓ (LEFT JOIN with WRK.flattened_diagnosis)
[Episode + Organization + Person + Diagnosis Data]
    ↓ (LEFT JOIN with Staging.Dim_leave_days_agg)
[Complete Enriched Dataset]
    ↓ (INSERT INTO)
[Staging.fact_episode]
```

## Dependencies

### Source Tables
- `[WRK].[episode]` - Primary episode data
- `staging.ref_mental_health_organisational_hierarchy` - Organization hierarchy
- `staging.dim_person` - Patient/person dimension
- `WRK.flattened_diagnosis` - Diagnosis information
- `[Staging].[Dim_leave_days_agg]` - Leave days aggregation

### Target Table
- `[Staging].[fact_episode]` - Final fact table

## Usage Instructions

### Prerequisites
1. Ensure all source tables are populated and up-to-date
2. Verify that dimension tables contain current reference data
3. Check that the target table schema matches the INSERT statement

### Execution Steps
1. **Backup**: Consider backing up the target table if needed
2. **Execute**: Run the entire script as a single transaction
3. **Validate**: Verify row counts and data quality after execution

### Example Execution
```sql
-- Execute the entire fact_episode.txt script
-- This will:
-- 1. Truncate the staging table
-- 2. Process all CTEs in sequence
-- 3. Insert the final enriched dataset

-- Validation queries after execution:
SELECT COUNT(*) FROM [Staging].[fact_episode];
SELECT TOP 10 * FROM [Staging].[fact_episode] ORDER BY created_datetime DESC;
```

## Performance Considerations

### Optimization Notes
- The script uses LEFT JOINs to preserve all episode records even if dimension matches are not found
- Temporal joins (date range conditions) may require proper indexing on date fields
- The final CTE approach allows for step-by-step data enrichment and easier debugging

### Recommended Indexes
- `WRK.episode`: Index on `episode_key`, `team_id`, `episode_start_date`
- `staging.ref_mental_health_organisational_hierarchy`: Index on `team_id`, `master_start_date`, `master_end_date`
- `staging.dim_person`: Index on `final_mental_health_person_group_id`
- `WRK.flattened_diagnosis`: Index on `episode_id`
- `Staging.Dim_leave_days_agg`: Index on `source_number`, `leave_start_date`

## Data Quality Considerations

### Null Handling
- Several fields are explicitly set to NULL as placeholders (e.g., `Gender`)
- LEFT JOINs ensure that missing dimension data doesn't exclude episode records

### Data Validation
- Episode dates should be validated for logical consistency
- Foreign key relationships should be verified after loading
- Duplicate episode keys should be monitored

### Known Issues
- Gender field is not yet populated (placeholder only)
- No direct linkage between establishment and episode tables (noted in comments)

## Troubleshooting

### Common Issues
1. **Missing dimension data**: Check if reference tables are current
2. **Date range mismatches**: Verify temporal join conditions
3. **Duplicate records**: May occur due to multiple diagnosis or leave records per episode

### Debug Queries
```sql
-- Check for episodes with multiple records
SELECT episode_key, COUNT(*) 
FROM leave_join 
GROUP BY episode_key 
HAVING COUNT(*) > 1;

-- Validate specific episode
SELECT * FROM leave_join WHERE episode_key = '746857';

-- Check leave days for specific episode
SELECT * FROM [Staging].[Dim_leave_days] 
WHERE source_number = '00996748';
```

## Maintenance

### Regular Tasks
1. Monitor execution performance
2. Validate data quality metrics
3. Update documentation when schema changes occur
4. Review and optimize join conditions periodically

### Change Management
- Any changes to source table schemas require corresponding updates to this ETL
- New dimension tables may require additional CTEs
- Field mapping changes should be documented and tested

---

*Last Updated: [Current Date]*
*Version: 1.0*
*Author: Data Engineering Team*