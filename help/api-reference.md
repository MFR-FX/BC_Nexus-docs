---
title: "BC Nexus — API Reference"
permalink: /help/api-reference/
---

# BC Nexus — API Reference

_Web service contract for Codeunit 50100 "FXNI Nexus Webservice"._

External systems interact with BC Nexus through standard BC web service mechanisms (OData v4 or SOAP). Before any external call can reach BC Nexus, you must manually publish Codeunit 50100 on the BC **Web Services** page — this step is not performed automatically on install (BC Online does not allow programmatic web service registration from within an extension).

---

## Authentication

All calls to the BC Nexus web service use standard BC authentication:
- **Production:** OAuth 2.0 via Microsoft Entra (App Registration)
- **Sandbox / Dev:** Basic authentication

BC Nexus does not manage inbound authentication — this is fully delegated to BC's own web service layer.

---

## Web Service Endpoint

### Manual publish (required after every install / reinstall)

Codeunit 50100 must be published manually on the BC **Web Services** page before any external call will work:

1. Open the **Web Services** page in BC (search for "Web Services")
2. Choose **New**
3. Set **Object Type** to `Codeunit`
4. Set **Object ID** to `50100`
5. Set **Service Name** to `FXNINexusWebservice`
6. Enable the **Published** checkbox

The row will appear in the list with a generated OData URL. Repeat this step in every environment (sandbox, production) and after any reinstall that removes the existing entry.

### Base URL

Base URL pattern (BC Online):
```
https://api.businesscentral.dynamics.com/v2.0/{tenantId}/{environment}/ODataV4/Company('{companyName}')/FXNINexusWebservice
```

### The parameter names on the wire are not the AL parameter names

OData exposes an AL parameter with its **first letter lowercased**. The AL signature
`Receive(InterfaceCode: Code[20]; RequestData: Text)` is therefore called as:

```json
{ "interfaceCode": "CURRENCYIMPORT", "requestData": "{...}" }
```

`"InterfaceCode"` answers `HTTP 400 BadRequest` with *"The parameter 'InterfaceCode' in the
request payload is not a valid parameter for the operation 'FXNINexusWebservice_Receive'."*
The AL signatures below are written the way AL spells them, because that is what an AL caller
uses; every one of them is lowercased at the first letter on the wire.

### Not every procedure below is reachable over OData

**A procedure with a `var` parameter is not exposed at all.** OData has no way to express an
output parameter on an unbound action, so BC leaves such procedures out of the metadata
entirely — there is no error to see, the operation simply does not exist.

| OData action | Parameters on the wire |
|---|---|
| `FXNINexusWebservice_Receive` | `interfaceCode`, `requestData` |
| `FXNINexusWebservice_Send` | `interfaceCode` |
| `FXNINexusWebservice_SendWithOptions` | `interfaceCode`, `requestData` |
| `FXNINexusWebservice_Publish` | `interfaceCode`, `requestData` |
| `FXNINexusWebservice_ReceiveAttachment` | `interfaceCode`, `recordKey`, `attachmentName`, `contentType`, `fileExtension`, `base64Content` |

The first four rows were verified against the published service's `$metadata`.
`SendWithOptions` is new in this version and has **not** been read back from a running
environment yet — it carries value parameters only, which is what keeps a procedure in the
metadata, but treat the row as expected rather than confirmed until `$metadata` says so.

**`ReceiveAndGetTransactionEntryNo` and `ReceiveWithMetadata` are AL-only.** Both carry
`var TransactionEntryNo: Integer`. They are documented below because another extension can call
them, and an external system cannot — such a caller uses `Receive` and correlates through its
own identifiers instead.

To see the list for a specific environment rather than trusting this table, run the
**BC Webservice Smoke** workflow: its first step prints every action of the published service
with the parameter names that environment expects.

---

## Functions

### Receive

Queues incoming data for asynchronous processing.

```
PROCEDURE Receive(InterfaceCode: Code[20]; RequestData: Text): Boolean
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| InterfaceCode | Code[20] | The Code of a configured Receive Interface Definition |
| RequestData | Text | The raw JSON or CSV payload to import |

**Returns:** `true` on success.

**Behavior:**
- Creates a new Transaction record (Status = Open)
- Stores RequestData in the transaction blob
- Returns immediately — actual processing happens asynchronously via job queue
- If IdempotencyKey is provided (via `ReceiveWithMetadata`), duplicate calls with the same key return the existing transaction entry number

**Errors:**
- Interface not found → Error "Interface definition 'X' does not exist."
- Interface blocked → Error "Interface 'X' is blocked and cannot process requests."

---

### ReceiveAndGetTransactionEntryNo

Receive that also hands back the entry number of the transaction it created.

```
PROCEDURE ReceiveAndGetTransactionEntryNo(
    InterfaceCode: Code[20];
    RequestData: Text;
    var TransactionEntryNo: Integer
): Boolean
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| InterfaceCode | Code[20] | The Code of a configured Receive Interface Definition |
| RequestData | Text | The raw JSON or CSV payload to import |
| TransactionEntryNo | Integer (var) | Returns the Entry No. of the created transaction |

**Returns:** `true` on success.

Identical to `Receive` in every other respect — it is `ReceiveWithMetadata` with the four tracing
fields left empty. Use it when the caller wants to poll the transaction it just queued without
having to supply an idempotency key or a correlation ID.

---

### ReceiveWithMetadata

Extended version of Receive with cross-system tracing fields.

```
PROCEDURE ReceiveWithMetadata(
    InterfaceCode: Code[20];
    RequestData: Text;
    CorrelationId: Text[100];
    IdempotencyKey: Text[100];
    ExternalMessageId: Text[100];
    ContractVersion: Code[20];
    var TransactionEntryNo: Integer
): Boolean
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| CorrelationId | Text[100] | Cross-system trace ID (stored, not processed) |
| IdempotencyKey | Text[100] | If set, duplicate calls with same key+interface return the existing entry |
| ExternalMessageId | Text[100] | Caller's own message ID (stored, not processed) |
| ContractVersion | Code[20] | Interface contract version (stored, not processed) |
| TransactionEntryNo | Integer (var) | Returns the Entry No. of the created (or existing) transaction |

---

### ReceiveAttachment

Convenience method to send a file attachment through an Attachment Mode Receive interface.

```
PROCEDURE ReceiveAttachment(
    InterfaceCode: Code[20];
    RecordKey: Text;
    AttachmentName: Text;
    ContentType: Text;
    FileExtension: Text;
    Base64Content: Text
): Boolean
```

Builds the required JSON payload internally and calls `Receive()`. The interface must have **Attachment Mode** enabled.

**JSON payload structure (generated internally):**
```json
{
  "recordKey": "CUST-001",
  "attachmentName": "Invoice.pdf",
  "contentType": "application/pdf",
  "fileExtension": "pdf",
  "content": "<base64-encoded file content>"
}
```

---

### Send

Triggers an immediate synchronous Send of BC data to an external endpoint.

```
PROCEDURE Send(InterfaceCode: Code[20]): Text
```

**Returns:** Response body from the external endpoint.

**The signature is unchanged and the call still means the same thing.** `Send` is now a one-line
delegate to `SendWithOptions(InterfaceCode, '')` — an existing caller keeps working, keeps its
parameter list, and keeps sending the whole configured set.

**Behavior:**
- Validates interface (must exist, not blocked, type = Send)
- Creates a transaction record
- Runs processing synchronously in the foreground
- Reads the records of the configured table — narrowed by the interface's own filter lines —
  builds the [response envelope](#response-envelope-send--publish), POSTs it to the endpoint
- Returns the endpoint's response body

**What did change is the shape of the outbound payload.** Until this version the body POSTed to
the endpoint was a bare JSON array. It is now the envelope object, on `Send` as on `Publish`.
A receiving system written against the array reads `value` instead of the body root — or, for a
partner system whose inbound format is fixed to the array, sends `"envelope": false` over `SendWithOptions` to keep
getting the bare array. See [Response envelope](#response-envelope-send--publish).

**Errors:**
- Interface not found / blocked → Error
- Interface not of type Send → Error "Interface 'X' is not of type 'Send'."
- HTTP request failed → Error with status code and response body
- More records than the interface's **Max Page Size** and no paging asked for → Error, see
  [The cap always applies](#the-cap-always-applies)

---

### SendWithOptions

`Send` with a retrieval options document. New in this version.

```
PROCEDURE SendWithOptions(InterfaceCode: Code[20]; RequestData: Text): Text
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| InterfaceCode | Code[20] | The Code of a configured Send Interface Definition |
| RequestData | Text | [Retrieval options](#retrieval-options-send--publish) as a JSON object. Empty is allowed and means "no options" |

**Returns:** Response body from the external endpoint, exactly as `Send`.

Identical to `Send` in every other respect. The options are stored on the transaction journal row
before processing starts, so the row afterwards says which filter, page size and token this call
actually ran with — there is no second field to look in, and no way to run a send whose options
were not recorded.

---

### Publish

Exposes BC data for an external system to pull.

```
PROCEDURE Publish(InterfaceCode: Code[20]; RequestData: Text): Text
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| InterfaceCode | Code[20] | The Code of a configured Publish Interface Definition |
| RequestData | Text | [Retrieval options](#retrieval-options-send--publish) as a JSON object. Empty is allowed and means "no options" |

**Returns:** The [response envelope](#response-envelope-send--publish) as the web service response
body.

**`RequestData` is no longer ignored.** Until this version it was stored on the transaction and
never read. It is now the options document, and the change has one visible consequence for
existing callers: a body that is valid JSON but is not an object — a bare array, a string, a
number — used to be accepted and disregarded, and is now an error. Sending `""` is unchanged and
still means "the whole set".

**Behavior:**
- Validates interface (must exist, not blocked, type = Publish)
- Parses the options; refuses anything it cannot read rather than falling back to "no options"
- Narrows the records: the interface's own filter lines first, the caller's conditions second
- Cuts the result to one page
- Builds the envelope from the field mappings and returns it

**Example response:**
```json
{
  "value": [
    { "itemNo": "ITEM-001", "description": "Widget A", "unitPrice": "10.50" },
    { "itemNo": "ITEM-002", "description": "Widget B", "unitPrice": "15.00" }
  ],
  "pageSize": 1000,
  "hasMore": false,
  "nextPageToken": null,
  "count": null
}
```

---

## <a id="retrieval-options-send--publish"></a>Retrieval options (Send / Publish)

`RequestData` of `Publish` and of `SendWithOptions` is a JSON **object**. Every member is
optional, and an empty or whitespace-only `RequestData` is the compatibility path: no filter, no
paging wish, the interface's own configuration and nothing else.

```json
{
  "filter": [
    { "key": "countryCode", "op": "eq",      "value": "DE" },
    { "key": "lastChanged", "op": "between", "value": "2026-01-01", "valueTo": "2026-06-30" }
  ],
  "page": { "size": 200, "token": "MTs...=" },
  "includeCount": true
}
```

| Member | Type | Default | Meaning |
|---|---|---|---|
| `filter` | Array of condition objects | none | Conditions the caller adds on top of the interface's own filter lines. `null` and an empty array both mean "no conditions" |
| `page.size` | Integer | the interface's Max Page Size | How many records this call may return. Never more than the interface's maximum |
| `page.token` | Text | none | Continuation token from a previous answer's `nextPageToken` |
| `includeCount` | Boolean | `false` | When true, `count` in the answer is the size of the filtered set |
| `envelope` | Boolean | `true` | When false, the answer/payload is the bare array of rows; refused together with paging or `includeCount` |

**An unknown member at the top level is ignored**, so a newer caller keeps working against an
older installation. `includeLines` is the first such member: it is read and has no effect in this
version. Nothing a caller can add at this level widens the answer, which is what makes ignoring
it safe.

**An unknown member inside a condition is an error.** `ops` for `op`, or `values` for `value`,
would leave the condition half-read and the answer wider than the caller believes it asked for.
A condition takes `key`, `op`, `value` and `valueTo` and nothing else.

### Filter conditions

| Member | Required | Notes |
|---|---|---|
| `key` | yes | The **Json Key** of a field mapping of this interface, matched case-insensitively. Max 50 characters |
| `op` | yes | One of the wire names below, matched case-insensitively |
| `value` | for every operator except where noted | Max 250 characters. Not trimmed — a leading blank is part of what you asked for |
| `valueTo` | only for `between` | Max 250 characters. Ignored by every other operator |

A `value` longer than 250 characters is refused rather than cut: a shortened filter value matches
**more** records than the caller asked for, and silently widening a filter is worse than an error.

### Operators

| Wire name | Comparison | Applies to |
|---|---|---|
| `eq` | equal | every filterable type |
| `ne` | not equal | every filterable type |
| `gt` | greater than | every filterable type |
| `ge` | greater than or equal | every filterable type |
| `lt` | less than | every filterable type |
| `le` | less than or equal | every filterable type |
| `between` | `value` to `valueTo`, both ends included | every filterable type |
| `in` | one of a `\|`-separated list in `value` | **Integer, BigInteger, Decimal, Date, DateTime, Option only** |
| `startsWith` | value at the start of the field | **Text and Code only** |
| `contains` | value anywhere in the field | **Text and Code only** |

The wire names are the contract. The captions on the setup page are translated and may be
renamed; these ten strings never move.

**`in` is not available for Text and Code fields.** A quoted `|`-list of text values is exactly
the filter-injection surface this design avoids everywhere else. Send one condition per value —
across several calls, since two conditions on one field are refused — or use `startsWith`.
At most 50 values in one `in` list.

**`startsWith` and `contains` reject filter syntax in the value.** These two are the only
operators whose value ends up next to a `*` in the filter string, so the value must not carry
syntax of its own. The characters `* ? | & < > = @ ' " ( )` and the two-character sequence `..`
are each refused with a message naming the offending character. For every other operator the
value is a typed placeholder, never concatenated text, so `*`, `|` and `..` in it are matched
literally.

**A Blob field cannot be filtered** with any operator.

**Two conditions on the same field are an error.** BC replaces a filter on a field instead of
ANDing it, so the second condition would silently make the first one disappear. Use `between`
for a range.

### What a caller may filter on

**Only fields the interface's field mapping exposes**, addressed by the mapping's **Json Key**.
That mapping is the whole allow-list — there is no second permission check, and no way for a
caller to reach a field the interface does not already put into its payload.

**An unknown key is an error, not an empty result.** `{"key": "creditLimit", ...}` on an interface
that does not map `creditLimit` fails the call; it does not return zero rows and it does not
return the unfiltered set. The message names the key and deliberately does **not** list the keys
that would have worked: listing them would answer "which fields does this interface expose" to
anyone who can call it, and would turn the error into an oracle — filter on a field that is not
in the payload, watch whether the message changes, and read the value off the answer instead of
off the data.

A mapping line that supplies a **static value** is not filterable either, and produces the same
error as an unknown key. This matters more than it sounds. A static value is the supported way to
keep a key in the published contract while hiding the number behind it — map `creditLimit` to the
real Credit Limit field, give the line a static `0`, and every published row reads `0`. Were such
a key filterable, the condition would be applied to the **real** field while the answer still
showed the constant, so row membership and `count` would leak the hidden value and a handful of
`between` calls would recover it exactly. Only mappings that actually publish the field's own
value can be filtered.

A mapping line with **Field No. 0** is not filterable either, for the simpler reason that it reads
nothing from the record at all.

### The caller can narrow, never widen

Filter lines configured on the interface are applied in BC filter group 2, the caller's
conditions in group 0. The platform ANDs the groups, and a filter in group 0 cannot widen one in
group 2. A caller whose condition points outside the configured filter therefore gets an **empty
`value` and no error** — the request was legal, it just matches nothing.

---

## Paging (Send / Publish)

Each interface definition carries **Max Page Size** (field 15, default 1000, range 1–100000). An
interface configured before that field existed reads 0, and 0 is read as 1000 at the point of use.

Effective page size = the caller's `page.size` when it is smaller than the interface's maximum,
otherwise the maximum. A caller can ask for less, never for more, and the answer always reports
which size was actually used in `pageSize`.

Records are read in **primary key order**, always, with no `SetCurrentKey` anywhere. That is what
makes a continuation token mean anything.

### <a id="the-cap-always-applies"></a>The cap always applies

A call that sends neither `page.size` nor `page.token` has not asked for paging. If the filtered
set is larger than the effective page size, that call **errors**. It does not return the first
page, and it does not return a short answer that looks complete — that is indistinguishable from
the whole result, and is the failure this refuses. The error message names the limit and the
continuation token of the last record that fit, so the retry does not have to start from the top.

### Continuation tokens

`nextPageToken` is the Base64-encoded position of the **last record delivered**. Send it back as
`page.token` to get the next page. Two ways it fails, both loudly:

- **Not readable** — the token is not valid Base64, or the position it decodes to is not a
  position of this table. Error.
- **Stale** — the record the token points at was deleted between the two calls. Error. Continuing
  from the following record instead would skip whatever was inserted in between and return a
  result that looks complete; the token is not silently restarted from the top either.

A token is only meaningful for the same interface and the same filter. Change the filter between
pages and the pages no longer describe one result set.

---

## <a id="response-envelope-send--publish"></a>Response envelope (Send / Publish)

**This is a breaking change against every version before it, and it is frozen from the first
AppSource release on.** Until now `Publish` returned, and `Send` POSTed, a bare JSON array. A bare
array cannot say whether it is the whole answer — every consumer that received one had to assume
it was. The envelope is what turns that assumption into something the receiving system can read.

```json
{
  "value": [ { "code": "EUR", "description": "Euro" } ],
  "pageSize": 200,
  "hasMore": true,
  "nextPageToken": "MDA7MTA7MTswOw==",
  "count": 4711
}
```

| Key | Type | Null when | Meaning |
|---|---|---|---|
| `value` | Array | **never** | The records of this page, one object per record, built from the field mappings. An empty array when nothing matched |
| `pageSize` | Integer | never | The **effective** page size this call ran with — not necessarily what the caller asked for |
| `hasMore` | Boolean | never | Whether at least one more record follows this page |
| `nextPageToken` | Text or `null` | `hasMore` is false | The token to pass as `page.token` for the next page |
| `count` | Integer or `null` | `includeCount` was not `true` | The size of the **filtered** set, before paging |

`nextPageToken` and `count` are written as an explicit JSON `null` when they do not apply, never
left out. A missing key and a null are two different contracts, and a typed consumer notices which
one it got.

`value` is always an array and always present, including on an interface with no matching records
and including when `hasMore` is true. Nothing else appears at the top level of the envelope in
this version.

**The envelope is the default and can be switched off per call.** Sending `"envelope": false`
in the options object turns the answer/payload into the bare array of rows — `value` and nothing
around it — for a partner system whose inbound format is fixed to that shape. Applies to `Send`
as on `Publish` alike, since both build their outgoing payload through the same envelope builder.
Paging and `count` are unavailable without the envelope, so `"envelope": false` combined with a
`page` or with `"includeCount": true` is refused rather than silently dropping the option: a bare
array has no place for `hasMore`, `nextPageToken` or `count`, and a short array that looks
complete is exactly the failure the envelope exists to prevent.

---

## JSON Payload Format (Receive)

For JSON interfaces, the incoming `RequestData` is a flat JSON object, one property per mapped
field:

```json
{
  "customerNo": "C00010",
  "name": "ACME Corp",
  "city": "Munich",
  "postCode": "80333"
}
```

Three shapes are accepted, and one transaction carries all the records in the payload:

| Shape | Batch Root Path | Result |
|-------|-----------------|--------|
| A single object, as above | empty | One record |
| An array of objects, `[{...}, {...}]` | empty | One record per element |
| An envelope object, `{"data": [{...}, {...}]}` | `data` | One record per element of that array |

An empty array is accepted and imports nothing. An array element that is not an object fails the
transaction, naming its position. A Batch Root Path that is configured but does not apply — the
payload is not an object, or has no such key — also fails rather than being ignored: the path and
the payload disagree, and only the operator can say which of the two is wrong.

Key matching is **case-insensitive** — `"customerNo"`, `"CustomerNo"`, and `"CUSTOMERNO"` all resolve to the same field mapping.

---

## CSV Payload Format (Receive)

For CSV interfaces, each line is one record and columns are identified by **Position** in the
field mapping.

```
C00010,ACME Corp,Munich,80333
C00011,Beta GmbH,Berlin,10115
```

**Delimiter** is configured per interface — comma (default), semicolon, tab or pipe. Semicolon
is the common case for files produced in German-speaking countries.

**Header row**: set `CSV Has Header Row` on the interface and the first non-empty line is
treated as column names and not imported.

**Quoting** follows RFC 4180. A value may be enclosed in double quotation marks, and a
delimiter inside such a value does not separate columns. `""` inside a quoted value is one
literal quotation mark, and a line break inside a quoted value belongs to the value. A
quotation mark only opens a quoted value at the start of a field — anywhere else it is an
ordinary character, so a value like `5" Rohr` imports unchanged. Quoted values keep their
outer whitespace, unquoted values are trimmed.

**A failing line aborts the whole transaction.** Processing is all-or-nothing: nothing from the
payload is written, and the error message names the source line number (1-based, header and
blank lines counted). Split a large file into one transaction per record if you need per-record
error tracking and partial success.

A line with fewer columns than the mapping reads is rejected rather than imported in part, and
an unterminated quotation mark is rejected rather than read as one large value.

One limit worth knowing: numeric and date values must use the invariant format (`1234.56`,
`2026-08-19`) — a per-interface format is not configurable yet. A UTF-8 byte order mark at the
start of the payload is removed before parsing and does not reach the first column.

---

## Field Type Handling

All incoming values arrive as text and are converted to the target field's data type:

| BC Field Type | Conversion |
|---------------|-----------|
| Text, Code | `CopyStr(value, 1, field length)` |
| Integer, Option | `Evaluate(IntValue, value)` |
| Decimal | `Evaluate(DecValue, value)` |
| Boolean | `Evaluate(BoolValue, value)` |
| Date | `Evaluate(DateValue, value)` |
| DateTime | `Evaluate(DateTimeValue, value)` |
| BigInteger | `Evaluate(BigIntValue, value)` |
| Guid | `Evaluate(GuidValue, value)` |
| Blob | Expects base64-encoded content |

**Every failed conversion is a hard error.** A value that does not parse as the target type,
a text value longer than the target field, and base64 content that does not decode all fail the
whole transaction with a message naming the field and the value. Nothing is skipped silently —
a field that could not be written would otherwise leave a record that looks imported and is not.

A missing key and an empty value are two different statements, and the import treats them
that way. Following JSON Merge Patch (RFC 7396), **an absent key means "leave this field
alone"**, while an explicit empty string or `null` is how a sender says "clear this". On top
of that, two configured escapes are taken before the conversion is attempted, and only these
two — **Mandatory** and **Skip If Empty**:

| Situation | Result |
|-----------|--------|
| Key absent, mapping is **Mandatory** | Transaction fails, naming the JSON key and the field |
| Key absent, mapping is not mandatory | Field is left unchanged — omitting a key never clears a field |
| Key present, value empty or `null`, mapping is **Mandatory** | Transaction fails, naming the JSON key and the field |
| Key present, value empty or `null`, mapping has **Skip If Empty** | Field is left as it is |
| Key present, value empty or `null`, neither switch set | Field is cleared — the source said "this is now empty" |
| Value present but not convertible | Transaction fails |

---

## Outbound Payload Format (Send / Publish)

The outbound payload is the [response envelope](#response-envelope-send--publish). Its `value`
member is the array of record objects that used to be the whole payload:

```json
{
  "value": [
    { "jsonKey1": "value1", "jsonKey2": "value2" },
    { "jsonKey1": "value3", "jsonKey2": "value4" }
  ],
  "pageSize": 1000,
  "hasMore": false,
  "nextPageToken": null,
  "count": null
}
```

The rules for the objects inside `value` are unchanged:

- If **Json Key** is empty for a field mapping line, the **Field Name** is used as the key
- Blob fields are base64-encoded
- Fields marked **Skip If Empty** are omitted from the object if the value is empty
- For **Entry Based Table** interfaces, the primary key field is excluded from the payload

---

## Error Responses

All errors follow the standard BC error response format. The BC Nexus webservice calls `Error()` with a Label text. The transaction is created with Status = Error and the error message stored in `Error Message` (Text[2048]).

---

## Examples

Ready-to-send request bodies for every function above, a curl example per call, and a
PowerShell smoke test that drives the whole sequence with a client-credentials token:
**[Examples]({{ site.baseurl }}/help/examples/)**.

The examples target the sample data (`CURRENCYIMPORT` on the standard Currency table), so a new
environment needs no data model of its own to try the service. The example README also covers
the two things this contract page does not spell out: `RequestData` is a JSON *string* and has
to be escaped inside the request body, and `Send`/`Publish` return `Text`, so the `value` member
of the OData response is a string that has to be parsed a second time.
