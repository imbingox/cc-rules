# cc-rules Minimal Hardening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Apply a minimal hardening patch that removes over-broad crypto rules, narrows suspicious stock IP rules, and switches template listeners to local-only defaults without changing the user's routing strategy.

**Architecture:** Patch the four existing files in place and keep rule ordering intact. Because this is a configuration-only change, use command-level red/green verification and YAML parsing instead of introducing a permanent test harness.

**Tech Stack:** Clash/Mihomo YAML configs, classical rule lists, Bash, `rg`, `git`, `python3` with PyYAML for syntax validation

---

### Task 1: Capture the red state for risky rules and listener defaults

**Files:**
- Modify: none
- Test: `/workspace/projects/cc-rules/rules/crypto.list`
- Test: `/workspace/projects/cc-rules/rules/stocks.list`
- Test: `/workspace/projects/cc-rules/clash.yaml`
- Test: `/workspace/projects/cc-rules/substore.yaml`

- [ ] **Step 1: Verify the over-broad crypto rules still exist**

Run:

```bash
rg -n "DOMAIN,app-measurement\\.com|DOMAIN-SUFFIX,amazon\\.com|DOMAIN-SUFFIX,amazonaws\\.com|DOMAIN-SUFFIX,appsflyer\\.com" /workspace/projects/cc-rules/rules/crypto.list
```

Expected: exit `0` with four matches from `rules/crypto.list`.

- [ ] **Step 2: Verify the suspicious `/24` stock rules still exist**

Run:

```bash
rg -n "119\\.28\\.37\\.77/24|43\\.153\\.233\\.212/24|124\\.156\\.233\\.21/24|170\\.106\\.49\\.68/24" /workspace/projects/cc-rules/rules/stocks.list
```

Expected: exit `0` with four matches from `rules/stocks.list`.

- [ ] **Step 3: Verify `clash.yaml` still defaults to LAN-exposed listeners**

Run:

```bash
rg -n "allow-lan: true|external-controller: 0\\.0\\.0\\.0:9090|listen: 0\\.0\\.0\\.0:53" /workspace/projects/cc-rules/clash.yaml
```

Expected: exit `0` with three matches from `clash.yaml`.

- [ ] **Step 4: Verify `substore.yaml` still defaults to LAN-exposed listeners**

Run:

```bash
rg -n 'allow-lan: true|bind-address: "\\*"|listen: 0\\.0\\.0\\.0:1053' /workspace/projects/cc-rules/substore.yaml
```

Expected: exit `0` with three matches from `substore.yaml`.

### Task 2: Remove only the clearly over-broad crypto entries

**Files:**
- Modify: `/workspace/projects/cc-rules/rules/crypto.list`
- Test: `/workspace/projects/cc-rules/rules/crypto.list`

- [ ] **Step 1: Edit `rules/crypto.list` to keep only crypto-specific domains**

Update the Bybit block from:

```text
DOMAIN,bybit-exchange.github.io
DOMAIN,app-measurement.com
DOMAIN-SUFFIX,monitor-frontend-collector.a.bybit-aws.com
DOMAIN-SUFFIX,bybit.com
DOMAIN-SUFFIX,bybit.biz
DOMAIN-SUFFIX,bycsi.com
DOMAIN-SUFFIX,bytick.com
DOMAIN-SUFFIX,byapis.com
DOMAIN-SUFFIX,bycbe.com
DOMAIN-SUFFIX,bymj.io
DOMAIN-SUFFIX,byabcde.com
DOMAIN-SUFFIX,byapps.net
DOMAIN-SUFFIX,byd3c3.com
DOMAIN-SUFFIX,bybdc6.com
DOMAIN-SUFFIX,ffbbbdc6d3c353211fe2ba39c9f744cd.com
DOMAIN-SUFFIX,ffe390afd658c19dcbf707e0597b846d.de
DOMAIN-SUFFIX,amazon.com
DOMAIN-SUFFIX,amazonaws.com
DOMAIN-SUFFIX,appsflyer.com
```

to:

```text
DOMAIN,bybit-exchange.github.io
DOMAIN-SUFFIX,monitor-frontend-collector.a.bybit-aws.com
DOMAIN-SUFFIX,bybit.com
DOMAIN-SUFFIX,bybit.biz
DOMAIN-SUFFIX,bycsi.com
DOMAIN-SUFFIX,bytick.com
DOMAIN-SUFFIX,byapis.com
DOMAIN-SUFFIX,bycbe.com
DOMAIN-SUFFIX,bymj.io
DOMAIN-SUFFIX,byabcde.com
DOMAIN-SUFFIX,byapps.net
DOMAIN-SUFFIX,byd3c3.com
DOMAIN-SUFFIX,bybdc6.com
DOMAIN-SUFFIX,ffbbbdc6d3c353211fe2ba39c9f744cd.com
DOMAIN-SUFFIX,ffe390afd658c19dcbf707e0597b846d.de
```

- [ ] **Step 2: Verify the removed crypto entries are gone**

Run:

```bash
rg -n "DOMAIN,app-measurement\\.com|DOMAIN-SUFFIX,amazon\\.com|DOMAIN-SUFFIX,amazonaws\\.com|DOMAIN-SUFFIX,appsflyer\\.com" /workspace/projects/cc-rules/rules/crypto.list
```

Expected: exit `1` with no matches.

- [ ] **Step 3: Verify the crypto-specific Bybit domains still remain**

Run:

```bash
rg -n "DOMAIN-SUFFIX,bybit\\.com|DOMAIN-SUFFIX,bytick\\.com|DOMAIN-SUFFIX,byapis\\.com" /workspace/projects/cc-rules/rules/crypto.list
```

Expected: exit `0` with matches for the kept exchange domains.

- [ ] **Step 4: Commit the crypto rule narrowing**

Run:

```bash
git -C /workspace/projects/cc-rules add rules/crypto.list
git -C /workspace/projects/cc-rules commit -m "fix: narrow crypto rule scope"
```

Expected: one new commit containing only `rules/crypto.list`.

### Task 3: Narrow stock IP rules from `/24` to `/32`

**Files:**
- Modify: `/workspace/projects/cc-rules/rules/stocks.list`
- Test: `/workspace/projects/cc-rules/rules/stocks.list`

- [ ] **Step 1: Edit the four stock IP rules to exact-host CIDRs**

Change:

```text
IP-CIDR,119.28.37.77/24,no-resolve
IP-CIDR,43.153.233.212/24,no-resolve
IP-CIDR,124.156.233.21/24,no-resolve
IP-CIDR,170.106.49.68/24,no-resolve
```

to:

```text
IP-CIDR,119.28.37.77/32,no-resolve
IP-CIDR,43.153.233.212/32,no-resolve
IP-CIDR,124.156.233.21/32,no-resolve
IP-CIDR,170.106.49.68/32,no-resolve
```

- [ ] **Step 2: Verify the `/24` rules are gone**

Run:

```bash
rg -n "119\\.28\\.37\\.77/24|43\\.153\\.233\\.212/24|124\\.156\\.233\\.21/24|170\\.106\\.49\\.68/24" /workspace/projects/cc-rules/rules/stocks.list
```

Expected: exit `1` with no matches.

- [ ] **Step 3: Verify the `/32` replacements exist**

Run:

```bash
rg -n "119\\.28\\.37\\.77/32|43\\.153\\.233\\.212/32|124\\.156\\.233\\.21/32|170\\.106\\.49\\.68/32" /workspace/projects/cc-rules/rules/stocks.list
```

Expected: exit `0` with four matches.

- [ ] **Step 4: Commit the stock IP narrowing**

Run:

```bash
git -C /workspace/projects/cc-rules add rules/stocks.list
git -C /workspace/projects/cc-rules commit -m "fix: narrow stock IP ranges"
```

Expected: one new commit containing only `rules/stocks.list`.

### Task 4: Switch template listeners to local-only defaults

**Files:**
- Modify: `/workspace/projects/cc-rules/clash.yaml`
- Modify: `/workspace/projects/cc-rules/substore.yaml`
- Test: `/workspace/projects/cc-rules/clash.yaml`
- Test: `/workspace/projects/cc-rules/substore.yaml`

- [ ] **Step 1: Edit `clash.yaml` to local-only listener defaults**

Change:

```yaml
allow-lan: true
external-controller: 0.0.0.0:9090
dns:
  enable: true
  listen: 0.0.0.0:53
```

to:

```yaml
allow-lan: false
external-controller: 127.0.0.1:9090
dns:
  enable: true
  listen: 127.0.0.1:53
```

- [ ] **Step 2: Edit `substore.yaml` to local-only listener defaults**

Change:

```yaml
allow-lan: true
bind-address: "*"
dns:
  enable: true
  cache-algorithm: arc
  listen: 0.0.0.0:1053
```

to:

```yaml
allow-lan: false
bind-address: 127.0.0.1
dns:
  enable: true
  cache-algorithm: arc
  listen: 127.0.0.1:1053
```

- [ ] **Step 3: Verify the old exposed listener values are gone**

Run:

```bash
rg -n "allow-lan: true|external-controller: 0\\.0\\.0\\.0:9090|listen: 0\\.0\\.0\\.0:53" /workspace/projects/cc-rules/clash.yaml
```

Expected: exit `1` with no matches.

Run:

```bash
rg -n 'allow-lan: true|bind-address: "\\*"|listen: 0\\.0\\.0\\.0:1053' /workspace/projects/cc-rules/substore.yaml
```

Expected: exit `1` with no matches.

- [ ] **Step 4: Verify the new local-only listener values exist**

Run:

```bash
rg -n "allow-lan: false|external-controller: 127\\.0\\.0\\.1:9090|listen: 127\\.0\\.0\\.1:53" /workspace/projects/cc-rules/clash.yaml
```

Expected: exit `0` with three matches.

Run:

```bash
rg -n "allow-lan: false|bind-address: 127\\.0\\.0\\.1|listen: 127\\.0\\.0\\.1:1053" /workspace/projects/cc-rules/substore.yaml
```

Expected: exit `0` with three matches.

- [ ] **Step 5: Parse both YAML files after the edits**

Run:

```bash
python3 - <<'PY'
import yaml
for path in (
    "/workspace/projects/cc-rules/clash.yaml",
    "/workspace/projects/cc-rules/substore.yaml",
):
    with open(path, "r", encoding="utf-8") as f:
        yaml.safe_load(f)
    print(f"OK {path}")
PY
```

Expected: exit `0` and two `OK ...` lines.

- [ ] **Step 6: Commit the listener hardening**

Run:

```bash
git -C /workspace/projects/cc-rules add clash.yaml substore.yaml
git -C /workspace/projects/cc-rules commit -m "fix: harden local listener defaults"
```

Expected: one new commit containing `clash.yaml` and `substore.yaml`.

### Task 5: Run final verification and create the integration commit

**Files:**
- Modify: none
- Test: `/workspace/projects/cc-rules/rules/crypto.list`
- Test: `/workspace/projects/cc-rules/rules/stocks.list`
- Test: `/workspace/projects/cc-rules/clash.yaml`
- Test: `/workspace/projects/cc-rules/substore.yaml`

- [ ] **Step 1: Re-run all rule-scope verification commands**

Run:

```bash
rg -n "DOMAIN,app-measurement\\.com|DOMAIN-SUFFIX,amazon\\.com|DOMAIN-SUFFIX,amazonaws\\.com|DOMAIN-SUFFIX,appsflyer\\.com" /workspace/projects/cc-rules/rules/crypto.list
```

Expected: exit `1` with no matches.

Run:

```bash
rg -n "119\\.28\\.37\\.77/32|43\\.153\\.233\\.212/32|124\\.156\\.233\\.21/32|170\\.106\\.49\\.68/32" /workspace/projects/cc-rules/rules/stocks.list
```

Expected: exit `0` with four matches.

- [ ] **Step 2: Re-run the YAML parse verification**

Run:

```bash
python3 - <<'PY'
import yaml
for path in (
    "/workspace/projects/cc-rules/clash.yaml",
    "/workspace/projects/cc-rules/substore.yaml",
):
    with open(path, "r", encoding="utf-8") as f:
        yaml.safe_load(f)
    print(f"OK {path}")
PY
```

Expected: exit `0` and two `OK ...` lines.

- [ ] **Step 3: Review the final diff for scope control**

Run:

```bash
git -C /workspace/projects/cc-rules diff -- rules/crypto.list rules/stocks.list clash.yaml substore.yaml
```

Expected: only the intended rule removals, CIDR narrowing, and listener-default changes appear.

- [ ] **Step 4: Create the final integration commit**

Run:

```bash
git -C /workspace/projects/cc-rules add rules/crypto.list rules/stocks.list clash.yaml substore.yaml
git -C /workspace/projects/cc-rules commit -m "fix: apply minimal rule hardening"
```

Expected: one final commit for the full patch if the earlier task commits were skipped or squashed locally. If the earlier commits were kept, skip this step and keep the three focused commits.
