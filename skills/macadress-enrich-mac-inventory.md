---
generated: '2026-08-28'
method: generated
name: Enrich a MAC address inventory
description: Resolve a list of MAC addresses to vendor, IEEE registration block, country and inferred device category, in batches of 100, without burning quota on retries.
api: openapi/macadress-openapi.yaml
operations: [healthz, lookupMACBatch, lookupMAC]
source: >-
  Grounded in openapi/macadress-openapi.yaml; every operationId below was read verbatim
  from that spec. Quota, limit and error rules cite rate-limits/macadress-rate-limits.yml,
  conventions/macadress-conventions.yml and errors/macadress-problem-types.yml.
---

# Enrich a MAC address inventory

Turn a raw list of MAC addresses — a DHCP lease table, an `arp -a` dump, an NAC export — into vendor, registration block, country and device class.

## Auth
- One API key, sent as `Authorization: Bearer mk_...`. Prefer the header: the `?api_key=` query form works everywhere but leaks the key into proxy and server logs. See `authentication/macadress-authentication.yml`.
- Keys are issued instantly at https://macadress.com/signup. No approval step.

## Before you start
- **Normalize nothing.** Colons, hyphens, spaces, Cisco dot-grouping and bare hex are all accepted and case is irrelevant. A `400` means the value is not twelve hex digits at all, not that it is formatted oddly.
- **Know your budget.** The cycle quota bills *per address resolved*, not per request. A 100-address batch costs 100 lookups. The per-minute cap bills *per request*, so a batch costs 1. See `rate-limits/macadress-rate-limits.yml`.

## Steps
1. **Check the service is up** — `healthz` (`GET /v1/healthz`). No key required, not counted against any quota, so it is free to call before a large job. `200 {"status":"ok"}` means the vendor database is reachable; `503` means it is not, and every lookup will fail.
2. **Chunk the inventory into groups of 100.** The batch endpoint's `macs` array is capped at 100 by the schema (`maxItems: 100`); 101 entries returns `400` on the envelope, not a partial result.
3. **Look up each chunk** — `lookupMACBatch` (`POST /v1/mac/batch`) with `{"macs": [...]}`. Results come back in the same order as the input, `count` entries, each carrying the original `input` string.
4. **Read per-item failures, not the HTTP status.** An unparseable address inside a well-formed batch does *not* fail the batch: that entry returns `valid: false` with its own `error` at HTTP `200`. Branch on the item, not the response code.
5. **Spot-check a single address** — `lookupMAC` (`GET /v1/mac/{mac}`) when you need to re-examine one result interactively. Same response shape.
6. **Record the data version.** Every response carries `X-Data-Version` (and `meta.database_version`), the UTC date the IEEE snapshot last synced. Store it with your enrichment run so a later re-run can tell whether the underlying registry moved or your inventory did.

## Reading the result
- `organization` is `null` for unregistered addresses *and* for blocks IEEE marks private. Use `vendor_lookup_reliable` to tell whether the name can be trusted as the manufacturer.
- `matched_prefix` and `prefix_length` are the block actually matched, at its real width (24/28/36 bits). `oui` is always the first 24 bits of the *address*, which is not the same thing for an MA-M or MA-S block. Key your inventory on `matched_prefix`, not `oui`.
- `device.category` is `"unknown"` for most of the registry today, and `device.exact_model_known` is a constant `false`. A MAC address never identifies a model; treat the category as a hint with a stated `confidence`, never as inventory truth.
- `is_private` is deprecated. Use `organization === null` together with `vendor_lookup_reliable`.

## Errors
- `401` — missing or invalid key. Not retryable; fix the credential. See `errors/macadress-problem-types.yml`.
- `429` covers **two different failures** and the only discriminator is the prose in the body. `"rate limit exceeded"` is retryable: back off and read `X-RateLimit-Reset` (seconds; there is no `Retry-After`). `"quota exceeded"` is **not** retryable — the account is paused until the rolling 30-day cycle renews or the plan is upgraded. An agent that blindly retries a 429 will loop until the cycle turns over.
- `503` on `healthz` is a provider-side dependency failure; retry after a delay.

## Notes
- Read-only API. Nothing here mutates provider state, so there is no idempotency key and nothing to reverse. The only irreversible consequence of a retry is spent quota. See `conventions/macadress-conventions.yml`.
- The same four lookups are available as MCP tools at `https://mcp.macadress.com/mcp` with the same key. See `mcp/macadress-mcp.yml`.
