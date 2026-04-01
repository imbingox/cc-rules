# cc-rules Minimal Hardening Design

Date: 2026-04-01

## Context

This repository is a small proxy-rule project centered on three user-facing configs:

- `clash.yaml`
- `shadowrocket.conf`
- `substore.yaml`

It also maintains four custom rule lists:

- `rules/crypto.list`
- `rules/stocks.list`
- `rules/direct.list`
- `rules/reject.list`

The user selected a conservative optimization path:

- Keep current routing intent and personal region preference
- Do not add README or broader documentation work
- Do not restructure the repository
- Apply only minimal correctness and safety fixes

## Goals

1. Reduce obvious routing overreach in custom rules.
2. Reduce default exposure of local services in template configs.
3. Preserve current strategy for Taiwan-preferring AI and crypto routing.
4. Keep the patch set small and low risk.

## Non-Goals

- No new README or usage docs
- No generator or templating system
- No directory reorganization
- No policy changes to Taiwan-focused proxy groups
- No broad rewrite of provider ordering or routing categories

## Proposed Changes

### 1. Narrow over-broad crypto rules

Update `rules/crypto.list` to remove entries that represent shared infrastructure rather than crypto-specific destinations.

Candidate removals:

- `DOMAIN,app-measurement.com`
- `DOMAIN-SUFFIX,amazon.com`
- `DOMAIN-SUFFIX,amazonaws.com`
- `DOMAIN-SUFFIX,appsflyer.com`

Rationale:

- These domains are used widely outside crypto products.
- Keeping them under the crypto rule set can force unrelated traffic through the proxy.
- The remaining exchange- and tool-specific domains still preserve the intended routing behavior for crypto services.

### 2. Tighten suspicious stock IP rules

Update `rules/stocks.list` to shrink several IP entries from `/24` to `/32` where the current entries appear to target single endpoints.

Candidate changes:

- `119.28.37.77/24` -> `119.28.37.77/32`
- `43.153.233.212/24` -> `43.153.233.212/32`
- `124.156.233.21/24` -> `124.156.233.21/32`
- `170.106.49.68/24` -> `170.106.49.68/32`

Rationale:

- `/24` expands a rule to 256 addresses and is unusually broad for entries that are written as exact host IPs.
- Shrinking to `/32` lowers the chance of accidentally direct-routing unrelated endpoints.

### 3. Harden default listener settings

Update template defaults in `clash.yaml` and `substore.yaml` so they are safer for local-only use.

Planned `clash.yaml` changes:

- `allow-lan: true` -> `allow-lan: false`
- `external-controller: 0.0.0.0:9090` -> `external-controller: 127.0.0.1:9090`
- `dns.listen: 0.0.0.0:53` -> `dns.listen: 127.0.0.1:53`

Planned `substore.yaml` changes:

- `allow-lan: true` -> `allow-lan: false`
- `bind-address: "*"` -> `bind-address: 127.0.0.1`
- `dns.listen: 0.0.0.0:1053` -> `dns.listen: 127.0.0.1:1053`

Rationale:

- The user explicitly does not want LAN-first defaults.
- These files are templates and should default to the safer local-only posture.
- This does not change routing policy; it only changes how broadly local services are exposed by default.

## Files In Scope

- `rules/crypto.list`
- `rules/stocks.list`
- `clash.yaml`
- `substore.yaml`

## Validation Plan

1. Parse `clash.yaml` and `substore.yaml` with a YAML parser to catch syntax issues.
2. Inspect modified rule lines to ensure format remains compatible with Clash-style classical rules.
3. Review the resulting diff to confirm:
   - crypto-specific domains remain intact
   - stock rules are only narrowed, not expanded
   - routing group logic is unchanged
   - listener defaults are local-only

## Risks

- Some removed shared-infrastructure domains may have been compensating for missing vendor-specific rules. If a crypto app stops matching correctly, targeted vendor domains may need to be added back later.
- Some `/24` stock rules may have been intentionally broad. Narrowing them may require follow-up if a provider actually uses multiple adjacent IPs that must remain direct.

## Rollback

Rollback is straightforward because the patch is limited to four files and does not change repository structure.
