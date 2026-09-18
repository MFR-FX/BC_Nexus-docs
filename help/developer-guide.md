---
title: "BC Nexus Developer Guide"
permalink: /help/developer-guide/
---

# BC Nexus Developer Guide

**Publisher:** fx-its | **Prefix:** FXNI | **ID Range:** 73710475–73710674  
**BC Version:** 27.0 (Runtime 16) | **Target:** Cloud (SaaS-first)

---

## Table of Contents

1. [Overview](#1-overview)
2. [Data Model Reference](#2-data-model-reference)
3. [Transaction Lifecycle](#3-transaction-lifecycle)
4. [Integration Events Reference](#4-integration-events-reference)
5. [Practical Examples](#5-practical-examples)
6. [Dependency and Naming Rules](#6-dependency-and-naming-rules)
7. [Public Codeunit API](#7-public-codeunit-api)
8. [Permission Sets](#8-permission-sets)

---

## 1. Overview

BC Nexus is a no-code data exchange layer for Business Central. It allows administrators to configure **Receive** (import), **Send** (export), and **Publish** (expose) interfaces through setup pages — no AL development is required for the common cases.

### When setup alone is sufficient

Use setup only when:

- The incoming payload maps directly to standard BC table fields.
- Field values need no transformation beyond what BC's `OnValidate` trigger provides (enable **Validate Field** on the field mapping line).
- Static default values can be expressed as plain text in the **Static Field Value** column.
- The PK lookup / upsert / always-insert behaviour built into BC Nexus covers your scenario.

### When you need an AL extension

Write a subscriber codeunit in a dependent extension when you need to:

- Set field values that derive from runtime state (current user, session variables, lookups across multiple tables).
- Transform data that cannot be expressed as a static value — for example, reformatting a date string from an external system.
- Build a custom outbound payload by querying tables that are not the interface's configured table.
- Conditionally skip processing for specific interface codes or contract versions.
- Redirect attachment storage to an external service (Azure Blob Storage, SharePoint, etc.).
- React to a completed insert or modify with side-effects such as posting, approval workflows, or telemetry.

Everything else — including authentication, HTTP transport, retry scheduling, idempotency, and JSON/CSV parsing — is handled by BC Nexus itself. Your extension only needs to implement the events that are relevant to your scenario.

---

## 2. Data Model Reference

### Table 73710475 "FXNI Nexus Setup"

Single-row configuration table. PK is always `'SETUP'`.

| Field | Type | Purpose |
|---|---|---|
| Primary Key | Code[10] | Always `'SETUP'`. |
| No. Of Retries per Transaction | Integer | How many times the job queue retries a failed transaction before setting Status to Error. |
| Auto. Process Transactions | Boolean | When true, a job queue entry (Codeunit 73710476) runs every minute to process Open transactions. |
| Enable Attachment Storage | Boolean | Master switch for attachment-mode interfaces. |
| Max Attachment Size (KB) | Integer | Maximum allowed attachment size. 0 = unlimited. |

### Table 73710477 "FXNI Nexus Interface Def."

One row per configured interface.

| Field | Type | Purpose |
|---|---|---|
| Code | Code[20] | PK. Passed to `Receive()`, `Send()`, and `Publish()` calls. |
| Description | Text[250] | Human-readable description. |
| Table No. | Integer | The BC table this interface reads from or writes to. |
| Table Name | Text[50] | Filled automatically from Table No. Read-only. |
| Interface Type | Enum "FXNI Interface Type" | `Receive`, `Send`, or `Publish`. |
| Blocked | Boolean | When true, all calls to this interface raise an error immediately. |
| Field Mapping Type | Enum "FXNI Field Mapping Type" | `JSON` or `CSV`. |
| CSV Delimiter | Enum "FXNI CSV Delimiter" | `Comma` (default), `Semicolon`, `Tab`, or `Pipe`. Column separator for a Receive interface's CSV payload. Only used when Field Mapping Type is CSV; a delimiter inside a double-quoted value does not separate columns. |
| CSV Has Header Row | Boolean | When true, the first non-empty line of the incoming CSV file is treated as column headers and skipped — it is not imported as a record. Only used when Field Mapping Type is CSV. |
| Always Create New Entries | Boolean | When true, every received transaction inserts a new record. No PK lookup is performed. |
| Entry Based Table | Boolean | For tables with auto-increment PKs (e.g. G/L Entry). Receive always inserts; Send/Publish omit the PK from the outbound payload. |
| Endpoint Code | Code[20] | FK to "FXNI Nexus Endpoint Definition". Required for Send and Publish. |
| Attachment Mode | Boolean | When true (Receive only), the transaction stores a file rather than writing to table fields. |

### Table 73710478 "FXNI Nexus Endpoint Definition"

One row per external HTTP endpoint.

| Field | Type | Purpose |
|---|---|---|
| Code | Code[20] | PK. |
| Description | Text[250] | Human-readable description. |
| Endpoint URL | Text[250] | Target URL for outbound requests. |
| Authentication Type | Enum "FXNI Authentication Type" | `None`, `Basic Auth`, or `OAuth 2.0`. |
| Client ID | Text[250] | OAuth client ID. |
| Token URL | Text[250] | OAuth token endpoint. |
| OAuth Auth Type | Enum "FXNI OAuth Auth Type" | `Client Credentials` — the only value the enum defines. Read and enforced by `AcquireOAuthToken`: an unhandled value raises an error instead of being ignored. |
| HTTP Timeout (ms) | Integer | Request timeout in ms for this endpoint, applied by `SendRequest` and `AcquireOAuthToken` alike. `0` = platform default, otherwise 1000–300000 (`OnValidate` enforces the range; `ApplyTimeout` re-checks it before use). Defaults to 30000 on new records. |
| Token Expires At | DateTime | Cached token expiry. Managed automatically. Read-only. |
| Token Request Body | Blob | JSON body sent to Token URL. Written via `SetTokenRequestBody()`. |
| Token Request Headers | Blob | Additional headers for token requests. Written via `SetTokenRequestHeaders()`. |
| Endpoint Request Headers | Blob | Additional headers added to every endpoint request. Written via `SetEndpointRequestHeaders()`. |
| Client Secret | IsolatedStorage | Set via `SetClientSecret()`. Never stored in a table field. |
| Access Token | IsolatedStorage | Managed automatically. Retrieved via `GetAccessToken()`. |

### Table 73710479 "FXNI Nexus Field Map Def."

One row per field mapping. Composite PK: `Interface Definition Code` + `Position`.

| Field | Type | Purpose |
|---|---|---|
| Interface Definition Code | Code[20] | FK to "FXNI Nexus Interface Def." |
| Position | Integer | Ordering key. For CSV, determines column position (1-based). |
| Field Mapping Type | Enum (FlowField) | Inherited from the interface definition. |
| Interface Type | Enum (FlowField) | Inherited from the interface definition. |
| Json Key | Text[50] | The JSON key name in the payload. Case-insensitive lookup. |
| Field No. | Integer | Field number in the target table. |
| Field Name | Text[50] | Filled automatically from Field No. Read-only. |
| Data Type | Text[50] | Filled automatically from Field No. Read-only. |
| Static Field Value | Text[50] | When set, this value is always written to the field — the JSON/CSV value is ignored. For Option/Enum fields, supply the integer value. |
| Mandatory | Boolean | Transaction fails if the mapped key is absent or empty in the payload. |
| Validate Field | Boolean | Calls `FieldRef.Validate()` after assignment, triggering the table field's `OnValidate`. |
| Skip If Empty | Boolean | Skips this field entirely when sending/publishing if the value is empty. |

### Table 73710480 "FXNI Nexus Transaction List"

One row per transaction, regardless of direction.

| Field | Type | Purpose |
|---|---|---|
| Entry No. | Integer | Auto-increment PK. |
| Interface Definition Code | Code[20] | Which interface owns this transaction. |
| Interface Type | Enum | Snapshot of the interface type at creation time. Preserved even if the interface is later deleted. |
| Transaction Datetime | DateTime | Set automatically on insert. |
| Status | Enum "FXNI Transaction Status" | `Open`, `Error`, `Processed`, or `Canceled`. |
| No. Of Remaining Tries | Integer | Decremented on each failure. When it reaches 0, Status becomes Error. |
| Next Try At DateTime | DateTime | Job queue only processes this transaction after this datetime. |
| Request | Blob | The raw JSON or CSV payload. Read via `GetRequest()`. |
| Response | Blob | The raw HTTP response or Publish output. Read via `GetResponse()`. |
| Error Message | Text[2048] | Last error from a failed processing attempt. |
| Correlation Id | Text[100] | Caller-provided tracing identifier. |
| Idempotency Key | Text[100] | Duplicate transactions with the same key for the same interface are suppressed. |
| External Message Id | Text[100] | Message identifier from the originating external system. |
| Contract Version | Code[20] | Caller-declared API contract version. |

### Table 73710481 "FXNI Nexus Attachment"

One row per attachment received through an attachment-mode interface.

| Field | Type | Purpose |
|---|---|---|
| Entry No. | Integer | Auto-increment PK. |
| Table No. | Integer | BC table the attachment belongs to. |
| Record Key | Text[250] | Pipe-delimited primary key of the parent record. |
| Attachment Name | Text[250] | File name or descriptive name. |
| Storage Location | Enum "FXNI Attach. Store" | `Local` or `External`. |
| Attachment Content | Blob | Binary content when Storage Location is Local. |
| Content Type | Text[100] | MIME type (e.g. `application/pdf`). |
| File Extension | Text[20] | e.g. `.pdf`, `.xlsx`. |
| File Size (Bytes) | Integer | Approximate decoded size. |
| External Reference | Text[500] | URL or storage key when Storage Location is External. |
| Interface Definition Code | Code[20] | Which interface created this attachment. |
| Transaction Entry No. | Integer | The transaction that created this attachment. |

---

## 3. Transaction Lifecycle

Understanding the lifecycle is essential for placing your subscriber code correctly.

### Receive flow

```
Caller → Codeunit 73710475 "FXNI Nexus Webservice"
  .Receive() / .ReceiveAndGetTransactionEntryNo() / .ReceiveWithMetadata()
    → OnBeforeReceive                          [event — can short-circuit]
    → Idempotency check (suppress duplicate if key matches)
    → Insert Transaction record (Status = Open)
    → OnAfterReceive                           [event]
    → Return (transaction is queued, not yet processed)

Job Queue → Codeunit 73710476 "FXNI Txn. Proc. Job"
  → Finds Open/Error transactions where Next Try At <= now
  → Codeunit.Run(73710477, TransactionRec) for each
    → OnBeforeProcessTransaction               [event — can short-circuit]
    → ValidateSetup
    → ProcessReceive:
        → Parse JSON or CSV
        → PK lookup (unless Always Create New or Entry Based)
        → If not found:
            → RecRef.Init()
            → OnBeforeInsertRecord             [event]
            → Apply field mappings
              → OnBeforeValidateField          [event — per field]
            → RecRef.Insert(true)
            → OnAfterInsertRecord              [event]
        → If found:
            → OnBeforeModifyRecord             [event]
            → Apply non-PK field mappings
              → OnBeforeValidateField          [event — per field]
            → RecRef.Modify(true)
            → OnAfterModifyRecord              [event]
    → Transaction.Status := Processed
    → OnAfterProcessTransaction                [event]
    → Commit
```

### Send flow

```
Caller → Codeunit 73710475 "FXNI Nexus Webservice"
  .Send()
    → Insert Transaction record (Status = Open)
    → Codeunit.Run(73710477, TransactionRec) — synchronous
        → OnBeforeProcessTransaction           [event — can short-circuit]
        → ValidateSetup
        → ProcessSend:
            → OnBeforeBuildSendPayload         [event — can replace payload]
            → If Payload still empty: build JSON from field mappings
            → HTTP POST to endpoint
            → Store response
        → Transaction.Status := Processed
        → OnAfterProcessTransaction            [event]
    → Return response text to caller
```

### Publish flow

The same as Send except the HTTP call is omitted: the payload is stored in `Transaction.Response` and returned to the caller. `OnBeforeBuildSendPayload` fires in the same position.

---

## 4. Integration Events Reference

All integration events use `[IntegrationEvent(false, false)]`. Neither parameter is `true`, meaning there is no global publisher and no inheritance. Subscribe with a standard `[EventSubscriber]` attribute.

BC Nexus exposes events across three codeunits. They are grouped below by codeunit.

### Namespaces and `using` directives

BC Nexus objects are organized into namespaces under `FxIts.BCNexus`. A subscriber object only sees a BC Nexus object it references — the publisher codeunit named in the `[EventSubscriber]` attribute, or a parameter type such as `Record "FXNI Nexus Transaction List"` — if it declares a matching `using` directive. This applies even when only a single type is referenced from a given namespace.

| Namespace | Contains (among others) |
|---|---|
| `FxIts.BCNexus` | FXNI Nexus Install, FXNI Nexus Upgrade, FXNI Nexus Record Key Mgt., all three permission sets |
| `FxIts.BCNexus.Setup` | Table and page "FXNI Nexus Setup", FXNI Nexus Setup Management |
| `FxIts.BCNexus.Interfaces` | "FXNI Nexus Interface Def.", "FXNI Nexus Field Map Def.", the enums "FXNI Interface Type" and "FXNI Field Mapping Type", the interface and field mapping pages |
| `FxIts.BCNexus.Transport` | "FXNI Nexus Endpoint Definition", the enums "FXNI Authentication Type" and "FXNI OAuth Auth Type", "FXNI Nexus HTTP Handler" |
| `FxIts.BCNexus.Transactions` | "FXNI Nexus Transaction List", the enum "FXNI Transaction Status", "FXNI Nexus Webservice", "FXNI Txn. Proc. Job", "FXNI Txn. Processing" |
| `FxIts.BCNexus.Attachments` | "FXNI Nexus Attachment", the enum "FXNI Attach. Store", "FXNI Attach. Processing", the ten page extensions |
| `FxIts.BCNexus.Test` | The test codeunit and the HTTP mock |

The subscriber stubs below are shown as standalone procedures, without the surrounding `using` directives of their host codeunit. Add the ones your subscriber object needs based on the table above; Section 5 shows this in full, working examples.

---

### 4.1 Codeunit 73710475 "FXNI Nexus Webservice"

#### `OnBeforeReceive`

| Attribute | Value |
|---|---|
| Fires in | `ReceiveWithMetadata()`, before any validation or transaction creation. |
| Effect of `IsHandled := true` | The entire receive call returns `true` immediately. No transaction is created. |

| Parameter | Direction | Type | Description |
|---|---|---|---|
| InterfaceCode | in | Code[20] | The interface code passed by the caller. |
| RequestData | var | Text | The raw request payload. You may modify it before the transaction is created. |
| IsHandled | var | Boolean | Set to `true` to suppress the default receive logic entirely. |

**Typical use cases:**
- Log or pre-validate payloads before BC Nexus processes them.
- Suppress processing for specific interface codes based on runtime conditions.
- Modify or canonicalize the raw payload (e.g. unwrap an envelope from a middleware).

**Subscriber stub:**
```al
[EventSubscriber(ObjectType::Codeunit, Codeunit::"FXNI Nexus Webservice", 'OnBeforeReceive', '', false, false)]
local procedure OnBeforeReceive(InterfaceCode: Code[20]; var RequestData: Text; var IsHandled: Boolean)
begin
    // Your logic here.
end;
```

---

#### `OnAfterReceive`

| Attribute | Value |
|---|---|
| Fires in | `ReceiveWithMetadata()`, after the transaction record has been inserted and the request blob written. |

| Parameter | Direction | Type | Description |
|---|---|---|---|
| InterfaceCode | in | Code[20] | The interface code. |
| TransactionRec | var | Record "FXNI Nexus Transaction List" | The newly created, unprocessed transaction. Status is `Open`. |

**Typical use cases:**
- Trigger immediate synchronous processing instead of waiting for the job queue.
- Emit telemetry after a transaction is successfully queued.
- Cross-reference the new `Entry No.` with an external audit system.

**Subscriber stub:**
```al
[EventSubscriber(ObjectType::Codeunit, Codeunit::"FXNI Nexus Webservice", 'OnAfterReceive', '', false, false)]
local procedure OnAfterReceive(InterfaceCode: Code[20]; var TransactionRec: Record "FXNI Nexus Transaction List")
begin
    // Your logic here.
end;
```

---

### 4.2 Codeunit 73710477 "FXNI Txn. Processing"

#### `OnBeforeProcessTransaction`

| Attribute | Value |
|---|---|
| Fires in | `ProcessTransaction()`, before interface validation and the type-dispatch case statement. |
| Effect of `IsHandled := true` | `ProcessTransaction()` returns `true` immediately. The transaction status is **not** updated to Processed — your subscriber is responsible for any status changes needed. |

| Parameter | Direction | Type | Description |
|---|---|---|---|
| TransactionRec | var | Record "FXNI Nexus Transaction List" | The transaction about to be processed. |
| IsHandled | var | Boolean | Set to `true` to bypass all default processing for this transaction. |

**Typical use cases:**
- Skip processing for specific interface codes or contract versions.
- Route transactions to a completely custom processing path.

**Subscriber stub:**
```al
[EventSubscriber(ObjectType::Codeunit, Codeunit::"FXNI Txn. Processing", 'OnBeforeProcessTransaction', '', false, false)]
local procedure OnBeforeProcessTransaction(var TransactionRec: Record "FXNI Nexus Transaction List"; var IsHandled: Boolean)
begin
    // Your logic here.
end;
```

---

#### `OnAfterProcessTransaction`

| Attribute | Value |
|---|---|
| Fires in | `ProcessTransaction()`, after the transaction status has been set to `Processed` and `Modify()` called. |

| Parameter | Direction | Type | Description |
|---|---|---|---|
| TransactionRec | var | Record "FXNI Nexus Transaction List" | The successfully processed transaction. Status is `Processed`. |

**Typical use cases:**
- Emit telemetry on successful processing.
- Trigger a downstream workflow (approval request, notification) after data has been written.

**Subscriber stub:**
```al
[EventSubscriber(ObjectType::Codeunit, Codeunit::"FXNI Txn. Processing", 'OnAfterProcessTransaction', '', false, false)]
local procedure OnAfterProcessTransaction(var TransactionRec: Record "FXNI Nexus Transaction List")
begin
    // Your logic here.
end;
```

---

#### `OnBeforeInsertRecord`

| Attribute | Value |
|---|---|
| Fires in | `ProcessReceive()`, after `RecRef.Init()` and before field mappings are applied to the new record. |
| When it fires | Only when a new record is being created (PK not found, or Always Create New Entries = true, or Entry Based Table = true). |

| Parameter | Direction | Type | Description |
|---|---|---|---|
| RecRef | var | RecordRef | An initialized (but not yet populated) RecordRef over the target table. You can set field values directly via `RecRef.Field(fieldNo).Value(...)`. |
| InterfaceDef | var | Record "FXNI Nexus Interface Def." | The interface definition. Use to check `InterfaceDef.Code`, `InterfaceDef."Table No."`, etc. |
| TransactionRec | var | Record "FXNI Nexus Transaction List" | The current transaction. Use `TransactionRec.GetRequest()` to access the raw payload. |

**Typical use cases:**
- Set fields that cannot be derived from the incoming payload, such as the current user's salesperson code, or a calculated default.
- Enforce company-specific defaults that are not appropriate to express in the field mapping setup.
- Read the raw payload to extract values that have no field mapping defined.

**Subscriber stub:**
```al
[EventSubscriber(ObjectType::Codeunit, Codeunit::"FXNI Txn. Processing", 'OnBeforeInsertRecord', '', false, false)]
local procedure OnBeforeInsertRecord(var RecRef: RecordRef; var InterfaceDef: Record "FXNI Nexus Interface Def."; var TransactionRec: Record "FXNI Nexus Transaction List")
begin
    // Your logic here.
end;
```

---

#### `OnAfterInsertRecord`

| Attribute | Value |
|---|---|
| Fires in | `ProcessReceive()`, immediately after `RecRef.Insert(true)` succeeds. |

| Parameter | Direction | Type | Description |
|---|---|---|---|
| RecRef | var | RecordRef | The RecordRef of the record that was just inserted. |
| InterfaceDef | var | Record "FXNI Nexus Interface Def." | The interface definition. |
| TransactionRec | var | Record "FXNI Nexus Transaction List" | The current transaction. |

**Typical use cases:**
- Post a follow-up document (e.g. release a sales order immediately after import).
- Trigger an approval workflow on the new record.
- Emit telemetry with the new record's PK.

**Subscriber stub:**
```al
[EventSubscriber(ObjectType::Codeunit, Codeunit::"FXNI Txn. Processing", 'OnAfterInsertRecord', '', false, false)]
local procedure OnAfterInsertRecord(var RecRef: RecordRef; var InterfaceDef: Record "FXNI Nexus Interface Def."; var TransactionRec: Record "FXNI Nexus Transaction List")
begin
    // Your logic here.
end;
```

---

#### `OnBeforeModifyRecord`

| Attribute | Value |
|---|---|
| Fires in | `ProcessReceive()`, after an existing record is found by PK lookup and before non-PK field mappings are applied. |
| When it fires | Only when an existing record is being updated. Never fires when Always Create New Entries = true or Entry Based Table = true. |

| Parameter | Direction | Type | Description |
|---|---|---|---|
| RecRef | var | RecordRef | A RecordRef positioned on the existing record. |
| InterfaceDef | var | Record "FXNI Nexus Interface Def." | The interface definition. |
| TransactionRec | var | Record "FXNI Nexus Transaction List" | The current transaction. |

**Typical use cases:**
- Prevent modification of specific fields on existing records (by resetting them after the field mappings run — use `OnAfterModifyRecord` for that pattern).
- Capture a before-state snapshot for audit purposes.

**Subscriber stub:**
```al
[EventSubscriber(ObjectType::Codeunit, Codeunit::"FXNI Txn. Processing", 'OnBeforeModifyRecord', '', false, false)]
local procedure OnBeforeModifyRecord(var RecRef: RecordRef; var InterfaceDef: Record "FXNI Nexus Interface Def."; var TransactionRec: Record "FXNI Nexus Transaction List")
begin
    // Your logic here.
end;
```

---

#### `OnAfterModifyRecord`

| Attribute | Value |
|---|---|
| Fires in | `ProcessReceive()`, immediately after `RecRef.Modify(true)` succeeds on an existing record. |

| Parameter | Direction | Type | Description |
|---|---|---|---|
| RecRef | var | RecordRef | The RecordRef of the record that was just modified. |
| InterfaceDef | var | Record "FXNI Nexus Interface Def." | The interface definition. |
| TransactionRec | var | Record "FXNI Nexus Transaction List" | The current transaction. |

**Typical use cases:**
- Trigger post-modify business rules (re-calculate totals, update related records).
- Emit a change log entry with the new state.

**Subscriber stub:**
```al
[EventSubscriber(ObjectType::Codeunit, Codeunit::"FXNI Txn. Processing", 'OnAfterModifyRecord', '', false, false)]
local procedure OnAfterModifyRecord(var RecRef: RecordRef; var InterfaceDef: Record "FXNI Nexus Interface Def."; var TransactionRec: Record "FXNI Nexus Transaction List")
begin
    // Your logic here.
end;
```

---

#### `OnBeforeValidateField`

| Attribute | Value |
|---|---|
| Fires in | `ApplyJsonFieldToRecord()` and `ApplyCsvValueToField()`, once per field mapping line, before the value is assigned to the FieldRef. |
| Fires for static values | Yes. When `Static Field Value` is set, the event fires with the static value as `Value` before it is assigned. |
| Fires when Skip If Empty applies | No. If the value is empty and Skip If Empty = true, the function returns before reaching the event. |

| Parameter | Direction | Type | Description |
|---|---|---|---|
| RecRef | var | RecordRef | The RecordRef being populated. You can read other fields on the record at this point. |
| FieldMapping | - | Record "FXNI Nexus Field Map Def." | The field mapping line currently being processed, passed as a copy. Use `FieldMapping."Field No."`, `FieldMapping."Json Key"`, etc. to identify which field is being processed. |
| Value | var | Text | The string value about to be assigned. Modify this to transform the value before assignment. |

`Value` is the only channel back into the assignment: the target `FieldRef` and the text to assign are resolved before this event fires, so changes made to `FieldMapping` — for example writing to `FieldMapping."Static Field Value"` — never reach the target field. That is why the parameter is passed as a copy rather than `var`: a subscriber that declares it as `var` fails to compile instead of silently having no effect.

**Typical use cases:**
- Transform date or number formats from an external system to the format BC's `Evaluate()` expects.
- Apply conditional lookups (e.g. translate an external code to a BC code).
- Mask or truncate values before they reach the field.

**Subscriber stub:**
```al
[EventSubscriber(ObjectType::Codeunit, Codeunit::"FXNI Txn. Processing", 'OnBeforeValidateField', '', false, false)]
local procedure OnBeforeValidateField(var RecRef: RecordRef; FieldMapping: Record "FXNI Nexus Field Map Def."; var Value: Text)
begin
    // Your logic here.
end;
```

---

#### `OnBeforeBuildSendPayload`

| Attribute | Value |
|---|---|
| Fires in | `ProcessSend()`, before the default JSON array is built from the field mappings. |
| Effect of setting Payload | If `Payload` is non-empty after the event, the default payload-building loop is skipped entirely. The value you set is sent directly to the endpoint. |

| Parameter | Direction | Type | Description |
|---|---|---|---|
| InterfaceDef | var | Record "FXNI Nexus Interface Def." | The interface definition. Use to read `InterfaceDef."Table No."` and `InterfaceDef.Code`. |
| TransactionRec | var | Record "FXNI Nexus Transaction List" | The current transaction. |
| Payload | var | Text | Initially empty. Set this to a non-empty string to replace the default payload. |

**Typical use cases:**
- Build a payload that joins data from multiple tables (the configured table plus related tables).
- Produce a payload format that does not match the BC Nexus JSON array convention (e.g. a SOAP envelope, or a vendor-specific schema).
- Filter which records are included in the payload at runtime.

**Subscriber stub:**
```al
[EventSubscriber(ObjectType::Codeunit, Codeunit::"FXNI Txn. Processing", 'OnBeforeBuildSendPayload', '', false, false)]
local procedure OnBeforeBuildSendPayload(var InterfaceDef: Record "FXNI Nexus Interface Def."; var TransactionRec: Record "FXNI Nexus Transaction List"; var Payload: Text)
begin
    // Your logic here.
end;
```

---

### 4.3 Codeunit 73710481 "FXNI Attach. Processing"

These events are relevant only for attachment-mode Receive interfaces.

#### `OnBeforeCreateAttachment`

| Attribute | Value |
|---|---|
| Fires in | `ProcessSingleAttachment()`, before the `"FXNI Nexus Attachment"` record is created. |
| Effect of `IsHandled := true` | The default attachment record creation and content storage are skipped entirely. |

| Parameter | Direction | Type | Description |
|---|---|---|---|
| InterfaceDef | var | Record "FXNI Nexus Interface Def." | The interface definition. |
| TransactionRec | var | Record "FXNI Nexus Transaction List" | The current transaction. |
| RecordKeyValue | var | Text | The `recordKey` value from the payload. |
| AttachmentName | var | Text | The `attachmentName` value from the payload. |
| Base64Content | var | Text | The base64-encoded file content. |
| IsHandled | var | Boolean | Set to `true` to replace default processing. |

#### `OnAfterCreateAttachment`

| Attribute | Value |
|---|---|
| Fires in | `ProcessSingleAttachment()`, after the attachment record has been created and content stored. |

| Parameter | Direction | Type | Description |
|---|---|---|---|
| InterfaceDef | var | Record "FXNI Nexus Interface Def." | The interface definition. |
| TransactionRec | var | Record "FXNI Nexus Transaction List" | The current transaction. |
| Attachment | var | Record "FXNI Nexus Attachment" | The newly created attachment record. |

#### `OnBeforeStoreContent`

| Attribute | Value |
|---|---|
| Fires in | `StoreAttachmentContent()`, before the base64 content is decoded and written to the local blob. |
| Effect of `IsHandled := true` | The local blob write is skipped. Your subscriber is responsible for storing the content and setting `Attachment."Storage Location"` and `Attachment."External Reference"` accordingly. |

| Parameter | Direction | Type | Description |
|---|---|---|---|
| Attachment | var | Record "FXNI Nexus Attachment" | The attachment record (not yet modified with content). |
| Base64Content | var | Text | The raw base64-encoded content. |
| IsHandled | var | Boolean | Set to `true` to replace default local storage. |

**Typical use case:** Route attachment content to Azure Blob Storage, SharePoint, or any other external file service. Set `Attachment."Storage Location"` to `External` and `Attachment."External Reference"` to the URL or storage key returned by the external service.

#### `OnBeforeRetrieveContent`

| Attribute | Value |
|---|---|
| Fires in | `RetrieveAttachmentContent()`, before the content is read from the local blob. |
| Effect of `IsHandled := true` | The local blob read is skipped. Your subscriber must provide a valid `InStr` pointing to the content. |

| Parameter | Direction | Type | Description |
|---|---|---|---|
| Attachment | var | Record "FXNI Nexus Attachment" | The attachment record. Check `Attachment."Storage Location"` to decide whether to intervene. |
| InStr | var | InStream | Provide a valid InStream over the content. |
| IsHandled | var | Boolean | Set to `true` to replace the default local read. |

---

## 5. Practical Examples

### 5.1 Set Salesperson Code from current user on incoming sales header records

**Scenario:** An external system sends sales orders to the `SALESORDERIMPORT` interface. The Salesperson Code is not in the payload — it must be set to the salesperson linked to the current BC user.

**Why `OnBeforeInsertRecord`:** This event fires after `RecRef.Init()` but before field mappings are applied, so any value set here is in place before standard mapping runs. If the field mapping also includes a Salesperson Code mapping, that mapping will overwrite what you set here, because field mappings run after this event. To guarantee your value wins, use `OnAfterInsertRecord` followed by a `RecRef.Modify(true)`, or ensure there is no field mapping for that field.

```al
using FxIts.BCNexus.Interfaces;
using FxIts.BCNexus.Transactions;

codeunit 50200 "MyExt Sales Order Subscribers"
{
    [EventSubscriber(ObjectType::Codeunit, Codeunit::"FXNI Txn. Processing", 'OnBeforeInsertRecord', '', false, false)]
    local procedure SetSalespersonOnSalesHeaderInsert(
        var RecRef: RecordRef;
        var InterfaceDef: Record "FXNI Nexus Interface Def.";
        var TransactionRec: Record "FXNI Nexus Transaction List")
    var
        SalespersonPurchaser: Record "Salesperson/Purchaser";
        UserSetup: Record "User Setup";
        FRef: FieldRef;
        SalespersonCodeFieldNo: Integer;
    begin
        // Only act on the specific interface and Sales Header table (Table 36)
        if InterfaceDef.Code <> 'SALESORDERIMPORT' then
            exit;
        if InterfaceDef."Table No." <> Database::"Sales Header" then
            exit;

        // Field 13 is "Salesperson Code" on Sales Header
        SalespersonCodeFieldNo := 13;
        if not RecRef.FieldExist(SalespersonCodeFieldNo) then
            exit;

        // Look up the salesperson linked to the current user in User Setup
        if not UserSetup.Get(UserId()) then
            exit;
        if UserSetup."Salespers./Purch. Code" = '' then
            exit;

        FRef := RecRef.Field(SalespersonCodeFieldNo);
        FRef.Value(UserSetup."Salespers./Purch. Code");
    end;
}
```

**Permission requirement:** Your extension must have at least `tabledata "User Setup" = R` in its permission set.

---

### 5.2 Transform an external date format in `OnBeforeValidateField`

**Scenario:** An external WMS sends dates as `YYYYMMDD` (e.g. `20260115`). BC's `Evaluate()` for a Date field expects `DD.MM.YYYY` (locale-dependent) or ISO 8601 (`2026-01-15`). The `WMSSHIPMENT` interface maps the `shipDate` JSON key to the `Shipment Date` field on the Sales Header.

**Why `OnBeforeValidateField`:** The event gives you the raw string value from the JSON payload before BC Nexus attempts to `Evaluate()` it into the field type. Replacing the value here is the correct point — there is no code between this event and `AssignFieldRefValue()`.

```al
using FxIts.BCNexus.Interfaces;
using FxIts.BCNexus.Transactions;

codeunit 50201 "MyExt WMS Field Transforms"
{
    [EventSubscriber(ObjectType::Codeunit, Codeunit::"FXNI Txn. Processing", 'OnBeforeValidateField', '', false, false)]
    local procedure TransformWmsDateFormat(
        var RecRef: RecordRef;
        FieldMapping: Record "FXNI Nexus Field Map Def.";
        var Value: Text)
    var
        Year: Text;
        Month: Text;
        Day: Text;
    begin
        // Only act on the WMS shipment interface
        if FieldMapping."Interface Definition Code" <> 'WMSSHIPMENT' then
            exit;

        // Only act on the shipDate key
        if FieldMapping."Json Key" <> 'shipDate' then
            exit;

        // Only transform values that look like YYYYMMDD (8 numeric characters)
        if StrLen(Value) <> 8 then
            exit;
        if not (Value[1] in ['1', '2']) then
            exit;

        Year  := CopyStr(Value, 1, 4);
        Month := CopyStr(Value, 5, 2);
        Day   := CopyStr(Value, 7, 2);

        // Produce ISO 8601 format, which BC's Evaluate() handles reliably
        Value := Year + '-' + Month + '-' + Day;
    end;
}
```

**Important:** `OnBeforeValidateField` fires for every field on every transaction processed by any interface. Always guard with an interface code check and a field key check to avoid unintended side effects. Keep the procedure fast — it is called in a loop.

---

### 5.3 Build a custom Send payload combining data from multiple tables

**Scenario:** The `CUSTOMEREXPORT` Send interface is configured for the Customer table. The external system requires a JSON object per customer that includes open invoice count — a value from `Cust. Ledger Entry` that is not a plain field on the Customer table. The standard field-mapping mechanism covers only fields from the configured table.

**Why `OnBeforeBuildSendPayload`:** Setting `Payload` to a non-empty value causes BC Nexus to skip its default payload-building loop entirely and send your value directly to the endpoint.

```al
using FxIts.BCNexus.Interfaces;
using FxIts.BCNexus.Transactions;

codeunit 50202 "MyExt Customer Export Payload"
{
    [EventSubscriber(ObjectType::Codeunit, Codeunit::"FXNI Txn. Processing", 'OnBeforeBuildSendPayload', '', false, false)]
    local procedure BuildCustomerExportPayload(
        var InterfaceDef: Record "FXNI Nexus Interface Def.";
        var TransactionRec: Record "FXNI Nexus Transaction List";
        var Payload: Text)
    var
        Customer: Record Customer;
        CustLedgerEntry: Record "Cust. Ledger Entry";
        JsonArray: JsonArray;
        JsonObj: JsonObject;
        ResultText: Text;
        OpenInvoiceCount: Integer;
    begin
        if InterfaceDef.Code <> 'CUSTOMEREXPORT' then
            exit;

        if not Customer.FindSet() then begin
            JsonArray.WriteTo(Payload);
            exit;
        end;

        repeat
            Clear(JsonObj);

            JsonObj.Add('customerNo', Customer."No.");
            JsonObj.Add('name', Customer.Name);
            JsonObj.Add('currencyCode', Customer."Currency Code");

            // Count open invoices from Cust. Ledger Entry
            CustLedgerEntry.SetRange("Customer No.", Customer."No.");
            CustLedgerEntry.SetRange("Document Type", CustLedgerEntry."Document Type"::Invoice);
            CustLedgerEntry.SetRange(Open, true);
            OpenInvoiceCount := CustLedgerEntry.Count();
            JsonObj.Add('openInvoiceCount', OpenInvoiceCount);

            JsonArray.Add(JsonObj);
        until Customer.Next() = 0;

        JsonArray.WriteTo(Payload);
    end;
}
```

**Performance note:** `CustLedgerEntry.Count()` inside the Customer loop produces one database round-trip per customer. For large datasets, build a temporary table indexed by customer number before the loop and look up the count from there. Do not call `CalcFields` inside the loop.

---

### 5.4 Skip specific interface codes using `IsHandled`

**Scenario:** During a maintenance window, processing of the `VENDORIMPORT` interface must be suspended without blocking the interface (which would prevent new transactions from being queued). Transactions should be left in `Open` status to be processed later.

**Why `OnBeforeProcessTransaction`:** Setting `IsHandled := true` causes `ProcessTransaction()` to return `true` immediately without changing the transaction status. The transaction remains `Open` and the job queue will attempt it again on the next run.

```al
using FxIts.BCNexus.Transactions;

codeunit 50203 "MyExt Maintenance Gate"
{
    [EventSubscriber(ObjectType::Codeunit, Codeunit::"FXNI Txn. Processing", 'OnBeforeProcessTransaction', '', false, false)]
    local procedure SkipDuringMaintenanceWindow(
        var TransactionRec: Record "FXNI Nexus Transaction List";
        var IsHandled: Boolean)
    var
        MySetup: Record "MyExt Setup";
    begin
        if TransactionRec."Interface Definition Code" <> 'VENDORIMPORT' then
            exit;

        // Read maintenance mode flag from a setup table in your own extension
        if not MySetup.Get() then
            exit;

        if MySetup."Maintenance Mode" then
            IsHandled := true;
        // IsHandled = true → ProcessTransaction returns true, transaction stays Open
    end;
}
```

**What IsHandled does and does not do:** When `IsHandled := true` is set in `OnBeforeProcessTransaction`, `ProcessTransaction()` returns `true` (no error) but does **not** set the transaction status to `Processed`. The transaction retains its current status (`Open` or `Error`). The job queue will pick it up again on its next run — typically within one minute — and call the event again. If your maintenance window lasts longer than the retry count multiplied by the retry interval, transactions will exhaust their remaining tries and move to `Error` status. Design accordingly.

---

## 6. Dependency and Naming Rules

### Declaring BC Nexus as a dependency

In your extension's `app.json`, add BC Nexus to the `"dependencies"` array using its exact `id`, `publisher`, `name`, and a minimum version:

```json
{
  "id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "name": "My Extension",
  "publisher": "My Publisher",
  "version": "1.0.0.0",
  "dependencies": [
    {
      "id": "b7749b8c-bd78-496d-b8b0-0d23ce1e94f9",
      "publisher": "fx-its",
      "name": "BC Nexus",
      "version": "27.0.0.0"
    }
  ]
}
```

The `id` value `b7749b8c-bd78-496d-b8b0-0d23ce1e94f9` is the BC Nexus app ID from its `app.json`. Do not change this value.

### ID range

BC Nexus occupies IDs **73710475–73710674**. Your dependent extension must use an ID range that does not overlap. If you are publishing to AppSource, you must use a Microsoft-assigned range. For partner/customer extensions, use any non-conflicting range above 50000 that is registered to your publisher.

### Object and field naming

- Use your own prefix, not `FXNI`. The `FXNI` prefix is reserved for BC Nexus objects.
- Do not create table extensions on BC Nexus tables unless you have a clear technical need. BC Nexus tables are not part of the stable public API surface — their field structure may change between major versions.
- Do not call `local procedure` members on BC Nexus codeunits. Only `procedure` (public) members are part of the supported API. See Section 7 for the full list.

### Permission sets in your extension

Your extension's permission sets must include permissions for the BC Nexus objects that your code accesses. At minimum, a subscriber codeunit that reads `"FXNI Nexus Interface Def."` needs:

```al
using FxIts.BCNexus.Interfaces;
using FxIts.BCNexus.Transactions;

permissionset 60100 "MyExt Permissions"
{
    Assignable = true;
    Permissions =
        tabledata "FXNI Nexus Interface Def." = R,
        tabledata "FXNI Nexus Transaction List" = R,
        codeunit "FXNI Txn. Processing" = X;
}
```

If your subscribers also read `"FXNI Nexus Field Map Def."`, add `R` access to that table as well.

### File naming convention

Follow BC Nexus conventions for file names:

- One object per file.
- File name format: `<ObjectName><Type>.al`, with the object name stripped of spaces and
  punctuation — `MyExtSalesOrderSubscribers.Codeunit.al`, `MyExtSetup.Table.al`. CodeCop rule
  AA0215 enforces this and names the expected file for you, so the compiler is the authority.
- Group files into folders that mirror your namespaces rather than folders per object type.
  BC Nexus itself is laid out that way — see the namespace table in section 4.

---

## 7. Public Codeunit API

Only `procedure` declarations (without the `local` modifier) are part of the supported public API. Calling `local procedure` members from outside BC Nexus is not supported and may break without notice.

### Codeunit 73710475 "FXNI Nexus Webservice"

| Procedure | Signature | Description |
|---|---|---|
| Receive | `procedure Receive(InterfaceCode: Code[20]; RequestData: Text): Boolean` | Queues a Receive transaction. Returns `true` on success. |
| ReceiveAndGetTransactionEntryNo | `procedure ReceiveAndGetTransactionEntryNo(InterfaceCode: Code[20]; RequestData: Text; var TransactionEntryNo: Integer): Boolean` | Same as `Receive()` but also returns the new transaction's Entry No. via an out-parameter. |
| ReceiveWithMetadata | `procedure ReceiveWithMetadata(InterfaceCode: Code[20]; RequestData: Text; CorrelationId: Text[100]; IdempotencyKey: Text[100]; ExternalMessageId: Text[100]; ContractVersion: Code[20]; var TransactionEntryNo: Integer): Boolean` | Full-featured receive call. Supports idempotency key deduplication and metadata fields. |
| ReceiveAttachment | `procedure ReceiveAttachment(InterfaceCode: Code[20]; RecordKey: Text; AttachmentName: Text; ContentType: Text; FileExtension: Text; Base64Content: Text): Boolean` | Convenience method for attachment-mode interfaces. Builds the required JSON payload and calls `Receive()`. |
| Send | `procedure Send(InterfaceCode: Code[20]): Text` | Creates and immediately processes a Send transaction. Returns the endpoint response text. Errors if the endpoint call fails. |
| Publish | `procedure Publish(InterfaceCode: Code[20]; RequestData: Text): Text` | Creates and immediately processes a Publish transaction. Returns the generated payload text. |

**Local procedures (not callable from outside):** `FindExistingByIdempotencyKey`, `OnBeforeReceive`, `OnAfterReceive`.

---

### Codeunit 73710476 "FXNI Txn. Proc. Job"

| Procedure | Signature | Description |
|---|---|---|
| ProcessOneTransaction | `procedure ProcessOneTransaction(var TransactionRec: Record "FXNI Nexus Transaction List")` | Processes exactly one already-positioned transaction and applies the resulting retry/error state transition — decrement the remaining tries, move to `Error` once exhausted, otherwise schedule the next attempt — persisting the result with `Modify()`. Contains **no** `Commit()`: the `Commit()` belongs to `OnRun`'s polling loop, so that one failing transaction does not roll back the successes of the same run. That makes this procedure safe to call from inside an ambient transaction, which `OnRun` itself is not. |

**Trigger:** `OnRun` selects due transactions (Status `Open` or `Error`, due at or before now, tries remaining) with `FindSet(false)` and calls `ProcessOneTransaction` for each. It commits **before** every record — not after — and once more after the loop. Committing first is what lets `ProcessOneTransaction` evaluate a `Codeunit.Run` return value: the AL runtime allows that only when no write transaction is open, and the error branch's `Modify()` of the previous record leaves one. The trailing commit exists because the last record is the only one the loop never commits.

---

### Codeunit 73710477 "FXNI Txn. Processing"

| Procedure | Signature | Description |
|---|---|---|
| ProcessTransaction | `procedure ProcessTransaction(var TransactionRec: Record "FXNI Nexus Transaction List"): Boolean` | Processes a single transaction record. Called by the job queue (Codeunit 73710476) and by `Send()` / `Publish()`. Safe to call directly in tests or from custom job logic. |
| ValidateSetup | `procedure ValidateSetup(var InterfaceDef: Record "FXNI Nexus Interface Def."; var TransactionRec: Record "FXNI Nexus Transaction List")` | Validates that the interface definition is complete and not blocked. Raises an error on any misconfiguration. |
| ProcessReceive | `procedure ProcessReceive(var InterfaceDef: Record "FXNI Nexus Interface Def."; var TransactionRec: Record "FXNI Nexus Transaction List")` | Executes the Receive flow: parses the request, performs PK lookup, and inserts or modifies the target record. |
| ProcessSend | `procedure ProcessSend(var InterfaceDef: Record "FXNI Nexus Interface Def."; var TransactionRec: Record "FXNI Nexus Transaction List")` | Executes the Send flow: builds the payload and HTTP-POSTs it to the endpoint. |
| ProcessPublish | `procedure ProcessPublish(var InterfaceDef: Record "FXNI Nexus Interface Def."; var TransactionRec: Record "FXNI Nexus Transaction List")` | Executes the Publish flow: builds the payload and stores it in the transaction response. |
| ParseJsonToFieldRef | `procedure ParseJsonToFieldRef(JsonKey: Text; JsonData: JsonObject; var FRef: FieldRef; DataType: Text)` | Utility: resolves a JSON key (case-insensitive) and assigns its value to the provided FieldRef. Useful in subscriber code that manually parses the request payload. |
| BuildJsonFromRecord | `procedure BuildJsonFromRecord(var RecRef: RecordRef; InterfaceCode: Code[20]): Text` | Utility: serializes a record to a JSON object string using the field mappings **of the given interface**, which it resolves itself. Useful in `OnBeforeBuildSendPayload` subscribers that want to build on top of the standard serialization. The interface code is a mandatory parameter rather than a pre-filtered record: the earlier signature took `var FieldMappings: Record` and discarded the caller's filter internally, so a payload could pick up another interface's JSON keys. |
| SplitCsvRecords | `procedure SplitCsvRecords(RequestText: Text; Delimiter: Char; var RecordLineNos: List of [Integer]): List of [Text]` | Utility: splits a raw CSV payload into logical records, respecting RFC 4180 quoting — a line break inside a quoted value does not end a record. Returns the record texts; `RecordLineNos` receives, for each returned record, the 1-based physical source line it starts on (header row and blank lines counted). |
| SplitCsvColumns | `procedure SplitCsvColumns(RecordText: Text; Delimiter: Char): List of [Text]` | Utility: splits one logical CSV record (as returned by `SplitCsvRecords`) into its columns, following RFC 4180 — a quoted value may contain the delimiter, doubled quotation marks (`""`) resolve to one literal quotation mark, quoted values keep their surrounding whitespace, unquoted values are trimmed. |

**Local procedures (not callable from outside):** `ProcessReceiveCSV`, `GetPrimaryKeyFieldNos`, `FindExistingRecordJSON`, `FindExistingRecordCSV`, `ApplyJsonFieldMappings`, `ApplyCsvFieldMappings`, `ApplyJsonFieldToRecord`, `ApplyCsvValueToField`, `TryGetJsonToken`, `AssignFieldRefValue`, `TryBase64ToTempBlob`, `BuildJsonFromRecordSkipPK`, and all `On*` event procedures.

---

### Codeunit 73710478 "FXNI Nexus HTTP Handler"

| Procedure | Signature | Description |
|---|---|---|
| SendRequest | `procedure SendRequest(var EndpointDef: Record "FXNI Nexus Endpoint Definition"; Method: Text; Body: Text; var ResponseText: Text; var HttpStatusCode: Integer): Boolean` | Sends an HTTP request to the endpoint. Handles authentication header injection. Returns `true` if the response is a 2xx status. |
| AcquireOAuthToken | `procedure AcquireOAuthToken(var EndpointDef: Record "FXNI Nexus Endpoint Definition"): Text` | Acquires or returns a cached OAuth token. Handles token expiry. Stores the token in IsolatedStorage via the endpoint table methods. |
| BuildBasicAuthHeader | `procedure BuildBasicAuthHeader(var EndpointDef: Record "FXNI Nexus Endpoint Definition"): SecretText` | Returns a base64-encoded `Basic <credentials>` header value for the endpoint. Returns `SecretText`, not `Text`, so the credentials never materialize as a plain string. |
| TestConnection | `procedure TestConnection(var EndpointDef: Record "FXNI Nexus Endpoint Definition"): Boolean` | Issues a GET request to the endpoint URL and returns true if the response is 2xx. Used by the Setup page to verify connectivity. |

**Local procedures:** `ParseAndAddHeaders`, `OnBeforeSendRequest`, `OnAfterSendRequest`, `OnBeforeAcquireToken`.

---

### Codeunit 73710479 "FXNI Nexus Setup Management"

| Procedure | Signature | Description |
|---|---|---|
| InitializeSetup | `procedure InitializeSetup()` | Creates the setup record with default values if it does not already exist. |
| CreateWebServiceEntry | `procedure CreateWebServiceEntry()` | Publishes the BC Nexus webservice entry. Raises events before and after for extensibility. |
| CreateJobQueueEntry | `procedure CreateJobQueueEntry()` | Creates the recurring job queue entry for Codeunit 73710476 if Auto. Process Transactions is enabled. |
| DeleteJobQueueEntry | `procedure DeleteJobQueueEntry()` | Removes all job queue entries for Codeunit 73710476. |
| LoadSampleData | `procedure LoadSampleData()` | Inserts the built-in sample endpoint and Currency import interface. Safe to call multiple times — uses `Get()` guards. |

---

### Codeunit 73710481 "FXNI Attach. Processing"

| Procedure | Signature | Description |
|---|---|---|
| ProcessAttachmentReceive | `procedure ProcessAttachmentReceive(var InterfaceDef: Record "FXNI Nexus Interface Def."; var TransactionRec: Record "FXNI Nexus Transaction List")` | Entry point for attachment-mode receive processing. Accepts a single JSON object or a JSON array of attachment objects. |
| StoreAttachmentContent | `procedure StoreAttachmentContent(var Attachment: Record "FXNI Nexus Attachment"; Base64Content: Text)` | Decodes base64 content and writes it to the attachment's local blob. Raises `OnBeforeStoreContent` so external storage providers can intercept. |
| RetrieveAttachmentContent | `procedure RetrieveAttachmentContent(var Attachment: Record "FXNI Nexus Attachment"; var InStr: InStream)` | Provides an InStream over the attachment content. Raises `OnBeforeRetrieveContent` so external storage providers can intercept. |
| CheckExternalStorageComplete | `procedure CheckExternalStorageComplete(var Attachment: Record "FXNI Nexus Attachment")` | Refuses an attachment whose Storage Location is External but whose External Reference is empty. Called after an `OnBeforeStoreContent` subscriber has handled the content, so a subscriber that moves a file out of Business Central without naming where it went fails loudly instead of leaving an unreadable attachment behind. |
| TryRetrieveExternalContent | `procedure TryRetrieveExternalContent(var Attachment: Record "FXNI Nexus Attachment"; var InStr: InStream): Boolean` | Fires `OnBeforeRetrieveContent` and reports whether a subscriber actually supplied the stream, instead of silently falling back to the local blob. `RetrieveAttachmentContent` delegates here, so the event is still raised in exactly one place. Call it when "no external handler installed" has to become a decision — for an externally stored attachment there is no local blob to fall back to. |

---

## 8. Permission Sets

BC Nexus ships with three assignable permission sets. The split follows one rule: whoever may execute the codeunit that resolves an endpoint's stored credentials can use those credentials to call any Send interface, whether or not they can ever see the credential itself in a table. The interactive business user and the technical account a foreign system authenticates as are therefore two separate roles, not one role with different table rights. Credentials live in Isolated Storage — no permission set governs access to that storage directly; codeunit execute rights are the only lever.

### FXNI Nexus Admin (Per 73710475)

Full RIMD access to all BC Nexus configuration and transaction tables, plus execute rights on all BC Nexus codeunits and pages. The only set that can execute `FXNI Nexus Setup Management` and page `FXNI Set Secret Dialog`, and therefore the only set that can register or replace endpoint credentials, publish the web service, create the job queue entry, or load the sample interface. Assign to the BC Nexus administrator user.

### FXNI Nexus User (Per 73710476)

Read-only access to configuration tables. Read and modify access to the transaction list (allowing users to cancel or reset transactions). No codeunit execute rights at all: this set cannot call `FXNI Nexus Webservice`, `FXNI Txn. Processing`, `FXNI Attach. Processing` or `FXNI Nexus HTTP Handler`, so it cannot run any interface — Send, Receive or Publish — and the "Process Manually" action on the transaction pages fails for it with a permission error by design. A failed transaction is retried with "Reset Status" instead, which only writes the journal row; the job queue then reprocesses the transaction under its own account. Assign to users who monitor integrations but do not configure them or trigger interfaces themselves.

### FXNI Nexus Integr. (Per 73710477, caption "BC Nexus Integration")

The technical account a foreign system authenticates as when it delivers data into BC Nexus or pulls a Publish interface — an S2S application registration or a dedicated service user, never a person signing in interactively. (The object name is abbreviated because a permission set identifier cannot exceed 20 characters; the caption is what an administrator sees when assigning it.) Executes the web service entry point, the processing codeunits and the HTTP handler, and creates and updates transaction and attachment records. No page permissions at all — this account has no user interface — and no write access to any configuration table, so it cannot register or change endpoints, redirect an endpoint URL, or touch credentials; those remain with `FXNI Nexus Admin`.

### Permission requirements for dependent extensions

When your extension's subscriber codeunits read BC Nexus tables, include those table permissions in your own permission set. Do not rely on the user already holding `FXNI Nexus Admin` — your extension should declare its own minimal permissions so it works correctly regardless of which BC Nexus permission set the user holds.
