---
name: myome-track-order-and-retrieve-report
description: Poll a MyOme order through the laboratory lifecycle and retrieve the finished report. Read-only and safe to run unattended.
api: MyOme API
generated: '2026-08-26'
method: generated
source: openapi/myome-openapi.json
operations:
  - myome.api.endpoints.requisition.requisition_list
  - myome.api.endpoints.requisition.requisition_get
  - myome.api.endpoints.order.order_get
---

# Track a MyOme order and retrieve its report

Base URL `https://api.myome.com/0/`. Every operation here is a GET — this skill is safe to run
unattended, unlike `myome-order-genomic-test`.

## Steps

1. **Find the requisitions in flight.** `GET /requisition` (operationId
   `myome.api.endpoints.requisition.requisition_list`). Paginate with `page` and `per_page` (both
   integers >= 1) and sort with `sort_field` plus `sort_order` (`ascending` / `descending`). Filter
   with `clinician_id`, `clinician_name`, `external_mrn`, `patient_id`, `patient_name`, `status` or
   `submitted_at`. There is no cursor and no total-count header, so page until a short page comes back.

2. **Open one requisition.** `GET /requisition/{requisition_id}` (operationId
   `myome.api.endpoints.requisition.requisition_get`), with the `RQ`-prefixed identifier. The response
   carries the subject, clinicians, consents, samples and the `orders` array with an `OR`-prefixed
   `order_id` per product.

3. **Poll each order.** `GET /order/{order_id}` (operationId
   `myome.api.endpoints.order.order_get`). Read `external_status`:

   | Status | Meaning | What to do |
   |---|---|---|
   | `SUBMITTED` | Order received | Keep polling |
   | `APPROVED` | Approved to proceed | Keep polling |
   | `AWAITING_SAMPLE` | Specimen has not arrived at the lab | Chase the specimen, not the API |
   | `ANALYZING` | Being processed | Keep polling |
   | `REFERRAL_LAB_LD_REVIEW` | In referral-lab review | Keep polling |
   | `CLINICIAN_REVIEW` | Results sent to the clinician, awaiting review | Keep polling; a human gate, not a lab delay |
   | `COMPLETED` | Done | Go to step 4 |
   | `DENIED` | Denied by the physician of record | Terminal. Stop polling |
   | `CANCELED` | Canceled by MyOme | Terminal. Read `status_reason` (e.g. `SAMPLE_NOT_RECEIVED`) and `status_message` |
   | `FAILED` | Order failed | Terminal. Read `status_reason` and escalate |

   Poll on a slow interval measured in hours. Turnaround is gated on physical specimen receipt and
   laboratory processing, MyOme publishes no SLA, and there are no rate-limit headers to guide you.
   Stop polling the moment you hit a terminal status.

4. **Retrieve the report.** A `COMPLETED` order exposes result access on the order response — report
   URIs with a `mimetype` and `description`, alongside sequencing data and structured interpretation
   data. `report_type` is one of BASELINE_RISK, MONOGENIC, NDD_CNA, RARE_DISEASE, PGX, PRS,
   FAMILY_VARIANT_TESTING, DATA_ONLY or CLINICAL_RISK_ONLY. Result URIs are pre-signed and
   short-lived: download promptly and never store or forward the URI itself as if it were a
   permanent link. An order normally has one report, zero if it failed, and several when a report has
   been amended or corrected — so take the latest, and check for more than one.

5. **Handle the contents as PHI.** Reports carry identified genomic findings. Deliver them to the
   destination your operator configured and nowhere else. Do not summarise clinical findings into a
   log, a ticket or a chat transcript.
