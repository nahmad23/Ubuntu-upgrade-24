# Ubuntu 22.04 → 24.04 Fleet Upgrade (Ansible)

Fully unattended in-place upgrade of Ubuntu 22.04 LTS (Jammy) to
Ubuntu 24.04 LTS (Noble) across a fleet of servers — **no prompts at any
stage**.

## Repository layout

```
├── ansible.cfg                        # SSH keepalives, logging, forks
├── inventory/hosts.ini                # Your server list (edit this)
├── upgrade-ubuntu-22-to-24.yml        # Step 1: the upgrade playbook
├── reenable-third-party-repos.yml     # Step 2: restore PPAs/vendor repos
└── README.md
```

## What the playbook does

Per host, in order:

1. **Pre-flight checks** — skips hosts already on 24.04, aborts on anything
   that isn't Ubuntu 22.04, requires ≥ 10 GB free on `/` **and ≥ 500 MB on
   `/boot`** (a full `/boot` is the most common way a release upgrade dies
   half-finished), aborts on held packages, and verifies the noble archive is
   reachable **via the mirror the host is actually configured against**.
2. **Quiesce the host** — masks `apt-daily*.timer` and
   `unattended-upgrades` so they cannot grab the dpkg lock mid-upgrade, waits
   for any in-flight apt/dpkg run to release its locks, then repairs any
   half-configured dpkg state (`dpkg --configure -a`, `apt-get --fix-broken`).
3. **Patch 22.04 first** — full `dist-upgrade` + `autoremove` (the release
   upgrader refuses to run on an out-of-date system), then reboots only if a
   new kernel requires it.
4. **Prepare the upgrader** — installs `update-manager-core`, sets
   `Prompt=lts` in `/etc/update-manager/release-upgrades`, clears stale
   upgrader state from any previous failed attempt, and confirms with
   `do-release-upgrade -c` that 24.04 is actually offered — otherwise the
   upgrade is a silent no-op that only surfaces as a confusing failure after
   the reboot.
5. **Release upgrade** — runs
   `do-release-upgrade -f DistUpgradeViewNonInteractive -m server`. See
   [Answering prompts safely](#answering-prompts-safely) below.
6. **Reboot & verify** — reboots into 24.04, re-gathers facts, **fails the
   host** if it still reports 22.04, and runs `dpkg --audit` to catch packages
   left half-installed.
7. **Cleanup** — purges leftover packages, does a final reboot only if
   required, and **removes the temporary non-interactive overrides** it
   installed in step 2.

## The `/usr` merge (a common hard blocker)

Ubuntu 24.04 refuses to upgrade a host that still has the historical split
layout, where `/bin`, `/sbin` and `/lib` are real directories rather than
symlinks into `/usr`:

```
ERROR Cannot upgrade system with unmerged /usr
```

`do-release-upgrade` aborts about ten seconds in. Because the non-interactive
frontend logs to `/var/log/dist-upgrade/main.log` rather than the terminal,
the visible symptom is a bare `rc=1` with **completely empty output** — which
is why this one is disproportionately hard to diagnose.

Hosts whose lineage predates 22.04 (installed as 18.04/20.04 and upgraded
since) are typically unmerged; images installed fresh as 22.04 already are.
That's why it hits some of a fleet and not others.

The playbook detects this and installs `usrmerge`, which performs the
conversion, before starting the release upgrade. Survey your fleet first:

```bash
ansible ubuntu_servers -m shell -a \
  'test -L /bin && echo merged || echo SPLIT-needs-usrmerge'
```

Two things worth knowing:

- `convert-usrmerge` **makes no changes at all** when it finds a file present
  in both `/lib` and `/usr/lib` (or `/bin` and `/usr/bin`). The package still
  installs "successfully", so the playbook re-checks the layout afterwards and
  fails the host rather than marching on into an upgrade that will abort.
  Those duplicates have to be resolved by hand.
- `usrmerge` lives in **universe** on some 22.04 images. If the install fails
  with "no package matching", enable universe on that host first.

Skip the whole step with `-e merge_usr=false`, which turns the situation into
an explicit pre-flight failure instead.

## Answering prompts safely

The upgrade is fully unattended, and every prompt is answered with the
*non-destructive* answer — nothing on the host is silently overwritten:

| Prompt | Handled by | Answer chosen |
| --- | --- | --- |
| dpkg conffile conflict (“keep or replace `/etc/foo.conf`?”) | `Dpkg::Options --force-confdef --force-confold` in a temporary `/etc/apt/apt.conf.d/99-ansible-release-upgrade` | **Keep the file on disk.** The maintainer's version is still written as `*.dpkg-dist` for later review |
| ucf-managed config files | `UCF_FORCE_CONFOLD=1` | Keep the existing file |
| “Which services should be restarted?” (whiptail checklist) | debconf preseed `libraries/restart-without-asking=true` | Restart them, rather than leave services running against deleted libraries |
| needrestart's interactive service list | `/etc/needrestart/conf.d/99-ansible-release-upgrade.conf` with `$nrconf{restart} = 'a'` | Restart automatically |
| “A newer kernel is available” pager | `$nrconf{kernelhints} = -1` | Suppressed; the playbook reboots explicitly anyway |
| apt-listchanges pager | `APT_LISTCHANGES_FRONTEND=none` | Skipped |
| Anything else | `DEBIAN_FRONTEND=noninteractive`, `DEBIAN_PRIORITY=critical`, `DistUpgradeViewNonInteractive` | Package default |

Two things worth knowing:

- **The overrides are temporary.** The apt and needrestart drop-ins are
  removed, and the apt timers unmasked, in an `always:` block — so this runs
  even when the upgrade fails. A host is never left permanently auto-answering
  config questions or with security updates disabled.
- **Nothing is auto-answered destructively.** `force-confold` never discards a
  config file you edited. Deliberately *not* preseeded is `grub-pc`'s
  `install_devices` question: answering it blindly can leave a host without a
  bootloader. If a host in your fleet asks it, handle that host by hand.

Add your own site-specific answers without editing the playbook:

```bash
ansible-playbook upgrade-ubuntu-22-to-24.yml \
  -e '{"extra_debconf_preseed": ["postfix postfix/main_mailer_type select No configuration"]}'
```

## Rolling batches (built-in safety for 100 servers)

The play runs with `serial: 10%` — 10 servers at a time for a 100-host
fleet. If more than `max_fail_percentage` (default 20%) of a batch fails,
the run stops before touching the remaining servers.

```bash
# Default: 10% of the fleet per batch
ansible-playbook upgrade-ubuntu-22-to-24.yml

# Fixed batch size of 5, stop if any batch has >10% failures
ansible-playbook upgrade-ubuntu-22-to-24.yml -e upgrade_batch=5 -e max_fail_pct=10

# Canary run against a couple of test servers first (recommended!)
ansible-playbook upgrade-ubuntu-22-to-24.yml --limit server01.example.com,server02.example.com
```

## Recommended rollout procedure

1. **Snapshot/backup every server first.** An in-place OS upgrade is not
   reversible — VM snapshots or verified backups are your rollback plan.
2. **Canary**: run against 1–2 non-critical servers with `--limit` and
   validate your applications on 24.04.
3. **Batch the rest**, watching each batch complete before the next starts
   (the playbook does this automatically via `serial`).
4. **Check `ansible-upgrade.log`** (written by `ansible.cfg`) and, on any
   failed host, `/var/log/dist-upgrade/` for the upgrader's own logs.

## Step 2: Re-enable third-party repositories

`do-release-upgrade` disables every third-party APT source (PPAs, Docker,
Grafana, HashiCorp, …) during the upgrade. Once a host is on 24.04, run:

```bash
ansible-playbook reenable-third-party-repos.yml

# --limit works as usual; batch with -e repo_batch=25%
ansible-playbook reenable-third-party-repos.yml -e repo_batch=25%
```

### Leaving third-party repos disabled

If you would rather **not** switch the vendor repos back on, and only need the
repositories that are already enabled to serve `noble`:

```bash
ansible-playbook reenable-third-party-repos.yml -e reenable_third_party=false
```

In that mode the playbook:

- leaves every source the upgrader disabled **disabled**, and prints which
  ones so the decision is visible in the run output;
- repoints sources that are **already enabled** but still name `jammy` — this
  is what `rewrite_enabled_sources` controls, and it is independent of
  `reenable_third_party`;
- leaves fixed suites (`stable`, and vendor suites like `pbiso`) alone,
  because there is no codename in them to rewrite;
- still runs `apt-get update` and fails the host on any repository that is
  genuinely broken.

Commented-out entries are never resurrected by the codename rewrite: the
search is anchored to active (non-`#`) lines only.

It finds everything the upgrader disabled in `/etc/apt/sources.list.d/`
(both classic `.list` files and deb822 `.sources` files), re-enables the
entries, rewrites the suite `jammy` → `noble`, then runs `apt-get update`
and **fails the host with a list of any repository that doesn't publish
`noble` packages yet** so you can handle those few by hand. Repos that use
a fixed suite (e.g. Google Chrome's `stable`) are re-enabled but not
rewritten. Use `-e rewrite_codename=false` if you want to re-enable
without touching suites at all.

Every file it edits is backed up alongside the original, so a bad rewrite is
easy to undo. The codename rewrite deliberately skips signing-key filenames —
`signed-by=/usr/share/keyrings/jammy-archive.gpg` keeps its name, because
renaming it to `noble-archive.gpg` would point the repo at a keyring that
doesn't exist. Suites (`jammy`, `jammy-updates`) and codenames in URLs are
still rewritten.

## Important notes / caveats
- **Runtime**: expect 30–90 minutes per server depending on package count,
  disk, and network. The playbook allows up to 2 h per host
  (`release_upgrade_timeout`) before giving up.
- **`poll` vs `timeout`** — these are often confused. `*_poll` is how often
  Ansible asks "are you done yet?"; lowering it does **not** make the upgrade
  faster, it only detects completion sooner and prints progress lines more
  often. `*_timeout` is what actually caps the runtime.

  ```bash
  # check every 5s instead of 15s (more progress output, no faster)
  ansible-playbook upgrade-ubuntu-22-to-24.yml -e release_upgrade_poll=5

  # allow 4 h per host instead of 2 h, for genuinely slow servers
  ansible-playbook upgrade-ubuntu-22-to-24.yml -e release_upgrade_timeout=14400
  ```
- **SSH resilience**: the release upgrade runs via Ansible `async`, and
  `ansible.cfg` sets aggressive SSH keepalives, so a brief `sshd` restart
  during the upgrade won't kill the run. An `async` job that runs out of time
  is treated as a failure — the playbook will not reboot a half-upgraded host.
- **On failure**, the playbook prints the tail of `/var/log/dist-upgrade/main.log`
  from the failed host before marking it failed, so the batch report usually
  tells you what went wrong without logging in.
- **`ansible.cfg` sets `host_key_checking = False`**, which accepts any host
  key presented. That's convenient for a fleet run but means the connection
  isn't authenticated — if you have a populated `known_hosts`, drop that line.
- **22.04 → 24.04 only.** The playbook hard-asserts the source version. For
  20.04 hosts you must go 20.04 → 22.04 first (LTS upgrades can't skip a
  release).
- **No collections required.** Every module used is `ansible.builtin.*`, so
  `ansible-core` alone is enough and there is nothing to install from Galaxy.
  `ansible.cfg` uses `callback_result_format = yaml` (ansible-core 2.13+) for
  readable output rather than the `community.general.yaml` callback, which was
  removed in community.general 12.0.0.
- Requires ansible-core ≥ 2.13 on the control node and Python 3 + `sudo` on the
  targets (standard on Ubuntu 22.04).
