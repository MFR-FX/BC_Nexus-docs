---
title: "Privacy Statement — BC Nexus"
permalink: /legal/privacy-statement/
---

# Privacy Statement — BC Nexus

**Publisher:** fx-its (Marco Frerix)  
**Contact:** support@fx-its.de  
**Last updated:** 2026-09-29

---

## 1. Overview

BC Nexus is a Microsoft Dynamics 365 Business Central extension that enables configurable data exchange (import, export, publish) between Business Central and external systems. This privacy statement explains what data the extension processes, how it is handled, and the rights of data subjects under the GDPR and applicable EU/German law.

## 2. Controller and Processor

Two roles have to be told apart.

**Customer data.** For personal data processed through BC Nexus integrations, the controller is the organization that installs and operates the extension in its Business Central environment. fx-its does not access data in that environment. Should fx-its access it in the course of support, this happens only as a processor under a data processing agreement in accordance with Article 28 GDPR, to be concluded before access.

**Diagnostics telemetry and support contact.** For the telemetry described in Section 5 and for personal data you send us when you contact support, the controller is:

fx-its — Marco Frerix  
Arlberger Straße 32, 47249 Duisburg, Germany  
E-mail: support@fx-its.de

## 3. Data Processed by the Extension

BC Nexus processes data solely at the direction of the installing organization. The extension's own code does not collect, store, or transmit data to fx-its; data leaves the customer's Business Central tenant only through the configured integration targets and through the optional Copilot feature described below. Specifically:

- **Business Central table data** — the extension reads from and writes to tables as configured by the administrator (field mappings, endpoints).
- **OAuth tokens and credentials** — stored in Business Central using the platform's `SecretText` / Isolated Storage mechanisms; never written to plain text or transmitted outside the configured endpoint.
- **Endpoint configuration** — base URLs, HTTP method settings, and mapping rules are stored in dedicated BC Nexus setup tables within the customer's Business Central tenant.
- **Transaction and error logs** — entry records (Nexus Entries) are stored in the customer's own database for auditing and troubleshooting.

### Optional Copilot feature (mapping assistant)

BC Nexus registers a Copilot capability, "FXNI Mapping Assistant", with the Business Central platform. It proposes field mappings for a sample payload that a user pastes on purpose. The feature is only available in Business Central online, and only when an administrator has activated the capability on the Copilot & AI Capabilities page. When a user presses Generate, the extension sends the following to the Azure OpenAI service that Microsoft manages for Business Central (Microsoft-managed resource; the extension carries no endpoint or key of its own):

- the name and number of the target table, the interface direction and the mapping format;
- the key names (JSON keys or CSV column positions) found in the pasted sample, each with **one** sample value truncated to 60 characters — the complete payload is not sent;
- the names of the fields of the target table that may be mapped.

The result is a proposal only; it is written to the mapping tables only after the user has confirmed it. The processing by Microsoft is governed by your agreements with Microsoft and Microsoft's documentation for Copilot in Business Central. fx-its does not receive the prompt or the answer. Because a pasted sample can contain personal data, the person pasting it should use example data or remove personal data first.

## 4. Data Retention

All data processed by BC Nexus remains within the customer's Business Central environment. Retention periods are governed by the customer's own policies and Business Central's built-in mechanisms. No data from the extension's own code is sent to fx-its; the only channel from the platform to an fx-its-operated system is the diagnostics telemetry described in Section 5, whose retention is stated there.

## 5. Data Sharing and Third-Party Transfers

BC Nexus makes HTTP calls to external endpoints **only** as configured explicitly by the customer administrator. The extension's own code does not analyze or share the customer's Business Central data with fx-its or third parties.

**Telemetry.**
The Business Central platform itself sends extension telemetry — errors, lifecycle events, and web service call metadata including tenant and environment identifiers, no business data — to the publisher's Application Insights resource in the Azure region Germany West Central. This is a platform mechanism, configured via `applicationInsightsConnectionString` in the app manifest, not code added by BC Nexus. Purpose: diagnostics and customer support. Retention: 30 days, after which the data is deleted. No record contents of the customer's Business Central data are transmitted through this channel.

## 6. Legal Bases

- **Article 6(1)(b) GDPR** (performance of a contract): handling of support requests and communication with you about the use of the Software.
- **Article 6(1)(f) GDPR** (legitimate interests): diagnostics telemetry — our interest is the stable operation of the Software, the analysis of errors and effective support. Telemetry does not contain business data. You can object at any time (see Section 8).

Customer data processed through integrations is processed by the customer on its own legal basis as controller.

## 7. Security Measures

- Credentials are stored using Business Central's `SecretText` and Isolated Storage — not in plain database fields.
- All HTTP communication uses the endpoint URLs and TLS settings as configured by the customer.
- Source code is not exposed in the distributed symbol file (`resourceExposurePolicy` is restricted).

## 8. GDPR Rights

Data subjects whose personal data is processed through a BC Nexus integration may exercise their rights (access, rectification, erasure, restriction of processing, portability, objection) by contacting the data controller — i.e., the organization that operates the Business Central environment. fx-its is not in a position to respond to such requests directly unless contracted as a processor; fx-its will forward requests it receives to the customer where this is possible.

For data that fx-its processes as controller (Section 2), you can exercise these rights towards fx-its via support@fx-its.de. You have the right to object to processing based on Article 6(1)(f) GDPR on grounds relating to your particular situation.

## 9. Right to Lodge a Complaint

You have the right to lodge a complaint with a data protection supervisory authority. The authority responsible for fx-its is: Landesbeauftragte für Datenschutz und Informationsfreiheit Nordrhein-Westfalen (LDI NRW), https://www.ldi.nrw.de.

## 10. Changes to This Statement

fx-its may update this privacy statement. The current version is always available at <https://mfr-fx.github.io/BC_Nexus-docs/legal/privacy-statement/>.

## 11. Contact

Questions about this privacy statement:  
**fx-its** — support@fx-its.de
