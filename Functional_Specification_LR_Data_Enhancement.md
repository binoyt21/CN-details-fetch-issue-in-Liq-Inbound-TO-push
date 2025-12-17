# Functional Specification
## LR Number and LR Date Enhancement for Liquid Inbound Transportation Orders

---

**Document Information**

| Field | Details |
|-------|---------|
| **Document Type** | Functional Specification |
| **Function Module** | Z_SCM_LIQUID_IB_GETDET |
| **Version** | 1.0 |
| **Date** | 2024 |
| **Author** | Functional Team |
| **Status** | Approved |

---

## 1. Executive Summary

### 1.1 Purpose
This document describes the functional requirements for enhancing the LR (Lorry Receipt) Number and LR Date fetching logic in the function module `Z_SCM_LIQUID_IB_GETDET`. The enhancement ensures that LR details are always populated when available, using a fallback mechanism to prevent blank values.

### 1.2 Scope
- **In Scope**: 
  - CN Data output structure (`ET_LIB_CN_DATA`)
  - Customer Document Data output structure (`ET_LIB_CUST_DOC_DATA`)
  - LR Number and LR Date assignment logic
  
- **Out of Scope**:
  - Changes to Order Header Data structure
  - Modifications to other CN data fields
  - Changes to existing RNM-specific logic

### 1.3 Business Justification
Currently, the system may return blank LR Number and LR Date values when OIGS table does not contain this information. This causes data quality issues and impacts downstream processes that depend on LR details. The enhancement ensures data completeness by implementing a fallback mechanism.

---

## 2. Business Requirements

### 2.1 Primary Requirements

**BR-001: LR Data Completeness**
- **Requirement**: LR Number and LR Date must not be blank when data is available in the system.
- **Priority**: High
- **Business Impact**: Prevents data quality issues and ensures downstream processes receive complete information.

**BR-002: Primary Data Source**
- **Requirement**: OIGS table must remain the primary source for LR Number and LR Date.
- **Priority**: High
- **Business Impact**: Maintains data consistency and follows existing business rules.

**BR-003: Fallback Data Source**
- **Requirement**: YTTSTX0002 table should act as a fallback source when OIGS does not have LR details.
- **Priority**: High
- **Business Impact**: Ensures maximum data availability and reduces manual intervention.

### 2.2 Business Rules

1. **Data Source Priority**:
   - Primary: OIGS table (Shipment Header)
   - Fallback: YTTSTX0002 table (Transportation Report Details)
   - Special Case: ZLRPO table (for RNM shipment types - existing logic remains unchanged)

2. **Override Condition**:
   - OIGS values override YTTSTX0002 values only when **both** LR_NO and LR_DT are populated in OIGS
   - If either field is blank in OIGS, YTTSTX0002 values are retained

3. **RNM Shipment Types**:
   - Existing RNM-specific logic using ZLRPO table takes precedence over both OIGS and YTTSTX0002
   - This ensures backward compatibility with existing RNM processes

---

## 3. Functional Design

### 3.1 Process Flow

```
┌─────────────────────────────────────────────────────────────┐
│ Start: Process Shipment for CN Data                          │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
        ┌──────────────────────────────┐
        │ Step 1: Fetch from YTTSTX0002│
        │ - Get REPORT_NO from         │
        │   YTTSTX0001 using SHNUMBER  │
        │ - Fetch LR_NO, LR_DT from    │
        │   YTTSTX0002 using REPORT_NO │
        │ - Assign as Initial Default  │
        └──────────────┬───────────────┘
                       │
                       ▼
        ┌──────────────────────────────┐
        │ Step 2: Check OIGS Table    │
        │ - Read OIGS-LR_NO            │
        │ - Read OIGS-LR_DT            │
        └──────────────┬───────────────┘
                       │
                       ▼
        ┌──────────────────────────────┐
        │ Step 3: Override Logic       │
        │ IF OIGS-LR_NO IS NOT INITIAL │
        │   AND OIGS-LR_DT IS NOT INITIAL│
        │ THEN                         │
        │   Override with OIGS values  │
        │ ENDIF                        │
        └──────────────┬───────────────┘
                       │
                       ▼
        ┌──────────────────────────────┐
        │ Step 4: RNM Check (Existing) │
        │ - If RNM shipment type,       │
        │   override with ZLRPO values  │
        └──────────────┬───────────────┘
                       │
                       ▼
        ┌──────────────────────────────┐
        │ End: Assign Final Values     │
        └──────────────────────────────┘
```

### 3.2 Data Flow

#### 3.2.1 Input Data Sources

**YTTSTX0001 (Transportation Report Header)**
- Key Field: `SHNUMBER` (Shipment Number)
- Output Field: `REPORT_NO` (Reporting Number)

**YTTSTX0002 (Transportation Report Details)**
- Key Field: `REPORT_NO` (from YTTSTX0001)
- Output Fields:
  - `LR_NO` (LR Number)
  - `LR_DT` (LR Date)

**OIGS (Shipment Header)**
- Key Field: `SHNUMBER` (Shipment Number)
- Output Fields:
  - `LR_NO` (LR Number)
  - `LR_DT` (LR Date)

#### 3.2.2 Output Data Structures

**ET_LIB_CN_DATA**
- `CN_NO` (Consignment Note Number) - Maps to LR Number
- `CN_DATE` (Consignment Note Date) - Maps to LR Date

**ET_LIB_CUST_DOC_DATA**
- `CN_NO` (Consignment Note Number) - Maps to LR Number
- `CN_DATE` (Consignment Note Date) - Maps to LR Date
- Applies to both `lw_cust_doc_data_td` and `lw_cust_doc_data` structures

### 3.3 Functional Logic Details

#### Step 1: Initial Default from YTTSTX0002

**Purpose**: Ensure LR details are available even if OIGS is incomplete.

**Process**:
1. Read YTTSTX0001 table using shipment number (`SHNUMBER`)
2. Extract `REPORT_NO` from YTTSTX0001
3. Read YTTSTX0002 table using `REPORT_NO`
4. Assign `LR_NO` and `LR_DT` from YTTSTX0002 to output structures
5. This serves as the initial default value

**Assignment**:
```abap
lw_cn_data-cn_no   = lw_yttstx0002-lr_no
lw_cn_data-cn_date = lw_yttstx0002-lr_dt
```

#### Step 2: Fetch from OIGS

**Purpose**: Retrieve LR details from primary source.

**Process**:
1. OIGS table is already fetched in the main processing loop
2. Access `OIGS-LR_NO` and `OIGS-LR_DT` fields
3. These values are available in the work area `lw_oigs`

#### Step 3: Override Logic

**Purpose**: Prioritize OIGS data when complete.

**Condition**: 
- Both `OIGS-LR_NO` AND `OIGS-LR_DT` must be populated

**Action**:
- If condition is met, override YTTSTX0002 values with OIGS values
- If condition is not met, retain YTTSTX0002 values

**Logic**:
```abap
IF lw_oigs-lr_no IS NOT INITIAL AND lw_oigs-lr_dt IS NOT INITIAL
  " Override with OIGS values
ENDIF
```

#### Step 4: RNM Special Handling (Existing Logic)

**Purpose**: Maintain existing RNM-specific behavior.

**Process**:
- If shipment type is RNM (determined by parameter table check)
- Override with values from ZLRPO table
- This takes precedence over both OIGS and YTTSTX0002

---

## 4. Affected Components

### 4.1 Function Module Sections

**Section 1: CN Data Processing**
- Location: Lines ~1785-1803
- Structure: `lw_cn_data`
- Fields: `cn_no`, `cn_date`

**Section 2: Customer Document Data Processing**
- Location: Lines ~2024-2045
- Structures: `lw_cust_doc_data_td`, `lw_cust_doc_data`
- Fields: `cn_no`, `cn_date`

### 4.2 Data Tables

| Table | Purpose | Key Fields |
|-------|---------|------------|
| YTTSTX0001 | Transportation Report Header | SHNUMBER, REPORT_NO |
| YTTSTX0002 | Transportation Report Details | REPORT_NO, LR_NO, LR_DT |
| OIGS | Shipment Header | SHNUMBER, LR_NO, LR_DT |
| ZLRPO | LR/PO Details (RNM) | SHNUMBER, LR_NO, CREATED_ON |

### 4.3 Output Structures

**ZSCE_CN_DATA_S** (CN Data Structure)
- `CN_NO`: Consignment Note Number (CHAR)
- `CN_DATE`: Consignment Note Date (DATS)

**ZSCE_CUST_DOC_DATA_S** (Customer Document Data Structure)
- `CN_NO`: Consignment Note Number (CHAR)
- `CN_DATE`: Consignment Note Date (DATS)

---

## 5. Business Scenarios

### Scenario 1: OIGS Has Complete Data
**Input**:
- OIGS-LR_NO = "LR123456"
- OIGS-LR_DT = "2024-01-15"
- YTTSTX0002-LR_NO = "LR789012"
- YTTSTX0002-LR_DT = "2024-01-14"

**Expected Output**:
- CN_NO = "LR123456" (from OIGS)
- CN_DATE = "2024-01-15" (from OIGS)

**Reason**: OIGS has both fields populated, so it overrides YTTSTX0002.

### Scenario 2: OIGS Has Incomplete Data
**Input**:
- OIGS-LR_NO = "LR123456"
- OIGS-LR_DT = "" (blank)
- YTTSTX0002-LR_NO = "LR789012"
- YTTSTX0002-LR_DT = "2024-01-14"

**Expected Output**:
- CN_NO = "LR789012" (from YTTSTX0002)
- CN_DATE = "2024-01-14" (from YTTSTX0002)

**Reason**: OIGS has only one field populated, so YTTSTX0002 values are retained.

### Scenario 3: OIGS Has No Data
**Input**:
- OIGS-LR_NO = "" (blank)
- OIGS-LR_DT = "" (blank)
- YTTSTX0002-LR_NO = "LR789012"
- YTTSTX0002-LR_DT = "2024-01-14"

**Expected Output**:
- CN_NO = "LR789012" (from YTTSTX0002)
- CN_DATE = "2024-01-14" (from YTTSTX0002)

**Reason**: OIGS has no data, so YTTSTX0002 values are used.

### Scenario 4: RNM Shipment Type
**Input**:
- Shipment Type = RNM (configured in parameter table)
- OIGS-LR_NO = "LR123456"
- OIGS-LR_DT = "2024-01-15"
- YTTSTX0002-LR_NO = "LR789012"
- YTTSTX0002-LR_DT = "2024-01-14"
- ZLRPO-LR_NO = "LR999999"
- ZLRPO-CREATED_ON = "2024-01-16"

**Expected Output**:
- CN_NO = "LR999999" (from ZLRPO)
- CN_DATE = "2024-01-16" (from ZLRPO)

**Reason**: RNM logic takes precedence over both OIGS and YTTSTX0002.

---

## 6. Acceptance Criteria

### 6.1 Functional Criteria

**AC-001**: When OIGS contains both LR_NO and LR_DT, these values must be used in the output.

**AC-002**: When OIGS does not contain both LR_NO and LR_DT, YTTSTX0002 values must be used.

**AC-003**: LR Number and LR Date must not be blank when data is available in either OIGS or YTTSTX0002.

**AC-004**: Existing RNM-specific logic must continue to work and take precedence.

**AC-005**: The enhancement must work for both CN Data and Customer Document Data structures.

### 6.2 Data Quality Criteria

**AC-006**: No data loss should occur - all available LR data should be captured.

**AC-007**: Data consistency must be maintained - same shipment should return same LR details across multiple calls.

**AC-008**: Performance should not degrade - existing query performance must be maintained.

### 6.3 Testing Criteria

**AC-009**: Unit tests must cover all business scenarios (Scenarios 1-4).

**AC-010**: Integration tests must verify end-to-end data flow.

**AC-011**: Regression tests must ensure existing functionality is not impacted.

---

## 7. Dependencies

### 7.1 Technical Dependencies
- Function Module: Z_SCM_LIQUID_IB_GETDET must be available
- Tables: YTTSTX0001, YTTSTX0002, OIGS must be accessible
- Data Structures: ZSCE_CN_DATA_S, ZSCE_CUST_DOC_DATA_S must exist

### 7.2 Data Dependencies
- YTTSTX0001 and YTTSTX0002 must be populated for shipments
- OIGS table must be maintained with shipment data
- Relationship between SHNUMBER and REPORT_NO must be valid

### 7.3 Process Dependencies
- Transportation reports must be created and maintained
- Shipment data must be properly maintained in OIGS
- Existing RNM processes must continue to function

---

## 8. Assumptions and Constraints

### 8.1 Assumptions
1. YTTSTX0002 table contains valid LR data for shipments
2. Relationship between YTTSTX0001 and YTTSTX0002 is maintained correctly
3. OIGS table is the primary source of truth for shipment data
4. Existing RNM logic will continue to be used for RNM shipment types

### 8.2 Constraints
1. Cannot modify existing RNM-specific logic
2. Must maintain backward compatibility
3. Cannot change function module interface
4. Must follow existing coding standards

---

## 9. Risks and Mitigation

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Data inconsistency between OIGS and YTTSTX0002 | High | Medium | Business rule: OIGS takes precedence when both fields are populated |
| Performance degradation due to additional table reads | Medium | Low | Tables are already fetched in main loop, only read operations added |
| RNM logic conflicts | High | Low | RNM logic takes precedence, tested separately |
| Missing data in both sources | Medium | Low | Business process ensures at least one source has data |

---

## 10. Approval

| Role | Name | Signature | Date |
|------|------|-----------|------|
| Functional Lead | | | |
| Business Owner | | | |
| Technical Lead | | | |

---

**Document End**

