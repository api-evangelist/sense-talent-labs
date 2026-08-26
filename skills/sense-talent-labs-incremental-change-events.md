---
name: sense-incremental-change-events
description: Keep a Sense tenant current after the initial sync by flushing batched change events from the source ATS, using upserts and partial updates.
api: Sense API
base_url: https://partner-api.sensehq.com/v1
operations:
  - postTokenAuthorizer
  - post_candidates
  - patch_candidates
  - post_placements
  - patch_placements
  - post_job-orders
  - patch_job-orders
  - post_submissions
  - patch_submissions
generated: '2026-08-26'
method: generated
source: openapi/sense-talent-labs-sense-api-openapi.json + https://developer.sensehq.com/
---

# Incremental change events into Sense

Keeps a Sense tenant up to date once `sense-initial-ats-sync` has run. Implement this as a hook on entity change in your source system.

## The watermark

Use the timestamp you recorded when the initial sync *started* as your first query bound. Query your source for everything changed since then, so nothing that moved during the long-running seed is missed.

## Flush policy

Accumulate changes and flush when **any** of these hits first — this is the policy Sense documents:

- 500 accumulated updates, or
- 256 KB after compression, or
- 5 minutes elapsed.

## Choosing POST vs PATCH

This is the decision that matters most, and getting it wrong silently destroys data.

| Situation | Operation | Why |
|---|---|---|
| You have the complete current entity | `post_<entity>` | Whole-entity UPSERT; converges on correct state and is safe to replay |
| You only know which fields changed | `patch_<entity>` | "Only fields present in the request body are updated; omitted fields retain their current values" |
| The record was deleted upstream | `post_<entity>` with `is_deleted = true` | Soft delete — there is no DELETE method |

**Never POST a partial entity.** POST replaces; every field you leave out is cleared.

## Ordering still applies

If a single flush contains new entities across types, respect the same dependency order as the initial sync (internal-users → candidates → client-contacts → companies → job-orders → submissions → placements → leads → certifications → appointments). A submission referencing a job order Sense has not processed yet will dangle.

## Idempotency and retries

Retries are safe because writes are keyed on your own `id`. A replayed POST converges rather than duplicating — provided the body is complete. Retry transient failures with exponential backoff, as the docs recommend.

## Token handling

Reuse the cached token from `sense-initial-ats-sync`. A long-running change-event process that requests a token per flush is exactly the pattern Sense warns leads to rate limiting or deactivation. One token, refreshed on `expires_in`, shared across all flushes.

## Monitoring

- Watch `warnings[]` on 201 responses, not just `errors[]` on 400s. A batch can be accepted while individual records raise warnings.
- Poll `https://status.sensehq.com/api/v2/summary.json` to distinguish a Sense incident from a bug in your flush loop.
- There is no delivery log, no replay, and no way to enumerate what Sense holds, so keep your own record of what you sent and when.
