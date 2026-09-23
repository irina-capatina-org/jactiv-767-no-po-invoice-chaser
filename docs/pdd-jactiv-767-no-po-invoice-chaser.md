# PDD - No-PO Invoice Chaser

## Document History

| Date | Version | Author | Role | Comments |
|------|---------|--------|------|----------|
| 2026-09-23 | 0.1 | uipath-analyst | Analyst | Initial analysis from request-work/request-details.md (UiPath Cartographer, v1.0, 16 Sep 2026) |

## 1. Document Control

| Field | Value |
|-------|-------|
| Document | PDD - No-PO Invoice Chaser |
| Story key | JACTIV-767 |
| Epic key | [SME REVIEW] |
| Source file | request-work/request-details.md |
| Branch | analysis-jactiv-767 |
| Author | uipath-analyst |
| Status | Draft - pending SME approval |
| Version | 0.1 |

## 2. Introduction

| Field | Value |
|-------|-------|
| Process name | No-PO Invoice Chaser |
| Process full name | NoPoInvoiceChaser |
| Business objective | Replace the daily manual Coupa review with a fully automated weekday run that identifies invoices with no linked PO and notifies the AP SME via Slack. |
| Owning department | Accounts Payable |

**Delivery Team**

| Role | Name / Contact |
|------|---------------|
| SME / Process Owner | Irina Capatina (irina.capatina@uipath.com, Slack ID WLX9BD8FN) |
| BA | uipath-analyst |
| Developer | [SME REVIEW] |

## 3. Process Overview

| Attribute | Value |
|-----------|-------|
| Process full name | NoPoInvoiceChaser |
| Function and department | Accounts Payable — invoice compliance monitoring |
| Short description | Queries Coupa daily for draft/new invoices in the past seven days with no properly linked PO, then sends one Slack DM to the AP SME with the count and a filtered Coupa link. |
| Required roles | Automation (unattended); SME receives notification only |
| Trigger and schedule | Weekday schedule, 10:00 Romania time (EET/EEST) |
| Volume (items per day / peak) | ~1 run/day; sample of 194 qualifying invoices cited in source |
| Average handling time | Manual: ~daily ad-hoc task; Automated target: <2 min [DEFAULT] |
| FTE effort | [SME REVIEW] |
| Estimated exception rate | Low — credit notes and description-only POs are deterministic exclusions |
| Input data | Coupa invoice list (status, invoice date, PO linkage, invoice type) |
| Output data | Slack Block Kit DM to SME with invoice count, policy note, action request, and filtered Coupa URL |

## 4. To-Be Process (High Level)

The automation runs unattended each weekday at 10:00 Romania time. It queries Coupa for invoices dated within the past seven days, retains only those with status **draft** or **new**, excludes credit notes, and checks whether each invoice has a properly linked purchase order (a PO number present only in the description field does not qualify). If qualifying invoices exist, the robot composes one Slack Block Kit direct message to the SME containing the total count, a policy reminder, an action request, and a link to the filtered Coupa invoice list, then sends it. If the count is zero, no message is sent.

**Steps eliminated from the manual process:**
- Manual Coupa filter and scroll
- Manual field copy and grouping by requester
- Manual Slack message drafting and sending
- Timing dependency on the AP team member's availability

**Stays human:** SME acts on the notification — raises or links the missing POs. Requester follow-up tracking is out of scope.

## 5. Detailed Process Steps

| Step | Action | Application | Expected Result | Remarks |
|------|--------|-------------|-----------------|---------|
| 1.1 | Scheduler triggers run at 10:00 Romania time on weekdays | Scheduler | Process starts | EET (UTC+2) / EEST (UTC+3); Saturday and Sunday excluded (BR-04) |
| 1.2 | Calculate date window: window_end = today, window_start = today − 7 days | Automation | Two date values available | Used in Coupa query and in the Slack message footer |
| 2.1 | Query Coupa invoice list with filter: invoice_date >= window_start AND invoice_date <= window_end AND status IN (draft, new) | Coupa | Paginated list of candidate invoices | Read-only access; BR-04 |
| 2.2 | For each invoice: check invoice type — if credit note, skip | Automation | Credit notes removed from candidate list | BR-03; log exclusion reason |
| 2.3 | For each remaining invoice: inspect PO linkage field — if no properly linked PO, mark as qualifying | Automation | Set of qualifying invoices | BR-01, BR-02; a PO number in the description field only does NOT satisfy linkage |
| 2.4 | Count qualifying invoices → invoice_count | Automation | Integer ≥ 0 | BR-05 |
| 3.1 | If invoice_count = 0, end run — send nothing | Automation | Run completes cleanly, no Slack message | BR-07; **DECISION POINT** — note: section 7.1 of source specifies a congratulatory message on clean day; contradicts BR-07 — see OQ-01 |
| 3.2 | Build Coupa filtered URL: base URL + invoice_date_gteq=window_start + invoice_date_lteq=window_end + status_eq=draft | Automation | coupa_url string | Source example: https://uipath-test.coupahost.com/invoices?q%5Binvoice_date_gteq%5D=...&q%5Bstatus_eq%5D=draft; PO linkage not filterable by URL (BR-05 note) |
| 3.3 | Compose Slack Block Kit payload: title ":receipt: {{invoice_count}} invoices need a purchase order"; body lines with :warning:, :no_entry:, :point_right: icons; primary button "Open the list in Coupa" linking to coupa_url; footer with window_start, window_end, run_date | Automation | Block Kit JSON payload ready | Recipient: Slack DM to member ID WLX9BD8FN (Irina Capatina); BR-06; exact JSON in source section 4 architectural notes |
| 4.1 | Send Slack DM to SME (Slack member ID WLX9BD8FN) via Slack HTTP Request activity | Slack | Message delivered; HTTP 200 response | BR-06; direct message, not channel post |
| 4.2 | Log run outcome (count, run_date, delivery status) | Automation | Run log entry written | [DEFAULT] |
| 4.3 | End run | Automation | Process completes | |

## 6. Applications and Systems

| Application | Interface type | Access method | Login method | Credential handling | Comments |
|-------------|---------------|---------------|--------------|---------------------|----------|
| Coupa | API | REST API (read-only) | API key / OAuth [SME REVIEW] | Orchestrator credential asset [DEFAULT] | Source URL: uipath-test.coupahost.com; confirm prod URL with SME |
| Slack | API | HTTP POST (Slack Incoming Webhook or Bot API) | Bot token [SME REVIEW] | Orchestrator credential asset [DEFAULT] | Recipient addressed by Slack member ID WLX9BD8FN; Block Kit payload; protocol = Slack Web API [SME REVIEW] |
| UiPath Orchestrator | Scheduler / credential store | Orchestrator | Robot machine credential | N/A | Trigger at 10:00 Romania time weekdays |

## 7. Business Rules

| ID | Rule | Source | Applies at step |
|----|------|--------|-----------------|
| BR-01 | Apply no-PO-no-pay policy: invoices without a properly linked PO qualify for notification. | BR-001 | 2.3 |
| BR-02 | A PO number typed into the invoice description only (not formally linked) does not satisfy the PO requirement. | BR-002 | 2.3 |
| BR-03 | Exclude credit notes from the qualifying population. | BR-003 | 2.2 |
| BR-04 | Include only invoices with status draft or new and an invoice date within the past seven days. | BR-004 | 2.1 |
| BR-05 | Report the count of qualifying invoices; individual invoices are not listed in the message. | BR-005 | 2.4, 3.3 |
| BR-06 | Send the Slack DM to Irina Capatina (Slack ID WLX9BD8FN) with count, policy note, action request, and Coupa link. | BR-006 | 3.3, 4.1 |
| BR-07 | Send nothing when a successful query returns zero qualifying invoices. | BR-007 | 3.1 |
| BR-08 | No retry, fallback or recovery behaviour is required; a run that cannot complete is reported as a failed run. | BR-008 | 4.2 |
| BR-09 | A run that cannot complete produces no notification. | BR-009 | 4.2 |
| BR-10 | The automation does not create or modify POs, approve invoices, change Coupa records, or track requester completion. | BR-010 | All |

## 8. Business Exceptions

| ID | Name | Trigger step | Trigger condition | Action |
|----|------|-------------|-------------------|--------|
| B1 | Credit note encountered | 2.2 | Invoice type = credit note | Exclude from qualifying set; log exclusion reason; continue |
| B2 | Description-only PO | 2.3 | PO text present in description field but no formal PO linkage | Treat as no linked PO; include in qualifying set if other rules pass (BR-02) |
| B3 | Zero qualifying invoices (clean day) | 3.1 | invoice_count = 0 after all filters | Send nothing (BR-07) — see OQ-01 for conflicting source instruction |
| B4 | Unhandled / catch-all exception | Any | Any unhandled error | Log error; mark run as failed; send no notification (BR-09) |

## 9. System Errors

| ID | Name | Trigger condition | Severity | Retry policy | Action |
|----|------|-------------------|----------|-------------|--------|
| S1 | Coupa API unresponsive | No HTTP response within timeout | High | No retry (BR-08) | Log; mark run failed; no Slack message sent |
| S2 | Coupa authentication failure | HTTP 401 / 403 from Coupa | High | No retry (BR-08) | Log; alert Orchestrator; mark run failed |
| S3 | Coupa unexpected response | Non-200 or malformed JSON from Coupa | High | No retry (BR-08) | Log; mark run failed |
| S4 | Slack delivery failure | Non-200 from Slack API | High | No retry (BR-08) | Log; mark run failed |
| S5 | Credential expiry | Orchestrator returns empty/null credential | High | No retry (BR-08) [DEFAULT] | Log; alert Orchestrator; abort |
| S6 | Unhandled exception | Any uncaught exception in workflow | High | No retry (BR-08) [DEFAULT] | Global handler logs stack trace; marks run failed |

## 10. Assumptions, Dependencies and Open Questions

1. **OQ-01 - Clean-day behaviour conflict.** [SME REVIEW] BR-07 says send nothing on a clean day; section 7.1 says send a congratulatory Slack message — confirm which applies.
2. **OQ-02 - Coupa API vs UI access.** [SME REVIEW] Confirm whether Coupa access is via REST API or UI automation, and provide the production hostname.
3. **OQ-03 - Coupa authentication method.** [SME REVIEW] Confirm API key, OAuth client-credentials, or another method for Coupa read access.
4. **OQ-04 - Slack integration method.** [SME REVIEW] Confirm whether the Slack DM is sent via Incoming Webhook, Bot token, or another method, and who provisions the credential.
5. **OQ-05 - Romania timezone handling.** [DEFAULT] Orchestrator trigger set to EET (UTC+2) / EEST (UTC+3); confirm Orchestrator machine timezone or use explicit UTC offset.
6. **OQ-06 - Coupa prod URL.** [SME REVIEW] Source cites uipath-test.coupahost.com; confirm production URL before go-live.
7. **OQ-07 - Block Kit JSON location.** Source references "exact Block Kit JSON in architectural considerations, section 4" — that section is not present in the supplied document; confirm whether a separate SDD appendix will carry it.
8. **OQ-08 - FTE saving baseline.** [SME REVIEW] Source does not quantify current FTE effort; provide if needed for business case sign-off.

## 11. Success Criteria

1. A weekday test run executes at 10:00 Romania time and completes without error.
2. Only invoices with status draft or new and invoice date within the past seven days are evaluated.
3. Credit notes are excluded from the qualifying count.
4. An invoice with a PO number only in the description field is counted as missing a linked PO.
5. One Slack DM is sent to Slack member ID WLX9BD8FN carrying the qualifying count and a working filtered Coupa link.
6. A successful run returning zero qualifying invoices sends no Slack message (pending OQ-01 resolution).
7. A run that cannot complete is recorded as failed and sends no Slack notification.
8. No Coupa record is created or modified during any run.
