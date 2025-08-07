# DT_RA_Project_Monstera_2024_Coal_LSL - SQL Analysis Documentation

## Overview

This document provides comprehensive documentation for the SQL analysis script used in the DT_RA_Project_Monstera_2024_Coal_LSL project. The script performs statistical analysis and data processing on employee compensation rates from various source systems (1SAP, GSAP, APOLLO) for coal operations.

## Database Schema

### Primary Tables

#### REF_COAL_Cashout_Rate
**Purpose**: Contains detailed cashout rate information for employees
**Records**: 4,730 entries covering 3,739 distinct employees

**Key Columns**:
- `EmployeeID`: Unique identifier for employees
- `effectiveRate`: The effective compensation rate
- `WTRRate`: Work Time Recording rate
- `compensationAmount`: Compensation amount value
- `Amount`: Alternative amount field
- `FlagPurged`: Indicates if record has been purged

**Usage Example**:
```sql
-- Get all cashout rates
SELECT * FROM REF_COAL_Cashout_Rate;

-- Get distinct employee count
SELECT DISTINCT(EmployeeID) FROM REF_COAL_Cashout_Rate; -- Returns 3739 employees
```

#### REF_COAL_RateTable
**Purpose**: Contains maximum rates and dates for each employee
**Records**: 3,739 distinct employees with their maximum rates

**Key Columns**:
- `EmployeeID`: Unique identifier for employees
- `IT0416maxrate`: Maximum IT0416 rate for the employee
- `WTRmaxrate`: Maximum WTR rate for the employee
- `IT0416maxdate`: Date of the maximum IT0416 rate

**Usage Example**:
```sql
-- Get all rate table entries
SELECT * FROM REF_COAL_RateTable;

-- Get distinct employee count
SELECT DISTINCT(EmployeeID) FROM REF_COAL_RateTable;
```

#### Source System Tables
- `STG_COAL_joinedIT0416_1SAP`: 1SAP system employee data
- `STG_COAL_joinedIT0416_GSAP`: GSAP system employee data  
- `STG_COAL_joinedIT0416_APOLLO`: APOLLO system employee data
- `WTR_request_20250721`: WTR (Work Time Recording) request data

## Core Functions and Queries

### 1. Statistical Analysis Functions

#### Rate Statistics Calculation
**Purpose**: Calculate comprehensive statistics for different rate types

**Function Pattern**:
```sql
WITH SortedRates_[TableName] AS (
    SELECT [RateColumn],
           ROW_NUMBER() OVER (ORDER BY [RateColumn]) as rn,
           COUNT(*) OVER () as cnt
    FROM [TableName]
)
SELECT 
    MIN([RateColumn]) AS rate_min,
    MAX([RateColumn]) AS rate_max,
    AVG([RateColumn]) AS rate_mean,
    (SELECT AVG([RateColumn])
     FROM SortedRates_[TableName]
     WHERE rn = (cnt + 1) / 2 OR (cnt % 2 = 0 AND rn = cnt / 2 + 1)) AS rate_med,
    VAR([RateColumn]) AS rate_var,
    STDEV([RateColumn]) AS rate_stdev,
    MAX([RateColumn]) - MIN([RateColumn]) AS rate_range
FROM SortedRates_[TableName]
```

**Supported Rate Types**:
- `effectiveRate` from REF_COAL_Cashout_Rate
- `WTRRate` from REF_COAL_Cashout_Rate  
- `IT0416maxrate` from REF_COAL_RateTable
- `WTRmaxrate` from REF_COAL_RateTable
- `FinalRate` from calculated final rates

**Example Usage**:
```sql
-- Get statistics for effective rates
WITH SortedRates_Cashout_Rate AS (
    SELECT effectiveRate,
           ROW_NUMBER() OVER (ORDER BY effectiveRate) as rn,
           COUNT(*) OVER () as cnt
    FROM REF_COAL_Cashout_Rate
)
SELECT 
    MIN(effectiveRate) AS rate_min,
    MAX(effectiveRate) AS rate_max,
    AVG(effectiveRate) AS rate_mean,
    (SELECT AVG(effectiveRate)
     FROM SortedRates_Cashout_Rate
     WHERE rn = (cnt + 1) / 2 OR (cnt % 2 = 0 AND rn = cnt / 2 + 1)) AS rate_med,
    VAR(effectiveRate) AS rate_var,
    STDEV(effectiveRate) AS rate_stdev,
    MAX(effectiveRate) - MIN(effectiveRate) AS rate_range
FROM SortedRates_Cashout_Rate;
```

### 2. Rate Analysis Functions

#### Employee Rate Count Analysis
**Purpose**: Analyze the number of different rates per employee

**Functions**:

##### Count Distinct Effective Rates per Employee
```sql
SELECT EmployeeID, COUNT(DISTINCT effectiveRate) as Number_of_Rates
FROM REF_COAL_Cashout_Rate
GROUP BY EmployeeID;
```

##### Count Distinct WTR Rates per Employee
```sql
SELECT EmployeeID, COUNT(DISTINCT WTRRate) as Number_of_WTR_Rates
FROM REF_COAL_Cashout_Rate
GROUP BY EmployeeID;
```

#### Zero vs Non-Zero Analysis
**Purpose**: Analyze compensation amounts that are zero vs non-zero

**Functions**:

##### Count Zero Compensation Amounts
```sql
SELECT EmployeeID, COUNT(*) as Number_of_zeroes
FROM REF_COAL_Cashout_Rate
WHERE compensationAmount = 0
GROUP BY EmployeeID;
```

##### Count Zero WTR Amounts
```sql
SELECT EmployeeID, COUNT(*) as Number_of_zeroes
FROM REF_COAL_Cashout_Rate
WHERE Amount = 0 OR Amount IS NULL
GROUP BY EmployeeID;
```

##### Count Distinct Non-Zero Compensation Amounts
```sql
SELECT EmployeeID, COUNT(DISTINCT compensationAmount) as Distinct_non_zeroes
FROM REF_COAL_Cashout_Rate
WHERE compensationAmount > 0
GROUP BY EmployeeID;
```

#### Comprehensive Employee Analysis
**Purpose**: Combine all rate analyses for each employee

```sql
SELECT 
    a.EmployeeID as EmployeeID, 
    a.Number_of_Rates as Number_of_Effective_Rates, 
    b.Number_of_WTR_Rates as Number_of_WTR_Rates, 
    COALESCE(c.Number_of_zeroes, 0) as Number_of_zeroes, 
    COALESCE(d.Number_of_zeroes, 0) as Number_of_zeroes_WTR, 
    COALESCE(e.Distinct_non_zeroes, 0) as Distinct_non_zeroes, 
    COALESCE(f.Distinct_non_zeroes, 0) as Distinct_non_zeroes_WTR
FROM #rates_Cashout_Rate a
FULL OUTER JOIN #rates_Cashout_WTR_Rate b ON a.EmployeeID = b.EmployeeID
FULL OUTER JOIN #zeroes_Cashout_Rate c ON a.EmployeeID = c.EmployeeID
FULL OUTER JOIN #zeroes_Cashout_Rate_WTR d ON a.EmployeeID = d.EmployeeID
FULL OUTER JOIN #non_zeroes_Cashout_Rate e ON a.EmployeeID = e.EmployeeID
FULL OUTER JOIN #non_zeroes_Cashout_Rate_WTR f ON a.EmployeeID = f.EmployeeID;
```

### 3. Rate Bucketing Functions

#### Rate Bucketing (Range-based)
**Purpose**: Categorize rates into predefined ranges for analysis

**Bucket Ranges**:
- `0`: Exactly zero
- `0-25`: 0 < rate ≤ 25
- `25-50`: 25 < rate ≤ 50
- `50-75`: 50 < rate ≤ 75
- `75-100`: 75 < rate ≤ 100
- `100-125`: 100 < rate ≤ 125
- `125-150`: 125 < rate ≤ 150
- `150-175`: 150 < rate ≤ 175
- `175-200`: 175 < rate ≤ 200
- `200-225`: 200 < rate ≤ 225
- `>225`: rate > 225

**Function Template**:
```sql
SELECT EmployeeID, [RateColumn],
    CASE WHEN [RateColumn] IS NULL THEN ''
         WHEN [RateColumn] = 0 THEN '0'
         WHEN [RateColumn] > 0 AND [RateColumn] <= 25 THEN '0-25'
         WHEN [RateColumn] > 25 AND [RateColumn] <= 50 THEN '25-50'
         WHEN [RateColumn] > 50 AND [RateColumn] <= 75 THEN '50-75'
         WHEN [RateColumn] > 75 AND [RateColumn] <= 100 THEN '75-100'
         WHEN [RateColumn] > 100 AND [RateColumn] <= 125 THEN '100-125'
         WHEN [RateColumn] > 125 AND [RateColumn] <= 150 THEN '125-150'
         WHEN [RateColumn] > 150 AND [RateColumn] <= 175 THEN '150-175'
         WHEN [RateColumn] > 175 AND [RateColumn] <= 200 THEN '175-200'
         WHEN [RateColumn] > 200 AND [RateColumn] <= 225 THEN '200-225'
         ELSE '>225'
         END AS Bucket,
    CASE WHEN [RateColumn] IS NULL THEN 'NULL'
         WHEN [RateColumn] = 0 THEN 'Zero'
         WHEN [RateColumn] > 0 AND [RateColumn] <= 25 THEN 'Low'
         WHEN [RateColumn] > 200 THEN 'High'
         ELSE 'Medium'
         END AS LowHighFlag
FROM [TableName]
ORDER BY EmployeeID;
```

#### Rate Bucketing (Numeric Scale)
**Purpose**: Assign numeric values to rate ranges for analysis

**Numeric Scale**:
- `-0.5`: Zero rates
- `1`: 0-25 range
- `2`: 25-50 range
- `3`: 50-75 range
- `4`: 75-100 range
- `5`: 100-125 range
- `6`: 125-150 range
- `7`: 150-175 range
- `8`: 175-200 range
- `9`: 200-225 range
- `10`: >225 range

**Function Template**:
```sql
SELECT EmployeeID, [RateColumn],
    CASE WHEN [RateColumn] IS NULL THEN ''
         WHEN [RateColumn] = 0 THEN '-0.5'
         WHEN [RateColumn] > 0 AND [RateColumn] <= 25 THEN '1'
         WHEN [RateColumn] > 25 AND [RateColumn] <= 50 THEN '2'
         WHEN [RateColumn] > 50 AND [RateColumn] <= 75 THEN '3'
         WHEN [RateColumn] > 75 AND [RateColumn] <= 100 THEN '4'
         WHEN [RateColumn] > 100 AND [RateColumn] <= 125 THEN '5'
         WHEN [RateColumn] > 125 AND [RateColumn] <= 150 THEN '6'
         WHEN [RateColumn] > 150 AND [RateColumn] <= 175 THEN '7'
         WHEN [RateColumn] > 175 AND [RateColumn] <= 200 THEN '8'
         WHEN [RateColumn] > 200 AND [RateColumn] <= 225 THEN '9'
         ELSE '10'
         END AS Bucket
FROM [TableName]
ORDER BY EmployeeID;
```

### 4. Source System Integration

#### Employee Source System Mapping
**Purpose**: Identify which source system each employee belongs to

```sql
SELECT employeeid, '1SAP' as Source_System 
FROM STG_COAL_joinedIT0416_1SAP
UNION
SELECT employeeid, 'GSAP' as Source_System 
FROM STG_COAL_joinedIT0416_GSAP
UNION
SELECT employeeid, 'APOLLO' as Source_System 
FROM STG_COAL_joinedIT0416_APOLLO
WHERE employeeid IN (SELECT DISTINCT(EmployeeID) FROM REF_COAL_Cashout_Rate);
```

### 5. Final Rate Calculation

#### Final Rate Logic
**Purpose**: Calculate the final rate using the maximum of IT0416 rate and WTR max rate

**Function**: `dbo.largerof(value1, value2)`
**Logic**: `MAX(COALESCE(IT0416maxrate,0), COALESCE(WTRmaxrate,0))`

```sql
SELECT employeeID, 
       It0416maxdate,
       dbo.largerof(COALESCE(IT0416maxrate,0),COALESCE(WTRmaxrate,0)) as FinalRate 
FROM REF_COAL_RateTable;
```

## Usage Examples

### Basic Data Exploration

```sql
-- Get basic counts
SELECT COUNT(*) FROM REF_COAL_Cashout_Rate; -- 4730 records
SELECT COUNT(DISTINCT EmployeeID) FROM REF_COAL_Cashout_Rate; -- 3739 employees

-- Find minimum non-zero rates
SELECT MIN(effectiveRate) FROM REF_COAL_Cashout_Rate WHERE effectiveRate <> 0; -- 1.38468
SELECT MIN(WTRRate) FROM REF_COAL_Cashout_Rate WHERE WTRRate <> 0;
SELECT MIN(IT0416maxrate) FROM REF_COAL_RateTable WHERE IT0416maxrate <> 0;
```

### Employee-Specific Analysis

```sql
-- Analyze specific employees
SELECT * FROM REF_COAL_RateTable
WHERE EmployeeID IN (00035211, 20002924, 20002981, 20007524, 20008954);

-- Get cashout details for specific employees
SELECT * FROM REF_COAL_Cashout_Rate
WHERE EmployeeID IN (00035211, 20002924, 20002981, 20007524, 20008954)
ORDER BY EmployeeID, FlagPurged;
```

### WTR Scope Analysis

```sql
-- Check WTR request scope
SELECT Employeeid, SourceSystemn 
FROM WTR_request_20250721
ORDER BY Employeeid;
```

## Temporary Tables Used

The script creates several temporary tables for intermediate calculations:

- `#rates_Cashout_Rate`: Employee effective rate counts
- `#rates_Cashout_WTR_Rate`: Employee WTR rate counts  
- `#zeroes_Cashout_Rate`: Zero compensation amount counts
- `#zeroes_Cashout_Rate_WTR`: Zero WTR amount counts
- `#non_zeroes_Cashout_Rate`: Non-zero compensation distinct counts
- `#non_zeroes_Cashout_Rate_WTR`: Non-zero WTR distinct counts
- `#Final_Rates`: Final calculated rates per employee

## Key Insights and Business Logic

1. **Rate Priority**: Final rates are calculated using `MAX(IT0416maxrate, WTRmaxrate)`
2. **Data Quality**: The analysis includes comprehensive zero vs non-zero amount tracking
3. **Multi-System Integration**: Data comes from three source systems (1SAP, GSAP, APOLLO)
4. **Rate Categorization**: Rates are bucketed into Low (0-25), Medium (25-200), and High (>200) categories
5. **Employee Coverage**: 3,739 distinct employees across all systems

## Performance Considerations

- Use appropriate indexes on `EmployeeID` columns for optimal performance
- Consider partitioning large tables by `EmployeeID` ranges if performance issues arise
- The statistical calculations use window functions which may require sufficient memory allocation
- Temporary tables are dropped and recreated to ensure clean runs

## Error Handling

- `COALESCE()` functions handle NULL values in rate calculations
- `FULL OUTER JOIN` operations ensure no employee data is lost during analysis
- Rate bucketing includes explicit NULL handling cases

## Maintenance Notes

- Update date references (currently 20250721) when processing new WTR request data
- Verify source system table names match current naming conventions
- Review rate bucket ranges periodically based on data distribution changes
- Monitor temporary table cleanup to prevent storage issues