---
title: "BC Nexus — Setup Guide"
permalink: /help/setup-guide/
---

# BC Nexus — Setup Guide

_Step-by-step guide to configure your first interfaces. No AL development needed._

---

## Prerequisites

- Business Central 27.0 or later (SaaS or on-premise)
- BC Nexus extension installed (`.app` file or published via AL Language extension)
- Admin access to the BC company

---

## 1. Enable HTTP Requests

After installing the extension:

1. Go to **Extension Management**
2. Find **BC Nexus** in the list
3. Open the extension details and enable **Allow HttpClient Requests**

This is required for outbound Send interfaces (BC → external endpoint).

---

## 2. Publish the Web Service

BC Online does not allow an extension to register web services programmatically. You must publish Codeunit 50100 manually before any external system can call `Receive`, `Send`, or `Publish`.

1. Open the **Web Services** page in BC (search bar: "Web Services")
2. Choose **New**
3. Fill in the row:

| Field | Value |
|-------|-------|
| Object Type | `Codeunit` |
| Object ID | `50100` |
| Service Name | `FXNINexusWebservice` |
| Published | ✓ (enable the checkbox) |

4. Confirm the row appears in the list with an OData URL — this confirms the service is live.

> **Repeat** this step in every environment (sandbox, production) and after any reinstall that removes the existing entry.

---

## 3. Permissions and Access

BC Nexus ships three permission sets. Assign exactly one per user or account — each is built for a different role, and none of them is meant to be layered on top of another.

| Permission Set | Caption | Assign to | What it can do |
|---|---|---|---|
| `FXNI Nexus Admin` | BC Nexus Admin | The person who sets BC Nexus up and keeps it running | Everything: setup, endpoints, interfaces, field mappings, and the operational actions on the transaction journal. It is also the only set that can register or replace endpoint credentials. |
| `FXNI Nexus User` | BC Nexus User | The business user who works with the results | Reads the configuration and the transaction journal; can retry a failed transaction. Cannot trigger an interface directly. |
| `FXNI Nexus Integr.` | BC Nexus Integration | The technical account a foreign system authenticates as when it calls the published web service | Calls the web service. No user interface, no configuration changes. |

A few points worth knowing before you assign these:

- **`FXNI Nexus User` cannot use the "Process Manually" action.** That action re-runs the interface immediately — for a Send interface, that includes the outbound call with the stored endpoint credentials, which this role must never trigger directly. If a business user needs to retry a failed transaction, they use **Reset Status** instead: it only resets the transaction to Open, without running anything. The job queue then reprocesses it under its own account (typically an account with `FXNI Nexus Admin` or `FXNI Nexus Integr.`).
- **`FXNI Nexus Integr.` is for the technical account, not for a person.** It grants no page access at all — nothing of BC Nexus is visible if someone signs in interactively with it. Use it for the S2S application registration or service user that the external system authenticates as when it calls the web service.
- **Why the split matters:** whoever can execute the web service codeunit can trigger any Send interface for any interface code — including the endpoint credentials stored for it, which they never see directly. That authorization is not visible on the endpoint or interface record itself, so assign `FXNI Nexus Integr.` only to the technical account that is meant to call the web service, never to a person as a shortcut.

**Setting up the calling account itself is a Business Central / Entra ID task, not a BC Nexus one.** An external system authenticates to the web service through Business Central's own OAuth 2.0 client-credentials flow: an Entra ID (Azure AD) app registration with a client secret, granted the Business Central API permission, and mapped to a Business Central user. Once that user exists, assign it the `FXNI Nexus Integr.` permission set above — BC Nexus does not create or manage the app registration itself.

---

## 4. Open BC Nexus Setup

Search for **BC Nexus Setup** in the BC search bar.

The Install codeunit creates a default Setup record automatically on first install. You will see:

- **No. Of Retries per Transaction** — how many times the job queue retries a failed transaction (default: 3)
- **Auto. Process Transactions** — enable this to have a Job Queue Entry process open transactions automatically

The Setup page also contains:
- **Endpoint Definitions** (sublist) — outbound connection credentials
- **Interface Definitions** (sublist) — all configured interfaces

---

## 5. Create a Receive Interface (Import into BC)

### <a id="interfaces"></a>Step 1 — Create the Interface Definition

In the Interface Definitions sublist on the Setup page, choose **New**:

| Field | Value |
|-------|-------|
| Code | `CUST-IMPORT` (your identifier) |
| Description | `Customer Import from ERP` |
| Table No. | Select the target table (e.g., Customer = 18) |
| Interface Type | `Receive` |
| Field Mapping Type | `JSON` or `CSV` |
| Blocked | Leave unchecked |
| Always Create New Entries | Enable for tables with auto-increment PKs (e.g. G/L Entry) |

> **CSV interfaces.** Choose `CSV` for Field Mapping Type when the data arrives as a text file with a fixed column order instead of JSON. Two more fields then appear on the interface line; they only take effect when Field Mapping Type is CSV:
>
> | Field | Values | Effect |
> |-------|--------|--------|
> | CSV Delimiter | Comma (default), Semicolon, Tab, Pipe | Character that separates the columns of the file. For the German-speaking market, Semicolon is the common case. |
> | CSV Has Header Row | Yes/No | When enabled, the first non-empty line is treated as a column header and is not imported as a record. |

### CSV Example — Set Up a Semicolon File with a Header Row

Assume the external system delivers customer master data as a file with semicolon as the delimiter and a header row:

```
Nummer;Name;Ort
K10001;Muster GmbH;Musterstadt
K10002;Beispiel AG;Beispielhausen
```

1. Create the Interface Definition as in Step 1, with Field Mapping Type = `CSV`.
2. Set CSV Delimiter to `Semicolon`.
3. Enable CSV Has Header Row — the line `Nummer;Name;Ort` is then skipped and not imported as a record.
4. In the Field Mapping (Step 2 below), create one line per column; Position matches the column number in the file (1 = Nummer, 2 = Name, 3 = Ort), regardless of whether a header row is present. Mappings with a static value (Static Field Value) read no column and do not count towards the required column number.
5. Use the **Processing Test** action and paste the sample file into the dialog that opens, as the request text, to create a test transaction.

**Values containing the delimiter or a line break:** if a value itself contains the delimiter or a line break, enclose it in double quotation marks, e.g. `"Musterstadt; Ortsteil Nord"`. A double quotation mark inside the value is represented by two double quotation marks (`""` stands for one literal `"`). A quotation mark only opens a quoted value when it is the very first character of the column — in the middle of a value (e.g. `5" Rohr`) it stays an ordinary character and is imported unchanged. Quoted values keep their leading/trailing whitespace; unquoted values are trimmed.

**Error messages:** if importing a line fails, the error message names the 1-based source line number (header row and blank lines counted) and the reason — for example too few columns or an unclosed quotation mark.

### <a id="field-mapping"></a>Step 2 — Define Field Mappings

Select the interface and choose the **Field Mapping** action.

The field mapping page header shows the Interface Code, Type, and Mapping Type (read-only).

**Tip:** Use the **Initialize from Table** action to auto-populate all table fields with position numbers and field names. Then delete the lines you don't need.

For each field you want to map:

| Field | Value |
|-------|-------|
| Position | Order in JSON/CSV (1, 2, 3, ...) |
| Json Key | The JSON key in the incoming payload (e.g., `customerNo`) |
| Field No. | Select the BC field from the target table |
| Mandatory | Enable for required fields |
| Validate Field | Enable to run BC validation triggers on this field |

**Json Path (Receive only, JSON only).** Leave it empty for the flat lookup above — Json Key alone is all most interfaces need. Fill it instead of Json Key when the value sits inside a nested JSON payload, with dot-separated segments and an optional fixed array index, e.g. `header.lines[0].itemNo`; a key that itself contains a dot or a bracket cannot be addressed this way. Setting it on a Send or Publish interface, on a CSV interface, or entering an expression that does not parse, is rejected when you leave the field.

**Tip:** Use the **Generate JSON Key** action to auto-generate valid JSON key names from field names.

### Step 3 — Test the Interface

Use the **Processing Test** action on the Interface Definition. A dialog opens with an empty text box; paste a sample payload in the format the interface expects — JSON or CSV, according to its Field Mapping Type — and confirm with OK. This creates a transaction with status Open that you can then process manually from the Transaction List; the text you pasted is stored unchanged as its request data.

**If Auto. Process Transactions (Step 4) is already enabled in this company**, the transaction is not waiting for you: the job queue picks it up on its next run and writes the test data into the target table. Use sample data, or switch auto-processing off while you test.

Cancelling the dialog creates nothing. Confirming an empty box — or one holding only blank space — is refused with an error message rather than creating a transaction that has no request data. On a new interface line that has not been saved yet — no Code entered — the action reports an error instead of opening the dialog.

### Step 4 — Enable Auto-Processing

On the Setup page, enable **Auto. Process Transactions**. This creates a Job Queue Entry that processes open transactions automatically.

### <a id="attachments"></a>Attachment Mode (optional)

Enable **Attachment Mode** on a Receive interface when the incoming call should store a file
instead of writing table fields — for example, a PDF or an image attached to a BC record. Two
setup-level fields control it, both on the **BC Nexus Setup** page:

| Field | Value |
|-------|-------|
| Enable Attachment Storage | Master switch for attachment-mode interfaces; must be on before any Attachment Mode interface can process a request |
| Max Attachment Size (KB) | Maximum allowed attachment size; `0` means unlimited |

With Attachment Mode enabled, a caller uses the `ReceiveAttachment` web service function instead
of `Receive` — see [API Reference]({{ site.baseurl }}/help/api-reference/) for the request shape.

---

## 6. Create a Send Interface (BC → External Endpoint)

### <a id="endpoints"></a>Step 1 — Create an Endpoint Definition

In the Endpoint Definitions sublist on the Setup page, choose **New** and fill in:

| Field | Value |
|-------|-------|
| Code | `MY-ENDPOINT` |
| Description | `DataLake Export Endpoint` |
| Endpoint URL | `https://api.example.com/ingest` — must start with `https://`, see below |
| HTTP Timeout (ms) | Default `30000`, see below |
| Authentication Type | `None`, `Basic Authentication`, or `OAuth 2.0` |

**HTTPS is mandatory.** Endpoint URL and Token URL only accept `https://` addresses — checked as soon as you leave the field, and again immediately before every request is sent, so a value entered by a configuration package cannot bypass it either. `http://localhost` is deliberately not exempted: a Business Central cloud environment runs its service tier in Microsoft's datacenter, which can never reach a developer's own machine, so an `http://` endpoint has no environment in which it would actually work. For local tests against a mock endpoint, subscribe to the `OnBeforeSendRequest` event instead of pointing at an unencrypted URL — that is how the extension's own test suite intercepts requests before they leave BC.

**HTTP Timeout (ms).** How long BC waits, in milliseconds, for a response before it abandons the request — for the endpoint call as well as for the OAuth token request. Default is `30000` (30 seconds). Enter `0` to use the platform default, or any value from `1000` to `300000`; a value outside that range is rejected when you leave the field. Consider lowering it for an endpoint that should always answer quickly: the job queue processes every open transaction in one run, one after another, so a single endpoint that hangs delays every other queued transaction behind it.

#### Authentication Type: OAuth 2.0

Selecting `OAuth 2.0` brings four more fields into play. They are hidden by default in the Endpoint Definitions list — use **Choose Columns** to show them:

| Field | Value |
|-------|-------|
| Client ID | The OAuth 2.0 client ID |
| Token URL | The token endpoint (`https://` only, checked the same way as Endpoint URL) |
| OAuth Auth Type | `Client Credentials` — the only supported grant type |
| Token Expires At | Read-only. Shows when the cached access token expires; BC acquires a new one automatically after this time. |

**Store the client secret with the Set Client Secret action, not in a field.** Choose **Set Client Secret** on the endpoint line. The value is written to Isolated Storage, never to a table field, and is not shown again afterwards. (The same action also stores the credential for Basic Authentication endpoints — there it is the complete `Basic base64(user:pass)` header value, not a plain password.)

**Token Request Body — use the `{{ "{{" }}client_secret}}` placeholder.** This field holds the body sent to the Token URL when a new access token is requested. Wherever the secret belongs, write the literal placeholder `{{ "{{" }}client_secret}}` — never the secret itself. Example:

```
grant_type=client_credentials&client_id=<your-client-id>&client_secret={{ "{{" }}client_secret}}&scope=<your-scope>
```

BC substitutes `{{ "{{" }}client_secret}}` with the value stored via Set Client Secret at the moment the token is actually requested; the secret is never written into this field. If the token endpoint needs no client secret parameter at all (for example mTLS or private_key_jwt), leave the placeholder out and the body is sent unchanged.

Two setups are rejected, both with an error naming the field, the endpoint code and the parameter — but never the value itself:

- **A literal secret in the body**, either as `client_secret=<value>` or as the JSON form `"client_secret": "<value>"`, is refused with an error telling you to replace it with `{{ "{{" }}client_secret}}`. Reason: Token Request Body is a database field. It is exported together with the company and readable in a RapidStart configuration package, so a literal secret there would defeat the entire point of storing it in Isolated Storage.
- **A placeholder with no stored secret** — `{{ "{{" }}client_secret}}` is present, but Set Client Secret was never used for this endpoint — is refused with an error pointing you at the Set Client Secret action.

Both checks run when the token is actually requested — on Test Connection, or the first Send/Receive call that needs a token — not while you are typing into the field.

**Known limitation — `client_secret_basic` is not securely supported.** Some token endpoints expect the client secret as an HTTP Basic header on the token request instead of as a body parameter. BC Nexus has no placeholder mechanism for that: a pre-encoded Basic header written into a token request header field would sit in the database in clear, recoverable form, with none of the protection described above. Only the `client_secret` in the request body, via the placeholder, is currently handled securely. Only `Client Credentials` is supported as the OAuth grant type.

**Editing the three text fields.** *Token Request Body*, *Token Request Headers* and *Endpoint
Request Headers* are blob fields, so they have no column on the endpoint list. Each is edited
through its own action on the endpoint line — **Edit Token Request Body**, **Edit Token Request
Headers** and **Edit Endpoint Request Headers** — which opens a multiline dialog showing the
current value.

Both header fields hold a JSON object with one property per header, for example
`{"X-Api-Version": "2"}`. Two mistakes there stop the request instead of being worked around,
each with a message naming the field and the endpoint:

- **Content that is not a JSON object** — a stray comma, a missing brace, or a header whose
  value is itself an object or an array rather than a single scalar. None of the headers
  configured in that field is sent; the call fails rather than going out half-configured.
- **An `Authorization` header set by hand, on an endpoint that authenticates.** With any
  authentication type other than **None**, the authorization header is built from the
  endpoint's authentication settings. A second one configured here collides with it, and the
  collision is refused rather than resolved in either direction — remove the entry. With
  authentication type **None** nothing is built, so an authorization header configured here is
  accepted and sent — an API key carried that way is a legitimate configuration. A pre-encoded
  credential does not belong in these fields anyway: they are stored in the database in clear
  text and are exported with the company.

Use the **Test Connection** action to verify the endpoint configuration works.

### Step 2 — Create the Interface Definition

| Field | Value |
|-------|-------|
| Code | `ITEM-EXPORT` |
| Description | `Item Master Export` |
| Table No. | Select source table (e.g., Item = 27) |
| Interface Type | `Send` |
| Field Mapping Type | `JSON` |
| Endpoint Code | `MY-ENDPOINT` |
| Max Page Size | `1000` (default) — the most records one call may read |

**Max Page Size is a ceiling, not a page size.** A caller can ask for fewer records, never for
more. A caller that asks for no paging at all and matches more records than this gets an **error**
naming the limit, not a truncated answer — a short result that looks complete is the failure this
setting exists to prevent. An interface that was configured before this field existed shows `0`,
which is read as 1000 at the point of use; set a real number when you want a different ceiling.

### Step 3 — Define Field Mappings

Use **Initialize from Table** and select the fields to include in the outbound payload.

- Set **Skip If Empty** on optional fields
- Set **Json Key** to control the key names in the outbound payload

The **Json Key** does one more thing from this version on: it is the name an external caller may
filter on, and the only one. A field with no mapping line cannot be filtered, and neither can a
line that supplies a static value.

### <a id="filters"></a><a id="interface-filters-fixed-server-side-limits"></a>Step 4 — Interface Filters (fixed, server-side limits)

Choose **Filters** on the interface definition. Each line is one condition that always applies,
whatever the caller sends:

| Field | Meaning |
|-------|---------|
| Field No. | The field of the target table. Selecting it fills **Field Name** |
| Operator | Equal, Not Equal, Greater Than, Greater or Equal, Less Than, Less or Equal, Between, Starts With, Contains, In |
| Value | The value compared against. Written as a value, never as a filter expression — `*` and `\|` are matched literally |
| Value To | Upper end of the range; only for **Between** |
| Description | Free text for you; no effect on processing |

The action is only enabled for **Send** and **Publish** interfaces — a Receive interface writes
into the table and has nothing to filter.

Two things worth knowing before you configure one:

- **A caller can only narrow these lines further, never widen them.** The interface's lines and
  the caller's conditions go into different BC filter groups, and the platform ANDs them. A
  caller whose condition falls outside your filter gets an empty result, not more data.
- **Filtering on a field that is part of no key makes every call read the whole table.** The page
  shows a message when you pick such a field. It is a hint, not a refusal — on a small table it
  does not matter, on a large one it does.

`Starts With` and `Contains` are available for text and code fields only, and `In` for numeric,
date and option fields only. `In` takes a `|`-separated list in **Value**, at most 50 entries.

### Step 5 — Trigger the Send

From AL code or the **Processing Test** action, call `Codeunit 50100 Nexus Webservice: Send("ITEM-EXPORT")`.

Use `SendWithOptions("ITEM-EXPORT", '{"page":{"size":200}}')` when the caller wants to add its own
filter, a page size or a continuation token. `Send` is unchanged and means "no options".

---

## 7. Create a Publish Interface (BC exposes data for external pull)

### Step 1 — Create an Endpoint Definition (required for Publish)

Same as for Send — Publish also needs an endpoint definition (used for the response routing).

### Step 2 — Create the Interface Definition

| Field | Value |
|-------|-------|
| Code | `ITEM-PUBLISH` |
| Interface Type | `Publish` |
| Table No. | Source table |
| Endpoint Code | Your endpoint |
| Max Page Size | `1000` (default) — see the note under Send, Step 2 |

### Step 3 — Define Field Mappings

Same as Send. Use **Skip If Empty** for optional fields. The **Json Key** of each line is also the
name a caller may filter on.

### Step 4 — Set Filters (optional)

Same **Filters** action as for a Send interface — see
[Interface Filters](#interface-filters-fixed-server-side-limits). Use it when the published set
should be smaller than the table, whatever the caller asks for.

### Step 5 — Call from External System

The external system calls the BC Nexus web service `Publish` function with the interface code and,
optionally, a retrieval options object as `RequestData`.

BC returns a **response envelope**, not a bare array:

```json
{
  "value": [ { "itemNo": "ITEM-001", "description": "Widget A" } ],
  "pageSize": 1000,
  "hasMore": false,
  "nextPageToken": null,
  "count": null
}
```

`value` holds the records, `hasMore` and `nextPageToken` say whether the answer is complete and
where to continue. A caller written against the old bare array reads `value` instead of the root.
The options document, the ten filter operators and the paging rules are in
[API Reference]({{ site.baseurl }}/help/api-reference/#retrieval-options-send--publish).

---

## <a id="transactions"></a>8. Transaction Monitoring

Go to **Transaction List** (accessible from the Setup page or via search).

| Status | Meaning |
|--------|---------|
| Open | Waiting to be processed by the job queue |
| Error | Failed; error message shows the cause; retries remaining |
| Processed | Successfully completed |
| Canceled | Manually canceled; will not be retried |

### Actions on Transaction List / Card

| Action | Effect |
|--------|--------|
| **Reset Status** | Sets status back to Open so it will be retried |
| **Process Manually** | Processes the transaction immediately in the foreground |
| **Cancel** | Sets status to Canceled; stops all retries |

You can also open a transaction card to view and modify the raw Request body before re-processing.

---

## 9. Standard Page Integrations

BC Nexus adds a **Send** action to the following standard pages for quick manual export:

- Sales Order List
- Purchase Order List
- Planned/Firm Planned/Released Production Orders
- Assembly Order List
- Purchase Invoice List
- Vendor List
- Item List
- Customer List

These actions are additive page extensions and can be customized in BC's own page personalization.

---

## Tips and Pitfalls

| Situation | Solution |
|-----------|----------|
| Transaction stuck in Error | Open Transaction Card, check Error Message, fix data or setup, Reset Status |
| Mandatory field failing unexpectedly | Check if the incoming JSON key matches exactly (lookup is case-insensitive but the key must exist) |
| Always Create New Entries vs. Entry Based Table | Use Always Create New Entries when your PK comes from a number series. Use Entry Based Table when the PK is an auto-increment integer — then never map the PK field |
| OAuth token not refreshing | Check Token Expires At in Endpoint Definition; use Test Connection action to force a new token |
| Interface blocked | Uncheck Blocked on the Interface Definition |
| CSV line rejected ("has only N columns") | The line has fewer columns than the field mapping expects. Check the column count of the source file; mappings with a static value (Static Field Value) do not count |
| CSV import aborts with "unterminated" | A quotation mark was opened but never closed. Check the file for a missing closing `"` |
| Numbers/dates from CSV are not recognized | The number/date format is not yet configurable per interface — values must already be in the invariant format (e.g. `1234.56`, `2026-08-19`) |
| CSV file has an invisible character in column 1 of the first line | A UTF-8 byte-order mark at the start of the payload is removed before parsing, so this no longer happens. A mark that sits anywhere other than at the very start is data and is kept |
| Endpoint or Token URL rejected on entry | The value does not start with `https://`. Only HTTPS addresses are accepted, including for local mock endpoints |
| "sets the client_secret parameter to a literal value" | Token Request Body contains the secret in clear text. Replace it with `{{ "{{" }}client_secret}}` and store the value with the Set Client Secret action |
| "contains the placeholder {{ "{{" }}client_secret}}, but no client secret is stored" | Token Request Body uses the placeholder correctly, but Set Client Secret was never run for this endpoint |

---

## Uninstalling BC Nexus

Before removing the extension:

1. On the **BC Nexus Setup** page, disable **Auto. Process Transactions**. This removes the Job
   Queue Entry that runs Codeunit 50101 (`FXNI Txn. Proc. Job`) on a schedule — an entry left
   behind keeps firing after the extension is gone and errors on every run.
2. Check for a Job Queue Entry pointing at Codeunit 50101 directly (search **Job Queue Entries**,
   filter Object Type to Run = Codeunit and Object ID to Run = 50101) in case one was created
   outside the Setup page action, and remove it as well.
3. On the **BC Nexus Setup** page, choose **Delete All Credentials**. This removes the stored
   Client Secret and Access Token of every endpoint in the current company and clears the token
   expiry stamp. It asks for confirmation first, and it keeps the endpoints themselves — the
   only cost of running it by mistake is entering the secrets again.

   **Repeat it in every company that has endpoints.** The keys are written with
   `DataScope::Company`, so one run covers one company.

   This is the step that has no automatic equivalent, and the reason is worth knowing: Isolated
   Storage cannot be enumerated. A key is only reachable through the code that rebuilds it from
   the endpoint code, so once the endpoint rows are gone — uninstalled, or the company deleted —
   nothing can find those keys again, not even a fresh installation of BC Nexus. Running this
   while the rows still exist is the only moment the values are reachable.
4. Delete the interfaces and endpoints you no longer need before uninstalling. Deleting an
   Endpoint Definition removes its stored Client Secret and Access Token from Isolated Storage.
   This covers a deletion that runs the table's triggers, which is what deleting a line on the
   Endpoint Definitions page does. A `DeleteAll()` without triggers, uninstalling the extension,
   and deleting the company do not run it, and leave both keys behind until the extension data
   is removed via **Delete Extension Data**.

**What uninstalling removes, and what it does not.** AL has no uninstall trigger for a
per-tenant extension on Business Central Online — there is no `OnUninstallAppPerCompany()` or
equivalent hook, so there is no automatic cleanup step the extension can run as part of removal.
Uninstalling BC Nexus through Extension Management stops the extension from running but leaves
all table data (setup, interfaces, endpoints, field mappings, the transaction log, and any
attachments) in place. To remove that data as well, use **Delete Extension Data** in Extension
Management after uninstalling — this is a separate, explicit step, not something uninstalling
does on its own.

---

## CSV Parsing Rules

BC Nexus parses CSV request bodies (Field Mapping Type = `CSV`) according to RFC 4180, applied
strictly — there are no Excel-style tolerances for malformed input:

- **Quoting.** A quotation mark only opens a quoted value when it is the first character of the
  column. A space or any other character before the opening quote means the quote is not
  special — it is imported as a literal character, and the value is not treated as quoted.
- **Trailing characters.** Characters that follow the closing quotation mark of a quoted value
  (before the next delimiter or line end) are appended to the value as plain text rather than
  causing an error.
- **Escaped quotes.** A double quotation mark inside a quoted value is represented by two
  consecutive double quotation marks (`""`), which is imported as one literal `"`.
- **Line breaks in values.** A line break inside a quoted value is part of the value, not a row
  separator — the parser keeps consuming lines until the closing quote is found.
- **Delimiter.** Configurable per interface (CSV Delimiter field): Comma, Semicolon, Tab, or
  Pipe. There is no auto-detection — the configured delimiter must match the file.
- **Leading BOM.** A leading UTF-8 byte-order mark on the file is removed before parsing, at the
  point where the stored payload becomes text, so it reaches neither the CSV parser nor the JSON
  parser. Only a mark at the very start is removed; anywhere else it is data.
- **Short lines.** A line with fewer columns than the field mapping expects aborts the import of
  that line with an error naming the line number, and rolls back the entire import of that
  transaction — there is no partial import; it is not padded or skipped silently.

This is deliberately strict. A file that a spreadsheet application would import without
complaint (stray characters after a quote, an unescaped quote in the middle of a field) may
still be rejected here — check the exact error message, which names the 1-based source line.
