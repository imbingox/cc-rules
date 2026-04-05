# cc-rules Surge Config Design

Date: 2026-04-05

## Context

This repository currently maintains:

- `clash.yaml` as the richest routing template
- `shadowrocket.conf` as a simpler rule-focused config
- custom rule lists under `rules/`

The user wants a new `surge.conf` derived from `clash.yaml`, but with Surge-native syntax and a complete profile structure based on a provided `[General]` template.

One key repository constraint is that `clash.yaml` does not contain actual nodes. It uses `proxies: ~`, which means the config is a routing and policy template rather than a self-contained node inventory. The user wants the Surge version to preserve that model: nodes will be injected externally, and `surge.conf` should keep `~` as the explicit placeholder for that injection point.

## Goals

1. Add a new `surge.conf` that mirrors the routing intent of `clash.yaml`.
2. Preserve the current region-and-function policy structure in Surge-native `[Proxy Group]` syntax.
3. Preserve current rule ordering and rule targets when converting to Surge `[Rule]` syntax.
4. Keep external node injection outside the repository-managed template by preserving a `~` placeholder.
5. Use the user-provided Surge `[General]` block as the authoritative base template.

## Non-Goals

- No attempt to embed real nodes in the repository
- No generator, script, or templating engine for node injection
- No changes to `clash.yaml` routing behavior
- No new rule categories beyond what `clash.yaml` already wires in
- No restoration of the currently deleted `substore.yaml`
- No attempt to prove runtime behavior inside Surge, since no nodes are present in the repo

## Proposed Changes

### 1. Add a new `surge.conf` template

Create a new top-level `surge.conf`.

Its structure will include:

- `[General]`
- `[Proxy]`
- `[Proxy Group]`
- `[Rule]`

The `[General]` section will be copied from the user-provided Surge base template rather than inferred from `clash.yaml`. This keeps the implementation aligned with the user's stated baseline and avoids making unsupported one-to-one assumptions for settings such as controller endpoints or DNS fallback behavior.

### 2. Preserve external node injection with a literal `~` placeholder

The `[Proxy]` section will contain:

- `DIRECT = direct`
- a standalone literal `~`

This `~` line is not intended as native Surge syntax. It is a repository-level placeholder that marks where an external process will inject actual node definitions before use. This mirrors the role of `proxies: ~` in `clash.yaml`.

Implications:

- The checked-in `surge.conf` is a template artifact.
- It is complete in structure and policy behavior, but not self-sufficient until the external node source expands the placeholder.
- The placeholder remains intentionally visible so downstream tooling has a stable insertion point.

### 3. Map Clash proxy groups to Surge proxy groups

The new `[Proxy Group]` section will preserve both the top-level functional groups and the region-specific groups from `clash.yaml`.

Top-level groups to preserve:

- `🚀 默认代理`
- `🤖 AI`
- `🪙 Crypto`
- `🍀 Google`
- `👨🏿‍💻 GitHub`
- `🎵 TikTok`
- `📲 Telegram`
- `🎬 流媒体`
- `🐟 漏网之鱼`
- `🌐 全部节点`

Region-specific groups to preserve:

- `🇨🇳 台湾-家宽`
- `🇨🇳 台湾-流媒体`
- `🇨🇳 台湾-其他`
- `🇭🇰 香港-家宽`
- `🇭🇰 香港-流媒体`
- `🇭🇰 香港-其他`
- `🇯🇵 日本-家宽`
- `🇯🇵 日本-流媒体`
- `🇯🇵 日本-其他`
- `🇸🇬 新加坡-家宽`
- `🇸🇬 新加坡-流媒体`
- `🇸🇬 新加坡-其他`
- `🇺🇸 美国-家宽`
- `🇺🇸 美国-流媒体`
- `🇺🇸 美国-其他`

Group type mapping:

- Clash `select` -> Surge `select`
- Clash `fallback` -> Surge `fallback`
- Clash `url-test` -> Surge `url-test`

Default composition mapping:

- `🌐 全部节点` will use `select` with `include-all-proxies=true`
- each `家宽` group will use `fallback`
- each `流媒体` group will use `url-test`
- each `其他` group will use `select`
- top-level functional groups will list the same region groups and fallback options as in `clash.yaml`

### 4. Translate Clash filtering into Surge regex filters

`clash.yaml` uses combinations of:

- `include-all: true`
- `filter`
- `exclude-filter`

The Surge version will express the same intent with:

- `include-all-proxies=true`
- `policy-regex-filter=...`

Regex translation principles:

- preserve region detection by matching Chinese names and common English abbreviations
- preserve function detection for `家宽` and `流媒体`
- preserve the current rule that node names ending with `-地区` should be attributed by the last region suffix when possible
- preserve exclusion of other regions through negative lookaheads
- preserve the `其他` behavior by excluding `家宽|宽带|住宅|家庭|流媒体|解锁|奈飞|netflix|影音`

This translation will be semantic rather than character-for-character. The goal is to keep the routing categories stable in Surge, not to reproduce Clash regex formatting exactly.

### 5. Convert Clash rule providers into Surge rule sets

The `[Rule]` section will keep the same order and same policy targets as `clash.yaml`.

Custom rule sources will remain on this repository:

- `reject.list`
- `direct.list`
- `crypto.list`
- `stocks.list`

Blackmatrix7 sources will switch from Clash paths to Surge paths where available.

Rule target mapping:

- `reject-custom` -> `REJECT`
- `direct-custom` -> `DIRECT`
- `crypto` -> `🪙 Crypto`
- `stocks` -> `DIRECT`
- `ads` -> `REJECT`
- `apple` -> `DIRECT`
- `china` -> `DIRECT`
- `telegram` -> `📲 Telegram`
- `openai` -> `🤖 AI`
- `claude` -> `🤖 AI`
- `gemini` -> `🤖 AI`
- `copilot` -> `🤖 AI`
- `google` -> `🍀 Google`
- `youtube` -> `🎬 流媒体`
- `netflix` -> `🎬 流媒体`
- `global` -> `🚀 默认代理`
- `GEOIP,CN` -> `DIRECT`
- `FINAL` -> `🐟 漏网之鱼`

The `👨🏿‍💻 GitHub` and `🎵 TikTok` groups will still be created in `[Proxy Group]`, but no new rule sets will be wired to them because `clash.yaml` also defines those groups without attaching rules.

## Architecture

The resulting repository structure stays simple:

- `clash.yaml` remains the source reference for routing semantics
- `surge.conf` becomes the Surge-native equivalent template
- `shadowrocket.conf` remains untouched
- `rules/` continues to host the project-specific classical rule lists

This keeps each client format independent while preserving one shared routing model.

## Data Flow

1. The repository provides `surge.conf` as a policy template.
2. An external process injects concrete node entries at the `~` placeholder in `[Proxy]`.
3. Surge loads those nodes.
4. `include-all-proxies=true` and `policy-regex-filter` populate the region-specific groups.
5. Top-level functional groups consume the region-specific groups.
6. `[Rule]` dispatches traffic to the same logical targets as `clash.yaml`.

## Error Handling And Compatibility Boundaries

- If no external nodes are injected, `surge.conf` remains a template and is not expected to be directly usable as a live profile.
- If injected node names do not match the expected naming patterns, region groups may be empty. This is acceptable and consistent with the naming-driven grouping strategy already used in `clash.yaml`.
- Top-level groups will keep `🌐 全部节点` or `DIRECT` as fallbacks where `clash.yaml` already does so, reducing the chance of dead-end selection paths.
- The implementation will not attempt one-to-one migration for Clash runtime settings such as `port`, `socks-port`, or `dns.fallback`; the user-provided Surge `[General]` template is the source of truth for those settings.

## Files In Scope

- `surge.conf`

## Validation Plan

1. Review the generated `surge.conf` structure for the expected four sections.
2. Verify that every top-level and region-specific group from `clash.yaml` exists in `surge.conf`.
3. Verify that every rule target from `clash.yaml` maps to the intended Surge policy name.
4. Verify that custom rule URLs still point to this repository and that Blackmatrix7 URLs use Surge rule-set paths.
5. Check for obvious placeholders, contradictions, or missing fallback policies.

## Risks

- Surge regex behavior may differ slightly from Clash regex behavior at the edges, especially around lookaheads and name-suffix matching.
- Blackmatrix7 may not publish every category under the exact same Surge path as Clash; implementation will need to use the closest available Surge rule-set path for each existing category.
- Because no real nodes exist in the repository, validation can only confirm structure and mapping, not runtime routing outcomes.

## Rollback

Rollback is straightforward because this work only adds a new `surge.conf` template and does not modify the existing routing files.
