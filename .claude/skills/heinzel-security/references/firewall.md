# Firewall

## Linux

Verify a firewall is installed, active, and the default incoming
policy is deny/drop. An inactive or missing firewall on a Linux
server is **CRITICAL** — aligned with the housekeeping severity.
Exception: Arch ships no firewall by default, so a missing
firewall there is **WARN** — see the Arch section below.

### Debian/Ubuntu (ufw)

```bash
ufw status verbose
```

- Not installed or inactive → **CRITICAL** "No active firewall"
- Active but default incoming is not `deny` → **WARN** "Firewall
  default incoming policy is not deny"
- Active and default deny → OK

### RHEL/Fedora/SUSE (firewalld)

```bash
firewall-cmd --state
firewall-cmd --get-default-zone
```

Then check the default zone's target:

```bash
firewall-cmd --zone=<zone> --get-target
```

- Not running → **CRITICAL** "No active firewall"
- Zone target is `ACCEPT` → **WARN** "Default zone target is
  ACCEPT (allows all incoming)"
- Zone target is `default` (reject/drop) → OK

### Arch (no default — detect what's in use)

Arch ships no firewall. Detect what is active, first hit
wins:

```bash
systemctl is-active nftables 2>/dev/null
systemctl is-active firewalld 2>/dev/null
systemctl is-active ufw 2>/dev/null
nft list ruleset 2>/dev/null
```

- No firewall tool active and no non-empty ruleset →
  **WARN** "No active firewall" — not CRITICAL: Arch
  ships none by default and the host may sit behind an
  external firewall. Say so in the report so the user
  can confirm the external protection.
- nftables active → check the input base chain:
  `nft list ruleset | grep -E "hook input"` must show
  `policy drop` (or `policy reject`). `policy accept`
  with no following drop rule → **WARN** "Firewall
  default incoming policy is not deny".
- firewalld or ufw present → use the checks in the
  RHEL / Debian sections above.
- Plain `iptables -S` rules exist but none of the
  services above is active → the rules are runtime-only
  and vanish on reboot → **WARN**.

## macOS

Check Application Firewall status:

```bash
/usr/libexec/ApplicationFirewall/socketfilterfw \
  --getglobalstate
```

- Disabled → **INFO** (not WARN — common on macOS behind NAT,
  consistent with housekeeping severity)
- Enabled → OK
