---
name: sense-receive-writebacks
description: Build and operate the HTTP endpoint that receives Sense Write-back events, meeting Sense's published receiver requirements and response contract.
api: Sense Write-back API
generated: '2026-08-26'
method: generated
source: https://developer.sensehq.com/ (Write-back API Specification)
---

# Receiving Sense write-backs

Sense pushes events **to you**. Unlike most webhook systems there is no subscription API to call — you build an endpoint to Sense's published spec, give Sense the URL, and Sense delivers to it. Configuration is done by Sense per agency, not self-serve.

## Endpoint requirements

Sense publishes a hard checklist. Your endpoint must:

1. Be RESTful and accept **POST**
2. Support HTTPS with **TLS 1.2** or better
3. Accept `application/json`
4. Emit accurate status codes
5. **Return an `id` in the response body on success**

Your URL must be **static**. Sense states it "cannot require any variable information to be provided for each writeback" — `/event/{candidate_id}` is explicitly invalid. A fixed query-string key such as `/event?key=12345` is allowed.

## Authentication

Pick one and tell Sense:

- **Bearer token** — a pre-shared key Sense embeds in every request, in the `Authorization` header or as a query parameter, your choice. Rotate at will.
- **OIDC** — give Sense a well-known OpenID Connect URL alongside the event URL.
- **OAuth 2.0** — give Sense an auth URL, scope, client id and client secret. `client_credentials` grant only. Tokens arrive as `Authorization: Bearer`.

Payloads are **not signed**. There is no HMAC to verify, so your bearer token or OAuth credential is the only thing separating a genuine write-back from a forged one. Treat it as a secret and prefer the header over the query parameter.

## The envelope

Every event, regardless of type:

```json
{
  "id": "xxxxxxxxx",
  "type": "ENGAGE_SENT_EVENT",
  "version": 1.0,
  "created_date": "2018-12-18T03:33:46+00:00",
  "data": { }
}
```

Branch on `type`. Check `version` before parsing `data`: major revisions (>= 1) are breaking, minor revisions (< 1) are not.

## Event types

`ENGAGE_SENT_EVENT`, `ENGAGE_EVENT_RESPONSE`, `MESSAGING_INCOMING_MESSAGE`, `MESSAGING_OUTGOING_MESSAGE`, `MESSAGING_DIGEST`, `CHATBOT_RESPONSE_SUMMARY`, `ENTITY_CREATE`, `ENTITY_UPDATE`, `ENTITY_DELETE`, `GENERIC_EVENT`.

`ENTITY_CREATE` is special: your response must include **`entity_id`** — the id of the record you just created — in addition to `id`.

## Responding

| Outcome | Status | Body |
|---|---|---|
| Success | 200 | `{ "id": "<unique_id>" }` |
| Bad auth | 401 | `{ "error": "invalid token or session" }` |
| Bad input | 4xx | `{ "error": "unable to store because of field x,y,z" }` |
| Your failure | 5xx | `{ "error": "unable to store, server issue x,y,z" }` |

**A 200 without an `id` counts as a failure** and Sense will redeliver. This is the single most common way a receiver ends up in a redelivery loop while appearing healthy.

## Redelivery and idempotency

Sense retries any failed write-back. No retry schedule, attempt limit, or dead-letter behaviour is published, and no ordering or exactly-once guarantee is given. **Deduplicate on the envelope `id`** and make your handler idempotent — that is the only protection available.

## Operational notes

- Return 200 fast and process asynchronously; a slow handler looks like a failure and earns a redelivery.
- Log the envelope `id`, `type` and `created_date` for every delivery — Sense provides no delivery log or replay you can query.
- Newer messaging write-back options are disabled by default and must be enabled per agency by Sense.
