# Technical Specification
## LR Number and LR Date Enhancement for Liquid Inbound Transportation Orders

---

**Document Information**

| Field | Details |
|-------|---------|
| **Document Type** | Technical Specification |
| **Function Module** | Z_SCM_LIQUID_IB_GETDET |
| **Version** | 1.0 |
| **Date** | 2024 |
| **Author** | Technical Team |
| **Status** | Approved |
| **Related FS** | Functional_Specification_LR_Data_Enhancement.md |

---

## 1. Overview

### 1.1 Purpose
This document provides technical design and implementation details for enhancing the LR Number and LR Date fetching logic in function module `Z_SCM_LIQUID_IB_GETDET`. The enhancement implements a fallback mechanism to ensure LR details are always populated when available.

### 1.2 Scope
- **Technical Changes**: 
  - Modification of LR data assignment logic in CN Data section
  - Modification of LR data assignment logic in Customer Document Data section
  - No changes to function module interface
  - No changes to data structures

### 1.3 Technical Objectives
1. Implement fallback logic using YTTSTX0002 table
2. Maintain OIGS as primary data source
3. Ensure backward compatibility
4. Maintain code performance
5. Follow ABAP coding standards

---

## 2. Technical Architecture

### 2.1 Current Architecture

```
┌─────────────────────────────────────────────────────────┐
│ Function Module: Z_SCM_LIQUID_IB_GETDET                 │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌─────────────────────────────────────┐                │
│  │ Main Processing Loop                │                │
│  │ - Fetch OIGS data                   │                │
│  │ - Process shipments                 │                │
│  └──────────────┬──────────────────────┘                │
│                 │                                         │
│     ┌───────────┴───────────┐                            │
│     │                       │                            │
│  ┌──▼────────┐      ┌───────▼────────┐                  │
│  │ CN Data   │      │ Customer Doc   │                  │
│  │ Section   │      │ Data Section   │                  │
│  └───────────┘      └────────────────┘                  │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### 2.2 Enhanced Architecture

```
┌─────────────────────────────────────────────────────────┐
│ Function Module: Z_SCM_LIQUID_IB_GETDET                 │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌─────────────────────────────────────┐                │
│  │ Main Processing Loop                │                │
│  │ - Fetch OIGS data                   │                │
│  │ - Fetch YTTSTX0001/YTTSTX0002 data │                │
│  │ - Process shipments                 │                │
│  └──────────────┬──────────────────────┘                │
│                 │                                         │
│     ┌───────────┴───────────┐                            │
│     │                       │                            │
│  ┌──▼────────┐      ┌───────▼────────┐                  │
│  │ CN Data   │      │ Customer Doc   │                  │
│  │ Section   │      │ Data Section   │                  │
│  │           │      │                 │                  │
│  │ Step 1:   │      │ Step 1:         │                  │
│  │ YTTSTX0002│      │ YTTSTX0002      │                  │
│  │           │      │                 │                  │
│  │ Step 2&3: │      │ Step 2&3:       │                  │
│  │ OIGS      │      │ OIGS            │                  │
│  │ Override  │      │ Override        │                  │
│  └───────────┘      └────────────────┘                  │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

---

## 3. Technical Design

### 3.1 Data Flow

#### 3.1.1 Data Sources

**YTTSTX0001 (Already Fetched)**
- Fetched in main processing loop (Lines ~879-936)
- Stored in internal table: `lt_yttstx0001`
- Key: `SHNUMBER` (Shipment Number)
- Output: `REPORT_NO`

**YTTSTX0002 (Already Fetched)**
- Fetched in main processing loop (Lines ~901-916)
- Stored in internal table: `lt_yttstx0002`
- Key: `REPORT_NO` (from YTTSTX0001)
- Output: `LR_NO`, `LR_DT`

**OIGS (Already Fetched)**
- Fetched in main processing loop (Lines ~630-711)
- Stored in internal table: `lt_oigs`
- Work area: `lw_oigs`
- Fields: `LR_NO`, `LR_DT`

#### 3.1.2 Processing Logic

**CN Data Section (Lines 1785-1803)**

```abap
"Step 1: Fetch LR Details from YTTSTX0002 (Initial Default)
READ TABLE lt_yttstx0001 INTO lw_yttstx0001 
  WITH KEY shnumber = lw_oigs-shnumber BINARY SEARCH.
IF sy-subrc IS INITIAL.
  READ TABLE lt_yttstx0002 INTO lw_yttstx0002 
    WITH KEY report_no = lw_yttstx0001-report_no BINARY SEARCH.
  IF sy-subrc IS INITIAL.
    lw_cn_data-cn_no   = lw_yttstx0002-lr_no.
    lw_cn_data-cn_date = lw_yttstx0002-lr_dt.
    lw_cn_data-cn_qty  = lw_yttstx0002-dlvry_qty1.
    lw_cn_data-cn_uom  = lw_yttstx0002-desp_uom.
  ENDIF.
ENDIF.

"Step 2 & 3: Fetch LR Details from OIGS and Override if both fields are populated
IF lw_oigs-lr_no IS NOT INITIAL AND lw_oigs-lr_dt IS NOT INITIAL.
  lw_cn_data-cn_no   = lw_oigs-lr_no.
  lw_cn_data-cn_date = lw_oigs-lr_dt.
ENDIF.
```

**Customer Document Data Section (Lines 2024-2045)**

```abap
"Step 1: Fetch LR Details from YTTSTX0002 (Initial Default)
READ TABLE lt_yttstx0001 INTO lw_yttstx0001 
  WITH KEY shnumber = lw_oigs-shnumber BINARY SEARCH.
IF sy-subrc = 0.
  READ TABLE lt_yttstx0002 INTO lw_yttstx0002 
    WITH KEY report_no = lw_yttstx0001-report_no BINARY SEARCH.
  IF sy-subrc = 0.
    lw_cust_doc_data_td-cn_no   = lw_yttstx0002-lr_no.
    lw_cust_doc_data_td-cn_date = lw_yttstx0002-lr_dt.
    lw_cust_doc_data-cn_no      = lw_yttstx0002-lr_no.
    lw_cust_doc_data-cn_date    = lw_yttstx0002-lr_dt.
  ENDIF.
ENDIF.

"Step 2 & 3: Fetch LR Details from OIGS and Override if both fields are populated
IF lw_oigs-lr_no IS NOT INITIAL AND lw_oigs-lr_dt IS NOT INITIAL.
  lw_cust_doc_data_td-cn_no   = lw_oigs-lr_no.
  lw_cust_doc_data_td-cn_date = lw_oigs-lr_dt.
  lw_cust_doc_data-cn_no      = lw_oigs-lr_no.
  lw_cust_doc_data-cn_date    = lw_oigs-lr_dt.
ENDIF.
```

### 3.2 Code Changes Summary

#### Change 1: CN Data Section

**Location**: Lines 1785-1803

**Before**:
```abap
READ TABLE lt_yttstx0001 INTO lw_yttstx0001 WITH KEY shnumber = lw_oigs-shnumber BINARY SEARCH.
IF sy-subrc IS INITIAL.
  READ TABLE lt_yttstx0002 INTO lw_yttstx0002 WITH KEY report_no = lw_yttstx0001-report_no BINARY SEARCH.
  IF sy-subrc IS INITIAL.
****    lw_cn_data-cn_no      =  lw_yttstx0002-lr_no.
****    lw_cn_data-cn_date    =  lw_yttstx0002-lr_dt.
    lw_cn_data-cn_qty     =  lw_yttstx0002-dlvry_qty1.
    lw_cn_data-cn_uom     =  lw_yttstx0002-desp_uom.
  ENDIF.
ENDIF.

lw_cn_data-cn_no      =  lw_oigs-lr_no.
lw_cn_data-cn_date    =  lw_oigs-lr_dt.
```

**After**:
```abap
"Step 1: Fetch LR Details from YTTSTX0002 (Initial Default)
READ TABLE lt_yttstx0001 INTO lw_yttstx0001 WITH KEY shnumber = lw_oigs-shnumber BINARY SEARCH.
IF sy-subrc IS INITIAL.
  READ TABLE lt_yttstx0002 INTO lw_yttstx0002 WITH KEY report_no = lw_yttstx0001-report_no BINARY SEARCH.
  IF sy-subrc IS INITIAL.
    lw_cn_data-cn_no      =  lw_yttstx0002-lr_no.
    lw_cn_data-cn_date    =  lw_yttstx0002-lr_dt.
    lw_cn_data-cn_qty     =  lw_yttstx0002-dlvry_qty1.
    lw_cn_data-cn_uom     =  lw_yttstx0002-desp_uom.
  ENDIF.
ENDIF.

"Step 2 & 3: Fetch LR Details from OIGS and Override if both fields are populated
IF lw_oigs-lr_no IS NOT INITIAL AND lw_oigs-lr_dt IS NOT INITIAL.
  lw_cn_data-cn_no      =  lw_oigs-lr_no.
  lw_cn_data-cn_date    =  lw_oigs-lr_dt.
ENDIF.
```

**Changes**:
1. Uncommented YTTSTX0002 assignment (lines 1791-1792)
2. Added conditional override logic for OIGS (lines 1799-1802)
3. Added descriptive comments

#### Change 2: Customer Document Data Section

**Location**: Lines 2024-2045

**Before**:
```abap
***        READ TABLE lt_yttstx0001 INTO lw_yttstx0001 WITH KEY shnumber = lw_oigs-shnumber BINARY SEARCH.
***        IF sy-subrc = 0.
***          READ TABLE lt_yttstx0002 INTO lw_yttstx0002 WITH KEY report_no = lw_yttstx0001-report_no BINARY SEARCH.
***          IF sy-subrc = 0.
***            lw_cust_doc_data_td-cn_no          = lw_yttstx0002-lr_no.
***            lw_cust_doc_data_td-cn_date        = lw_yttstx0002-lr_dt.
***
***            lw_cust_doc_data-cn_no             = lw_yttstx0002-lr_no.
***            lw_cust_doc_data-cn_date           = lw_yttstx0002-lr_dt.
***          ENDIF.
***        ENDIF.

        lw_cust_doc_data_td-cn_no          = lw_oigs-lr_no.
        lw_cust_doc_data_td-cn_date        = lw_oigs-lr_dt.

        lw_cust_doc_data-cn_no             = lw_oigs-lr_no.
        lw_cust_doc_data-cn_date           = lw_oigs-lr_dt.
```

**After**:
```abap
"Step 1: Fetch LR Details from YTTSTX0002 (Initial Default)
READ TABLE lt_yttstx0001 INTO lw_yttstx0001 WITH KEY shnumber = lw_oigs-shnumber BINARY SEARCH.
IF sy-subrc = 0.
  READ TABLE lt_yttstx0002 INTO lw_yttstx0002 WITH KEY report_no = lw_yttstx0001-report_no BINARY SEARCH.
  IF sy-subrc = 0.
    lw_cust_doc_data_td-cn_no          = lw_yttstx0002-lr_no.
    lw_cust_doc_data_td-cn_date        = lw_yttstx0002-lr_dt.

    lw_cust_doc_data-cn_no             = lw_yttstx0002-lr_no.
    lw_cust_doc_data-cn_date           = lw_yttstx0002-lr_dt.
  ENDIF.
ENDIF.

"Step 2 & 3: Fetch LR Details from OIGS and Override if both fields are populated
IF lw_oigs-lr_no IS NOT INITIAL AND lw_oigs-lr_dt IS NOT INITIAL.
  lw_cust_doc_data_td-cn_no          = lw_oigs-lr_no.
  lw_cust_doc_data_td-cn_date        = lw_oigs-lr_dt.

  lw_cust_doc_data-cn_no             = lw_oigs-lr_no.
  lw_cust_doc_data-cn_date           = lw_oigs-lr_dt.
ENDIF.
```

**Changes**:
1. Uncommented YTTSTX0002 assignment (lines 2030-2034)
2. Added conditional override logic for OIGS (lines 2039-2045)
3. Added descriptive comments

---

## 4. Technical Implementation Details

### 4.1 Data Structures

**No Changes Required**
- All data structures remain unchanged
- Work areas and internal tables already exist
- No new fields added

**Existing Work Areas Used**:
- `lw_yttstx0001`: Type `lty_yttstx0001`
- `lw_yttstx0002`: Type `lty_yttstx0002`
- `lw_oigs`: Type `lty_oigs`
- `lw_cn_data`: Type `zsce_cn_data_s`
- `lw_cust_doc_data_td`: Type `zsce_cust_doc_data_s`
- `lw_cust_doc_data`: Type `zsce_cust_doc_data_s`

### 4.2 Internal Tables

**No Changes Required**
- `lt_yttstx0001`: Already populated (Lines 879-894)
- `lt_yttstx0002`: Already populated (Lines 901-916)
- `lt_oigs`: Already populated (Lines 630-711)

**Table Access**:
- All tables are accessed using `READ TABLE` with `BINARY SEARCH`
- Tables are pre-sorted for optimal performance
- No additional SELECT statements required

### 4.3 Logic Flow

#### 4.3.1 CN Data Processing Flow

```
LOOP AT lt_oigs INTO lw_oigs
  │
  ├─> Initialize lw_cn_data structure
  │
  ├─> [Step 1] Read YTTSTX0001 using SHNUMBER
  │   │
  │   └─> IF found
  │       │
  │       └─> [Step 1a] Read YTTSTX0002 using REPORT_NO
  │           │
  │           └─> IF found
  │               │
  │               └─> Assign: cn_no = YTTSTX0002-LR_NO
  │                   Assign: cn_date = YTTSTX0002-LR_DT
  │
  ├─> [Step 2 & 3] Check OIGS-LR_NO and OIGS-LR_DT
  │   │
  │   └─> IF both are NOT INITIAL
  │       │
  │       └─> Override: cn_no = OIGS-LR_NO
  │           Override: cn_date = OIGS-LR_DT
  │
  └─> [Step 4] RNM Check (Existing Logic - Unchanged)
      │
      └─> IF RNM shipment type
          │
          └─> Override with ZLRPO values
```

#### 4.3.2 Customer Document Data Processing Flow

```
LOOP AT lt_oigs INTO lw_oigs
  │
  └─> LOOP AT lt_oigsi INTO lw_oigsi
      │
      ├─> Initialize lw_cust_doc_data_td and lw_cust_doc_data
      │
      ├─> [Step 1] Read YTTSTX0001 using SHNUMBER
      │   │
      │   └─> IF found
      │       │
      │       └─> [Step 1a] Read YTTSTX0002 using REPORT_NO
      │           │
      │           └─> IF found
      │               │
      │               └─> Assign: cn_no = YTTSTX0002-LR_NO (both structures)
      │                   Assign: cn_date = YTTSTX0002-LR_DT (both structures)
      │
      ├─> [Step 2 & 3] Check OIGS-LR_NO and OIGS-LR_DT
      │   │
      │   └─> IF both are NOT INITIAL
      │       │
      │       └─> Override: cn_no = OIGS-LR_NO (both structures)
      │           Override: cn_date = OIGS-LR_DT (both structures)
      │
      └─> [Step 4] RNM Check (Existing Logic - Unchanged)
          │
          └─> IF RNM shipment type
              │
              └─> Override with ZLRPO values
```

### 4.4 Performance Considerations

#### 4.4.1 Optimization Techniques

1. **Binary Search**: All table reads use `BINARY SEARCH` for optimal performance
2. **Pre-sorted Tables**: Internal tables are sorted before processing
3. **No Additional SELECTs**: Uses existing data fetched in main loop
4. **Minimal Overhead**: Only adds conditional checks and assignments

#### 4.4.2 Performance Impact

**Before Enhancement**:
- 2 READ TABLE operations per shipment (YTTSTX0001, YTTSTX0002)
- 1 direct assignment from OIGS

**After Enhancement**:
- 2 READ TABLE operations per shipment (YTTSTX0001, YTTSTX0002) - Same
- 1 conditional check (OIGS fields)
- 1 conditional assignment (OIGS values)

**Impact**: Negligible - Only adds 2 conditional checks per shipment

#### 4.4.3 Memory Usage

- No additional internal tables
- No additional work areas
- No memory overhead

---

## 5. Code Quality Standards

### 5.1 ABAP Coding Guidelines

**Compliance**:
- ✅ Uses descriptive comments
- ✅ Follows naming conventions
- ✅ Uses BINARY SEARCH for table reads
- ✅ Proper indentation and formatting
- ✅ Clear logic flow

**Code Review Checklist**:
- [x] Comments explain business logic
- [x] Variable names are meaningful
- [x] No hard-coded values
- [x] Error handling considered
- [x] Performance optimized

### 5.2 Error Handling

**Current Implementation**:
- Uses `sy-subrc` checks after READ TABLE operations
- Handles missing data gracefully
- No exceptions raised

**Recommendations**:
- Current error handling is sufficient
- No additional error handling required

---

## 6. Testing Strategy

### 6.1 Unit Testing

**Test Case 1: OIGS Complete Data**
```abap
" Setup
lw_oigs-lr_no = 'LR123456'
lw_oigs-lr_dt = '20240115'
lw_yttstx0002-lr_no = 'LR789012'
lw_yttstx0002-lr_dt = '20240114'

" Execute
" [Code execution]

" Verify
ASSERT lw_cn_data-cn_no = 'LR123456'
ASSERT lw_cn_data-cn_date = '20240115'
```

**Test Case 2: OIGS Incomplete Data**
```abap
" Setup
lw_oigs-lr_no = 'LR123456'
lw_oigs-lr_dt = '' " Blank
lw_yttstx0002-lr_no = 'LR789012'
lw_yttstx0002-lr_dt = '20240114'

" Execute
" [Code execution]

" Verify
ASSERT lw_cn_data-cn_no = 'LR789012'
ASSERT lw_cn_data-cn_date = '20240114'
```

**Test Case 3: OIGS No Data**
```abap
" Setup
lw_oigs-lr_no = '' " Blank
lw_oigs-lr_dt = '' " Blank
lw_yttstx0002-lr_no = 'LR789012'
lw_yttstx0002-lr_dt = '20240114'

" Execute
" [Code execution]

" Verify
ASSERT lw_cn_data-cn_no = 'LR789012'
ASSERT lw_cn_data-cn_date = '20240114'
```

**Test Case 4: RNM Shipment Type**
```abap
" Setup
" RNM shipment type configured
lw_oigs-lr_no = 'LR123456'
lw_oigs-lr_dt = '20240115'
lw_zlrpo-lr_no = 'LR999999'
lw_zlrpo-created_on = '20240116'

" Execute
" [Code execution including RNM logic]

" Verify
ASSERT lw_cn_data-cn_no = 'LR999999'
ASSERT lw_cn_data-cn_date = '20240116'
```

### 6.2 Integration Testing

**Test Scenarios**:
1. End-to-end function module call with various shipment types
2. Verify output structures are populated correctly
3. Verify both CN Data and Customer Document Data sections
4. Verify RNM logic still works correctly

### 6.3 Regression Testing

**Test Areas**:
1. Existing functionality for all shipment types
2. Invoice calculation logic
3. Business ID and Sub-business ID assignment
4. Route and vehicle information
5. All other output fields

---

## 7. Deployment Plan

### 7.1 Pre-Deployment

1. **Code Review**: Technical and functional review
2. **Unit Testing**: Execute all unit test cases
3. **Integration Testing**: Execute integration test scenarios
4. **Performance Testing**: Verify no performance degradation

### 7.2 Deployment Steps

1. **Transport Request**: Create transport request
2. **Development System**: Deploy to DEV
3. **Testing**: Execute test cases in DEV
4. **Quality Assurance**: Deploy to QAS
5. **User Acceptance Testing**: Execute UAT
6. **Production**: Deploy to PRD after approval

### 7.3 Post-Deployment

1. **Monitoring**: Monitor function module execution
2. **Validation**: Validate output data quality
3. **Documentation**: Update user documentation if required

---

## 8. Rollback Plan

### 8.1 Rollback Scenarios

**Scenario 1: Data Quality Issues**
- Rollback: Revert to previous version
- Impact: Low - only affects LR data assignment

**Scenario 2: Performance Issues**
- Rollback: Revert to previous version
- Impact: Low - minimal performance impact expected

### 8.2 Rollback Procedure

1. Identify transport request to rollback
2. Create reverse transport request
3. Deploy reverse transport to affected systems
4. Validate rollback success

---

## 9. Dependencies and Prerequisites

### 9.1 Technical Prerequisites

- SAP SCM system with Transportation Management
- Function Module: Z_SCM_LIQUID_IB_GETDET
- Tables: YTTSTX0001, YTTSTX0002, OIGS
- Data Structures: ZSCE_CN_DATA_S, ZSCE_CUST_DOC_DATA_S

### 9.2 Data Prerequisites

- YTTSTX0001 and YTTSTX0002 tables must be populated
- OIGS table must contain shipment data
- Valid relationship between SHNUMBER and REPORT_NO

---

## 10. Approval

| Role | Name | Signature | Date |
|------|------|-----------|------|
| Technical Lead | | | |
| Code Reviewer | | | |
| Quality Assurance | | | |

---

## 11. Appendix

### 11.1 Code Locations

**CN Data Section**:
- Start: Line 1785
- End: Line 1803
- Function: LR data assignment for CN Data structure

**Customer Document Data Section**:
- Start: Line 2024
- End: Line 2045
- Function: LR data assignment for Customer Document Data structures

### 11.2 Related Documentation

- Functional Specification: Functional_Specification_LR_Data_Enhancement.md
- Business Requirement: Prompt - soft implementation.txt
- Function Module Documentation: Z_SCM_LIQUID_IB_GETDET

---

**Document End**

