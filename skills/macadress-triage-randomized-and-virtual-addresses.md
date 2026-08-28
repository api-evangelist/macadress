---
generated: '2026-08-28'
method: generated
name: Triage randomized and virtual MAC addresses
description: Decide whether an address is a factory-assigned NIC, an OS privacy-randomized address, a hypervisor/container interface, or a reserved protocol address — and how much to trust that verdict.
api: openapi/macadress-openapi.yaml
operations: [lookupMAC, lookupMACBatch]
source: >-
  Grounded in openapi/macadress-openapi.yaml (components.schemas.Result.randomization,
  .virtualization, .special_use, .slap_quadrant) and the field reference at
  https://macadress.com/docs. Vocabulary values cite vocabulary/macadress-vocabulary.yml.
---

# Triage randomized and virtual MAC addresses

The question behind most MAC lookups on a modern network is not "who made this" but "is this even a real device". This skill reads the four signals that answer it.

## Auth
- `Authorization: Bearer mk_...`. See `authentication/macadress-authentication.yml`.

## Steps
1. **Look the address up** — `lookupMAC` (`GET /v1/mac/{mac}`), or `lookupMACBatch` (`POST /v1/mac/batch`) for a whole subnet's worth at once.
2. **Read `administration_type` first.** `universally_administered` means the U/L bit says this is a factory-burned address from a registered block. `locally_administered` means software chose it — which could be privacy randomization, a hypervisor, a container bridge, or a deliberate override. Everything below only matters in the locally administered case.
3. **Read `randomization`, not just `potentially_randomized`.** The object carries `confidence` (`none` / `possible` / `likely`), the `signals` that drove the call, and `alternative_explanations`. A locally administered address that matches a known hypervisor prefix lists `virtual_machine` / `container` as alternatives rather than asserting randomization.
4. **Cross-check `virtualization`.** `confidence: exact` is reserved for prefixes IEEE itself registered to a hypervisor vendor (VMware, Xen, Hyper-V, VirtualBox, Parallels). Conventions that are not IEEE registrations — libvirt's QEMU/KVM default, Docker's bridge driver — score lower on purpose. Do not treat `low` as a negative; treat it as "convention, not registration".
5. **Check `special_use` before raising an alert.** Broadcast, IPv4/IPv6 multicast mapping, VRRP, HSRP, STP, LACP, 802.1X and LLDP addresses are protocol traffic, not devices. Each match names the IEEE/IETF/vendor `source` that documents it.
6. **Read `slap_quadrant` for structured local addresses.** `AAI`, `ELI`, `SAI` or `reserved` under IEEE 802c. An `ELI` address is an extended local identifier derived from a real company id — meaningfully different from a random `AAI`.
7. **Escalate on `vendor_lookup_reliable: false`.** That flag is the API telling you the `organization` string is not safe to attribute, whatever it says.

## Reading confidence honestly
Every judgment on this surface is graded and the grades are closed enums, not prose. `device.confidence` and `vendor_location.parse_confidence` cap where the evidence caps — `parse_confidence` never exceeds `medium` because the provider refuses to guess where a street address ends in IEEE's space-delimited format. See `vocabulary/macadress-vocabulary.yml` for every value.

## Errors
- Same catalog as every other lookup. Note that `400` is only returned for input that is not a MAC address at all; an address that is valid but unregistered returns `200` with `registered: false`. See `errors/macadress-problem-types.yml`.

## Notes
- `explanation` is the same plain-English sentence the website shows. It is safe to surface verbatim to a human, and it is the field to cite rather than paraphrasing the bit flags.
