# Mental Health Episode Reporting - Methodology Document

## 1. Executive Summary

This methodology document outlines the data processing and reporting approach for mental health episode analytics. The system transforms raw episode data from multiple sources into a structured fact table (`fact_episode`) that supports comprehensive mental health service reporting and analysis.

## 2. Data Sources and Architecture

### 2.1 Primary Data Sources

**Source System**: Working (WRK) database containing raw episode data
- **Table**: `[WRK].[episode]`
- **Key Identifier**: `episode_key`
- **Patient Identifier**: `final_mental_health_person_group_id`
- **Organization Identifier**: `team_id`

### 2.2 Reference Data Sources

**Organizational Hierarchy**: `staging.ref_mental_health_organisational_hierarchy`
- Provides organizational context and team mappings
- Includes temporal validity through `master_start_date` and `master_end_date`

**Person Dimension**: `staging.dim_person`
- Contains patient demographic and identification information
- Links via `final_mental_health_person_group_id`

**Diagnosis Dimension**: `staging.dim_diagnosis`
- Provides diagnosis classification and coding
- Links episodes to standardized diagnostic categories

**Leave Dimension**: `staging.dim_leave`
- Captures leave-related information for episodes
- Supports analysis of service interruptions

## 3. Data Transformation Process

### 3.1 Episode Data Extraction

The transformation process begins with extracting core episode information from the WRK system:

**Core Episode Attributes**:
- Episode identification (`episode_id`, `episode_number`)
- Temporal information (`episode_start_date`, `episode_end_date`)
- Episode characteristics (`episode_type`, `episode_start_mode`, `episode_end_mode`)
- Service delivery metrics (`episode_leave_days`)

**Patient Demographics**:
- Indigenous status (mapped values)
- Marital status (mapped values)
- Sex and gender information
- Area of usual residence (SA2 classification)

**Clinical Information**:
- Mental health legal status
- Care referral information
- Primary and additional diagnoses (initially null, populated through joins)

### 3.2 Multi-Stage Join Process

#### Stage 1: Organization Data Integration
- **Purpose**: Link episodes to organizational hierarchy
- **Join Logic**: Team ID matching with temporal validity
- **Key Constraint**: Episode start date must fall within organization's active period
- **Result**: Episodes enriched with `organisation_key`

#### Stage 2: Patient Data Integration
- **Purpose**: Link episodes to patient master data
- **Join Logic**: Matching on `final_mental_health_person_group_id`
- **Result**: Episodes enriched with `patient_key`

#### Stage 3: Diagnosis Data Integration
- **Purpose**: Associate episodes with diagnostic information
- **Join Logic**: Links to diagnosis dimension
- **Result**: Episodes enriched with `diagnosis_key`

#### Stage 4: Leave Data Integration
- **Purpose**: Incorporate leave-related information
- **Join Logic**: Links to leave dimension
- **Result**: Episodes enriched with `leave_key`

### 3.3 Data Quality and Validation Rules

**Temporal Consistency**:
- Episode start dates must precede or equal end dates
- Organization assignments validated against active periods
- Historical data integrity maintained through timestamp tracking

**Referential Integrity**:
- All foreign keys validated against dimension tables
- Null values explicitly handled where data is unavailable
- Source system tracking maintained for audit purposes

**Business Rules**:
- Episode numbers converted to varchar for consistency
- Mapped values used for standardized categorical data
- Created and updated timestamps automatically generated

## 4. Target Data Structure

### 4.1 Fact Table Schema (`[Staging].[fact_episode]`)

**Primary Keys**:
- `episode_key`: Unique episode identifier

**Foreign Keys**:
- `organisation_key`: Links to organizational hierarchy
- `patient_key`: Links to patient dimension
- `diagnosis_key`: Links to diagnosis classification
- `leave_key`: Links to leave information

**Temporal Attributes**:
- `episode_start_date`, `episode_end_date`: Episode duration
- `created_datetime`, `last_updated_datetime`: Audit timestamps

**Categorical Dimensions**:
- `episode_type`: Type classification
- `mental_health_legal_status`: Legal status categorization
- `referral`: Referral source information
- `indigenous_status`, `marital_status`, `sex`, `gender`: Demographics

**Geographic Attributes**:
- `area_of_usual_residence_sa2`: Statistical area classification

**Operational Attributes**:
- `episode_leave_days`: Service interruption metrics
- `source_system_id`: Data lineage tracking

## 5. Data Processing Methodology

### 5.1 Extract, Transform, Load (ETL) Process

**Extract Phase**:
- Full truncate and reload of staging fact table
- Source data extracted from WRK system
- Reference data validated for currency

**Transform Phase**:
- Multi-stage join process as outlined in Section 3.2
- Data type conversions and standardization
- Business rule application and validation

**Load Phase**:
- Atomic insertion into staging fact table
- Audit trail creation with timestamps
- Data quality checks and validation

### 5.2 Data Refresh Strategy

**Frequency**: Based on source system update patterns
**Method**: Complete refresh (truncate and reload)
**Validation**: Post-load data quality checks

## 6. Reporting and Analytics Framework

### 6.1 Supported Analysis Dimensions

**Temporal Analysis**:
- Episode duration and patterns
- Seasonal variations in service delivery
- Historical trend analysis

**Geographic Analysis**:
- Service delivery by statistical area
- Population-based utilization rates
- Access and equity analysis

**Demographic Analysis**:
- Service utilization by population groups
- Indigenous health service delivery
- Gender and age-based analysis

**Clinical Analysis**:
- Diagnosis-based service patterns
- Legal status and service delivery
- Referral pathway analysis

**Organizational Analysis**:
- Team and service performance
- Resource utilization patterns
- Service delivery efficiency

### 6.2 Key Performance Indicators

**Volume Metrics**:
- Total episodes processed
- Episode counts by type and duration
- Patient unique counts

**Quality Metrics**:
- Data completeness rates
- Referential integrity compliance
- Temporal consistency validation

**Operational Metrics**:
- Average episode duration
- Leave day utilization
- Service continuity measures

## 7. Data Governance and Quality Assurance

### 7.1 Data Lineage

**Source Tracking**: All records maintain source system identification
**Audit Trail**: Creation and modification timestamps for all records
**Version Control**: Change management through staging environment

### 7.2 Quality Controls

**Validation Rules**:
- Mandatory field completeness checks
- Referential integrity validation
- Business rule compliance verification

**Exception Handling**:
- Null value management strategy
- Data anomaly identification and resolution
- Error logging and monitoring

### 7.3 Privacy and Security

**De-identification**: Patient identifiers appropriately managed
**Access Controls**: Role-based access to sensitive information
**Compliance**: Adherence to health data privacy regulations

## 8. Technical Implementation Notes

### 8.1 Performance Considerations

**Indexing Strategy**: Optimized for common query patterns
**Partitioning**: Temporal partitioning for large datasets
**Query Optimization**: Efficient join strategies and query plans

### 8.2 Scalability

**Volume Handling**: Designed for large-scale episode data
**Processing Efficiency**: Optimized ETL procedures
**Storage Management**: Appropriate data retention policies

## 9. Limitations and Assumptions

### 9.1 Data Limitations

- Diagnosis information initially null, populated through dimensional joins
- Gender information noted as placeholder for future implementation
- Establishment-episode linkage not yet implemented

### 9.2 Methodological Assumptions

- Episode data completeness varies by source system
- Temporal joins assume accurate date information
- Organizational hierarchy changes captured through effective dating

## 10. Future Enhancements

### 10.1 Planned Improvements

- Implementation of establishment-episode linkage
- Enhanced gender data capture and reporting
- Real-time data processing capabilities

### 10.2 Analytical Extensions

- Predictive modeling for service planning
- Advanced clinical outcome analysis
- Population health trend forecasting

---

**Document Version**: 1.0  
**Last Updated**: Generated from SQL analysis  
**Review Cycle**: Annual or as required by system changes  
**Approved By**: [To be completed by appropriate authority]