---
name: sense-initial-ats-sync
description: Seed a Sense tenant with a full applicant-tracking-system data set for the first time, in the dependency order Sense requires, using batch upserts.
api: Sense API
base_url: https://partner-api.sensehq.com/v1
operations:
  - post_internal-users
  - post_candidates
  - post_client-contacts
  - post_companies
  - post_job-orders
  - post_submissions
  - post_placements
  - post_leads
  - post_certifications
  - post_appointments
generated: '2026-08-26'
method: generated
source: openapi/sense-talent-labs-sense-api-openapi.json + https://developer.sensehq.com/
---

# Initial ATS sync into Sense

Seeds a Sense tenant from scratch. Run this once; use `sense-incremental-change-events` to keep it current afterwards.

## Before you start

- You need an OAuth 2.0 `client_id` / `client_secret` issued by the Sense team. There is no self-serve signup and no sandbox — the first call you make is a real write to a real tenant.
- **Record the wall-clock time the sync starts.** You will need it as the watermark for the incremental process. Sense documents this explicitly.
- Confirm you can supply the *complete* entity for every record. POST is a whole-entity replace.

## Step 1 — Get a token

`POST https://partner-auth.sensehq.com/oauth2/token` (`postTokenAuthorizer`), form-encoded with `client_id`, `client_secret`, `grant_type=client_credentials`.

Read `expires_in` from the response and cache the token for that long. Do not hardcode 300 seconds — the docs warn the lifetime may change without notice, and that hammering the token endpoint "may lead to rate limiting or deactivation." Refresh only when the token is about to expire.

## Step 2 — Upsert each entity type in dependency order

Sense has no server-side ordering. A reference to an id that has not been processed yet will dangle, so the order below is mandatory, not advisory:

1. `post_internal-users` — no outbound references, always first
2. `post_candidates`
3. `post_client-contacts`
4. `post_companies` — Company and ClientContact reference each other; contacts go first to break the cycle
5. `post_job-orders`
6. `post_submissions`
7. `post_placements`
8. `post_leads`
9. `post_certifications`
10. `post_appointments`

For each type, page through your source system and POST batches:

- Maximum **500 entities** per request (declared as `maxItems` in the contract).
- Maximum **256 KB after compression** per request.
- Exceeding either returns **413**. Split the batch and resend — the retry is safe.
- Send the *complete* entity every time. `id` is your system's primary key; Sense does not mint it.
- Datetimes must be ISO 8601 or the update is rejected with 400.

## Step 3 — Handle responses

- **201** — accepted for async processing. This does **not** mean the data is stored. Expect roughly 30 minutes before records appear in the product. Do not read back immediately and treat a 404 as failure.
- **400** — validation failure. Read `errors[]`; each item gives `entity_id`, `message` and `path` so you can locate the bad record inside the batch. Also read `warnings[]` — a batch can be accepted with warnings. Since 2023-03-15 an unknown or invalid field is a hard 400 rather than being ignored.
- **413** — split and resend.
- **401 / 429 / 5xx** are not declared in the contract but can occur. Treat any non-2xx you cannot parse as transient and retry with exponential backoff.

## Step 4 — Verify a sample

Spot-check with `get_candidates` (`GET /candidates/{id}`) using ids you sent, but only after allowing the processing delay. There are no list endpoints, so you cannot enumerate what landed — you can only check ids you already know.

## Gotchas

- **A partial POST is destructive.** Omitted fields are cleared. If you cannot supply a complete entity, use PATCH instead.
- **There is no delete.** To remove a record, send it with `is_deleted = true`. No restore operation is documented, so treat this as one-way.
- **One credential, full access.** There is no scope model; the credential can write every entity in the tenant. Scope your own automation accordingly.
