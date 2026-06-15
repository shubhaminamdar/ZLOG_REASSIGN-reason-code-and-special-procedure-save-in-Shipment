# Vendor Change Type — Z_SCM_VENDOR_SAVE_REASSIGN

**Function Module:** `Z_SCM_VENDOR_SAVE_REASSIGN`  
**Function Group:** `ZSCM_TRUCK_VENDOR_REASSIGN`  
**System:** RD2 (source verified via SAP MCP)  
**Config Table:** `ZLOG_EXEC_VAR`

## Summary

Before calling `BAPI_SHIPMENT_CHANGE`, read `ZLOG_EXEC_VAR` to map the input reason code (`I_RES_CODE`) to a special procedure, and pass the reason code to the shipment header.

| Req | Description |
|-----|-------------|
| 1 | Read `ZLOG_EXEC_VAR` where `NAME = ZSCM_REASSIGN_VEND_CHGNE_TYPE`, `REMARKS = I_RES_CODE`, `ACTIVE = X`; get `ERRORMSG` |
| 2 | Before `BAPI_SHIPMENT_CHANGE`: set `gw_header-special_procedure_id` from `ERRORMSG`, `gw_headerx-special_procedure_id = 'C'`, and `gw_header-zzres_des = I_RES_CODE` |

## Current State (RD2)

The BAPI block in `Z_SCM_VENDOR_SAVE_REASSIGN` currently only updates the vendor (`service_agent_id`). It does **not**:

- Read `ZSCM_REASSIGN_VEND_CHGNE_TYPE` from `ZLOG_EXEC_VAR`
- Set `special_procedure_id`
- Set `zzres_des`

## Prerequisite — Configuration (Basis / SM30)

Create and maintain entries in `ZLOG_EXEC_VAR` for parameter `ZSCM_REASSIGN_VEND_CHGNE_TYPE` (not yet present in RD2 at time of analysis).

| NAME | NUMB | REMARKS | ERRORMSG | ACTIVE |
|------|------|---------|----------|--------|
| `ZSCM_REASSIGN_VEND_CHGNE_TYPE` | 0001 | `R01` | `PTPK` | X |
| `ZSCM_REASSIGN_VEND_CHGNE_TYPE` | 0002 | `R02` | `PREM` | X |

- **REMARKS** — reason code passed as `I_RES_CODE`
- **ERRORMSG** — value for `gw_header-special_procedure_id` (maps to `VTTK-SDABW`; typically 4-char codes such as `PTPK` / `PREM`)

> **Note:** `ZLOG_EXEC_VAR-ERRORMSG` is `NATXT` (73 chars), but `special_procedure_id` is `SDABW` (~4 chars). Store short SDABW codes in config, not full text.

## Objects to Change

| ID | Location | Change |
|----|----------|--------|
| CH-VC01 | CONSTANTS section | Add `lc_vend_chgne_type` |
| CH-VC02 | SELECT from `ZLOG_EXEC_VAR` | Add `lc_vend_chgne_type` to `WHERE name IN (...)` |
| CH-VC03 | Before `BAPI_SHIPMENT_CHANGE` | READ config; set `special_procedure_id` and `zzres_des` |

---

## Change 1 — Add Constant

In the **CONSTANTS** section, add:

```abap
lc_vend_chgne_type TYPE rvari_vnam VALUE 'ZSCM_REASSIGN_VEND_CHGNE_TYPE'.
```

---

## Change 2 — Include Config Key in Bulk SELECT

The FM already reads `errormsg` from `ZLOG_EXEC_VAR` into `lt_zlog_exec_var`. Add the new key to the existing `WHERE name IN (...)` clause:

```abap
SELECT  name
        shtyp
        shtype
        mfrgr
        active
        remarks
        transplpt
        spart
        rfcdest
        ewb_uom_d
        zzpmatkl1
        errormsg
        lifnr
  FROM zlog_exec_var
  INTO TABLE lt_zlog_exec_var
 WHERE name IN ( gc_business,
                 gc_product_cat,
                 gc_subform_id,
                 gc_servcat,
                 gc_business_id,
                 gc_rate_validate,
                 gc_validate_shtyp,
                 gc_pol_subform_id,
                 gc_vendor_change,
                 gc_subformatid,
                 lc_zscm_efr_check_reassign,
                 lc_z_scm_get_business,
                 lc_business,
                 lc_subbusiness,
                 lc_scm_get_sub_prod_id,
                 lc_zscm_get_rpl_sub,
                 lc_subformat,
                 lc_spot_vend_list,
                 lc_spot_vend_reassign,
                 lc_vend_chgne_type )    " << ADD THIS
   AND active = abap_true.
```

The global `active = abap_true` filter already satisfies **ACTIVE = X**.

---

## Change 3 — Read Config by Reason Code (Req 1)

Before `BAPI_SHIPMENT_CHANGE`, read the matching config entry:

```abap
DATA: lw_vend_chgne_cfg TYPE lty_zlog_exec_var.

CLEAR lw_vend_chgne_cfg.
READ TABLE lt_zlog_exec_var INTO lw_vend_chgne_cfg
  WITH KEY name    = lc_vend_chgne_type
           remarks = i_res_code.
" active = X already guaranteed by bulk SELECT filter
```

| Filter | Source |
|--------|--------|
| `NAME` | `ZSCM_REASSIGN_VEND_CHGNE_TYPE` |
| `REMARKS` | `I_RES_CODE` |
| `ACTIVE` | `X` (via bulk SELECT) |
| Value to use | `ERRORMSG` |

---

## Change 4 — Set BAPI Header Fields Before `BAPI_SHIPMENT_CHANGE` (Req 2)

Insert **after** the `service_agent_id` logic and **before** `CALL FUNCTION 'BAPI_SHIPMENT_CHANGE'`:

```abap
"--- Req 1 & 2: Map reason code to special procedure via ZLOG_EXEC_VAR
IF sy-subrc = 0 AND lw_vend_chgne_cfg-errormsg IS NOT INITIAL.
  gw_header-special_procedure_id  = lw_vend_chgne_cfg-errormsg.
  gw_headerx-special_procedure_id = 'C'.
ENDIF.

"--- Req 2: Pass reason code to shipment header
IF i_res_code IS NOT INITIAL.
  gw_header-zzres_des  = i_res_code.
  gw_headerx-zzres_des = 'C'.
ENDIF.

CALL FUNCTION 'BAPI_SHIPMENT_CHANGE'
  EXPORTING
    headerdata       = gw_header
    headerdataaction = gw_headerx
  TABLES
    return           = lt_retrn[].
```

---

## Complete BAPI Block (After Changes)

```abap
DATA: lv_lifnr           TYPE tdlnr,
      gw_header          TYPE bapishipmentheader,
      gw_headerx         TYPE bapishipmentheaderaction,
      lt_retrn           TYPE TABLE OF bapiret2,
      lw_retrn           TYPE bapiret2,
      lw_vend_chgne_cfg  TYPE lty_zlog_exec_var.

lv_lifnr = i_lifnr.
IF lv_lifnr IS INITIAL.
  gw_header-service_agent_id  = lv_lifnr.
  gw_headerx-service_agent_id = 'D'.
ELSE.
  CALL FUNCTION 'CONVERSION_EXIT_ALPHA_INPUT'
    EXPORTING
      input  = lv_lifnr
    IMPORTING
      output = lv_lifnr.
  gw_header-service_agent_id  = lv_lifnr.
  gw_header-shipment_num      = i_shipno.
  gw_headerx-service_agent_id = 'C'.
ENDIF.

" Req 1: Read ZLOG_EXEC_VAR for reason-code → special procedure mapping
CLEAR lw_vend_chgne_cfg.
READ TABLE lt_zlog_exec_var INTO lw_vend_chgne_cfg
  WITH KEY name    = lc_vend_chgne_type
           remarks = i_res_code.

" Req 2: Set special_procedure_id from ERRORMSG
IF sy-subrc = 0 AND lw_vend_chgne_cfg-errormsg IS NOT INITIAL.
  gw_header-special_procedure_id  = lw_vend_chgne_cfg-errormsg.
  gw_headerx-special_procedure_id = 'C'.
ENDIF.

" Req 2: Set reason code on shipment
IF i_res_code IS NOT INITIAL.
  gw_header-zzres_des  = i_res_code.
  gw_headerx-zzres_des = 'C'.
ENDIF.

CALL FUNCTION 'BAPI_SHIPMENT_CHANGE'
  EXPORTING
    headerdata       = gw_header
    headerdataaction = gw_headerx
  TABLES
    return           = lt_retrn[].
```

---

## Important Notes

1. **`gw_headerx-zzres_des`** — The requirement specifies `gw_header-zzres_des = I_RES_CODE`. Also set `gw_headerx-zzres_des = 'C'` so the BAPI actually updates the field (consistent with the enhanced vendor reassign pattern).

2. **When config is missing** — If no matching `ZSCM_REASSIGN_VEND_CHGNE_TYPE` entry exists, skip `special_procedure_id` but still set `zzres_des = I_RES_CODE` when `I_RES_CODE` is not initial.

3. **No FM interface change** — All changes are internal; `I_RES_CODE` is already an importing parameter.

4. **Reference pattern** — Mirrors the existing spot vendor config read (`ZSCM_SPOT_VEND_LIST` + `REMARKS = I_RES_CODE`) already in the FM, and the `apply_special_procedure` logic in `ZLOG_CHNGE_TRUCK_VEND_CLS`.

## BAPI Field Mapping

| BAPI Field | Action Flag | Source |
|------------|-------------|--------|
| `special_procedure_id` | `gw_headerx-special_procedure_id = 'C'` | `ZLOG_EXEC_VAR-ERRORMSG` |
| `zzres_des` | `gw_headerx-zzres_des = 'C'` | `I_RES_CODE` |
| `service_agent_id` | (existing logic) | `I_LIFNR` |
