---
name: myome-order-genomic-test
description: Submit a sequencing requisition to MyOme's clinical laboratory and confirm the orders it created. High-consequence, irreversible, and requires human confirmation before the write.
api: MyOme API
generated: '2026-08-26'
method: generated
source: openapi/myome-openapi.json
operations:
  - myome.api.endpoints.product.product_get
  - myome.api.endpoints.consent.consent_get
  - myome.api.endpoints.requisition.requisition_post
  - myome.api.endpoints.requisition.requisition_get
---

# Order a genomic test from MyOme

Base URL `https://api.myome.com/0/` (sandbox `https://api.sbx.myome.com/0/`).

## Before you start

- Authenticate with `Authorization: Bearer <jwt>`. The JWT comes from MyOme's Keycloak at
  auth.myome.com using partner credentials MyOme issued by secure email. It refreshes hourly, so
  refresh before a long-running flow rather than mid-flight.
- **This flow is irreversible.** `POST /requisition` creates a clinical laboratory order against a
  patient sample and is billable. MyOme publishes no cancel, void or refund operation and states no
  reversal window. Get explicit human confirmation before step 4, every time.
- **There is no idempotency key.** If a request times out, do NOT retry it. Go to step 5 and list
  requisitions to find out whether the first attempt landed.
- Everything you send and receive is identified PHI under HIPAA. Do not log request or response bodies.

## Steps

1. **List what you can order.** `GET /product` (operationId
   `myome.api.endpoints.product.product_get`). Returns the products available to your account, each
   with a `product_id` prefixed `PR`, a `name` and a `description`. Pick the `product_id` values the
   clinician actually ordered — do not infer a product from a test name.

2. **Find the consents you must record.** `GET /consent` (operationId
   `myome.api.endpoints.consent.consent_get`) with the required `consents` query parameter, an array
   of consent identifiers (e.g. `consents=DNAVISIT&consents=PR-2019`). Consent types are CONTACT,
   IRB, RESEARCH and TESTING. The response tells you which consent URIs to send on the requisition.

3. **Assemble the requisition.** You need the subject (name, demographics including date of birth,
   contact info and address), the ordering clinician (including their NPI, a 9- or 10-digit CMS
   National Provider Identifier), the granted consents, the sample, the product ids, and billing.
   Billing is polymorphic: `PayorType` is INSTITUTIONAL, INSURANCE or SELF, and the payload shape
   follows (BillingInstitutional, BillingInsurance with the Insurance and Relationship objects, or
   BillingSelf). If the product needs risk-model covariates — the Tyrer-Cuzick breast-cancer inputs,
   or the coronary-artery-disease, type-2-diabetes or prostate-cancer input sets — collect them here.
   Never invent a clinical value; leave it out and say what is missing.

4. **CONFIRM WITH A HUMAN, then submit.** `POST /requisition` (operationId
   `myome.api.endpoints.requisition.requisition_post`). A success returns `201 Created` with the
   `requisition_id` (prefixed `RQ`) plus the clinicians, consents granted, subject and the `orders`
   array — one Order per Product requested, each with an `order_id` prefixed `OR`. Record every
   identifier immediately; they are your only handle on the work.

5. **Read it back.** `GET /requisition/{requisition_id}` (operationId
   `myome.api.endpoints.requisition.requisition_get`) to confirm the orders and their initial status.
   Use this same call, not a retry, whenever you are unsure a submission landed.

## Handling failures

- `400` — the runtime returns `application/problem+json` (RFC 9457): `{type, title, detail, status}`.
  The spec instead declares `application/json` `{title, detail, code?, errors?, extra?}` where
  `errors` maps field names to messages (and may nest one level). Parse defensively for both shapes.
- `401` — `{"detail": "No authorization token provided"}` or an expired token. Refresh the JWT and
  retry the READ. Never blind-retry a write.
- `403` — the product or campaign is not enabled for your account. Stop; this is an account
  configuration question for MyOme, not something to work around.
- `415` — send `Content-Type: application/json`.
- No rate limit is published and no `Retry-After` is emitted, so pace conservatively and treat any
  unexplained failure as a reason to stop and ask a human.
