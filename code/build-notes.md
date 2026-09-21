# Build Notes — NoPoInvoiceChaser

Single API Workflow that queries Coupa for invoices without a linked PO (draft/new, past 7 days), filters in-memory, and sends one Slack DM to the AP SME when qualifying invoices exist.

## Task Status

| Task | Project | Status | Notes |
|---|---|---|---|
| T1 — Verify IS connections | NoPoInvoiceChaser | done | Both connections confirmed as program infrastructure facts in org architectural considerations §4; no cloud login needed — confirmed by the note dated 2026-09-21 in §4 |
| T2 — Build NoPoInvoiceChaser workflow | NoPoInvoiceChaser | done | All 7 SDD steps implemented; validate passes (1 expected warning — empty #Else branch on clean-day path, by design per BR-07) |
| T3 — Testing | NoPoInvoiceChaser | partial | Static validation passes; live run requires IS connections and would send real Slack DMs — not executed in this runner (no credentials) |
| T4 — Pack and publish | NoPoInvoiceChaser | partial | `uip solution pack` succeeds locally (`jactiv-707-no-po-invoice-chaser_0.0.1.zip`); publish and schedule activation require `uip login` — not executed in this runner |

## Deviations from the SDD

None. All seven steps from §4, all business rules (BR-01 through BR-10), all error scenarios (S1–S5) are implemented as specified. The SDD's OQ defaults are applied verbatim:

- OQ-01: No message on clean day (BR-07, `If_1#Else` branch is empty)
- OQ-02: No retry, no error Slack notification; single catch at workflow edge logs + rethrows
- OQ-03: Coupa URL uses `status_eq=draft` only, verbatim from SDD §4 Step 5

## Left for a human

| Item | File | SDD Section | Notes |
|---|---|---|---|
| Coupa pagination | `NoPoInvoiceChaser/Workflow.json` | §1 Assumptions, OQ-11 | `limit=50` is set per SDD. If Coupa returns paginated responses and the invoice count exceeds 50, a pagination loop must be added. Developer must verify Coupa API page-size limit against the live tenant. |
| PO-linkage field name | `NoPoInvoiceChaser/Workflow.json` (`Javascript_FilterAndCount`) | §4 Step 3, OQ-12 | Implementation uses `invoice-lines[].po-number` and `invoice-lines[].order-header-num` per org §4 live-run note. Developer must confirm these are the correct field names against the Coupa API schema for this tenant. |
| Slack channel identifier | `NoPoInvoiceChaser/Workflow.json` (`HTTP_Request_Slack`) | §4 Step 7 | The `channel` field uses `irina.capatina@uipath.com` (BR-06). Developer must confirm the `slack-product-test-app` connection has `chat:write` scope and that the Slack API accepts an email as the channel identifier for this workspace. |
| Live run + test execution | — | §8 Testing Strategy | T-01 through T-05, T-B1 through T-B3, T-S1 through T-S5 require IS connections and would trigger real Slack DMs. Execute in a credentialed environment after deployment. |
| Publish and schedule activation | — | §7 Deployment Target | `uip login` → `uip solution publish` → activate weekday 10:00 Europe/Bucharest schedule in `Fusion2026`. |
| Open OQ items | — | §15 / §9 | OQ-01, OQ-02, OQ-03, OQ-05, OQ-11, OQ-12 — all carry SDD defaults; SME confirmation needed before production sign-off. |

## How to test this

```bash
# Static validation (offline, no credentials needed)
uip api-workflow validate code/jactiv-707-no-po-invoice-chaser/NoPoInvoiceChaser/Workflow.json --output json
# Expected: Result: Success, Status: Valid

# Pack the solution (offline, no credentials needed)
uip solution pack code/jactiv-707-no-po-invoice-chaser /tmp/buildcheck \
  --name jactiv-707-no-po-invoice-chaser --version 0.0.1 --output json
# Expected: Result: Success, Code: SolutionPack

# Live run (requires credentials; WILL send a real Slack DM if qualifying invoices exist)
# uip login
# uip api-workflow run code/jactiv-707-no-po-invoice-chaser/NoPoInvoiceChaser/Workflow.json --output json
```
