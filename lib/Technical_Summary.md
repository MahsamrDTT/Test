# Technical Summary: fact_episode.txt SQL Script

## Overview
This SQL script is a data integration and ETL (Extract, Transform, Load) process that populates the `[Staging].[fact_episode]` table by combining data from multiple source tables in a mental health data warehouse.

## Script Structure

### 1. Data Truncation
```sql
TRUNCATE TABLE [Staging].[fact_episode];
```
- Clears existing data from the target fact table before loading new data
- Ensures a clean slate for the data refresh process

### 2. Common Table Expressions (CTEs)

#### A. `episode_data` CTE (Lines 4-41)
**Purpose**: Extracts base episode data from the WRK schema
**Source**: `[WRK].[episode]`
**Key Fields**:
- Episode identifiers (episode_key, episode_id, episode_number)
- Patient identifiers (patient_id, combined_agency_patient_id)
- Temporal data (episode_start_date, episode_end_date)
- Categorical data (episode_type, mental_health_legal_status, referral)
- Demographics (indigenous_status, marital_status, sex)

#### B. `Org_table_join` CTE (Lines 47-90)
**Purpose**: Enriches episode data with organizational hierarchy information
**Join Logic**: 
- Links episodes to organizations via team_id
- Applies temporal constraints to ensure organization data is valid for the episode period
**Source**: `staging.ref_mental_health_organisational_hierarchy`

#### C. `person_join` CTE (Lines 94-139)
**Purpose**: Links episodes to patient dimension data
**Join Logic**: Uses `final_mental_health_person_group_id` as the linking key
**Source**: `staging.dim_person`

#### D. `diagnosis_join` CTE (Lines 141-189)
**Purpose**: Associates episodes with diagnosis information
**Join Logic**: Links via episode_id
**Source**: `WRK.flattened_diagnosis`

#### E. `leave_join` CTE (Lines 190-242)
**Purpose**: Incorporates leave/absence data for episodes
**Join Logic**: 
- Links via episode_number (source_number)
- Ensures leave periods fall within episode date ranges
**Source**: `[Staging].[Dim_leave_days_agg]`

### 3. Final Data Load (Lines 248-318)
**Target**: `[Staging].[fact_episode]`
**Operation**: INSERT INTO with SELECT from the final CTE
**Audit Fields**: Adds `created_datetime` and `last_updated_datetime` with `GETDATE()`

## Data Flow Architecture

```
[WRK].[episode] (Base Data)
    ↓
+ [staging].[ref_mental_health_organisational_hierarchy] (Organization Context)
    ↓
+ [staging].[dim_person] (Patient Demographics)
    ↓
+ [WRK].[flattened_diagnosis] (Medical Diagnoses)
    ↓
+ [Staging].[Dim_leave_days_agg] (Leave/Absence Data)
    ↓
[Staging].[fact_episode] (Final Fact Table)
```

## Key Technical Features

### Temporal Data Integrity
- Organization assignments are validated against date ranges
- Leave periods are constrained within episode boundaries
- Audit timestamps track data lineage

### Data Relationships
- **One-to-Many**: Episode → Diagnoses (via flattened structure)
- **One-to-One**: Episode → Organization (time-bounded)
- **One-to-One**: Episode → Person
- **One-to-Many**: Episode → Leave periods

### NULL Handling
- Explicit NULL assignments for unmapped foreign keys
- LEFT JOINs preserve episodes even when related data is missing

## Performance Considerations

### Potential Optimizations
1. **Indexing**: Ensure indexes exist on join columns:
   - `team_id` in organizational hierarchy table
   - `final_mental_health_person_group_id` in person dimension
   - `episode_id` in diagnosis table
   - `source_number` in leave dimension

2. **Statistics**: Update table statistics before running for optimal query plans

3. **Partitioning**: Consider date-based partitioning for large fact tables

### Estimated Volume
- Comment indicates ~1,055,005 rows processed from base episode data
- Final row count may be higher due to one-to-many relationships with diagnoses and leave periods

## Data Quality Checks

### Built-in Validations
- Date range validation for organization assignments
- Temporal constraints for leave periods
- Foreign key relationships maintained through structured joins

### Recommended Additional Checks
- Validate episode start/end date logic
- Check for duplicate episodes after all joins
- Monitor NULL rates in key foreign key fields

## Dependencies

### Source Systems
- **WRK Schema**: Working/staging area for raw episode and diagnosis data
- **Staging Schema**: Processed dimension tables and reference data

### Prerequisites
- All source tables must be populated and current
- Organizational hierarchy must be maintained with accurate date ranges
- Person dimension must be synchronized with source systems

## Maintenance Notes
- Script includes commented debugging queries (lines 320-321)
- Consider implementing logging for row counts at each CTE step
- Monitor for data drift in join key relationships over time

---

*This technical summary provides implementation details for database administrators and developers working with the mental health data warehouse.*