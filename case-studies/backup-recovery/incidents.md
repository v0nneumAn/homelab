# Backup & Recovery: Incident Investigations

These investigations support the [main case study](README.md). They record the failures that shaped the architecture and the controls that prevent recurrence.

Real repository identifiers, hostnames, paths, and credentials are omitted or replaced with role-based examples.

## Incident 1: `keep-last=3` did not stop the root filesystem filling

### Symptom

A weekly Proxmox backup run failed after producing only a small number of guest archives. The hypervisor root filesystem had reached 100%, and several guests had recovery points older than expected.

### Evidence

The default `local` storage was being used for backup archives. It was not a separate backup disk: it was a directory on the 94 GB root filesystem.

The growth pattern was:

```text
one full archive run ≈ 47 GB
+ installation images
+ another archive run
≈ root filesystem capacity
```

The run that failed left log files without corresponding archives for several guests. Those recovery points were stale and shared the same full disk.

### Root cause

`keep-last=3` created a false sense of capacity safety. Pruning removes an older archive **after** a new archive is written. When the next archive cannot fit alongside the previous one, the job fails before pruning can reclaim space.

The underlying problem was the storage schema: root-backed `local` accepted backup content even though root was sized for the OS rather than repeated full guest archives.

### Minimal correction

- Move existing archives to a dedicated file-level volume after checking the copy.
- Repoint backup jobs to that volume.
- Remove `backup` from root-backed `local` so future jobs fail at configuration time instead of filling root.
- Restore fresh archives for affected guests rather than assuming old ones were adequate.
- Verify that the new archive path does not increase root usage.

### Verification

- Root usage dropped from 100% to 26% during recovery and is 29% at the latest check.
- A test archive landed on the dedicated volume without growing root.
- All covered guests received fresh archives.
- `local` no longer lists `backup` as accepted content.

### Reusable lessons

1. **Retention is not a capacity guarantee.** Understand when pruning runs relative to archive creation.
2. **A storage name is not a physical safety boundary.** Inspect the underlying filesystem and content schema.
3. **Fix the recurring configuration path, not just the full disk.**

---

## Incident 2: An unmounted backup volume could recreate the original failure

### Symptom and design risk

Moving backups to a mounted dedicated volume solved the immediate space issue, but it introduced a subtle recurrence path:

```text
backup volume is not mounted
  → Proxmox sees a directory storage path anyway
  → writes land in an empty mountpoint on root
  → the original root-exhaustion failure returns
```

There was also a storage-allocation constraint: the thin pool had no free extents for a conventional separate volume, and thin pools cannot be shrunk to create them.

### Evidence and root cause

A directory storage path is only a pathname. Proxmox does not automatically guarantee that the path refers to a mounted, independent filesystem before writing.

The correct fix needed to address both:

- a thin LV formatted with ext4 so backup files had a real file-level home;
- a failure guard for the times when that LV was not mounted.

### Minimal correction

- Create a thin logical volume inside the existing pool, format it with ext4, and mount it as the backup filesystem.
- Make the underlying mountpoint mode `000` so an unmounted path cannot accept backup writes.
- Add `nofail` and mount checks to the operational path so a missing volume fails closed rather than silently falling back to root.

### Verification

- The dedicated LV is mounted as ext4 and active in Proxmox.
- The unmounted mountpoint is intentionally unwritable.
- A test archive landed on the LV without changing root usage.
- Current status shows the backup volume at 24% used and root at 29%.

### Reusable lessons

1. **A mounted filesystem is not the same as a directory path.** Verify the mount before relying on it.
2. **Fail closed on the failure you already experienced.** The unmounted case must not silently become a root write.
3. **Thin allocation changes the mechanism, not the failure domain.** The LV separates schemas, not the physical array.

---

## Incident 3: SFTP access needed service boundaries, not general SSH access

### Symptom and design risk

Device backups needed a central transport for Proxmox hosts, the DNS host, router configuration, and workstation data. Using general administrative SSH would have given routine backup processes broader access than necessary and made key rotation/revocation harder.

There were also implementation traps:

- `sudo` stripped the restic password-file environment variable;
- an external SFTP subsystem ran through the login shell and complicated scripted access;
- repository ownership could differ between local and remote writers;
- heredoc-generated scripts lost backslashes if escaping was not checked carefully;
- `useradd --home-dir` did not create the intended service home without `-m`.

### Root cause

Backup transport was being treated as ordinary administrator access rather than a constrained service with its own identity, permissions, and lifecycle.

### Minimal correction

- Use a dedicated `restic` service account with `ForceCommand internal-sftp`.
- Give each client a dedicated key and require `restrict` during manual key authorization.
- Route even the host's own backup through the same restricted SFTP path so repository ownership stays uniform.
- Keep repository password handling out of `sudo` by exporting the variable explicitly in the intended execution context.
- Check generated scripts rather than assuming heredoc escaping is correct.
- Create service directories with the intended ownership/mode rather than relying on `useradd` defaults.
- Verify a real backup and forced-SFTP behavior before removing an old key during rotation.

### Verification

- Real client backups completed through the constrained path.
- Restricted SFTP worked without granting a general interactive administrative channel.
- Repository files retained uniform ownership.
- Replacement client keys were verified through real backups and forced-SFTP checks before old enrollments were removed.
- Timers continued running after key rotation.

### Reusable lessons

1. **Backup access should be a narrow service, not a spare administrator account.**
2. **Per-client identities make revocation and rotation practical.**
3. **Test the actual transport and write path.** `restrict` on an authorization line is not enough without a real backup and forced-SFTP check.
4. **Provisioning is not complete until generated scripts and service state are verified.**

---

## Incident 4: Cluster quorum changed while a backup was running

### Symptom

During consolidation of the CA VMs onto the primary Proxmox node, the secondary node was powered off while a vzdump ran. One backup failed with a permission error while Proxmox's cluster filesystem was read-only.

### Evidence

The secondary node went offline before `pvecm expected 1` was applied. In that interval, the cluster temporarily lost quorum and `pmxcfs` became read-only. A backup task then failed while trying to write a temporary configuration file.

The same migration also showed why a later corosync restart needs another quorum check: expected votes can return to two while only one node remains online.

### Root cause

Quorum-affecting actions overlapped with a backup job. The backup itself was not the root cause; it was a casualty of changing cluster availability while the cluster filesystem could not accept writes.

### Minimal correction

- Apply intentional single-node expected votes **before** or **after** backup windows, not during them.
- Re-check quorum before starting further cluster or backup operations.
- Re-run the failed archive only after `pmxcfs` was writable and the cluster was healthy.
- Keep both CA VMs on the active node temporarily and update their backup coverage.
- After every relevant corosync restart while the second node is offline, re-check expected votes, total votes, quorum, services, and the mounted cluster filesystem.

### Verification

- The cluster returned to intentional single-node quorum.
- `/etc/pve` was mounted and writable.
- Fresh CA VM archives completed on the active node.
- Reverse-migration preparation is documented rather than assuming the secondary node can be powered on with its old configuration.

### Reusable lessons

1. **Backup jobs depend on cluster health, not just free disk space.**
2. **Sequence quorum changes away from backup windows.**
3. **Re-check expected votes after corosync restarts; do not assume the previous single-node exception persists.**

---

## Incident 5: RAID cache failure distorted backup and restore performance

### Symptom

Restoring a VM disk took far longer than expected, with high I/O delay and a storage job that appeared to hang. RAID-5 sequential writes measured about **13.5 MiB/s**.

### Evidence

The Smart Array controller's write cache was permanently disabled because its backup power source was absent. Writes and parity operations therefore reached the disks synchronously.

Comparison measurements showed:

- RAID-5 before workaround: **13.5 MiB/s**;
- RAID-0 control: **91.6 MiB/s**;
- RAID-5 after enabling physical drive write cache: **93.3 MiB/s**.

### Root cause

The controller battery/module was dead, disabling the intended write-back cache. This was a hardware/cache issue, not evidence that Proxmox or the backup format was inherently slow.

### Minimal correction and accepted risk

- Enable physical drive write cache as a temporary workaround and avoid concurrent heavy disk jobs on the array.
- Record the trade-off explicitly: drive-cache enablement is not equivalent to restoring the controller's protected write-back cache.
- Plan a battery/module replacement, then disable drive write cache, restore the intended controller cache behaviour, and re-benchmark.

### Verification

- RAID-5 sequential write throughput improved from 13.5 to 93.3 MiB/s.
- VM disks could use the array without the earlier pathological stalls.
- The battery replacement and cache-model reversal remain open hardware work rather than a completed fix.

### Reusable lessons

1. **Benchmark before treating backup slowness as an application problem.**
2. **Record hardware workarounds as risk acceptance, not permanent repair.**
3. **Do not infer backup reliability from throughput alone.** Faster writes do not remove the need for independent copies and restore tests.

## Cross-incident method

The recurring method is:

1. Identify whether the failure is capacity, mount state, transport, identity, cluster health, hardware, or application consistency.
2. Inspect the relevant runtime evidence rather than assuming a generic backup failure.
3. Add a guard at the layer that can actually prevent recurrence.
4. Verify real archives, transport, and recovery—not just configuration.
5. Record residual risk instead of describing every workaround as a complete fix.

## Related documents

- [Main case study](README.md)
- [Verification and residual risk](verification.md)
- [Selected implementation artifacts](https://github.com/v0nneumAn/homelab-backup-recovery)
- [Homelab case-study index](../../README.md)
