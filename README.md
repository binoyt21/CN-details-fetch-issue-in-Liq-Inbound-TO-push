# CN-details-fetch-issue-in-Liq-Inbound-TO-push
CN details fetch issue in Liq Inbound TO push
# Z_SCM_LIQUID_IB_GETDET - Liquid Inbound Transportation Order Details

## Overview

This ABAP Function Module (`Z_SCM_LIQUID_IB_GETDET`) is designed to fetch comprehensive details for liquid inbound transportation orders in SAP SCM (Supply Chain Management) systems. It retrieves shipment header data, consignment note (CN) data, and customer document data for liquid transportation orders.

## Description

The function module processes liquid inbound shipments and extracts detailed information including:
- **Order Header Data**: Shipment details, vehicle information, route details, dates, and transportation mode
- **CN (Consignment Note) Data**: LR numbers, dates, quantities, weights, and billing information
- **Customer Document Data**: Invoice details, material information, quantities, and amounts

## Function Signature

```abap
FUNCTION Z_SCM_LIQUID_IB_GETDET
  IMPORTING
    I_TKNUM TYPE ZSCE_TKNUM_RANGE_TT OPTIONAL
    I_DISPT_DT TYPE ZSCE_DATBG_RANGE_TT
    I_TPLST TYPE ZSCE_TPLST_RANGE_TT OPTIONAL
  EXPORTING
    ET_LIB_HEADER_DATA TYPE ZSCE_ORDER_HDR_DATA_T
    ET_LIB_CN_DATA TYPE ZSCE_CN_DATA_T
    ET_LIB_CUST_DOC_DATA TYPE ZSCE_CUST_DOC_DATA_T
```

## Parameters

### Import Parameters

| Parameter | Type | Description | Required |
|-----------|------|-------------|----------|
| `I_TKNUM` | `ZSCE_TKNUM_RANGE_TT` | Range of shipment/tracking numbers | Optional |
| `I_DISPT_DT` | `ZSCE_DATBG_RANGE_TT` | Date range for dispatch date | **Required** |
| `I_TPLST` | `ZSCE_TPLST_RANGE_TT` | Range of transportation planning status | Optional |

### Export Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `ET_LIB_HEADER_DATA` | `ZSCE_ORDER_HDR_DATA_T` | Table of order header data |
| `ET_LIB_CN_DATA` | `ZSCE_CN_DATA_T` | Table of consignment note data |
| `ET_LIB_CUST_DOC_DATA` | `ZSCE_CUST_DOC_DATA_T` | Table of customer document data |

## Key Features

### 1. Multi-Source LR Data Fetching
- **Primary Source**: OIGS table (Shipment Header)
- **Fallback Source**: YTTSTX0002 table (Transportation Report)
- Ensures LR Number and LR Date are always populated when available

### 2. Shipment Type Support
- Chemical Inbound shipments
- RNM (Returnable Material) shipments
- Port-to-Plant (TTC2) shipments
- Storage Location to Storage Location transfers

### 3. Business Logic
- Supports multiple business units and sub-business units
- Handles different billing regimes (FCM, GTA, etc.)
- Calculates invoice values and quantities
- Supports unit conversions (KG to MT)

### 4. Vehicle and Route Management
- Fetches vehicle details from OIGSV
- Retrieves route information and transit durations
- Supports QWIK vehicle type integration

## Recent Implementation

### LR Number and LR Date Enhancement (Latest)

**Business Requirement:**
- LR Number and LR Date must not be blank
- OIGS remains the primary source
- YTTSTX0002 acts as a fallback source when OIGS does not have LR details

**Implementation Logic:**

1. **Step 1: Initial Default from YTTSTX0002**
   - Fetches LR details from YTTSTX0002 using REPORT_NO
   - Assigns `cn_no` and `cn_date` as initial default values

2. **Step 2: Override from OIGS**
   - Checks OIGS table for LR_NO and LR_DT
   - If both fields are populated in OIGS, overrides the YTTSTX0002 values

3. **Step 3: RNM Special Handling**
   - For RNM shipment types, ZLRPO table takes precedence

**Affected Sections:**
- CN Data assignment (Lines 1785-1803)
- Customer Document Data assignment (Lines 2024-2045)

## Technical Details

### Main Tables Used

- **OIGS**: Shipment Header
- **OIGSI**: Shipment Items
- **OIGSS**: Shipment Stops
- **OIGSV**: Shipment Vehicles
- **YTTSTX0001**: Transportation Report Header
- **YTTSTX0002**: Transportation Report Details
- **ZLRPO**: LR/PO Details
- **LIPS**: Delivery Item
- **LIKP**: Delivery Header
- **EKKO/EKPO**: Purchase Order Header/Item
- **VBELN**: Sales Document
- **T001W**: Plant Master
- **ADRC**: Address Data

### Key Constants

- `lc_chem_shtyp`: Chemical shipment types
- `lc_rnm_shtyp`: RNM shipment types
- `lc_business`: Business ID configuration
- `lc_trans_movt`: Transportation movement type
- `lc_servcat`: Service category
- `lc_cnqty_fetch`: CN quantity fetch configuration
- `lc_liqd_subtype`: Liquid sub-business type

## Usage Example

```abap
DATA: lt_header_data TYPE zsce_order_hdr_data_t,
      lt_cn_data     TYPE zsce_cn_data_t,
      lt_cust_doc    TYPE zsce_cust_doc_data_t,
      lr_dispt_dt    TYPE zsce_datbg_range_tt.

" Prepare date range
lr_dispt_dt = VALUE #( ( sign = 'I' option = 'BT' low = '20240101' high = '20241231' ) ).

" Call function module
CALL FUNCTION 'Z_SCM_LIQUID_IB_GETDET'
  EXPORTING
    i_dispt_dt        = lr_dispt_dt
    i_tplst           = VALUE #( ( sign = 'I' option = 'EQ' low = 'A' ) )
  IMPORTING
    et_lib_header_data = lt_header_data
    et_lib_cn_data     = lt_cn_data
    et_lib_cust_doc_data = lt_cust_doc.
```

## File Structure

```
.
├── README.md                          # This file
├── Z_SCM_LIQUID_IB_GETDET.txt        # Main function module source code
└── Prompt - soft implementation.txt   # Business requirement document
```

## Prerequisites

- SAP SCM system with Transportation Management module
- Access to custom tables: ZSCE_*, YTTSTX*, ZLRPO, ZPTC_LRPO, ZTRSTLMNT
- Required function modules:
  - `Z_SCM_QWIK_SHIP_CHECK`
  - `Z_SCM_GET_QWIKVEH`
  - `Z_SCM_LIQUID_CN_WEIGHT`
  - `Z_SCM_FCM_VEND_ELGBTY_CHK`
  - `/OBIZ/RIMP_GET_STOBOE`

## Change History

| Date | Author | Description |
|------|--------|-------------|
| 30.04.2020 | Eswara Reddy | Initial creation |
| 19.05.2020 | - | QWIK vehicle type support |
| 29.10.2020 | Swapnesh | CN quantity fetch enhancement |
| 12.08.2021 | Ravi M | LR number/date logic update |
| 31.01.2024 | Kalpesh | Billing regime enhancements |
| 27.06.2024 | Kalpesh/Biswa | FCM vendor mapping |
| Latest | - | LR data fallback logic (YTTSTX0002) |

## Authors

- **Technical Author**: Eswara Reddy
- **Functional Author**: Minal Tike
- **Contributors**: Various team members (see change history in code)

## Notes

- This function module handles client-specific data (uses `CLIENT SPECIFIED` in SELECT statements)
- Performance optimized with proper sorting and binary search where applicable
- Supports both standard and custom shipment types
- Includes extensive error handling and data validation

## Support

For issues or questions related to this function module, please contact the SCM development team or refer to the technical documentation.

## License

This is proprietary SAP code. All rights reserved.

---

**Version**: 1.0  
**Last Updated**: 2024  
**SAP Module**: SCM - Transportation Management

