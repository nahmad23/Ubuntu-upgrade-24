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
   that isn't Ubuntu 22.04, requires ≥ 10 GB free on `/`, and verifies the
   Ubuntu archive is reachable.
2. **Patch 22.04 first** — full `dist-upgrade` + `autoremove` (the release
   upgrader refuses to run on an out-of-date system), then reboots only if a
   new kernel requires it.
3. **Prepare the upgrader** — installs `update-manager-core`, sets
   `Prompt=lts` in `/etc/update-manager/release-upgrades`, clears stale
   upgrader state from any previous failed attempt.
4. **Release upgrade** — runs
   `do-release-upgrade -f DistUpgradeViewNonInteractive` with
   `DEBIAN_FRONTEND=noninteractive` and `NEEDRESTART_MODE=a`, so every
   question (config-file conflicts, service restarts, "remove obsolete
   packages?") is answered automatically. Existing config files are kept
   (`confold` behaviour).
5. **Reboot & verify** — reboots into 24.04, re-gathers facts, and **fails
   the host** if it still reports 22.04.
6. **Cleanup** — purges leftover packages and does a final reboot only if
   required.

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
# Same batching/--limit options apply
ansible-playbook reenable-third-party-repos.yml
```

It finds everything the upgrader disabled in `/etc/apt/sources.list.d/`
(both classic `.list` files and deb822 `.sources` files), re-enables the
entries, rewrites the suite `jammy` → `noble`, then runs `apt-get update`
and **fails the host with a list of any repository that doesn't publish
`noble` packages yet** so you can handle those few by hand. Repos that use
a fixed suite (e.g. Google Chrome's `stable`) are re-enabled but not
rewritten. Use `-e rewrite_codename=false` if you want to re-enable
without touching suites at all.

## Important notes / caveats
- **Runtime**: expect 30–90 minutes per server depending on package count,
  disk, and network. The playbook allows up to 2 h per host
  (`release_upgrade_timeout`) before giving up.
- **SSH resilience**: the release upgrade runs via Ansible `async`, and
  `ansible.cfg` sets aggressive SSH keepalives, so a brief `sshd` restart
  during the upgrade won't kill the run.
- **22.04 → 24.04 only.** The playbook hard-asserts the source version. For
  20.04 hosts you must go 20.04 → 22.04 first (LTS upgrades can't skip a
  release).
- Requires Ansible ≥ 2.12 on the control node and Python 3 + `sudo` on the
  targets (standard on Ubuntu 22.04).
