---
title: "Privacy Statement — BC Nexus"
permalink: /legal/privacy-statement/
---

# Privacy Statement — BC Nexus

**Publisher:** fx-its (Marco Frerix)  
**Contact:** support@fx-its.de  
**Last updated:** 2026-09-13

---

## 1. Overview

BC Nexus is a Microsoft Dynamics 365 Business Central extension that enables configurable data exchange (import, export, publish) between Business Central and external systems. This privacy statement explains what data the extension processes, how it is handled, and the rights of data subjects under the GDPR and applicable EU/German law.

## 2. Data Controller

The data controller for BC Nexus is the organization that installs and operates the extension in their Business Central environment. fx-its (the publisher) acts as a data processor only when delivering support services under a separate agreement.

## 3. Data Processed by the Extension

BC Nexus processes data solely at the direction of the installing organization. The extension itself does not collect, store, or transmit data to fx-its or any third party outside the configured integration targets. Specifically:

- **Business Central table data** — the extension reads from and writes to tables as configured by the administrator (field mappings, endpoints).
- **OAuth tokens and credentials** — stored in Business Central using the platform's `SecretText` / Isolated Storage mechanisms; never written to plain text or transmitted outside the configured endpoint.
- **Endpoint configuration** — base URLs, HTTP method settings, and mapping rules are stored in dedicated BC Nexus setup tables within the customer's Business Central tenant.
- **Transaction and error logs** — entry records (Nexus Entries) are stored in the customer's own database for auditing and troubleshooting.

## 4. Data Retention

All data processed by BC Nexus remains within the customer's Business Central environment. Retention periods are governed by the customer's own policies and Business Central's built-in mechanisms. No data is sent to fx-its or any fx-its-operated system.

## 5. Data Sharing and Third-Party Transfers

BC Nexus makes HTTP calls to external endpoints **only** as configured explicitly by the customer administrator. The extension's own code does not analyze or share the customer's Business Central data with fx-its or third parties.

**Telemetry.** <!-- Legal review pending: telemetry wording (Marco) -->
The Business Central platform itself sends extension telemetry — errors, lifecycle events, and web service call metadata including tenant and environment identifiers, no business data — to the publisher's Application Insights resource in the Azure region Germany West Central. This is a platform mechanism, configured via `applicationInsightsConnectionString` in the app manifest, not code added by BC Nexus. Purpose: diagnostics and customer support. Retention: as configured in Azure Monitor; default 90 days. No record contents of the customer's Business Central data are transmitted through this channel.

## 6. Security Measures

- Credentials are stored using Business Central's `SecretText` and Isolated Storage — not in plain database fields.
- All HTTP communication uses the endpoint URLs and TLS settings as configured by the customer.
- Source code is not exposed in the distributed symbol file (`resourceExposurePolicy` is restricted).

## 7. GDPR Rights

Data subjects whose personal data is processed through a BC Nexus integration may exercise their rights (access, rectification, erasure, portability, objection) by contacting the data controller — i.e., the organization that operates the Business Central environment. fx-its is not in a position to respond to such requests directly unless contracted as a processor.

## 8. Changes to This Statement

fx-its may update this privacy statement. The current version is always available at <https://mfr-fx.github.io/BC_Nexus-docs/legal/privacy-statement/>.

## 9. Contact

Questions about this privacy statement:  
**fx-its** — support@fx-its.de
