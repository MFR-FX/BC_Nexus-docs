---
title: "Privacy Statement — BC Nexus"
permalink: /legal/privacy-statement/
---

# Privacy Statement — BC Nexus

**Publisher:** fx-its (Marco Frerix)  
**Contact:** support@fx-its.de  
**Last updated:** 2026-10-16

---

## 1. Overview

BC Nexus is a Microsoft Dynamics 365 Business Central extension that enables configurable data exchange (import, export, publish) between Business Central and external systems. This privacy statement explains what data the extension processes, how it is handled, and the rights of data subjects under the GDPR and applicable EU/German law.

BC Nexus is installed together with FXI Core, a library extension by fx-its that holds the endpoint credentials and performs the outgoing HTTP calls. This statement covers FXI Core as part of BC Nexus.

## 2. Controller and Processor

Two roles have to be told apart.

**Customer data.** For personal data processed through BC Nexus integrations, the controller is the organization that installs and operates the extension in its Business Central environment. fx-its does not access data in that environment. Should fx-its access it in the course of support, this happens only as a processor under a data processing agreement in accordance with Article 28 GDPR, to be concluded before access.

**Diagnostics telemetry, support, sales contacts and documentation website.** For the telemetry described in Section 5 and for the personal data described in Section 6 (support requests, enquiries through AppSource "Contact me", offers and contracts, visits to this documentation website), the controller is:

fx-its — Marco Frerix  
Arlberger Straße 32, 47249 Duisburg, Germany  
E-mail: support@fx-its.de

## 3. Data Processed by the Extension

BC Nexus processes data solely at the direction of the installing organization. It requires FXI Core, a library app from the same publisher that provides the HTTP transport, the endpoint configuration and the activity log. The code of neither app collects, stores, or transmits data to fx-its; data leaves the customer's Business Central tenant only through the configured integration targets and through the optional Copilot feature described below. Specifically:

- **Business Central table data** — the extension reads from and writes to tables as configured by the administrator (field mappings, endpoints).
- **Credentials** — the client secret or Basic authentication credentials of an endpoint and the OAuth access token obtained with them are held as `SecretText` in the Isolated Storage of the FXI Core app, per company, not in table fields. They are sent only to the token URL or endpoint URL configured for that endpoint.
- **Endpoint configuration** — endpoint URLs, authentication type, client ID, token URL, timeouts and additional request and token request headers, including any API key an administrator enters there, are stored in the endpoint table of the FXI Core app; interface definitions, field mappings and filters are stored in BC Nexus tables. Both are part of the customer's Business Central database.
- **Transaction journal** — every BC Nexus transaction is recorded in the BC Nexus Transaction List with its request and response payload, status and error message, for auditing, retries and troubleshooting.
- **Attachments** — if attachment storage is enabled in BC Nexus Setup, files received with a transaction are stored in the BC Nexus Attachment table together with file name, content type, size and the ID of the user who processed the transaction, or only as a reference when another extension stores them externally.
- **Activity log** — requests sent to an endpoint are recorded in the Activity Log of the FXI Core app with endpoint, URL, method, HTTP status, duration, error text and the ID of the user who sent them. Request and response bodies are stored there only when **Store Payloads** is switched on in FXI Core Setup (off by default), and only if a body does not exceed **Max Payload Size (KB)** (64 KB by default); a larger body is not stored. The **Purge** action on the Activity Log deletes entries older than **Retention Days** (30 by default); entries are not deleted automatically.

### Optional Copilot feature (mapping assistant)

BC Nexus registers a Copilot capability with the Business Central platform. It is listed as "Interface Field Mapping" on the **Copilot & agent capabilities** page and proposes field mappings for a sample payload that a user pastes on purpose. The feature is only available in Business Central online. It is registered as generally available and is therefore active by default once BC Nexus is installed; an administrator can deactivate it on the **Copilot & agent capabilities** page at any time.

The language model is called through the Azure OpenAI resources that Microsoft manages for Copilot in Business Central (Business Central AI resources). fx-its operates no Azure OpenAI resource for this feature, and the extension carries no endpoint or key of its own. When a user presses Generate, the extension sends:

- fixed instructions that describe the task and the expected answer format — they contain no customer data;
- the number and caption of the target table, the interface direction (Receive, Send or Publish) and the sample format (JSON or CSV);
- the key names found in the pasted sample — for JSON the keys of the first record that hold a simple value, for CSV the column headings, or "Column 1", "Column 2" … where the file has no header row or a heading is empty — each with its column position (CSV only) and **one** value from the first data record, truncated to 60 characters. The rest of the sample is not sent;
- for each field of the target table that may be mapped: its number, its name, its caption in the user's language, its data type, its field class (for example FlowField) and, for option and enum fields, the names of the allowed values.

The answer is a proposal only and is held in a temporary list; a discarded proposal is not stored. When the user confirms it with Keep it, only the accepted mapping lines are written to the field mapping: for JSON the key name, for CSV the column position, each with its target field and the Mandatory, Validate Field and Skip If Empty settings. The sample values, the rest of the pasted sample and the model's confidence and reasoning are never stored. The processing by Microsoft — including the geography in which it takes place, which administrators control with the **Allow data movement** setting on the same page — is governed by your agreements with Microsoft and Microsoft's documentation for Copilot in Business Central. fx-its does not receive the prompt or the answer. Because a pasted sample can contain personal data, the person pasting it should use example data or remove personal data first.

## 4. Data Retention

All data that BC Nexus and FXI Core store remains within the customer's Business Central environment, except attachment content that another extension stores externally (Section 3). BC Nexus does not delete entries in its transaction journal or stored attachments automatically and does not yet offer a retention policy or a purge function for them; these records, including the request and response payloads they hold, remain in the customer's database. For the activity log of FXI Core, see Retention Days in Section 3. Retention of all other data follows the customer's own policies. No data from the code of either app is sent to fx-its; the only channel from the platform to an fx-its-operated system is the diagnostics telemetry described in Section 5, whose retention is stated there. For the data fx-its processes as controller in support and sales, see Section 6.

## 5. Data Sharing and Third-Party Transfers

Apart from the Copilot feature described in Section 3, BC Nexus — through FXI Core — makes HTTP calls to external endpoints **only** as configured explicitly by the customer administrator. The code of neither app analyzes or shares the customer's Business Central data with fx-its or third parties.

**Telemetry.**
The Business Central platform itself sends extension telemetry — errors, lifecycle events, and web service call metadata including tenant and environment identifiers, no business data — to the publisher's Application Insights resource in the Azure region Germany West Central. This is a platform mechanism, configured via `applicationInsightsConnectionString` in the app manifest, not code added by BC Nexus. Purpose: diagnostics and customer support. Retention: 30 days, after which the data is deleted. No record contents of the customer's Business Central data are transmitted through this channel.

## 6. Support, Sales Contacts and Documentation Website

**Support requests.** When you write to support@fx-its.de or open a request in the ticket portal, we process your name, e-mail address, organization, the content of your request and any attachments, as well as the history of the request. Every support e-mail is logged as a request in our ticket system, Jira Service Management, operated by Atlassian (Atlassian Pty Ltd and Atlassian US, Inc.) as our processor. Our e-mail is provided by Microsoft 365 (Microsoft Ireland Operations Limited) as our processor. Purpose: handling your request and providing the support owed under the contract.

**Enquiries through "Contact me".** When you press "Contact me" on the AppSource listing of BC Nexus, Microsoft passes the contact details you entered there to us through Microsoft Partner Center (source of these data, Article 14(2)(f) GDPR) — typically name, e-mail address, phone number, organization, country and job title. Purpose: answering your enquiry and sending you an offer. We process offers, order confirmations and invoices in our offer and accounting software; its provider acts as our processor.

**Documentation website.** This documentation website is hosted on GitHub Pages by GitHub, Inc. When you open a page, GitHub processes your IP address and technical request data (server log) to deliver the page and to protect its service; fx-its has no access to these logs. The website sets no cookies of its own and uses no analytics or tracking.

**Transfers to third countries.** Atlassian, Microsoft and GitHub may process data in the USA and other countries outside the EU/EEA. Atlassian, Microsoft and GitHub (as a Microsoft subsidiary) are certified under the EU-U.S. Data Privacy Framework, for which the European Commission has adopted an adequacy decision (Article 45 GDPR); in addition, their data processing agreements contain the EU Standard Contractual Clauses (Article 46(2)(c) GDPR), which also cover transfers to countries without an adequacy decision.

**Retention.**

- Enquiries that do not lead to a contract: deleted six months after the last contact.
- Support requests: deleted three years after the end of the calendar year in which the request was closed (the regular limitation period, Section 195 BGB), unless they are needed longer to assert or defend legal claims or are business letters that must be retained under Section 257 HGB (for example a notice of defect or a termination), which are retained for six years.
- Offers, order confirmations, invoices and other business letters: retained for the statutory periods of Section 257 HGB and Section 147 AO (six years for business letters, eight years for accounting vouchers such as invoices), then deleted.
- Website server logs: retained by GitHub under its own policies; fx-its does not store them.

**Provision of data and automated decisions.** You are not obliged to provide personal data. Without your contact details and the content of your request, however, we cannot answer an enquiry, send an offer or handle a support request, and without the data needed for the contract we cannot conclude it. fx-its makes no decisions based solely on automated processing, including profiling (Article 22 GDPR).

## 7. Legal Bases

- **Article 6(1)(b) GDPR** (performance of a contract and pre-contractual measures at your request): handling of support requests, answering enquiries through "Contact me", offers, order processing and communication with you about the use of the Software.
- **Article 6(1)(c) GDPR** (legal obligation): retention of business letters and accounting records under Section 257 HGB and Section 147 AO.
- **Article 6(1)(f) GDPR** (legitimate interests): support requests from persons who are not themselves our contractual partner (for example employees of a customer) — our interest is handling the request of the customer they act for; delivery and security of the documentation website.
- **Article 6(1)(f) GDPR** (legitimate interests): diagnostics telemetry — our interest is the stable operation of the Software, the analysis of errors and effective support. Telemetry does not contain business data. You can object at any time (see Section 9).

Customer data processed through integrations is processed by the customer on its own legal basis as controller.

## 8. Security Measures

- Client secrets, Basic authentication credentials and access tokens are stored as `SecretText` in Isolated Storage, not in table fields. Values an administrator enters as additional request or token request headers, for example an API key in an Authorization header, are stored in plain text in the endpoint table of FXI Core.
- All HTTP communication uses the endpoint URLs and TLS settings as configured by the customer.
- Source code is not exposed in the distributed symbol file (`resourceExposurePolicy` is restricted).

## 9. GDPR Rights

Data subjects whose personal data is processed through a BC Nexus integration may exercise their rights (access, rectification, erasure, restriction of processing, portability, objection) by contacting the data controller — i.e., the organization that operates the Business Central environment. fx-its is not in a position to respond to such requests directly unless contracted as a processor; fx-its will forward requests it receives to the customer where this is possible.

For data that fx-its processes as controller (Section 2), you can exercise these rights towards fx-its via support@fx-its.de. You have the right to object to processing based on Article 6(1)(f) GDPR on grounds relating to your particular situation.

## 10. Right to Lodge a Complaint

You have the right to lodge a complaint with a data protection supervisory authority. The authority responsible for fx-its is: Landesbeauftragte für Datenschutz und Informationsfreiheit Nordrhein-Westfalen (LDI NRW), https://www.ldi.nrw.de.

## 11. Changes to This Statement

fx-its may update this privacy statement. The current version is always available at <https://mfr-fx.github.io/BC_Nexus-docs/legal/privacy-statement/>.

## 12. Contact

Questions about this privacy statement:  
**fx-its** — support@fx-its.de
