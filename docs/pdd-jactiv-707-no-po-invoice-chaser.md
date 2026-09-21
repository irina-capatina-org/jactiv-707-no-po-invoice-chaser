# PDD - No-PO Invoice Chaser

## Document History

| Date | Version | Author | Role | Comments |
|---|---|---|---|---|
| 2026-09-21 | 0.1 | uipath-analyst | Analyst | Initial PDD created from docs/jactiv-707-request-details.docx. |

| Date | Version | Author | Role | Comments |
|------|---------|--------|------|----------|
| 2026-09-21 | 0.1 | uipath-analyst | Analyst | Initial analysis from request-work/request-details.md (converted from docs/jactiv-707-request-details.docx, source version 1.0 dated 16 September 2026 by UiPath Cartographer) |

## 1. Document Control

| Field | Value |
|-------|-------|
| Document | PDD - No-PO Invoice Chaser |
| Story key | JACTIV-707 |
| Epic key | [SME REVIEW] |
| Source file | request-work/request-details.md |
| Branch | analysis-jactiv-707 |
| Author | uipath-analyst |
| Status | Draft - pending SME approval |
| Version | 0.1 |

## 2. Introduction

**Process name:** No-PO Invoice Chaser

| Field | Value |
|-------|-------|
| Process Full Name | NoPoInvoiceChaser |
| Business objective | Replace the daily manual Coupa review with a fully automated weekday control that identifies invoices with no properly linked purchase order and notifies the AP responsible by Slack, eliminating handling effort and ensuring consistent, timely visibility of missing-PO exceptions under the no-PO-no-pay policy |
| Owning department | Accounts Payable (Finance) |

**Delivery Team**

| Role | Name / Contact |
|------|---------------|
| SME / Process Owner | Irina Capatina (irina.capatina@uipath.com) |
| BA | uipath-analyst |
| Developer | [SME REVIEW] |

## 3. Process Overview

| Attribute | Value |
|-----------|-------|
| Process full name | NoPoInvoiceChaser |
| Function and department | Accounts Payable – Finance |
| Short description | Automated weekday control that queries Coupa for invoices with status draft or new dated within the past seven days, excludes credit notes and invoices with a properly linked PO, counts the remainder, and sends one Slack direct message to the SME with the count and a filtered Coupa link; sends nothing when the count is zero |
| Required roles | Unattended robot (scheduler-triggered); SME receives output |
| Trigger and schedule | Weekday schedule at 10:00 Romania time (Europe/Bucharest) |
| Volume (items per day / peak) | Up to 194 invoices evaluated per run (sample value from live run cited in source); typical qualifying count [SME REVIEW] |
| Average handling time (manual vs automated target) | Manual: approximately one review per day at unspecified duration [SME REVIEW]; Automated target: short linear run — read, filter, format, send |
| FTE effort | [SME REVIEW] |
| Estimated exception rate | Low; data is structured and rules are deterministic per source |
| Input data | Coupa invoice list: status, invoice date, invoice type, PO-linkage field on invoice lines, invoice date window (today minus 7 days to today) |
| Output data | Slack direct message to irina.capatina@uipath.com containing qualifying invoice count and filtered Coupa URL; or no message if count is zero |

## 4. Scope

**In scope**

- Weekday execution triggered at 10:00 Romania time (Europe/Bucharest)
- Querying the Coupa invoice list for invoices with status `draft` or `new`
- Filtering to invoice date within the past seven calendar days
- Excluding credit notes from the qualifying population
- Detecting missing or invalid PO linkage on invoice lines (a PO number in the description field only does not satisfy the requirement)
- Counting qualifying invoices
- Composing and sending one Slack direct message to the SME (Irina Capatina) containing the count, a policy reminder, an action request, and a filtered Coupa link
- Suppressing the message on a clean run (count = 0) — **see Open Question OQ-01 for a documented contradiction**
- Reporting the run as failed when the automation cannot complete

**Out of scope**

- Purchase-order creation in any system
- Invoice approval or payment release in Coupa
- Modification of any Coupa record
- Direct communication with suppliers
- Requester follow-up tracking or any second notification channel
- Full invoice lifecycle automation
- Audit-retention design
- Retry, fallback and error-recovery behaviour (per BR-08) — **see Open Question OQ-02 for a diagram contradiction**
- Listing individual invoices in the Slack message
- Any AI classification or human approval step in the normal path

## 5. To-Be Process (High Level)

The future process is a fully automated, unattended weekday control. At 10:00 Romania time the robot starts without human intervention. It reads the Coupa invoice list, applies a deterministic set of filters (date window, status, document type, PO-linkage), counts what qualifies, and sends a single Slack direct message to the AP responsible — or sends nothing if nothing qualifies.

**Steps that disappear in the automated process:**

- Manual daily review of the Coupa invoice list by an AP team member
- Manual field copying from Coupa into a message draft
- Manual grouping of invoice entries by requester
- Manual composition and sending of the Slack message
- Dependence on a supplier follow-up to prompt the review

**What stays human:**

- Acting on the notification: the SME (Irina Capatina) receives the message and ensures purchase orders are raised and linked for the identified invoices
- Requester engagement and PO-creation action (out of scope for the robot)

The automation does not classify documents with AI, does not require human approval at any step in the normal path, and does not modify Coupa data or create purchase orders. The process is a short linear sequence: schedule fires → query Coupa → filter → count → compose message → send to Slack.

## 6. Detailed Process Steps

| Step | Action | Application | Expected Result | Remarks |
|------|--------|-------------|-----------------|---------|
| 1.0 | Weekday schedule fires at 10:00 Romania time (Europe/Bucharest) | Orchestrator / scheduler | Run starts; execution context is initialised with today's date, window-start date (today minus 7 days), and window-end date (today) | Trigger is time-based, weekdays only. BR-04 governs the date window |
| 1.1 | Calculate date window: window_end = today (run date); window_start = today minus 7 calendar days | Automation | Date variables available for query and for constructing the Coupa URL | BR-04 |
| 2.0 | Query Coupa invoice list with filter: invoice_date >= window_start AND invoice_date <= window_end AND status IN (draft, new) | Coupa | Paginated list of invoice records matching the filter | Interface type and access method [SME REVIEW] — see section 7. Pagination strategy [SME REVIEW] |
| 2.1 | **Decision:** Is the Coupa response valid (no system error, parseable payload)? | Coupa | Branch: Yes → step 3.0; No → step S1 handler | Per future-state diagram. BR-08 states no retry is required; diagram contradicts this — see OQ-02 |
| 3.0 | For each invoice record returned: read invoice type field | Coupa | Invoice type value available for credit-note check | Start of per-invoice evaluation loop |
| 3.1 | **Decision:** Is invoice type = credit note? | Automation | Branch: Yes → step 3.2 (exclude); No → step 3.3 | BR-03 |
| 3.2 | Exclude record; record exclusion reason as "credit note" | Automation | Record dropped from qualifying set; exclusion reason logged | BR-03. Exclusion reason logging is inferred from section 7.1 "records the exclusion reason" |
| 3.3 | Read PO-linkage field on each invoice line | Coupa | PO linkage value(s) available for evaluation | PO linkage is held on invoice lines, not the invoice header — source section 4.3 |
| 3.4 | **Decision:** Is at least one invoice line properly linked to a purchase order? A PO number present only in the invoice description field does NOT satisfy this check | Automation | Branch: Yes → step 3.5 (exclude, has PO); No → step 3.6 (include, missing PO) | BR-01, BR-02 |
| 3.5 | Exclude record; invoice has a properly linked PO | Automation | Record dropped from qualifying set | Normal exclusion — no exception |
| 3.6 | Add record to qualifying set; increment qualifying count | Automation | Qualifying count += 1 | |
| 3.7 | **Loop end:** repeat steps 3.0–3.6 for next invoice record | Automation | All returned records evaluated | |
| 4.0 | **Decision:** Is qualifying count > 0? | Automation | Branch: Yes → step 5.0; No → step 4.1 | BR-07. **OQ-01:** section 4.3 and BR-07 say send nothing; section 7.1 says send a congratulations message — SME must resolve |
| 4.1 | **Clean day:** do not send any Slack message; end run successfully | Automation | Run completes; no Slack message sent | BR-07 (text) — pending OQ-01 resolution |
| 5.0 | Construct Coupa filtered URL using window_start and window_end: `https://uipath-test.coupahost.com/invoices?q%5Binvoice_date_gteq%5D=<window_start>&q%5Binvoice_date_lteq%5D=<window_end>&q%5Bstatus_eq%5D=draft` | Automation | URL string ready for inclusion in Slack message | URL pattern taken verbatim from source section 4.3. The URL filters only to draft; SME should confirm whether a combined draft+new URL is needed — OQ-03 |
| 5.1 | Compose Slack message body using the template from source section 4.3: qualifying count + policy sentence + action request + Coupa URL | Automation | Message string ready | BR-05, BR-06. Individual invoice lines are NOT listed — BR-05 |
| 5.2 | Send Slack direct message to Irina Capatina (irina.capatina@uipath.com) | Slack | Message delivered to SME's DM | BR-06. Slack interface type [SME REVIEW] — see section 7 |
| 5.3 | **Decision:** Was the Slack message delivered successfully? | Slack | Branch: Yes → step 6.0 (end); No → step S2 handler | |
| 6.0 | End run; record run as successful | Orchestrator | Run status = success; no further action | |

## 7. Applications and Systems

| Application | Interface type | Access method | Login method | Credential handling | Comments |
|-------------|---------------|---------------|--------------|--------------------|---------| 
| Coupa | API [SME REVIEW] | REST API or web UI scraping [SME REVIEW] | Service account or OAuth [SME REVIEW] | Orchestrator credential asset [DEFAULT] | Source states access type = read. Coupa URL pattern `uipath-test.coupahost.com` suggests a test tenant — production URL [SME REVIEW]. PO linkage is on invoice lines, not the header, which affects the query design |
| Slack | API [SME REVIEW] | Slack API / Bot token [SME REVIEW] | Bot OAuth token [SME REVIEW] | Orchestrator credential asset [DEFAULT] | Source states access type = write. Destination is a direct message to irina.capatina@uipath.com. Protocol/method (Incoming Webhook vs Bot API) [SME REVIEW] |
| Orchestrator / Scheduler | Orchestration | UiPath Orchestrator [SME REVIEW] | Robot service account [DEFAULT] | Managed by Orchestrator [DEFAULT] | Trigger is weekday schedule at 10:00 Europe/Bucharest. Delivery model [SME REVIEW] — see section 12 |

## 8. Business Rules

| ID | Rule | Source | Applies at step |
|----|------|--------|----------------|
| BR-01 | The no-PO-no-pay policy applies to invoices without a properly linked purchase order | Source BR-001 | 3.4 |
| BR-02 | A PO number typed into the invoice description but not properly linked to the invoice does not satisfy the PO requirement | Source BR-002 | 3.4 |
| BR-03 | Credit notes are excluded from the notification population | Source BR-003 | 3.1, 3.2 |
| BR-04 | Include only invoices with status `draft` or `new` and an invoice date within the past seven calendar days (relative to the run date) | Source BR-004 | 1.1, 2.0 |
| BR-05 | The Slack message reports the total count of qualifying invoices; individual invoices are not listed | Source BR-005 | 5.1 |
| BR-06 | The Slack message is sent to the SME (Irina Capatina, irina.capatina@uipath.com) and must include: the qualifying count, a sentence on why a linked PO matters under the no-PO-no-pay policy, a request to ensure a PO exists and is linked, and a filtered Coupa URL | Source BR-006 | 5.1, 5.2 |
| BR-07 | When a successful query returns zero qualifying invoices, no Slack message is sent | Source BR-007 | 4.0, 4.1 — **contradicted by source section 7.1; see OQ-01** |
| BR-08 | No retry, fallback or error-recovery behaviour is required; a run that cannot complete is reported as a failed run | Source BR-008 | All steps — **contradicted by future-state diagram; see OQ-02** |
| BR-09 | A run that cannot complete produces no notification; there is no second message path | Source BR-009 | S1, S2 handlers — **contradicted by future-state diagram which shows an error Slack message; see OQ-02** |
| BR-10 | The automation does not create or modify purchase orders, approve invoices, change Coupa records, or track requester completion | Source BR-010 | All steps |

## 9. Business Exceptions

| ID | Name | Trigger step | Trigger condition | Action |
|----|------|-------------|-------------------|--------|
| B1 | Credit note encountered | 3.1 | Invoice type field equals credit note | Exclude record from qualifying set; log exclusion reason as "credit note"; continue to next record |
| B2 | Description-only PO | 3.4 | PO number appears in description field only; no proper PO linkage on invoice lines | Treat as missing PO; include in qualifying set if other rules pass (BR-02) |
| B3 | Clean day — no qualifying invoices | 4.0 | Qualifying count = 0 after all records processed | Per BR-07 (text): end run successfully with no Slack message. Pending OQ-01: source section 7.1 states a congratulations message should be sent instead — SME must resolve before build |

## 10. System Errors

| ID | Name | Trigger condition | Severity | Retry policy | Action |
|----|------|------------------|----------|-------------|--------|
| S1 | Coupa query failure | Coupa returns an error response, times out, or returns an unparseable payload (step 2.0–2.1) | High | BR-08 (text): no retry required. Future-state diagram shows retry up to 3 attempts — see OQ-02; apply [SME REVIEW] | Per BR-08/BR-09: report run as failed; send no notification. If OQ-02 resolves to allow retries, implement per diagram |
| S2 | Slack delivery failure | Slack API returns an error or message undeliverable (step 5.2–5.3) | High | BR-08 (text): no retry. Future-state diagram shows retry up to 3 attempts then send error Slack message — see OQ-02; apply [SME REVIEW] | Per BR-08/BR-09: report run as failed; send no further notification |
| S3 | Application unresponsive | Target application (Coupa or Slack) is unreachable or the session cannot be established | High | [DEFAULT] No retry per BR-08; mark run failed | Log failure; end process; report run as failed in Orchestrator |
| S4 | Credential expiry | Stored credential for Coupa or Slack is rejected during authentication | High | [DEFAULT] No retry | Log failure with credential reference; end process; report run as failed |
| S5 | Unhandled exception | Any unexpected runtime exception not caught by a specific handler | High | [DEFAULT] No retry | Log full exception details; end process; report run as failed in Orchestrator |

## 11. Data Definitions

| Field | Type | Source | Target | Validation | Required |
|-------|------|--------|--------|-----------|---------|
| invoice_date | Date | Coupa invoice record | Filter condition | Must be >= window_start and <= window_end | Yes |
| status | String / Enum | Coupa invoice record | Filter condition | Must be `draft` or `new` | Yes |
| invoice_type | String / Enum | Coupa invoice record | Credit-note check (step 3.1) | If value = credit note, exclude record | Yes |
| po_linkage | Reference / Object | Coupa invoice **line** (not header) | PO-linkage check (step 3.4) | Must be a proper linked PO reference; text in description field only does not satisfy | Yes |
| invoice_description | String | Coupa invoice record | PO-linkage check (step 3.4) | Read-only; a PO number here without line linkage is treated as missing | No |
| qualifying_count | Integer | Computed by automation | Slack message body | >= 0; if 0 → no message (pending OQ-01) | Yes |
| window_start | Date | Computed: run date minus 7 days | Coupa query, Coupa URL | Must be a valid date; format [SME REVIEW] | Yes |
| window_end | Date | Computed: run date | Coupa query, Coupa URL | Must be a valid date; format [SME REVIEW] | Yes |
| slack_recipient | String (email / user ID) | Hardcoded per BR-06 | Slack API | irina.capatina@uipath.com | Yes |
| slack_message_body | String | Composed at step 5.1 | Slack API | Must contain: count, policy sentence, action request, Coupa URL | Yes |
| coupa_filtered_url | String (URL) | Constructed at step 5.0 | Slack message body | Pattern: `https://uipath-test.coupahost.com/invoices?q%5Binvoice_date_gteq%5D=<window_start>&q%5Binvoice_date_lteq%5D=<window_end>&q%5Bstatus_eq%5D=draft` | Yes |
| exclusion_reason | String | Automation | Log / audit trail | "credit note" for B1 exclusions | No |

**Coupa URL pattern (verbatim from source section 4.3):**

`https://uipath-test.coupahost.com/invoices?q%5Binvoice_date_gteq%5D=<window_start>&q%5Binvoice_date_lteq%5D=<window_end>&q%5Bstatus_eq%5D=draft`

Note: The URL filters to status `draft` only. Whether a combined draft+new filter is achievable and needed is OQ-03.

**Slack message template (verbatim from source section 4.3):**

> "194 invoices from the last seven days have no purchase order linked. An invoice without a linked PO cannot be matched or paid under our no-PO-no-pay policy, and payment to the supplier stalls until it is fixed. Please make sure a purchase order exists for these invoices and is correctly linked to each one. Open the list in Coupa: https://uipath-test.coupahost.com/invoices?q%5Binvoice_date_gteq%5D=\<window start\>&q%5Binvoice_date_lteq%5D=\<window end\>&q%5Bstatus_eq%5D=draft"

The count `194` is the sample value from the live run cited in the source. The message template substitutes the actual qualifying count and computed dates at runtime.

## 12. Environment and Constraint Signals

| Attribute | Signal |
|-----------|--------|
| Delivery model | [SME REVIEW] — Automation Cloud, Automation Suite or standalone UiPath not stated in source |
| Product exclusions | No AI classification, no Document Understanding, no human-in-the-loop step required (source section 4.4 explicit) |
| Orchestration constraints | Weekday schedule trigger at 10:00 Europe/Bucharest required; single unattended robot run |
| Document storage | Not applicable — process produces no stored documents; Coupa is the system of record |
| Signing modality | Not applicable — no document signing in scope |
| Robot attendance | Unattended — triggered by schedule, requires no human interaction during the run (source section 4.1, 4.4) |

## 13. Canonical Test Data

| Field | Value | Role | Source location |
|-------|-------|------|----------------|
| qualifying_count (sample) | 194 | Input / validation — live sample count cited in source | Source section 4.3 notification content example |
| slack_recipient | irina.capatina@uipath.com | Expected output — hardcoded recipient | Source sections 4.3, 5 (BR-006), 6, 8 |
| coupa_base_url | https://uipath-test.coupahost.com | Input / validation — test tenant base URL | Source section 4.3 |
| coupa_url_date_gte_param | q%5Binvoice_date_gteq%5D | Validation — URL query parameter for window start | Source section 4.3 |
| coupa_url_date_lte_param | q%5Binvoice_date_lteq%5D | Validation — URL query parameter for window end | Source section 4.3 |
| coupa_url_status_param | q%5Bstatus_eq%5D=draft | Validation — URL query parameter for status | Source section 4.3 |
| invoice_status_include_1 | draft | Input — qualifying status value | Source BR-004, section 8 acceptance criteria |
| invoice_status_include_2 | new | Input — qualifying status value | Source BR-004, section 8 acceptance criteria |
| invoice_type_exclude | credit note | Input — document type triggering exclusion | Source BR-003, section 7.1 |
| date_window_days | 7 | Input — number of calendar days in the lookback window | Source BR-004, section 4.3 |
| schedule_time | 10:00 | Input — trigger time | Source section 1.2, 4.2, 8 acceptance criteria |
| schedule_timezone | Europe/Bucharest (Romania time) | Input — timezone for schedule | Source section 1.2, 4.2 |
| slack_message_sample | "194 invoices from the last seven days have no purchase order linked. An invoice without a linked PO cannot be matched or paid under our no-PO-no-pay policy, and payment to the supplier stalls until it is fixed. Please make sure a purchase order exists for these invoices and is correctly linked to each one. Open the list in Coupa: https://uipath-test.coupahost.com/invoices?q%5Binvoice_date_gteq%5D=\<window start\>&q%5Binvoice_date_lteq%5D=\<window end\>&q%5Bstatus_eq%5D=draft" | Expected output — full message template with live sample count | Source section 4.3 |
| sme_name | Irina Capatina | Validation — named SME recipient | Source sections 4.3, 6.1, 8, 9 |

## 14. Decomposition Signals

- **Distinct processing stages:** Three clear stages — (1) Coupa query and ingestion, (2) per-invoice filtering and counting, (3) Slack notification composition and delivery. These could be separate workflow sequences or a single linear sequence given the low volume and simple rules.
- **Per-item transactional processing:** Yes — each invoice record is evaluated individually against credit-note and PO-linkage rules (steps 3.0–3.7). Volume is low (up to 194 records per sample run); a simple in-memory loop is sufficient; no queue-based transactional pattern is indicated.
- **Document Understanding with human validation:** Not present in source. Source explicitly excludes AI classification and human approval (section 4.4).
- **Multiple output channels:** Single output channel only — one Slack direct message to one recipient. No email, no report file, no secondary channel.
- **Reporting:** Not present in source. Run success or failure is reported via Orchestrator job status only.
- **Queue / batch mentions:** Not present in source. The source describes a single daily linear run, not a queued batch pattern.

## 15. Assumptions, Dependencies and Open Questions

1. **OQ-01 [SME REVIEW] — Contradiction: clean-day behaviour.** Source section 4.3 (notification content table, "On a clean day") and BR-07 both state "Nothing is sent when the count is zero." Source section 7.1 (Known Exceptions table) states "Send a slack message saying congrats that all invoices have a po assigned." These are directly contradictory. The PDD applies BR-07 (no message) as the default pending SME resolution. SME must confirm which behaviour is required before build.

2. **OQ-02 [SME REVIEW] — Contradiction: retry and error notification behaviour.** Source section 4.4 and BR-08 explicitly state "No retry, fallback or recovery behaviour is required" and "a run that cannot complete is simply reported as a failed run." Source BR-09 states "A run that cannot complete produces no notification." The future-state diagram (image2.png) contradicts this by showing: retry Coupa query up to 3 attempts, retry Slack message up to 3 attempts, and a "Send error message" Slack step after 3 failed retries. The PDD applies the text rules (BR-08, BR-09) as the authoritative source. SME must confirm which behaviour is required. If the diagram is authoritative, retry counts (3) and error message content must be defined.

3. **OQ-03 [SME REVIEW] — Coupa URL status filter.** The sample URL in source section 4.3 filters to `status_eq=draft` only. BR-04 includes both `draft` and `new`. SME to confirm whether the Coupa filtered link should cover both statuses, and whether a combined URL parameter exists for this.

4. **OQ-04 [SME REVIEW] — Coupa access interface.** Source section 2.2 states access type = read but does not specify whether the integration uses the Coupa REST API, a UI web scraping approach, or another mechanism. SME / technical team to confirm and provide API endpoint details or access credentials.

5. **OQ-05 [SME REVIEW] — Coupa production URL.** The sample URL uses `uipath-test.coupahost.com`. The production tenant URL is not stated. SME to confirm.

6. **OQ-06 [SME REVIEW] — Slack integration method.** Source states Slack as the destination (write access) but does not specify the integration type: Slack Bot API, Incoming Webhook, or another mechanism. SME / technical team to confirm and provide token or webhook URL.

7. **OQ-07 [SME REVIEW] — Date format for Coupa URL parameters.** The sample URL uses placeholder tokens `<window start>` and `<window end>`. The required date format for the Coupa query parameters (ISO 8601, YYYY-MM-DD, or other) is not specified. SME to confirm.

8. **OQ-08 [SME REVIEW] — Slack message date and amount format.** Source section 9 (Next Steps item 3) lists "Agree the final reminder wording and a readable date and amount format" as an open action. Final wording and formatting convention have not been confirmed.

9. **OQ-09 [SME REVIEW] — FTE effort figure.** The source states the review is "approximately one per day" but does not quantify time per run. Required for the Process Overview AHT row.

10. **OQ-10 [SME REVIEW] — UiPath delivery model.** Automation Cloud, Automation Suite or standalone is not stated. This gates licensing and infrastructure choices.

11. **OQ-11 [SME REVIEW] — Coupa pagination.** It is not confirmed whether the Coupa API or UI returns all results in a single response or paginates. The automation must handle pagination if results can exceed a page limit.

12. **OQ-12 [SME REVIEW] — PO-linkage field name.** The exact Coupa field name for PO linkage on invoice lines is not stated in the source. Developer must confirm with SME or Coupa API documentation.

13. **[DEFAULT]** Credentials for both Coupa and Slack will be stored as Orchestrator credential assets and not hardcoded in the workflow.

14. **[DEFAULT]** The run timezone is interpreted as Europe/Bucharest (UTC+2 standard / UTC+3 DST) consistent with "Romania time" in the source.

15. **Referenced-but-unavailable input:** Source section 9 (Next Steps) item 1 references confirmation of "the reminder wording and the Coupa link" — this confirmation is not included in the source document. Item 3 references agreement on "final reminder wording and a readable date and amount format" — not confirmed in the available document.

16. **Referenced-but-unavailable input:** Source "Appendix A: Process Map and Diagrams" refers to current-state and future-state maps embedded in sections 3.4 and 4.1 respectively. The source document as supplied contains these maps as image1.png (current state) and image2.png (future state), both of which were readable. No further appendix or attachment content was referenced but absent.

## 16. Success Criteria

1. A scheduled weekday run executes at 10:00 Europe/Bucharest time and does not execute on Saturday or Sunday.
2. Only invoice records with status `draft` or `new` are evaluated; records with any other status are not included.
3. Only invoice records with an invoice date within the past seven calendar days (inclusive of today's run date) are evaluated; records outside this window are not included.
4. Invoice records with type = credit note are excluded from the qualifying set; at least one credit-note record in test data must be demonstrably absent from the Slack message count.
5. An invoice record that has a PO number in the description field only (no proper PO linkage on invoice lines) is counted as missing a PO and included in the qualifying count.
6. An invoice record with a properly linked PO on at least one invoice line is excluded from the qualifying count.
7. The Slack direct message is sent to irina.capatina@uipath.com and to no other recipient.
8. The Slack message body contains: the total qualifying invoice count, a sentence referencing the no-PO-no-pay policy, a request to ensure a PO is raised and linked, and a working hyperlink to the Coupa invoice list filtered to the same seven-day window used by the run.
9. Individual invoice identifiers are not listed in the Slack message body.
10. When the qualifying count is zero and OQ-01 is resolved as "send nothing": no Slack message is sent, and the run completes with success status.
11. A run that encounters a system error (Coupa unreachable, Slack delivery failure, credential error, unhandled exception) is reported as a failed run in Orchestrator and produces no business notification (pending OQ-02 resolution).
12. The automation makes no modifications to any Coupa record and does not create or modify any purchase order; post-run Coupa audit log must show no writes from the robot service account.
