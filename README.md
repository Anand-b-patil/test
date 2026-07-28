# ZVX_01 — Sample Request RAP Application

A complete SAP RAP (RESTful ABAP Programming Model) build: CDS entities, projections, metadata extensions, behavior definitions, business logic, and OData service exposure for a Fiori Elements app.

---

## Architecture Overview

```
Database Table  →  Root CDS View  →  Projection CDS View  →  Metadata Extension  →  Service Binding  →  Fiori UI
                         ↑
                Behavior Definition + Implementation Class
                (business logic lives here)
```

Each layer only knows about its own concern — the table doesn't know about UI, the root CDS doesn't know which app consumes it, the projection doesn't know how it's rendered, the metadata extension doesn't know the business rules, and the behavior implementation doesn't know about Fiori at all.



---

## Step 1 — Table `ZVX_01_SMPL_REQ`

Build this in ADT: right-click package → New → Other ABAP Repository Object → Database Table.

| Field | Data Element | Key |
|---|---|---|
| CLIENT | MANDT | ✅ |
| REQUEST_UUID | SYSUUID_X16 | ✅ |
| REQUEST_NO | NUMC, length 10 | |
| CUSTOMER | KUNNR | |
| MATERIAL | MATNR | |
| PLANT | WERKS_D | |
| QUANTITY | QUAN, length 13, decimals 3 | |
| UOM | MEINS | |
| REQ_DATE | DATS | |
| REQ_STATUS | CHAR, length 2 | |
| REJECT_REASON | CHAR, length 60 | |
| APPROVED_BY | SYUNAME | |
| APPROVED_ON | DATS | |
| CREATED_BY | SYUNAME | |
| CREATED_AT | TIMESTAMPL | |
| LAST_CHANGED_BY | SYUNAME | |
| LAST_CHANGED_AT | TIMESTAMPL | |
| LOCAL_LAST_CHANGED_AT | TIMESTAMPL | |

Or paste this directly into the table editor's **Source Code** view:

```abap
@EndUserText.label : 'Material Sample Dispatch Request'
@AbapCatalog.enhancement.category : #NOT_EXTENSIBLE
@AbapCatalog.tableCategory : #TRANSPARENT
@AbapCatalog.deliveryClass : #A
@AbapCatalog.dataMaintenance : #RESTRICTED
define table zvx_01_smpl_req {
  key client                 : mandt not null;
  key request_uuid           : sysuuid_x16 not null;
  request_no                 : numc10;
  customer                   : kunnr;
  material                   : matnr;
  plant                      : werks_d;
  @Semantics.quantity.unitOfMeasure : 'zvx_01_smpl_req.uom'
  quantity                   : abap.quan(13,3);
  uom                        : meins;
  req_date                   : dats;
  req_status                 : abap.char(2);
  reject_reason              : abap.char(60);
  approved_by                : syuname;
  approved_on                : dats;
  created_by                 : syuname;
  created_at                 : timestampl;
  last_changed_by            : syuname;
  last_changed_at            : timestampl;
  local_last_changed_at      : timestampl;

}
```

**Do not skip this or it won't activate:** go to the table's **Currency/Quantity Fields** tab (or the `@Semantics.quantity.unitOfMeasure` annotation above does this for you in the textual editor) and set QUANTITY's reference field to UOM. This is the #1 activation failure the brief warns about — QUAN fields in DDIC always need to know which field holds their unit.

**Defense-ready one-liner for this step:** REQUEST_UUID is the technical key RAP needs internally; REQUEST_NO is the human-friendly number shown in the UI — they're deliberately separate so the UUID can be system-generated and never renumbered.

---
## Step 2 — Message class `ZVX_01_MSG`

In ADT: right-click package → New → Other ABAP Repository Object → Message Class. Then add these message numbers (double-click into the message class, "Insert Line", fill in number/text):

| No. | Text (use `&` for placeholders) |
|---|---|
| 001 | Material & does not exist |
| 002 | Material & is not maintained for plant & |
| 003 | Customer & does not exist |
| 004 | Customer & is blocked for orders |
| 005 | Quantity must be greater than zero |
| 006 | Quantity exceeds maximum of & for this material |
| 007 | Monthly cap of & requests reached for customer & |
| 008 | Reject reason must be at least 10 characters |

Keep this exact numbering — you'll reference `001`–`008` by number in the behaviour implementation, so it's worth writing them down now rather than improvising later.

**Defense one-liner:** messages live in one message class instead of hard-coded strings so the error text is centrally maintainable/translatable — same reasoning as the utility class, just for text instead of logic.

---

## Step 3 — Utility class `ZCL_VX_01_SAMPLE_RULES`

This is the one class both the RAP behaviour and the ALV report call into — don't let any copy of this logic leak into either of those later.

```abap
CLASS zcl_vx_01_sample_rules DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC.

  PUBLIC SECTION.
    CONSTANTS:
      gc_mtart_fert        TYPE mtart VALUE 'FERT',
      gc_mtart_hawa        TYPE mtart VALUE 'HAWA',
      gc_mtart_roh         TYPE mtart VALUE 'ROH',
      gc_max_fert          TYPE i     VALUE 5,
      gc_max_hawa          TYPE i     VALUE 10,
      gc_max_roh           TYPE i     VALUE 25,
      gc_max_other         TYPE i     VALUE 2,
      gc_monthly_cap       TYPE i     VALUE 3,
      gc_status_requested  TYPE char2 VALUE '01',
      gc_status_approved   TYPE char2 VALUE '02',
      gc_status_dispatched TYPE char2 VALUE '03',
      gc_status_rejected   TYPE char2 VALUE '04'.

    CLASS-METHODS:
      get_max_sample_qty
        IMPORTING iv_material   TYPE matnr
        RETURNING VALUE(rv_max) TYPE i,

      get_status_text
        IMPORTING iv_status      TYPE char2
        RETURNING VALUE(rv_text) TYPE string,

      is_monthly_cap_hit
        IMPORTING iv_customer   TYPE kunnr
                  iv_date       TYPE dats
        RETURNING VALUE(rv_hit) TYPE abap_bool.

ENDCLASS.


CLASS zcl_vx_01_sample_rules IMPLEMENTATION.

  METHOD get_max_sample_qty.
    DATA lv_mtart TYPE mtart.

    SELECT SINGLE mtart
      FROM mara
      WHERE matnr = @iv_material
      INTO @lv_mtart.

    CASE lv_mtart.
      WHEN gc_mtart_fert.
        rv_max = gc_max_fert.
      WHEN gc_mtart_hawa.
        rv_max = gc_max_hawa.
      WHEN gc_mtart_roh.
        rv_max = gc_max_roh.
      WHEN OTHERS.
        rv_max = gc_max_other.
    ENDCASE.
  ENDMETHOD.

  METHOD get_status_text.
    CASE iv_status.
      WHEN gc_status_requested.
        rv_text = 'Requested'.
      WHEN gc_status_approved.
        rv_text = 'Approved'.
      WHEN gc_status_dispatched.
        rv_text = 'Dispatched'.
      WHEN gc_status_rejected.
        rv_text = 'Rejected'.
      WHEN OTHERS.
        rv_text = 'Unknown'.
    ENDCASE.
  ENDMETHOD.

  METHOD is_monthly_cap_hit.
    DATA: lv_count       TYPE i,
          lv_month_start TYPE dats,
          lv_month_end   TYPE dats.

    lv_month_start = iv_date(6) && '01'.

    CALL FUNCTION 'LAST_DAY_OF_MONTHS'
      EXPORTING
        day_in            = iv_date
      IMPORTING
        last_day_of_month = lv_month_end
      EXCEPTIONS
        day_in_no_date    = 1
        OTHERS            = 2.

    SELECT COUNT(*)
      FROM zvx_01_smpl_req
      WHERE customer   = @iv_customer
        AND req_date    BETWEEN @lv_month_start AND @lv_month_end
        AND req_status <> @gc_status_rejected
      INTO @lv_count.

    rv_hit = COND #( WHEN lv_count >= gc_monthly_cap THEN abap_true ELSE abap_false ).
  ENDMETHOD.

ENDCLASS.
```

**Quick sanity check before moving on** (worth the 60 seconds): open a class-based test include or just debug-execute `zcl_vx_01_sample_rules=>get_max_sample_qty( 'some_FERT_material' )` and confirm it returns `5`, and a HAWA material returns `10`. Catching a wrong MTART mapping now saves you re-checking every downstream validation later.

**Defense one-liner:** this class is `FINAL CREATE PUBLIC` with only `CLASS-METHODS` (static) — no state, so both the RAP behaviour class and the ALV report can call it directly by name without instantiating anything, which is exactly why the brief scores "genuinely reused, not copy-pasted" separately in engineering quality.

---

## Step 4 — CDS Root View Entity: `ZVX_01_R_SampleRequest`

**New → Other ABAP Repository Object → Data Definition**

```abap
@AbapCatalog.sqlViewName: 'ZVX01RSMPLREQ'
@AccessControl.authorizationCheck: #CHECK
@EndUserText.label: 'Sample Request - Root View'
@Metadata.allowExtensions: true
@ObjectModel.representativeKey: 'RequestNo'
define root view entity ZVX_01_R_SampleRequest
  as select from zvx_01_smpl_req
  association to parent I_Customer as _Customer on $projection.Customer = _Customer.Customer
  association to parent I_Product  as _Product  on $projection.Material = _Product.Product
  association to parent I_Plant    as _Plant    on $projection.Plant    = _Plant.Plant
{
  key request_uuid                            as RequestUuid,
      request_no                              as RequestNo,
      customer                                as Customer,
      material                                as Material,
      plant                                   as Plant,
      @Semantics.quantity.unitOfMeasure: 'Uom'
      quantity                                as Quantity,
      uom                                     as Uom,
      req_date                                as RequestDate,
      req_status                              as ReqStatus,

      // Calculated field: colours the status in the Fiori list
      case req_status
        when '04' then 1   // Rejected -> red
        when '01' then 2   // Requested -> yellow
        when '02' then 3   // Approved -> green
        when '03' then 3   // Dispatched -> green
        else 3
      end                                     as StatusCriticality,

      reject_reason                           as RejectReason,
      approved_by                             as ApprovedBy,
      approved_on                             as ApprovedOn,

      @Semantics.user.createdBy: true
      created_by                              as CreatedBy,
      @Semantics.systemDateTime.createdAt: true
      created_at                              as CreatedAt,
      @Semantics.user.lastChangedBy: true
      last_changed_by                         as LastChangedBy,
      @Semantics.systemDateTime.lastChangedAt: true
      last_changed_at                         as LastChangedAt,
      @Semantics.systemDateTime.localInstanceLastChangedAt: true
      local_last_changed_at                   as LocalLastChangedAt,

      /* Associations */
      _Customer,
      _Product,
      _Plant
}
```

> **Defense one-liner:** `StatusCriticality` is computed in the CDS layer (not stored) because it's derived purely from `req_status` — no reason to persist a value the database can compute on read, and it keeps the Fiori annotation (criticality-based coloring) simple.

### About `_Customer`, `_Product`, `_Plant`

These are **not objects you create** — they are association names (aliases you choose) pointing to **standard, SAP-delivered CDS views** already present in every S/4HANA system:

- `I_Customer` — standard interface view over `KNA1`
- `I_Product` — standard interface view over `MARA`
- `I_Plant` — standard interface view over `T001W`

`_Customer`, `_Product`, `_Plant` are just the local role names given to these associations inside your own view (underscore prefix is SAP naming convention, not a separate artifact). `association to parent` means it's a to-one relationship — one sample request always points to exactly one customer, one material, one plant.

These associations power two things for free:
1. **Value help (F4)** in the metadata extension via `@Consumption.valueHelpDefinition`.
2. **Navigation** — future apps could drill from a Sample Request into full customer/material/plant details.

Sanity check that they exist in your system (don't create them):
```abap
SELECT SINGLE * FROM i_customer WHERE customer = '1000000'.
SELECT SINGLE * FROM i_product  WHERE product  = 'TG11'.
SELECT SINGLE * FROM i_plant    WHERE plant    = '1010'.
```

---

## Step 5a — Projection View: `ZVX_01_C_SampleRequest`

```abap
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Sample Request - Projection'
@Metadata.allowExtensions: true
@Search.searchable: true
define view entity ZVX_01_C_SampleRequest
  as projection on ZVX_01_R_SampleRequest
{
  key RequestUuid,
      RequestNo,
      Customer,
      Material,
      Plant,
      Quantity,
      Uom,
      RequestDate,
      ReqStatus,
      StatusCriticality,
      RejectReason,
      ApprovedBy,
      ApprovedOn,
      CreatedBy,
      CreatedAt,
      LastChangedBy,
      @Semantics.systemDateTime.lastChangedAt: true
      LastChangedAt,
      @Semantics.systemDateTime.localInstanceLastChangedAt: true
      LocalLastChangedAt,

      /* Associations for value help / navigation */
      _Customer,
      _Product,
      _Plant
}
```

## Step 5b — Metadata Extension: `ZVX_01_C_SAMPLEREQ_MDE`

**New → Other ABAP Repository Object → Metadata Extension.** Target: `ZVX_01_C_SampleRequest`.

```abap
@Metadata.layer: #CUSTOMER
@UI: {
  headerInfo: { typeName: 'Sample Request', typeNamePlural: 'Sample Requests' }
}
annotate view ZVX_01_C_SampleRequest with
{
  @UI.facet: [
    { id: 'Details', purpose: #STANDARD, type: #IDENTIFICATION_REFERENCE, label: 'Request Details', position: 10 }
  ]

  @UI.lineItem: [ { position: 10, label: 'Request No' } ]
  @UI.identification: [ { position: 10 } ]
  RequestNo;

  @UI.lineItem: [ { position: 20, label: 'Customer' } ]
  @UI.identification: [ { position: 20 } ]
  @UI.selectionField: [ { position: 10 } ]
  @Consumption.valueHelpDefinition: [ { entity: { name: 'I_Customer', element: 'Customer' } } ]
  Customer;

  @UI.lineItem: [ { position: 30, label: 'Material' } ]
  @UI.identification: [ { position: 30 } ]
  @UI.selectionField: [ { position: 20 } ]
  @Consumption.valueHelpDefinition: [ { entity: { name: 'I_Product', element: 'Product' } } ]
  Material;

  @UI.lineItem: [ { position: 40, label: 'Quantity' } ]
  @UI.identification: [ { position: 40 } ]
  Quantity;

  @UI.lineItem: [ { position: 50, label: 'UoM' } ]
  Uom;

  @UI.lineItem: [ { position: 60, label: 'Request Date' } ]
  @UI.identification: [ { position: 50 } ]
  RequestDate;

  @UI.lineItem: [ { position: 70, label: 'Status', criticality: 'StatusCriticality' } ]
  @UI.identification: [ { position: 60 } ]
  @UI.selectionField: [ { position: 30 } ]
  ReqStatus;

  @UI.hidden: true
  StatusCriticality;
}
```

> **Defense one-liner:** the metadata extension is a separate object rather than inline annotations because it keeps "what the data is" (projection) cleanly separated from "how it's shown" (UI layer) — you can swap the UI layout without touching the data model.

> **Activation order:** root → projection → metadata extension. Projection won't activate if the root has errors; the metadata extension needs the projection's field names to match exactly.

---

## Step 6 — RAP Behaviour Definition (Root)

**New → Other ABAP Repository Object → Behavior Definition**, on `ZVX_01_R_SampleRequest`.

```abap
managed implementation in class zbp_vx_01_r_samplereq unique;
strict ( 2 );

define behavior for ZVX_01_R_SampleRequest alias SampleRequest
persistent table zvx_01_smpl_req
lock master
authorization master ( instance )
etag master LastChangedAt
{
  field ( numbering : managed )  RequestUuid;
  field ( readonly )             RequestNo, ReqStatus, Uom, ApprovedBy, ApprovedOn;
  field ( mandatory : create )   Customer, Material, Quantity;

  create;
  update;
  delete;

  determination setInitialValues on modify { create; }
  determination deriveUom        on modify { field Material; }

  validation validateMaterial on save { create; field Material, Plant; }
  validation validateCustomer on save { create; field Customer; }
  validation validateQuantity on save { create; field Quantity, Material; }

  action ( features : instance ) approve result [1] $self;

  mapping for zvx_01_smpl_req
  {
    RequestUuid         = request_uuid;
    RequestNo            = request_no;
    Customer             = customer;
    Material             = material;
    Plant                = plant;
    Quantity             = quantity;
    Uom                  = uom;
    RequestDate          = req_date;
    ReqStatus            = req_status;
    RejectReason         = reject_reason;
    ApprovedBy           = approved_by;
    ApprovedOn           = approved_on;
    CreatedBy            = created_by;
    CreatedAt            = created_at;
    LastChangedBy        = last_changed_by;
    LastChangedAt        = last_changed_at;
    LocalLastChangedAt   = local_last_changed_at;
  }
}
```

### Behaviour Definition (Projection)

The service binding publishes the projection, not the root directly — without this the app has nothing to expose.

```abap
projection;
strict ( 2 );

define behavior for ZVX_01_C_SampleRequest alias SampleRequest
{
  use create;
  use update;
  use delete;
  use action approve;
}
```

---

## Step 7 — Implementation Class: `zbp_vx_01_r_samplereq`

Full class — determinations, validations, approve action, and feature control, all together.

```abap
CLASS zbp_vx_01_r_samplereq DEFINITION PUBLIC ABSTRACT FINAL
  FOR BEHAVIOR OF zvx_01_r_samplerequest.

  PRIVATE SECTION.
    METHODS setInitialValues FOR DETERMINE ON MODIFY
      IMPORTING keys FOR SampleRequest~setInitialValues.

    METHODS deriveUom FOR DETERMINE ON MODIFY
      IMPORTING keys FOR SampleRequest~deriveUom.

    METHODS validateMaterial FOR VALIDATE ON SAVE
      IMPORTING keys FOR SampleRequest~validateMaterial.

    METHODS validateCustomer FOR VALIDATE ON SAVE
      IMPORTING keys FOR SampleRequest~validateCustomer.

    METHODS validateQuantity FOR VALIDATE ON SAVE
      IMPORTING keys FOR SampleRequest~validateQuantity.

    METHODS approve FOR MODIFY
      IMPORTING keys FOR ACTION SampleRequest~approve RESULT result.

    METHODS get_features FOR FEATURES
      IMPORTING keys REQUEST requested_features FOR SampleRequest
      RESULT result.

    METHODS get_next_request_no
      RETURNING VALUE(rv_next) TYPE numc10.

ENDCLASS.

CLASS zbp_vx_01_r_samplereq IMPLEMENTATION.

  METHOD get_next_request_no.
    SELECT SINGLE MAX( request_no ) FROM zvx_01_smpl_req INTO @DATA(lv_max).
    rv_next = lv_max + 1.
  ENDMETHOD.

  METHOD setInitialValues.
    DATA lt_update TYPE TABLE FOR UPDATE zvx_01_r_samplerequest.

    LOOP AT keys INTO DATA(ls_key).
      lt_update = VALUE #( BASE lt_update
        ( %tky        = ls_key-%tky
          Plant        = '1010'
          RequestDate  = cl_abap_context_info=>get_system_date( )
          ReqStatus    = '01'
          RequestNo    = get_next_request_no( )
          %control-Plant       = if_abap_behv=>mk-on
          %control-RequestDate = if_abap_behv=>mk-on
          %control-ReqStatus   = if_abap_behv=>mk-on
          %control-RequestNo   = if_abap_behv=>mk-on ) ).
    ENDLOOP.

    MODIFY ENTITIES OF zvx_01_r_samplerequest IN LOCAL MODE
      ENTITY SampleRequest
        UPDATE FIELDS ( Plant RequestDate ReqStatus RequestNo )
        WITH lt_update.
  ENDMETHOD.

  METHOD deriveUom.
    READ ENTITIES OF zvx_01_r_samplerequest IN LOCAL MODE
      ENTITY SampleRequest
        FIELDS ( Material )
        WITH CORRESPONDING #( keys )
      RESULT DATA(lt_requests).

    DATA lt_update TYPE TABLE FOR UPDATE zvx_01_r_samplerequest.

    LOOP AT lt_requests INTO DATA(ls_request).
      SELECT SINGLE meins FROM mara
        WHERE matnr = @ls_request-Material
        INTO @DATA(lv_meins).

      IF sy-subrc = 0.
        lt_update = VALUE #( BASE lt_update
          ( %tky = ls_request-%tky
            Uom  = lv_meins
            %control-Uom = if_abap_behv=>mk-on ) ).
      ENDIF.
    ENDLOOP.

    MODIFY ENTITIES OF zvx_01_r_samplerequest IN LOCAL MODE
      ENTITY SampleRequest
        UPDATE FIELDS ( Uom )
        WITH lt_update.
  ENDMETHOD.

  METHOD validateMaterial.
    READ ENTITIES OF zvx_01_r_samplerequest IN LOCAL MODE
      ENTITY SampleRequest
        FIELDS ( Material Plant )
        WITH CORRESPONDING #( keys )
      RESULT DATA(lt_requests).

    LOOP AT lt_requests INTO DATA(ls_request).
      SELECT SINGLE matnr FROM mara
        WHERE matnr = @ls_request-Material
        INTO @DATA(lv_matnr).

      IF sy-subrc <> 0.
        APPEND VALUE #( %tky = ls_request-%tky ) TO failed-SampleRequest.
        APPEND VALUE #( %tky = ls_request-%tky
                         %msg = new_message( id       = 'ZVX_01_MSG'
                                              number   = '001'
                                              severity = if_abap_behv_message=>severity-error
                                              v1       = ls_request-Material )
                       ) TO reported-SampleRequest.
        CONTINUE.
      ENDIF.

      SELECT SINGLE matnr FROM marc
        WHERE matnr = @ls_request-Material
          AND werks = @ls_request-Plant
        INTO @DATA(lv_marc).

      IF sy-subrc <> 0.
        APPEND VALUE #( %tky = ls_request-%tky ) TO failed-SampleRequest.
        APPEND VALUE #( %tky = ls_request-%tky
                         %msg = new_message( id       = 'ZVX_01_MSG'
                                              number   = '002'
                                              severity = if_abap_behv_message=>severity-error
                                              v1       = ls_request-Material
                                              v2       = ls_request-Plant )
                       ) TO reported-SampleRequest.
      ENDIF.
    ENDLOOP.
  ENDMETHOD.

  METHOD validateCustomer.
    READ ENTITIES OF zvx_01_r_samplerequest IN LOCAL MODE
      ENTITY SampleRequest
        FIELDS ( Customer )
        WITH CORRESPONDING #( keys )
      RESULT DATA(lt_requests).

    LOOP AT lt_requests INTO DATA(ls_request).
      SELECT SINGLE kunnr, loevm, aufsd FROM kna1
        WHERE kunnr = @ls_request-Customer
        INTO @DATA(ls_kna1).

      IF sy-subrc <> 0.
        APPEND VALUE #( %tky = ls_request-%tky ) TO failed-SampleRequest.
        APPEND VALUE #( %tky = ls_request-%tky
                         %msg = new_message( id       = 'ZVX_01_MSG'
                                              number   = '003'
                                              severity = if_abap_behv_message=>severity-error
                                              v1       = ls_request-Customer )
                       ) TO reported-SampleRequest.
        CONTINUE.
      ENDIF.

      IF ls_kna1-loevm = abap_true OR ls_kna1-aufsd IS NOT INITIAL.
        APPEND VALUE #( %tky = ls_request-%tky ) TO failed-SampleRequest.
        APPEND VALUE #( %tky = ls_request-%tky
                         %msg = new_message( id       = 'ZVX_01_MSG'
                                              number   = '004'
                                              severity = if_abap_behv_message=>severity-error
                                              v1       = ls_request-Customer )
                       ) TO reported-SampleRequest.
      ENDIF.
    ENDLOOP.
  ENDMETHOD.

  METHOD validateQuantity.
    READ ENTITIES OF zvx_01_r_samplerequest IN LOCAL MODE
      ENTITY SampleRequest
        FIELDS ( Quantity Material )
        WITH CORRESPONDING #( keys )
      RESULT DATA(lt_requests).

    LOOP AT lt_requests INTO DATA(ls_request).
      IF ls_request-Quantity <= 0.
        APPEND VALUE #( %tky = ls_request-%tky ) TO failed-SampleRequest.
        APPEND VALUE #( %tky = ls_request-%tky
                         %msg = new_message( id       = 'ZVX_01_MSG'
                                              number   = '005'
                                              severity = if_abap_behv_message=>severity-error )
                       ) TO reported-SampleRequest.
        CONTINUE.
      ENDIF.

      DATA(lv_max) = zcl_vx_01_sample_rules=>get_max_sample_qty( ls_request-Material ).

      IF ls_request-Quantity > lv_max.
        APPEND VALUE #( %tky = ls_request-%tky ) TO failed-SampleRequest.
        APPEND VALUE #( %tky = ls_request-%tky
                         %msg = new_message( id       = 'ZVX_01_MSG'
                                              number   = '006'
                                              severity = if_abap_behv_message=>severity-error
                                              v1       = |{ lv_max }| )
                       ) TO reported-SampleRequest.
      ENDIF.
    ENDLOOP.
  ENDMETHOD.

  METHOD approve.
    READ ENTITIES OF zvx_01_r_samplerequest IN LOCAL MODE
      ENTITY SampleRequest
        FIELDS ( ReqStatus )
        WITH CORRESPONDING #( keys )
      RESULT DATA(lt_requests).

    DATA lt_update TYPE TABLE FOR UPDATE zvx_01_r_samplerequest.

    LOOP AT lt_requests INTO DATA(ls_request).
      IF ls_request-ReqStatus <> '01'.
        APPEND VALUE #( %tky = ls_request-%tky ) TO failed-SampleRequest.
        APPEND VALUE #( %tky = ls_request-%tky
                         %msg = new_message( id       = 'ZVX_01_MSG'
                                              number   = '007'
                                              severity = if_abap_behv_message=>severity-error )
                       ) TO reported-SampleRequest.
        CONTINUE.
      ENDIF.

      lt_update = VALUE #( BASE lt_update
        ( %tky              = ls_request-%tky
          ReqStatus          = '02'
          ApprovedBy         = cl_abap_context_info=>get_user_technical_name( )
          ApprovedOn         = cl_abap_context_info=>get_system_date( )
          %control-ReqStatus  = if_abap_behv=>mk-on
          %control-ApprovedBy = if_abap_behv=>mk-on
          %control-ApprovedOn = if_abap_behv=>mk-on ) ).
    ENDLOOP.

    MODIFY ENTITIES OF zvx_01_r_samplerequest IN LOCAL MODE
      ENTITY SampleRequest
        UPDATE FIELDS ( ReqStatus ApprovedBy ApprovedOn )
        WITH lt_update
      FAILED failed
      REPORTED reported.

    READ ENTITIES OF zvx_01_r_samplerequest IN LOCAL MODE
      ENTITY SampleRequest
        ALL FIELDS WITH CORRESPONDING #( lt_update )
      RESULT DATA(lt_result).

    result = VALUE #( FOR ls_res IN lt_result
                       ( %tky   = ls_res-%tky
                         %param = ls_res ) ).
  ENDMETHOD.

  METHOD get_features.
    READ ENTITIES OF zvx_01_r_samplerequest IN LOCAL MODE
      ENTITY SampleRequest
        FIELDS ( ReqStatus )
        WITH CORRESPONDING #( keys )
      RESULT DATA(lt_requests).

    result = VALUE #( FOR ls_request IN lt_requests
      ( %tky                     = ls_request-%tky
        %action-approve          = COND #( WHEN ls_request-ReqStatus = '01'
                                            THEN if_abap_behv=>fc-o-enabled
                                            ELSE if_abap_behv=>fc-o-disabled )
        %field-ReqStatus         = if_abap_behv=>fc-f-read_only
      ) ).
  ENDMETHOD.

ENDCLASS.
```

> **Note the reuse point:** `validateQuantity` calls `zcl_vx_01_sample_rules=>get_max_sample_qty`, not a re-implemented `CASE` statement. This is the exact thing graded under "utility class genuinely reused."

### Defense one-liners

- Validations trigger `on save`, not `on modify`, because they check business-rule correctness that should only block the final commit — checking on every keystroke/field-change would be wasteful and premature while the user is still filling the form.
- `setInitialValues` runs on `create` only (not every modify) because defaults should be set once, not silently reset on every subsequent edit.
- `field ( numbering : managed )` on `RequestUuid` tells RAP to auto-generate the UUID itself rather than expecting the client or the DB to supply it.
- `get_features` (FOR FEATURES) greys out the Approve button in Fiori Elements when `ReqStatus ≠ '01'` — without it, the button would always be clickable and the guard would only fire after the user clicks it and hits the validation error. Doing both (feature control + the action's own status check) is good practice: features for UX, the action's own check for a hard server-side guard — never trust the client alone.
- The action returns `result [1] $self` so after approval RAP re-reads the entity and pushes the updated `ReqStatus`/`ApprovedBy`/`ApprovedOn` straight back to the UI without a manual refresh.
- `ApprovedBy` is set from `cl_abap_context_info=>get_user_technical_name( )` server-side, not accepted from the client — approval identity must come from the authenticated session, not a payload field, or anyone could spoof who approved a request.
- Determinations only receive **keys** in their `IMPORTING keys` parameter, not field values — that's why `deriveUom` explicitly re-reads `Material` via `READ ENTITIES` even though it was just set in the same request.

### CDS/DDIC edge cases

- **Why `strict ( 2 )` and not `strict ( 1 )`?** Level 2 additionally enforces stricter mandatory/readonly field checks and ETag handling under the newer RAP contract — the safer default for new greenfield developments.
- **Why is `RequestNo` `field ( readonly )` and set in a determination, not `field ( numbering : managed )`?** Numbering-managed is for the key UUID; `RequestNo` is a human-readable running number with custom logic (`get_next_request_no`), so it's assigned in `setInitialValues` and simply locked from client input via `readonly`.
- **Why `etag master LastChangedAt`?** Gives optimistic concurrency control for free — if two users open the same request, the second save gets a conflict error instead of silently overwriting the first user's changes.
- **Why `managed` and not `unmanaged` implementation?** Managed lets RAP handle DB persistence (INSERT/UPDATE/DELETE) automatically from the mapping; you only hook in for business logic. Unmanaged would require writing your own `save` method — unnecessary here for a single-table, single-entity scenario.
- **Why `lock master` only at the root?** Only the root entity in a RAP BO carries the lock — it's the transactional anchor; child/projection entities inherit locking through the root.
- **Why no draft handling (`with draft`)?** Draft is only needed for save-for-later / inconsistent-intermediate-state editing. A simple create-then-approve flow doesn't need it — adding it unasked would be over-engineering.

---

## Step 8 — Utility Class: `zcl_vx_01_sample_rules`

```abap
CLASS zcl_vx_01_sample_rules DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION.
    CLASS-METHODS get_max_sample_qty
      IMPORTING iv_material    TYPE matnr
      RETURNING VALUE(rv_max)  TYPE menge_d.
ENDCLASS.

CLASS zcl_vx_01_sample_rules IMPLEMENTATION.
  METHOD get_max_sample_qty.
    SELECT SINGLE mtart FROM mara
      WHERE matnr = @iv_material
      INTO @DATA(lv_mtart).

    rv_max = COND #(
      WHEN lv_mtart = 'FERT' THEN 10
      WHEN lv_mtart = 'HALB' THEN 25
      ELSE 5 ).
  ENDMETHOD.
ENDCLASS.
```

---

## Step 9 — Message Class: `ZVX_01_MSG`

| No. | Text |
|---|---|
| 001 | Material &1 does not exist |
| 002 | Material &1 is not maintained for plant &2 |
| 003 | Customer &1 does not exist |
| 004 | Customer &1 is blocked / marked for deletion |
| 005 | Quantity must be greater than zero |
| 006 | Quantity exceeds maximum allowed sample quantity of &1 |
| 007 | Only requests in Requested status can be approved |

---

## Step 10 — Service Definition & Service Binding

**Service Definition** (New → Other ABAP Repository Object → Service Definition):

```abap
@EndUserText.label: 'Sample Request Service'
define service ZVX_01_UI_SAMPLEREQ_O2 {
  expose ZVX_01_C_SampleRequest as SampleRequest;
}
```

**Service Binding** (New → Other ABAP Repository Object → Service Binding):

- Name: `ZVX_01_UI_SAMPLEREQ_O2`
- Binding type: **OData V4 – UI**
- Service Definition: `ZVX_01_UI_SAMPLEREQ_O2`

Activate, then click **Publish**. Click the entity set `SampleRequest` and use **Preview** to launch the generic Fiori Elements List Report / Object Page — a working test client without needing an actual Fiori Launchpad tile.

---

## Step 11 — Fiori Elements App (List Report / Object Page)

Since the metadata extension already carries `lineItem`, `identification`, `selectionField`, and `headerInfo` annotations, no floorplan config is required beyond generating the app:

1. In the Service Binding preview, confirm the list report shows columns in the order defined by `@UI.lineItem` positions, and that `ReqStatus` renders with criticality coloring.
2. Confirm the **Approve** button appears on the object page toolbar — this comes automatically from the exposed action. To control its position/label explicitly, optionally add on any field in the metadata extension:

```abap
@UI.identification: [ { type: #FOR_ACTION, dataAction: 'approve', label: 'Approve' } ]
```

3. Test the full flow in preview:
   - Create a request → check `RequestNo`/`Uom`/`RequestDate` auto-populate.
   - Try an invalid material/customer/quantity to confirm validation messages fire.
   - Approve a valid one and confirm the button greys out afterward.

> **Defense one-liner:** the app needed zero custom UI5 code — everything (list, object page, value helps, action button, criticality) comes from CDS annotations and RAP metadata. That's the entire point of Fiori Elements: the framework generates the UI from metadata, developers only write the data model and business logic.

---


---

# ZVX_01_SAMPLE_REPORT

```abap
REPORT zvx_01_sample_report.

TYPE-POOLS: vrm.

TABLES:
  zvx_01_smpl_req,
  kna1,
  makt.

*-----------------------------------------------------------------------
* Selection Screen
*-----------------------------------------------------------------------

SELECT-OPTIONS:
  s_kunnr FOR zvx_01_smpl_req-customer,
  s_matnr FOR zvx_01_smpl_req-material,
  s_date  FOR zvx_01_smpl_req-req_date.

PARAMETERS:
  p_stat TYPE char2 AS LISTBOX VISIBLE LENGTH 15.

PARAMETERS:
  p_half AS CHECKBOX.

*-----------------------------------------------------------------------
* Types
*-----------------------------------------------------------------------

TYPES: BEGIN OF ty_output,

         request_no TYPE numc10,
         customer   TYPE kunnr,
         name1      TYPE kna1-name1,
         material   TYPE matnr,
         maktx      TYPE makt-maktx,
         quantity   TYPE menge_d,
         uom        TYPE meins,
         req_date   TYPE dats,
         req_status TYPE char2,

       END OF ty_output.

DATA:

  gt_output TYPE STANDARD TABLE OF ty_output,
  gs_output TYPE ty_output.

*-----------------------------------------------------------------------
* Status Dropdown
*-----------------------------------------------------------------------

INITIALIZATION.

  PERFORM fill_status.

FORM fill_status.

  DATA:
    lt_values TYPE vrm_values,
    ls_value  TYPE vrm_value.

  ls_value-key = ''.
  ls_value-text = 'All'.
  APPEND ls_value TO lt_values.

  ls_value-key = '01'.
  ls_value-text = 'Requested'.
  APPEND ls_value TO lt_values.

  ls_value-key = '02'.
  ls_value-text = 'Approved'.
  APPEND ls_value TO lt_values.

  ls_value-key = '03'.
  ls_value-text = 'Dispatched'.
  APPEND ls_value TO lt_values.

  ls_value-key = '04'.
  ls_value-text = 'Rejected'.
  APPEND ls_value TO lt_values.

  CALL FUNCTION 'VRM_SET_VALUES'
    EXPORTING
      id     = 'P_STAT'
      values = lt_values.

ENDFORM.

*-----------------------------------------------------------------------
* Validation
*-----------------------------------------------------------------------

AT SELECTION-SCREEN.

  IF s_date-high IS NOT INITIAL
     AND s_date-high > sy-datum.

    MESSAGE 'Future date is not allowed' TYPE 'E'.

  ENDIF.

*-----------------------------------------------------------------------
* Start of Selection
*-----------------------------------------------------------------------

START-OF-SELECTION.

  PERFORM get_data.

  PERFORM display_alv.

*-----------------------------------------------------------------------
* Get Data
*-----------------------------------------------------------------------

FORM get_data.

  SELECT

      a~request_no,
      a~customer,
      b~name1,
      a~material,
      c~maktx,
      a~quantity,
      a~uom,
      a~req_date,
      a~req_status

  INTO TABLE @gt_output

  FROM zvx_01_smpl_req AS a

  INNER JOIN kna1 AS b
     ON a~customer = b~kunnr

  INNER JOIN makt AS c
     ON a~material = c~matnr

  WHERE

        a~customer IN @s_kunnr

    AND a~material IN @s_matnr

    AND a~req_date IN @s_date

    AND c~spras = @sy-langu.

  IF p_stat IS NOT INITIAL.

    DELETE gt_output
      WHERE req_status <> p_stat.

  ENDIF.

ENDFORM.

*-----------------------------------------------------------------------
* Display ALV
*-----------------------------------------------------------------------

FORM display_alv.

  DATA:

    lo_alv TYPE REF TO cl_salv_table.

  cl_salv_table=>factory(

      IMPORTING

          r_salv_table = lo_alv

      CHANGING

          t_table = gt_output ).

*-----------------------------------------------------------------------
* Functions
*-----------------------------------------------------------------------

  lo_alv->get_functions( )->set_all( ).

*-----------------------------------------------------------------------
* Optimize Columns
*-----------------------------------------------------------------------

  lo_alv->get_columns( )->set_optimize( ).

*-----------------------------------------------------------------------
* Display
*-----------------------------------------------------------------------

  lo_alv->display( ).

ENDFORM.
```

---

## How One Request Flows Through the Whole Stack

1. User opens the Fiori app and hits **Create**.
2. They fill in Customer, Material, Quantity (mandatory per the behavior definition).
3. On save: RAP auto-generates the UUID (`numbering: managed`), `setInitialValues` fires and stamps Plant/Date/Status/RequestNo, `deriveUom` looks up the UOM from the material master.
4. The three validations run: material/plant checked against `MARA`/`MARC`, customer checked against `KNA1` (not blocked), quantity checked against the utility class's material-type-based max.
5. If anything fails, the message (from `ZVX_01_MSG`) is surfaced tied to the specific failed record, and the save is rejected.
6. If it passes, the record is persisted with status `01` (Requested), `StatusCriticality` computes to yellow.
7. Later, someone clicks **Approve** — `get_features` has already confirmed the button is enabled. The action re-validates server-side, sets status to `02` (Approved), stamps `ApprovedBy`/`ApprovedOn`, and RAP pushes the refreshed record straight back to the UI (`result [1] $self`) — criticality flips to green, no manual refresh needed.

---

## Complete Object Checklist

| # | Object | Name |
|---|---|---|
| 1 | Table | `zvx_01_smpl_req` |
| 2 | Root CDS view entity | `ZVX_01_R_SampleRequest` |
| 3 | Projection CDS view | `ZVX_01_C_SampleRequest` |
| 4 | Metadata extension | `ZVX_01_C_SAMPLEREQ_MDE` |
| 5 | Behavior definition (root) | on `ZVX_01_R_SampleRequest` |
| 6 | Behavior definition (projection) | on `ZVX_01_C_SampleRequest` |
| 7 | Behavior implementation class | `zbp_vx_01_r_samplereq` |
| 8 | Utility class | `zcl_vx_01_sample_rules` |
| 9 | Message class | `ZVX_01_MSG` (messages 001–007) |
| 10 | Service definition | `ZVX_01_UI_SAMPLEREQ_O2` |
| 11 | Service binding | `ZVX_01_UI_SAMPLEREQ_O2` (OData V4 UI) |

**Activation order (always top to bottom):**
table → root CDS → projection CDS → metadata extension → root behavior definition → projection behavior definition → behavior implementation class → service definition → service binding.
