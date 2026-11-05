# DESIGN: possible extensions

Notes on extensions to `nagios-pacman-check`. All four items below are
now implemented. The script stays simple `sh`/`oksh`-style (POSIX `test`,
backticks, `expr` — no bashisms). Syntax can be checked with:

    oksh -n ./nagios-pacman-check

## 1. Relevant vs. less-relevant packages — Implemented

`pacman -Qu` alone doesn't carry repo/importance info. Turns out
`pacman -Qi` (local DB query) does *not* expose a `Repository` field —
only `pacman -Si` (sync DB query) does, pulled straight from whatever is
configured in `pacman.conf`, no hardcoding needed. A pending update is
"relevant" if `pacman -Si <pkg>` resolves to a repo; "less relevant"
otherwise (see item 3).

Counts are surfaced in the status line as `N relevant, M less relevant`.

## 2. Package names in the status output — Implemented

The per-package detail (name, old → new version, repo tag, and any CVE
fix tag) is written to a temp file and printed unconditionally after the
summary line, Nagios "long output" style (first line = summary shown in
the overview, following lines = detail shown on drill-down) — no longer
gated behind `-v`/`--verbose`.

## 3. Per-repo / official vs. foreign distinction — Implemented

`pacman -Si <pkg>`'s `Repository` field handles official and custom
repos generically (core/extra/multilib/archlinuxaba/...) without
special-casing repo names in the script. When `pacman -Si` finds no
match (package not in any configured sync db — the AUR/foreign case,
same set `pacman -Qm` would list), it's tagged `foreign/AUR`.

A `sort | uniq -c` breakdown per repo name is included in the status
line, e.g. `core: 1, foreign/AUR: 1`.

## 4. CVE / security advisories (no HTML scraping) — Implemented

Fetches `https://security.archlinux.org/issues/all.json` (the Arch
Security Tracker's JSON API, not scraped) with `curl`, parsed with `jq`.
Cached to `/var/tmp/nagios-pacman-check-security.json`, refreshed only
when older than 6h, falling back to a stale cache (or skipping cleanly,
no `UNKNOWN`) if the fetch fails.

Two signals, both using `vercmp` (ships with `pacman`) to compare the
installed version against the tracker's `affected` version correctly:

- Pending updates whose target version resolves a tracker entry with
  `status=="Fixed"` are tagged `fixes CVE-xxxx(severity)` and counted.
- Installed packages matching a tracker entry with `status=="Vulnerable"`
  (no fix shipped yet) are reported as unpatched known vulnerabilities —
  independent of whether `pacman -Qu` shows an update, since no update
  would resolve these. This alone can trigger `STATE_WARNING`.

Both signals are scoped to packages actually installed on the system
(`pacman -Qq`/the pending-update's own package name) — the unpatched-scan
output also intersects a tracker group's package list against installed
packages before printing, so a group like `jre8-openjdk-headless,jdk8-openjdk`
only ever shows the subset that's really installed, never the whole group.

## 5. Steering security output (severity/category filters) — Implemented

Four new flags, since all-or-nothing reporting of every severity got noisy:

- `--unpatched-only` / `--fixes-only` / `--no-security` — pick which of
  the two signals from item 4 run at all (default: both). Opting a signal
  out also opts it out of affecting the exit state.
- `--min-severity=critical|high|medium|low` — display/count threshold
  ("this or more severe"), via a small `severity_rank()` helper
  (critical=4 ... unknown=0) reused by both signals.
- A Critical-severity unpatched vulnerability now escalates the exit
  state to `STATE_CRITICAL` (tracked unconditionally via
  `nof_critical_unpatched`, independent of `--min-severity`, so a display
  filter can never hide the fact that something critical triggered the
  alert). Everything else still only reaches `STATE_WARNING`, matching
  the original convention of this script.

## 6. `--cache-only` flag — Implemented

`check_yum.pl` (HariSekhon/Nagios-Plugins) has a `-C`/`--cache-only` option
that skips the metadata refresh and runs entirely from the existing cache,
for when a separate cron job (`yum makecache`) already keeps it warm.
`nagios-pacman-check` now offers the same: `--cache-only` skips `sudo
pacman -Sy` and checks `pacman -Qu` against whatever sync db is already
on disk. Useful if a timer already runs `pacman -Sy` separately, or to
avoid the sudo/network round-trip on every check invocation. This is
distinct from the existing security-cache logic in item 4, which already
has its own independent 6h freshness window — `--cache-only` only
affects the pacman sync db step.

## Decision: no `-W`-style "warn only on security updates" flag — not implemented

Considered adding a `check_yum.pl`-style `-W`/`--warn-on-any-update`
parity flag. `check_yum.pl` tags every update as security/non-security
(binary, from yum's security plugin) and by default only escalates on
security updates — `-W` is what makes plain non-security updates
contribute to WARNING at all. `nagios-pacman-check` already warns on
*any* pending update unconditionally (the `nof_updates -gt 0` branch of
the final exit-state `if`), independent of whether it's CVE-tagged —
there's no off switch today. Note this isn't the same axis as
`--min-severity`, which only filters tracker severity among updates
already known to relate to a CVE; a plain update with no CVE relevance
has no severity for `--min-severity` to threshold. Decided not to add
this for now — would mean either flipping the current default (less
noisy, but a behavior change) or adding an opt-out flag for it; no clear
need yet.

## Note: kernel/reboot-needed is a separate concern, not merged in

Came up while comparing `nagios-arch-installed-kernel` (checks `uname -r`
vs the installed kernel package version, i.e. "do I need to reboot?")
against this script's "are updates pending?" question. Decision: keep
them as two separate Nagios services, not one merged plugin — a system
can have zero pending updates and still need a reboot from a kernel
update days ago, and Nagios services are meant to isolate independently
alertable concerns.

**Update:** reconsidered — see items 7 and 8 below. `nagios-pacman-check`
now has `--check-kernel` and `--check-restart` as opt-in flags, default
off, so the "one service per concern" default behavior is unchanged and
this stays purely additive for anyone who wants a single combined
check instead.

Checked how `check_yum.pl` handles this for RHEL: it does *not*
independently compare running vs. installed kernel. It piggybacks on a
diagnostic string yum's own `yum-plugin-security` happens to print
during `--security check-update` (`Security: kernel-... is an installed
security update`) and short-circuits to CRITICAL/WARNING when it sees
it. That's narrower than our kernel check (security-kernel-updates only,
not any kernel bump) and incidental rather than deliberate (fragile to
yum's exact wording — the script is full of "format may have changed"
UNKNOWN fallbacks because of this). Pacman has no equivalent free
side-channel, so replicating this in `nagios-pacman-check` would mean
adding a deliberate `uname -r` vs `pacman -Qi` comparison — i.e.
duplicating `nagios-arch-installed-kernel` inside this script. Not worth
it; if a single aggregated Nagios service line is ever wanted, that's a
Nagios-side concern (e.g. a check that runs both and combines), not a
reason to merge the scripts.

## 7. `--check-kernel` flag — Implemented

Ports `nagios-arch-installed-kernel`'s check (running `uname -r` vs. the
installed kernel package's version, per-arch package name list) directly
into `nagios-pacman-check` as an opt-in flag, default off — the
standalone plugin is unaffected and still exists for anyone running it
as its own separate service (see the separate-concern note above). When
`--check-kernel` is set: a mismatch appends `kernel: running X, installed
Y (reboot needed)` to the status line and escalates the exit to at least
`WARNING` (never downgrades an existing `CRITICAL`). The original had a
bug where `armv6l|armv7l` set `KERNEL_PACKAGES` to `linux-raspberrypi`
then immediately overwrote it with `linux-rpi-legacy`, so only the
second name was ever tried — see the bugfix note below for the full
fix (expanded candidate list, plus a found-nothing false-positive that
surfaced on a real ARM board).

## 8. `--check-restart` flag — Implemented

Generalizing the reboot check beyond just the kernel turned out not to
need reimplementing RHEL's install-timestamp-vs-boot-time technique (the
idea previously sketched here) or writing a `needrestart`-style
`/proc/*/maps` scanner from scratch — `needrestart`
(https://github.com/liske/needrestart; present on this system, and in
the AUR for Arch, confirmed to support pacman) already ships its own
built-in Nagios plugin mode: `needrestart -p` (`-b` for
plain batch/machine-readable mode without Nagios framing). It must run
as root (`sudo`, same as the existing `pacman -Sy` call) and its exit
codes already line up with `utils.sh`'s `STATE_*` values exactly (tested:
no-root → exit 3/UNKNOWN; a stale kernel + stale services → exit
2/CRITICAL) — no translation needed.

`--check-restart`, default off, shells out to `sudo needrestart -p`,
takes its first output line as a status-line tag and the remaining
lines (the `Services:`/`Sessions:`/`Containers:` lists) as long output,
and folds its exit code into ours via a `worse_state()` helper (OK <
WARNING < UNKNOWN < CRITICAL) so a `needrestart` finding is never
silently downgraded by, nor silently downgrades, the pacman-update
signals. If `needrestart` isn't installed, reports that and contributes
`UNKNOWN` rather than silently skipping (consistent with how missing
`pacman`/`sudo` failures are already handled elsewhere in this script).

Covers strictly more than `--check-kernel` (kernel *and* glibc/systemd/
dbus-linking services *and* sessions/containers), so the two flags
overlap on the kernel signal when both are used together — harmless,
just two sources tagging the same reboot-needed fact.

## Bugfix: `--check-kernel` false-positive on Arch Linux ARM — fixed

Reported in the field on an ARM board: `kernel: running 6.12.45,
installed (reboot needed)` — the installed version came back empty, yet
it still claimed a reboot was needed. Two bugs, both now fixed in
`check_kernel()` (`nagios-pacman-check`) and the equivalent code in
`nagios-arch-installed-kernel`:

1. Arch Linux ARM renamed its kernel package over time; the running
   system had `linux-rpi` installed, which wasn't in our
   `armv6l|armv7l` candidate list (`linux-raspberrypi`,
   `linux-rpi-legacy` — themselves stale/legacy names, compounded by the
   original script's `armv6l|armv7l` case assigning
   `KERNEL_PACKAGES` twice, so only the second name was ever tried).
   Candidate list is now `linux-rpi linux-raspberrypi
   linux-raspberrypi4 linux-rpi-legacy` for armv6l/armv7l and
   `linux-aarch64 linux-rpi linux-raspberrypi4` for aarch64, covering
   current and legacy ALARM naming; still arch-generic (first match
   wins), no hardcoding beyond the candidate names.
2. When *no* candidate package resolved (e.g. the list is stale again
   someday), the code compared the running version against an empty
   string, which is always "not equal" — a guaranteed false "reboot
   needed". Now an empty lookup result is reported honestly instead
   (`installed kernel package not found (tried: ...)`) without setting
   the mismatch flag; in the standalone `nagios-arch-installed-kernel`
   this returns `UNKNOWN` instead of a false `WARNING`.

## Future idea: human-readable advisory enrichment — not implemented

Found while exploring the tracker API: `security.archlinux.org` also
exposes `/CVE-xxxx.json` (adds a plain-English `description`, `vector`)
and `/advisory/feed.atom` — a real Atom feed of official Arch Security
Advisories (ASA-xxxx), each entry a full human bulletin (Summary,
Resolution with the exact `pacman -Syu "pkg>=fixedver"` command, Impact).
Much more end-user-readable than the terse `all.json` dump this script
uses today, but ASA coverage is a small curated subset — most tracked
issues never get a formal advisory — so it would complement, not
replace, the current `all.json`-based approach. Would need Atom parsing
(e.g. `xmllint --xpath`), an added dependency not yet taken.

## Tradeoffs

Items 1-3 are free: pure `pacman` queries, no new dependencies. Item 4
took the `curl` + `jq` dependency over hand-rolled JSON parsing in shell.
Item 5 is free: builds on item 4's dependencies and cache, no new ones.
