# Arch Linux (and derivatives)

Rules for Arch Linux and Arch-based derivatives
(Manjaro, EndeavourOS). Arch is a **rolling release**:
there is no stable branch, no version pinning, and
no distro-side firewall or auto-update mechanism by
default.

## Package Manager

- Use `pacman` for all official-repo packages.
- **Always upgrade with a full sync:**
  `pacman -Syu <package>`
- **Never run `pacman -Sy <package>`** — see the
  Partial Upgrades taboo below.
- Review pending upgrades without syncing:
  `checkupdates` (pacman-contrib) — safe, uses a
  throwaway database. For one package:
  `pacman -Sp <package>` prints what it would install
  without installing it.
- Non-interactive install (with the full upgrade):
  `pacman --noconfirm -Syu <package>`
- Query installed package:
  `pacman -Qi <package>` (detailed) or
  `pacman -Q <package>` (one line)
- List files of a package:
  `pacman -Ql <package>`
- Which package owns a file:
  `pacman -Qo /path/to/file`

### Partial Upgrades (taboo)

`pacman -Sy <package>` refreshes the sync database
but upgrades only the named package. The rest of the
system keeps old versions while the database points
at new ones — the resulting version mismatch breaks
dependency resolution and can leave the system
partially upgraded in ways that are difficult to
untangle.

This is the single most dangerous routine command on
Arch. If a package must be installed, always do the
full upgrade at the same time: `pacman -Syu
<package>`.

Note: `pacman -Sy` as a *standalone* database refresh
followed later by `pacman -Su` is equally forbidden —
the dangerous window is the moment the database is
newer than the installed packages, regardless of
what follows.

## AUR (Arch User Repository)

- The AUR is user-generated content. Build **only as
  a normal user, never as root** — `makepkg` must not
  run with elevated privileges.
- Build manually when possible:
  `git clone https://aur.archlinux.org/<pkg>.git`,
  inspect the `PKGBUILD` and install scripts, then
  `makepkg -si` in the clone directory.
- AUR helpers (`paru`, `yay`) are **not** in official
  repos. Use them only on explicit user request, and
  only after the user has reviewed what they install.
- After building, verify the package landed in the
  local database: `pacman -Qi <package>`.
- Log any AUR install to the server memory file —
  AUR packages do not receive Arch security team
  coverage and must be tracked manually (see
  `rules/version-check.md`).

## Rolling Release Discipline

There is no stable branch and no "hold back" policy —
but there are real manual steps before and after
upgrading:

- **Before major upgrades, check Arch news:**
  fetch <https://archlinux.org/news/> (web search or
  smart fetch) and look for manual-intervention
  posts. Famous examples: kernel/module signature
  changes, filesystem layout moves, mandatory
  config migrations. Upgrading across a
  manual-intervention release without reading the
  news is the classic Arch breakage.
- **Keyring refresh:** if `pacman -Syu` fails with
  signature errors on a long-unattended host,
  refresh the keyring first:
  `pacman -Sy archlinux-keyring && pacman -Su`
  (order matters — keyring must be upgraded before
  the rest).
- **`.pacnew` / `.pacorig` / `.pacsave` files:**
  after upgrading, merge config changes:
  `pacdiff` (from `pacman-contrib`) or review
  manually. Untouched `.pacnew` files mean the
  system runs old config against new binaries.
- **Housekeeping cadence:** an Arch host left
  un-upgraded for months is riskier than on a
  point-release distro — the eventual upgrade gap
  grows. Flag long gaps to the user.

## Firewall

Arch installs **no firewall** by default — none of
ufw, firewalld, or nftables is part of a base
install, and nothing filters traffic until it is set
up. Expected posture on a managed server: **nftables**
with a default-deny input chain, configured in
`/etc/nftables.conf` and loaded by `nftables.service`.

Detection order on an audit (first hit wins):

1. `systemctl is-active nftables` — native
   nftables service.
2. `systemctl is-active firewalld` — firewalld
   front-end (uses nftables backend).
3. `systemctl is-active ufw` — ufw front-end
   (uses iptables-nft compat layer).
4. `nft list ruleset` non-empty — rules present
   but not managed by any of the services above
   (e.g. docker's own rules, or a hand-rolled
   ruleset).

None of the above → **flag it to the user** as a
finding (housekeeping/severity: **WARN**, not
CRITICAL — many Arch servers run behind an external
firewall; see the security skill's firewall
reference for the exact wording).

Verify an active ruleset rejects unsolicited
incoming traffic:

```
nft list ruleset | grep -E "hook input"
```

Look for a base chain with `type filter hook input`
and `policy drop` (or `policy reject`). If the input
base chain policy is `accept` and no explicit drop
rule follows, the firewall is decorative — treat as
a finding.

**Critical (lockout risk):** before enabling
`nftables.service` on a remote host, make sure the
ruleset accepts SSH. The shipped
`/etc/nftables.conf` is a deny-by-default ruleset
that already includes `tcp dport ssh accept` —
enabling it as-is keeps SSH reachable and traffic
filtered. The danger is a *custom* ruleset without
the SSH accept rule: the moment the service loads
it, the session dies and the host is unreachable.
Validate any ruleset before loading it with
`nft -c -f /etc/nftables.conf` (syntax check only,
no kernel changes), then
`systemctl enable --now nftables`.

Do not mix `iptables` commands with a native
nftables ruleset — they create separate rule
objects and will conflict.

## Automatic Security Updates

Arch has **no official auto-update mechanism** and
no security-only archive — every update is part of
the same rolling snapshot, and Arch does not
guarantee binary backwards compatibility mid-stream.
Auto-applying a full `pacman -Syu` unattended is
therefore a risk trade-off, not a clear win like on
point-release distros.

- If no update mechanism exists at all, flag it to
  the user (**WARN** in housekeeping).
- If the user wants automation, discuss first, and
  prefer a **notified** workflow: a timer that runs
  `checkupdates` (from `pacman-contrib`) and mails
  or logs the result, while the actual upgrade
  stays manual. See `rules/scheduled-housekeeping.md`
  for the timer pattern.
- **Security exposure check:** `arch-audit`
  (official repos, extra) lists installed packages
  with known CVEs, based on the Arch Security Team
  tracker. Run it during security audits; it needs
  network access to fetch
  <https://security.archlinux.org/all.json>.
  `arch-audit -u` limits output to packages with a
  fixed version available (i.e. fixed by upgrading).

## Service Manager

- `systemctl` (systemd)
- Check service: `systemctl status <service>`
- Logs: `journalctl -u <service>`
- Reload vs restart: prefer `systemctl reload` when
  the service supports it. See
  `rules/service-reload.md` for the auto-proceed
  policy and `memory/service-policy.md` opt-out /
  opt-in lists.

## Mandatory Access Control

Arch has **no MAC (SELinux/AppArmor) enabled by
default**. Do not expect `getenforce` or
`aa-status` to return anything meaningful; do not
flag their absence as a finding beyond an INFO note
in security audits.

## Directory Conventions

- Config files: `/etc/`
- Web roots: `/srv/http/` (distro default) or
  `/var/www/` (admin choice — check what's in
  use before assuming)
- Logs: `/var/log/` (plus `journalctl`)
- Nginx config: `/etc/nginx/nginx.conf` with
  `include /etc/nginx/conf.d/*.conf;` — **no
  sites-available/sites-enabled pattern** on Arch.
- SSH config: `/etc/ssh/sshd_config` (never
  modify — see CLAUDE.md taboos); note Arch's
  sshd uses `sshd_config.d/` drop-ins too.

## Version Detection

- `/etc/os-release` — `ID=arch`, `BUILD_ID=rolling`
- `uname -r` — kernel version, which changes
  constantly; not a release version in the
  point-release sense. Record kernel + `pacman -Q
  linux` in the server memory file instead of a
  "distro version".
- Derivatives report their own `ID` (e.g.
  `manjaro`, `endeavouros`) — treat `ID_LIKE=arch`
  as Arch-family and read this file, but prefer
  the derivative's own repos for anything
  kernel-related.

## Notes

- `pacman-contrib` provides `checkupdates`,
  `paccache`, `pacdiff`, `pactree` — worth
  installing on any managed Arch host (ask the
  user first, per house rules).
- `paccache -r` keeps the last 3 package versions
  in the cache and removes older ones — safe cache
  hygiene if `/var/cache/pacman/pkg` grows large.
  Dry-run first with `paccache -d -r`.
- Arch Linux ARM (`ID=archarm`) follows the same
  rules, with its own repos and mirrors.

## Common Pitfalls

- `pacman -Sy <pkg>` — never. Always `-Syu <pkg>`.
  See Partial Upgrades above.
- "nftables is installed" is not the same as "a
  firewall is active" — the ruleset loads only when
  `nftables.service` starts, and a custom ruleset
  may differ from the shipped default. Always
  inspect `nft list ruleset`.
- No MAC, no auto-updates, no firewall by default
  — absence of these is expected on Arch, not a
  misconfiguration, but the *firewall* absence is
  still flagged WARN per the Firewall section.
- After every upgrade, check for `.pacnew` files
  (`pacdiff` or
  `find /etc -name '*.pacnew'`). Unmerged
  `.pacnew` files are silent config drift.
- nginx on Arch uses `conf.d/` only. Do not
  create `sites-available/` out of habit.
- Derivatives (Manjaro etc.) may lag Arch or
  carry their own kernel patches — check
  `os-release` before kernel-related work.
- `pacman --noconfirm` still asks nothing on
  remove-or-upgrade prompts inside `-Syu`; review
  the dry-run output first if the upgrade list is
  large.
