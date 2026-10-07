---
title: "Support — BC Nexus"
permalink: /support/
---

# Support — BC Nexus

**Publisher:** fx-its (Marco Frerix)  
**Support email:** support@fx-its.de  
**Support portal:** https://fx-its.atlassian.net/servicedesk/customer/portal/1  
**Documentation:** https://mfr-fx.github.io/BC_Nexus-docs/help/  
**Last updated:** 2026-10-07

---

## How to Get Support

### Step 1 — Check the documentation

Before contacting support:

- Read the [Setup Guide]({{ site.baseurl }}/help/setup-guide/) for configuration instructions.
- Read the [API Reference]({{ site.baseurl }}/help/api-reference/) for integration details.
- Browse the full documentation at https://mfr-fx.github.io/BC_Nexus-docs/help/ — your question may already be answered.

### Step 2 — Email support

Send an email to **support@fx-its.de** with the subject line `[BC Nexus] <brief description>` and include:

- BC Nexus version (visible in the Extension Management page in Business Central)
- Business Central version (Help → About)
- Which Interface Type is affected (Receive / Send / Publish)
- Steps to reproduce the issue
- What you expected to happen
- What actually happened
- Any error messages from the Nexus Transaction List

Incomplete reports will be asked to provide the missing information before investigation begins.

Every email to this address is logged as a request in our ticket system (Jira Service Management).

### Step 3 — Ticket portal

As an alternative to email, open a request in the ticket portal: https://fx-its.atlassian.net/servicedesk/customer/portal/1

On your first request you create an account with your email address; afterwards you can follow all your requests there. Requests from the portal and from email (Step 2) are handled the same way.

---

## Response Times

| Issue type | Target first response |
|------------|----------------------|
| Critical bug (extension not loading / data loss risk) | 2 business days |
| Standard bug (documented feature broken) | 5 business days |
| Support / configuration question | 5 business days |
| Feature request | Reviewed, no guaranteed timeline |

Response times are best-effort commitments. For customers with active support agreements, different SLAs apply.

## Included Support and Services

Each licensed production environment includes **2 hours of support per contract year** through the portal or by email: configuration questions, analysis of integration errors and help with the documented features. Bug fixes in BC Nexus itself never count against these hours.

Beyond the included hours, support is billed at **120 EUR per hour**, either per request after your confirmation or as a prepaid hour contingent. Setting up interfaces for you is a separate service, offered on request and billed by effort at the same rate. Unused included hours do not carry over to the next contract year. All prices are final prices: as a small business under § 19 UStG, fx-its does not charge VAT.

---

## Warranty Coverage

For the first 90 days after installation, bugs in core functionality are covered under the [Limited Warranty]({{ site.baseurl }}/warranty/). After that period, support continues on a best-effort basis.

---

## Security Vulnerabilities

Report security vulnerabilities like any other request — by email to **support@fx-its.de** (Step 2) or in the [ticket portal](https://fx-its.atlassian.net/servicedesk/customer/portal/1) (Step 3). Keep the report confidential: do not copy anyone else on the email, keep the portal request private (do not share it with your organization or add participants), and do not disclose details publicly until a fix is available or we have agreed on a disclosure date. A request handled this way is visible only to you and to us.

**Subject line (email) / Summary (portal):** `[BC Nexus] Security: <brief description>`

---

## Out of Scope

The following are outside the support scope:

- Custom AL code written by the customer or third parties
- Issues caused by third-party extensions conflicting with BC Nexus
- Business Central platform bugs (report these to Microsoft)
- Consulting, implementation, or project services (contact support@fx-its.de for a separate engagement)
