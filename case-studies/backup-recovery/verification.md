# Backup & Recovery: Verification and Residual Risk

This document supports the [main case study](README.md). It separates the original migration evidence, current status checks, restore-test evidence, and the limits of what those checks prove.

No repository credentials, private keys, production archives, or real internal identifiers are included.

## 1. Acceptance principles

The project used four evidence classes:

1. **Storage evidence:** the destination filesystem, content schema, mount state, and actual archive location.
2. **Runtime evidence:** timers, repository status, fresh snapshots, and service health.
3. **Recovery evidence:** real restore tests and fresh restorable archives.
4. **Negative controls:** wrong destinations are rejected, unmounted paths are unwritable, and backup access is constrained.

A successful scheduled task is useful, but it is not equivalent to recovery proof.

## 2. Original migration verification

| Check | Evidence | Result |
|---|---|---|
| Existing archive migration | Copied archives were compared before originals were removed | Passed; file counts matched and dry-run diff was empty |
| Root filesystem recovery | Root usage before and after migration | Dropped from 100% to 26% |
| Dedicated volume | Thin LV formatted with ext4 and registered as file-level storage | Passed |
| New archive destination | A real vzdump archive was generated on the new volume | Passed without growing root |
| Recurrence prevention | `backup` removed from root-backed `local` content | Confirmed |
| Unmounted-volume guard | Underlying mountpoint deliberately made unwritable | Confirmed as a fail-closed control |
| Guest coverage | Fresh archives produced for all covered guests after migration | Passed |
| Host/device coverage | Real restic backups completed through restricted SFTP | Passed |
| Restore tests | Host and device restore tests were performed after deployment | Passed |

The original root measurement was 26% immediately after recovery. The current 29% measurement below is a later state, not a contradiction or a claim that usage never changes.

## 3. Current read-only status

These checks were performed on **2026-11-26** without starting a backup, restore, prune, or configuration change.

| Check | Current evidence | Interpretation |
|---|---|---|
| Hypervisor root | 94 GB filesystem, 29% used | Root is not approaching the original full-disk failure state |
| Dedicated backup volume | Approximately 1 TB thin volume, 24% used | Mounted and active with available capacity |
| Restic repository | 178 snapshots | Repository has versioned recovery history |
| Latest restic snapshot | 2026-11-26 at 02:30 | The daily client schedule produced a current snapshot |
| Backup timer | Active and scheduled | Scheduling is currently enabled |
| Prune timer | Scheduled and last ran on 2026-11-22 | Retention is operating, subject to its next scheduled runs |
| Secondary-node storage | Disabled | Expected while the secondary node is offline; not active redundancy |

Snapshot output was deliberately reduced to count and timestamp. No repository paths, hostnames, or credential contents were displayed.

## 4. Restore and transport evidence

### Restore testing

Restore tests were completed for host and device data after deployment. That is stronger evidence than an archive count because it proves data can be read and used for recovery.

No fresh production restore was needed for this publication. The case study does not claim that every application, database, or future guest has since been restored again.

### Restricted transport

The transport checks included:

- a real backup through the restricted SFTP path;
- forced-SFTP behavior rather than a general interactive administrative session;
- uniform repository ownership across local and remote writers;
- dedicated per-client identities;
- real backup and forced-SFTP checks after key replacement before old enrollment removal.

These are operational checks, not merely an `authorized_keys` configuration statement.

### Retention

Retention uses daily, weekly, and monthly history with scheduled pruning. The documented policy is **7 daily / 4 weekly / 6 monthly**.

Retention controls growth and history. It does not guarantee capacity before a new archive is written, and it does not replace the root-destination and mountpoint guards.

## 5. Control verification matrix

| Control | Intended failure prevented | Verification |
|---|---|---|
| `backup` removed from root-backed `local` | New backup job accidentally fills root through `local` | Configuration rejects that destination |
| Dedicated file-level volume | Backup files are confused with thin-pool block storage | Real archives land on the mounted ext4 volume |
| Mode-000 underlying mountpoint | Missing backup mount silently writes onto root | Unmounted path fails closed |
| Restricted SFTP service | Backup client gains unnecessary interactive administration | Forced-SFTP and real backup checks pass |
| Per-client identities | One shared credential prevents practical revocation | Clients can be rotated/revoked independently |
| Uniform repository ownership | Local/remote writers create permission drift | Both paths use the same restricted write model |
| Retention/pruning | Unbounded version growth | Daily/weekly/monthly retention and scheduled prune are active |
| Password-manager copy | Encrypted repository becomes unrecoverable with lost password | Password availability is explicitly tracked as a recovery dependency |
| Restore tests | Archives exist but cannot be used | Host/device restore tests passed |

## 6. Negative and boundary checks

A healthy backup architecture also needs to demonstrate what it does not allow:

- Root-backed `local` does not accept new backup content.
- An unmounted backup path is not a writable fallback.
- A backup client does not receive a general interactive administrative channel.
- Public artifacts contain no repository password, private keys, production archives, or live repository configuration.
- An offline secondary node is not counted as an active backup copy.
- A snapshot is not treated as a portable backup.
- A disk archive is not automatically treated as an application-consistent database backup.

## 7. Evidence-quality rules

| Observation | What it proves | What it does not prove |
|---|---|---|
| Backup timer is active | A schedule is enabled | Every archive completed or can be restored |
| Snapshot count increases | New repository snapshots exist | Offsite redundancy or application consistency |
| Archive file exists | A file was produced | It is complete, current, portable, or restorable |
| Root usage is low now | Root has current capacity | The destination schema can never regress |
| Mountpoint is present | A directory exists | The dedicated filesystem is mounted |
| Restore test passed | The tested data could be restored | Every application or future workload has equivalent recovery coverage |
| Restricted key line exists | The intended access control was configured | The replacement key works until a real forced-SFTP/backup check passes |
| Throughput improved | The storage performance workaround helped | RAID reliability, backup independence, or restore correctness improved equally |

## 8. Residual risk and follow-ups

### No active offsite or secondary copy

The primary repository shares a physical host and array with production workloads. The secondary-node mirror is suspended and its storage is disabled while that node is offline.

A hypervisor, array, site, or correlated administrative failure could therefore affect production data and the primary backup repository together. This is not a complete 3-2-1 or offsite disaster-recovery design.

### Shared physical array

The dedicated thin LV fixes the root-storage and mountpoint failure modes, but not physical disk/array loss. Hardware failure domains remain shared.

### RAID battery/cache repair remains open

The physical-drive write-cache workaround improved RAID-5 sequential writes from 13.5 to 93.3 MiB/s, but it is not equivalent to a protected controller write-back cache. The battery/module replacement, cache-model reversal, and follow-up benchmarks remain open work.

### Application consistency is separate

A Proxmox disk archive does not automatically quiesce every application or guarantee transactional consistency for every live database. Workload-specific dumps, quiescing, and restore tests remain necessary where relevant.

### Password availability remains critical

Encrypted restic data is unrecoverable without its repository password. Keeping that password in a password manager is an operational recovery requirement and must not be replaced by a public example or a note in the repository.

### Restore coverage needs maintenance

The restore tests prove the tested host/device paths at the time they were run. They do not prove every future workload, every application dependency, or every later archive indefinitely. Restore testing should be repeated deliberately as the environment changes.

## 9. Acceptance statement

The Backup & Recovery architecture is accepted as production because the original root-destination failure was corrected at the configuration and mount layers, real guest archives and encrypted device backups were produced, restricted transport was exercised, retention was configured, and restore tests passed.

The accepted design still has explicit shared-host, shared-array, offsite, application-consistency, hardware, and password-availability limitations. It should be described as **restore-tested central backup infrastructure with residual disaster-recovery work**, not as a fully redundant offsite recovery system.

## Related documents

- [Main case study](README.md)
- [Incident investigations](incidents.md)
- [Selected implementation artifacts](https://github.com/v0nneumAn/homelab-backup-recovery)
- [Homelab case-study index](../../README.md)
- [Portfolio project](https://jasper.devsnull.net/projects/backup-recovery)
