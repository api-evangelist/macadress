---
generated: '2026-08-28'
method: generated
name: Audit a vendor's registered MAC block footprint
description: Search the IEEE vendor/block directory by organization name and country, and know exactly where the API's reachable results stop and the bulk downloads must take over.
api: openapi/macadress-openapi.yaml
operations: [searchVendors, lookupMAC]
source: >-
  Grounded in openapi/macadress-openapi.yaml (searchVendors,
  components.schemas.VendorSearchResult / VendorBlock) and
  https://macadress.com/docs. Pagination ceiling recorded in
  conventions/macadress-conventions.yml.
---

# Audit a vendor's registered MAC block footprint

Answer "which IEEE blocks does this organization hold, and where are they registered".

## Auth
- `Authorization: Bearer mk_...`. A directory search bills as **one lookup**, the same as a single MAC. See `rate-limits/macadress-rate-limits.yml`.

## Steps
1. **Search the directory** — `searchVendors` (`GET /v1/vendors`) with any combination of:
   - `query` — case-insensitive substring match against the organization name.
   - `country` — an exact ISO 3166-1 alpha-2 code (`US`, `DE`, `JP`).
   - `limit` — default 10, hard maximum 50.
2. **Read `total` and `blocks` as different things.** `total` is how many blocks match the filter overall; `blocks` is only the page you were given. They diverge immediately.
3. **Respect the reachability ceiling.** There is no `offset` and no cursor. With `query` and `country` both omitted, `total` reports the entire registry (~58,000 non-private blocks) and you can still only retrieve 50. Narrow the filter, or stop using the API for this job — see step 5.
4. **Match on name deliberately.** `organization` is the exact registered string. The provider states that its canonical vendor id is a mechanical slug of that string and **never** merges spelling variants, subsidiaries or acquisitions. "Apple, Inc." and a differently spelled sibling registration are two organizations here. If you need a corporate rollup, you must build it yourself and own the judgment.
5. **For a complete footprint, use the bulk files instead.** The whole vendor database is published as static downloads at https://macadress.com/downloads — CSV, JSON, Cisco `vendorMacs.xml`, plus drop-in replacements for Wireshark's `manuf` and nmap's `nmap-mac-prefixes` — rebuilt from the live registry twice daily. That is the supported path for exhaustive analysis; the search endpoint is for interactive narrowing.
6. **Confirm a specific block** — `lookupMAC` (`GET /v1/mac/{mac}`) on any address inside the block returns `matched_prefix`, `prefix_length`, `range_start`/`range_end`, `address_capacity` and a `vendor` object with `block_count` and a `lookup_url` for the human page.

## Reading the result
- `prefix_int` plus `mask_bits` (24 / 28 / 36) identify a block. `mask_bits` tells you the assignment class: 24 for MA-L/IAB/CID, 28 for MA-M, 36 for MA-S.
- `is_private: true` blocks have no disclosed organization; they are excluded from search results entirely.
- `first_seen_at` and `last_changed_at` are **this deployment's own sync history**, not IEEE dates. Do not read them as when a block was registered — no authoritative IEEE registration date exists in the source feed, which is why `assignment.registered_at` is documented as always `null`.

## Errors
- `401`, `429` (rate limit *or* quota — check the body text) as per `errors/macadress-problem-types.yml`. `searchVendors` declares no `400`: an omitted filter is a valid whole-registry match, not an error.

## Notes
- The MCP server exposes `search_vendors` with the same filters, plus a `get_vendor_block` tool that has **no REST equivalent** — it returns full detail for one block including the other blocks the same organization holds. If you need that, use MCP. See `mcp/macadress-tool-crosswalk.yml`.
