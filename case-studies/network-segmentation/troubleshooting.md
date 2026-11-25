# Network Segmentation: Troubleshooting Investigations

These investigations are part of the [production network-segmentation case study](README.md). They record failures and recovery decisions because the diagnostic method is as important as the finished architecture.

Each record separates observed evidence, confirmed causes, and remaining uncertainty. Commands are illustrative and use role-based placeholders; they do not reproduce production addresses, hostnames, wireless credentials, or network identifiers.

## Investigation 1: Guest and IoT DNS returned “connection refused”

### Symptom

After the internal resolver moved from the legacy network into the management zone:

- guest clients still received DHCP leases;
- internet access by IP address still worked;
- UDP and TCP DNS failed immediately;
- both guest wireless paths showed the same behaviour;
- an already-trusted workstation could still resolve through the resolver.

The immediate failure was important. This looked less like an upstream timeout and more like an active refusal somewhere along the internal path.

### First evidence—and why it was insufficient

Initial checks showed that:

- the resolver listened on UDP and TCP port 53;
- its host firewall accepted the relevant input;
- the guest firewall rule counters incremented;
- the resolver worked from another client zone.

None of those facts proved the complete guest forwarding path.

A listening socket proves the application is bound locally. A host firewall rule proves one gate permits the traffic. A router rule counter proves a packet encountered that rule. None alone proves that the router's final accept chain matched the packet and forwarded it through the correct interface.

### Root cause

The DNS forwarding rules had the correct new resolver address but retained the old destination zone.

Conceptually, the rule said:

```text
source zone: guest
destination address: <internal-resolver>
destination zone: <legacy-zone>
service: DNS over UDP and TCP
```

The resolver was now reached through the management interface, not the legacy interface. The rendered firewall rule therefore jumped into an accept chain that expected legacy-zone egress. Packets did not satisfy that chain and fell through to rejection.

OpenWrt's reject behaviour explained the client-visible result:

- TCP DNS received an active reset.
- UDP DNS received an active rejection.

That produced `connection refused` rather than a silent timeout.

### Minimal correction

I changed only the destination zone on the guest and IoT DNS pinholes:

```text
destination zone: <legacy-zone> → <management-zone>
```

The resolver address, service ports, source zones, and the rest of the policy stayed unchanged. No broad guest-to-management or IoT-to-management forwarding was introduced.

### Verification

After reloading the firewall, I checked:

- the rendered rules now targeted the management-zone accept chain;
- guest UDP DNS succeeded;
- guest TCP DNS succeeded;
- internal and external names resolved;
- the corrected rule counters advanced on real requests;
- guest probes to unrelated management services remained blocked.

The positive and negative checks mattered equally. Restoring DNS must not silently restore broad management access.

### Reusable lessons

1. **A destination address and destination zone form one policy decision.** Changing only the address leaves the rendered forwarding path inconsistent.
2. **A rule counter is not an accept verdict.** Follow the packet into the target chain and inspect the expected egress interface.
3. **Refusal and timeout are different signals.** An active rejection can come from firewall policy; it does not prove the application is down.
4. **Test the client path that failed.** A healthy trusted-client query does not prove guest forwarding works.

---

## Investigation 2: NTP worked on the legacy network but not the new zones

### Symptom

During management migration, real NTP probes produced a split result:

- the legacy gateway answered normally;
- the new management gateway did not answer;
- the new trusted gateway did not answer.

The router already synchronized with upstream time sources, so upstream synchronization was not the missing piece.

### Choosing the intended time source

Before changing firewall rules, I identified which system was actually meant to serve NTP.

The DNS host was a time client only; it had no NTP listener. OpenWrt already had upstream synchronization and server capability. Using the router avoided adding another service and another cross-zone dependency.

This prevented a plausible but incorrect fix: opening forwarding to a host that was never serving NTP in the first place.

### Evidence

A socket listing made the service look broadly available, but the runtime service definition showed an interface restriction:

```text
ntpd ... -l -I <legacy-bridge> ...
```

In this command:

- `-l` enables server behaviour;
- `-I <legacy-bridge>` restricts the service to the legacy bridge.

The socket listing alone had hidden that interface-level constraint. Real client-mode NTP requests then confirmed the split: legacy succeeded while management and trusted requests failed.

### Root cause

The NTP server remained bound to the legacy interface after the new zone gateways were introduced. The problem was service binding first, not an assumed lack of broad NTP forwarding.

### Minimal correction

I removed the legacy interface restriction from the NTP service configuration. Restricted zones then received narrow router-input rules for UDP NTP, and DHCP was configured to advertise each zone's own router gateway as the time source.

An obsolete forwarding rule toward the non-existent DNS-host NTP service was removed rather than preserved “just in case.”

### Verification

Verification used real NTP protocol exchanges, not only port checks:

1. Send a client-mode NTP request.
2. Receive a valid server-mode response.
3. Confirm the responding address is the intended zone gateway.
4. Repeat from each zone included in the acceptance scope.

Management, trusted, and server paths passed during migration. A later real IoT client also passed NTP along with DHCP, UDP/TCP DNS, internet, and isolation checks. Tests for other deliberately inactive zones remained deferred rather than being inferred from those results.

### Reusable lessons

1. **Identify the actual server before opening access to it.**
2. **A wildcard-looking socket does not prove interface availability.** Inspect the runtime service command and bindings.
3. **Use a real protocol request.** An NTP client/server exchange proves more than a port counter or a zero-byte probe.
4. **Scope completion to the clients tested.** One zone's success is not evidence for an inactive zone.

---

## Investigation 3: Cluster configuration changed, but runtime quorum broke

### Symptom

The Proxmox cluster's communication address needed to move into the management zone. The configuration was edited and its version incremented, but the running cluster link initially remained on the legacy address.

After the cluster communication service was restarted to apply the change, cluster activity briefly became blocked even though the active node itself was healthy.

### Evidence

There were two separate state transitions to investigate.

#### 1. Stored configuration versus running link

The updated configuration had propagated, but runtime inspection still showed the old link. This proved that writing and distributing the configuration had not changed the active cluster communication path.

A service restart was required for the running link to adopt the new address.

#### 2. Restart versus expected votes

The second cluster node was intentionally offline. Before the restart, the cluster had been deliberately configured to operate with one expected vote.

After the restart, expected votes returned to two. With only one node online, runtime status reported the equivalent of:

```text
Quorate: No
Activity blocked
```

This explained why a correct network change could still produce a cluster-wide operational failure.

### Root cause

The immediate cause was not simply “the wrong IP address.” It was a combination of:

- configuration state not equalling runtime state;
- the restart being necessary to apply the new communication link;
- the restart changing the expected-vote state while the second node was intentionally offline.

The cluster therefore lost its intentional single-node quorum even though the new network path was available.

### Minimal correction

After confirming that the second node was intentionally offline and no cluster operation depended on it, I restored the deliberate single-node expected-vote setting:

```bash
pvecm expected 1
```

This was not a substitute for the network correction. The new communication link and the expected-vote state both had to be correct.

### Verification

Final checks covered more than the changed address:

```bash
corosync-cfgtool -s
pvecm status
systemctl is-active pve-cluster corosync pveproxy
findmnt /etc/pve
```

The evidence confirmed:

- the running link used the management path;
- cluster membership matched the intended active node;
- expected and total votes were correct for single-node operation;
- quorum was restored;
- required services were active;
- the cluster filesystem was mounted and usable.

Only then was it safe to continue with further production changes.

### Reusable lessons

1. **A newer configuration file does not prove a running service adopted it.** Check the live link.
2. **A service restart can change more than the setting being targeted.** Re-check expected votes and quorum after every relevant restart while a node is offline.
3. **Do not perform backup or cluster changes while quorum is unhealthy.** Restore a known operational state first.
4. **Rollback must respect cluster configuration versioning.** Blindly overwriting with an older configuration is not a safe rollback strategy.

---

## Investigation 4: A strong wireless signal led clients into an empty path

### Symptom

During the wireless migration, household clients experienced severe connectivity and performance problems.

One access point had returned to advertising the main wireless network even though earlier documentation described it as IoT-only. Clients could see a strong network, authenticate, and associate, but then failed to obtain a useful service path. Retry and failure counters were severe.

### Evidence

The investigation separated three layers:

1. **Wireless association:** the client could see and join the SSID.
2. **Bridge membership:** the wireless interface needed to be attached to the bridge connected to the intended VLAN.
3. **End-to-end network service:** DHCP, gateway, DNS, and internet traffic had to cross that bridge and the upstream trunk.

Runtime inspection showed that the access point's wireless interface targeted the trusted-zone network, but the corresponding bridge did not exist with working membership. The SSID was therefore advertising a path that the access point could not actually provide.

Useful checks included:

```bash
iw dev
for bridge in /sys/class/net/br-*; do
    echo "=== $bridge ==="
    ls "$bridge/brif/"
done
```

Other evidence prevented over-correction:

- router CPU and memory were healthy;
- WAN and managed-switch trunk health were clean;
- a client using the main OpenWrt access point achieved a healthy throughput control;
- a separate access point also showed poor rates and retries, which was not automatically the same fault.

### Root cause

The confirmed configuration fault was an enabled SSID whose wireless interface was not attached to a working trusted-zone bridge.

Strong signal proved radio reachability only. It did not prove DHCP, Layer-2 forwarding, or IP connectivity.

The separate low-rate/retry behaviour remained a distinct RF/driver/roaming question; it was not explained away by the bridge fault.

### Minimal correction

Recovery deliberately reduced the amount of changing state:

1. Disable the broken main-wireless interfaces so clients stopped being attracted into the failed path.
2. Temporarily simplify wireless advertising while assessing coverage and service.
3. Restore a minimal main SSID on the affected access points with the correct trusted-zone bridge and VLAN membership.
4. Ensure the daisy-chained access-point link carried the required VLAN on both sides.
5. Leave unrelated management and native-VLAN behaviour unchanged.
6. Re-test real client association, DHCP, and service before declaring recovery.

The main router/access point alone could not cover the household, so simply leaving the other access points off was not a viable long-term recovery.

### Verification and remaining uncertainty

Read-only runtime checks later confirmed that the affected wireless and VLAN interfaces were members of the intended bridge on both access points. Jasper reported acceptable household client service after the recovery.

That closes the confirmed client-trap configuration fault. It does **not** prove a permanent fix for the separate wireless performance issue:

- both access points still shared a channel;
- low rates and high retry/failure counters were still observed;
- some signal evidence was weak;
- rebooting one access point improved behaviour but did not diagnose its cause.

If symptoms recur, the next step is a controlled one-AP-at-a-time test with an awake client, recording BSSID, signal, negotiated rates, retries, and latency before making further changes.

### Reusable lessons

1. **An enabled SSID without a working bridge is a client trap.**
2. **Runtime state wins over the last documented AP role.** GUI edits or later changes can invalidate a handoff.
3. **Strong signal is not end-to-end connectivity.** Check association, bridge membership, DHCP, and traffic separately.
4. **A reboot can clear state without diagnosing it.** Record what remains unproven.
5. **Extra SSIDs add beacon and airtime overhead; they do not directly lower SNR.** Simplification helped incident isolation, but it is not a universal RF explanation.

---

## Investigation 5: OpenWrt upgrade recovery and selective ACME restoration

### Symptom and objective

A later attended OpenWrt upgrade was intended to preserve the production network, installed packages, custom scripts, certificates, and ACME state.

The upgrade exposed three separate problems:

1. the real ACME client and its dependencies disappeared from the custom image;
2. WAN DHCP repeatedly failed during the failed boot sequences;
3. internal DNS failed downstream of the WAN outage and eventually exhausted its concurrent-query capacity.

These were investigated as separate causal chains rather than treating every symptom as one firmware bug.

### Preparation that made recovery possible

Before upgrading, I created private backups and baselines including:

- a normal system backup;
- a separate safety archive containing ACME account state and the custom CA trust material;
- package and enabled-service manifests;
- hashes of critical configuration and script files;
- a checksum-verified rollback image.

The normal backup inclusion list was inspected rather than assumed complete. The generated archives were downloaded and validated before the upgrade.

Those artifacts allowed later recovery to be selective instead of destructive.

### Confirmed cause 1: dependency pruning removed the ACME implementation

The top-level `acme` package was a wrapper that supplied no actual client files. The implementation and support came from dependencies including:

- `acme-acmesh`;
- `acme-common`;
- `socat`;
- `openssl-util`.

The first custom image removed the wrapper without explicitly retaining the implementation. A package check reported no unresolved package names, but that did not prove the required dependency subtree would remain in the built image.

The target manifest and installed system later confirmed that the actual ACME executable and supporting packages were missing.

A corrected image explicitly included the implementation package. However, once the router was already running the new release, installing the missing package in place would have been lower risk than performing another flash. That process mistake is part of the record.

### WAN DHCP failure: recovered, but trigger unproven

During the failed sequences:

- LAN and management access could work while WAN had no lease;
- repeated DHCP discovery did not obtain a lease;
- shutdown logs were orderly rather than evidence of a kernel panic or watchdog reset;
- both custom images showed WAN/DHCP trouble, so adding ACME packages did not explain it;
- restoring the known-good network configuration and rebooting produced a valid lease.

What was not captured was enough early packet and first-boot evidence to identify why the upstream DHCP server did not answer.

Therefore, none of these claims are made:

- that the ACME correction caused the WAN failure;
- that one edited network option definitely caused or fixed it;
- that the underlying first-boot/DHCP behaviour has a proven permanent fix.

If it recurs, the correct response is to freeze further writes and capture WAN state and DHCP packets before another reboot.

### Confirmed cause 2: DNS failure was a downstream cascade

While WAN was unavailable:

- the DNS-over-HTTPS upstream timed out or reported unreachable networks;
- the internal resolver could not complete upstream queries;
- concurrent queries accumulated until the resolver reached its configured limit.

This explained the household DNS failure without requiring a theory that the router upgrade had erased the resolver's configuration.

After WAN recovery, restarting the DNS/DoH containers cleared the stuck backlog. That was recovery of downstream state, not proof that DNS had caused the WAN problem.

### Confirmed cause 3: ACME account state and custom trust were missing

The known-good network recovery preserved the working network configuration and corrected packages, but the active system still lacked:

- the ACME account/domain state;
- the custom private-CA addition to the system trust bundle.

The renewal scripts, scheduled jobs, and deployed certificate remained present. This meant a full configuration restore would have overwritten working state unnecessarily.

### Minimal correction

Jasper selectively restored only the missing ACME account state and custom CA trust material from the validated private safety archive.

No additional firmware flash was performed. The working network configuration was left intact.

### Verification

Final verification covered the complete application path:

- the ACME implementation and expected files were installed;
- the package manifest contained the required dependency set;
- the ACME account and domain state were readable;
- TLS verification against the private CA directory succeeded;
- the renewal script completed and correctly reported that renewal was not due;
- its always-run cleanup restarted the web service;
- router HTTPS returned a successful application response;
- WAN, UDP/TCP DNS, and public HTTPS were healthy.

### Process lessons

1. **A zero-missing package check is not a completeness proof.** Inspect the image manifest, installed package list, and actual executable.
2. **Do not treat a wrapper package as the implementation.** Understand what its dependencies supply before removing it.
3. **Prefer in-place package repair over another production-router flash when possible.**
4. **Readiness checks must include WAN and DNS health.** A target version string and one binary are insufficient.
5. **Unexpected production behaviour freezes further writes.** Understand the first incident before layering another correction on top.
6. **Restore the minimum missing state.** Selective recovery preserved a known-good network instead of overwriting it.
7. **Keep backup inclusion lists explicit and validate the archive.** The recovery worked because the required state was actually present and retrievable.

## Cross-incident method

The investigations share a repeatable method:

1. Define the exact failing path and client.
2. Separate application, host firewall, router policy, Layer-2, and alternate-route evidence.
3. Inspect runtime state rather than assuming configuration intent.
4. Change the smallest variable supported by the evidence.
5. Verify both the restored service and the boundaries that must remain denied.
6. Record unresolved causes instead of converting recovery into a false root-cause claim.

## Related documents

- [Main case study](README.md)
- [Verification and acceptance](verification.md)
- [Homelab case-study index](../../README.md)
- [Portfolio project page](https://jasper.devsnull.net/projects/network-segmentation)
