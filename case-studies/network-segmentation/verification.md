# Network Segmentation: Verification, Acceptance, and Rollback

This document supports the [production network-segmentation case study](README.md). It defines what “working” meant, which paths were tested, how isolation was checked, and which limitations remained after migration.

All examples are sanitised. Placeholders such as `<internal-resolver>` and `<management-gateway>` represent roles rather than real production identifiers.

## 1. Acceptance principles

A stage was accepted only when the relevant runtime evidence supported it.

### Positive verification

Prove that required behaviour works:

- a client receives the intended DHCP settings;
- internal and external names resolve over UDP and TCP;
- a real NTP client receives a valid server response;
- certificate validation and renewal paths complete;
- logs and metrics reach their collectors;
- intended administrators can use HTTPS and SSH;
- the public site remains reachable;
- the virtualisation cluster remains quorate and usable.

### Negative verification

Prove that the boundaries still hold:

- guest and IoT clients cannot administer management systems;
- a server cannot use an exception intended for a different infrastructure role;
- an unintended remote user cannot become an administrator;
- privileged remote SSH remains refused where policy forbids it;
- removing legacy paths does not silently re-enable broad interim forwarding.

### Completion is scoped

A successful test proves the path and conditions tested. It does not automatically prove:

- an inactive zone with no current client;
- every application on every client;
- a future offline device whose configuration has not yet been reviewed;
- a permanent fix for an intermittent wireless or upstream ISP symptom.

## 2. Core service acceptance matrix

| Area | Test and evidence | Result |
|---|---|---|
| DHCP and zone placement | Real clients received an address, gateway, DNS, and time settings appropriate to the intended zone | Passed for the active paths tested |
| Internal DNS | Queried local infrastructure names through the internal resolver | Passed |
| External DNS over UDP | Resolved public names from trusted, management, server, guest, and IoT paths included in testing | Passed after the relevant policy corrections |
| External DNS over TCP | Repeated resolution over TCP rather than assuming UDP proved both transports | Passed for tested guest, IoT, and infrastructure paths |
| NTP | Sent client-mode NTP requests and validated server-mode responses from intended zone gateways | Passed for management, trusted, server, and later real IoT acceptance |
| Private PKI / ACME | Verified CA directory TLS, account state, validation reachability, renewal behaviour, and the live served certificate | Passed |
| Logging | Confirmed the log collector was ready and current sources produced queryable records | Passed for configured active sources |
| Metrics | Confirmed collector-to-endpoint scrapes and resulting Prometheus target state | Passed for active targets; intentionally offline systems were not treated as failures |
| Management HTTPS | Connected through intended local and remote paths with valid certificate chains | Passed |
| Administrative SSH | Confirmed the intended unprivileged administrative user could authenticate through the approved remote path | Passed |
| Privileged remote SSH | Attempted the explicitly forbidden privileged login path | Refused as intended |
| Public portfolio | Requested the public HTTPS site through its Cloudflare/Tunnel/Caddy path | Passed |
| Proxmox cluster | Inspected running communication link, membership, expected votes, quorum, services, and cluster filesystem | Passed after restoring intentional single-node quorum |
| Legacy discovery | Ran a final discovery scan and inspected relevant hosts for legacy interfaces/routes/advertisements | Zero legacy hosts; old production addressing retired |

## 3. Isolation and policy checks

Security boundaries were tested alongside service restoration.

| Source | Intended permission | Boundary checked |
|---|---|---|
| Guest clients | Internet and explicit shared DNS/NTP services | No broad management or server access |
| IoT clients | Internet and selected shared/application services | Management isolation and no unrelated server access |
| Trusted administrator | Approved management interfaces and internal services | No implication that every trusted device should administer every system |
| Server workloads | Only documented dependencies and service flows | No use of infrastructure-only exceptions assigned to another role |
| Logging collector | Inbound logs and approved outbound metric scrapes | No administrative access inferred from monitoring permission |
| Private CA | Explicit validation and renewal paths | No broad access to unrelated client or management services |
| Remote administrator | Approved subnet routes, user-specific SSH, and management HTTPS | Unintended users and privileged remote SSH refused |

### Example: DNS success did not prove management exposure

After the guest DNS zone correction, the acceptance test included both:

```text
Guest client → internal resolver over UDP and TCP: allowed
Guest client → unrelated management service: denied
```

This prevented a successful DNS fix from hiding an overly broad forwarding change.

### Example: monitoring access did not imply administration

A metrics collector needed to reach a specific endpoint on a workstation. The rule was added for that source and service only. A local metrics response, a remote collector scrape, and the resulting monitoring target were then checked separately.

The permission did not grant the collector general administrative access to the workstation.

## 4. Remote-access acceptance

Remote access was tested from outside the household network rather than inferred from a VPN's connected state.

The acceptance path covered:

1. remote-client authentication;
2. approved subnet-route availability;
3. split DNS for the intended private names;
4. management HTTPS with a valid certificate;
5. SSH as the intended unprivileged administrative user;
6. refusal of the forbidden privileged SSH user;
7. access to the public portfolio through its normal public path.

A direct VPN connection alone was not enough. Split DNS, subnet routes, internal router policy, destination host firewalls, and application authentication were independent checks.

### Controlling alternate paths during client tests

A dual-homed test device can invalidate an isolation result. If a laptop has both a test-VLAN connection and a remote-access route, traffic may reach an internal destination through the second path even when the tested VLAN correctly denies it.

Later tests therefore controlled this explicitly by:

- disabling unrelated network paths where practical; or
- binding requests to the interface under test while retaining a separate administrative control path.

This produced stronger evidence without tearing down the connection needed to operate the test.

## 5. Evidence-quality rules

Several checks looked convincing but were insufficient on their own.

| Observation | What it proves | What it does not prove |
|---|---|---|
| Firewall counter increments | A packet encountered a rule | A downstream chain accepted and forwarded it |
| Listening socket | A process is bound locally | Every intended zone can reach it |
| Successful ping | One ICMP path works | DNS, TCP, TLS, or application behaviour works |
| HTTP application response | The application path was reached | The certificate or every policy path is correct |
| Configuration file saved | Intended state was written | The running service adopted it |
| Backup command completed | A command returned | An archive exists, is complete, and can be restored |
| Strong wireless signal | Radio association is possible | Bridge membership, DHCP, routing, or internet access works |
| Container is running | A process exists | The application is ready or its external dependencies are healthy |
| Old documentation says a device has a role | A previous intended state was recorded | The device still has that runtime role |

These distinctions repeatedly shortened investigations by directing the next check toward the uncertain layer rather than another unrelated configuration change.

## 6. Rollback and change controls

### Fresh backups before risky stages

Backups were taken before significant changes rather than only at the beginning of the project. Where archives were involved, their contents and hashes were checked and copies were stored privately off the source device.

No credentials or private backup artifacts are included in this public record.

### Temporary network application

Where supported, network changes used a temporary application mechanism such as `netplan try`. This provided an automatic path back if connectivity was not confirmed.

### Scheduled switch rollback

For lockout-risk managed-switch changes, a scheduled reload was armed before changing management state. After the new path was verified, the scheduled rollback was cancelled and the configuration was saved.

The important sequence was:

```text
backup → arm rollback → change → verify new path → cancel rollback → save
```

### Proxmox network checks

Proxmox network changes were inspected and dry-run where supported before application. Guest VLAN changes used clean stop/start cycles when runtime application was unreliable.

Cluster changes additionally required runtime link, expected-vote, quorum, service, and cluster-filesystem checks.

### Additive migration and staged removal

New networks and management paths were created before old ones were removed. The legacy environment remained a recovery path until dependency tests showed it was no longer carrying production traffic.

Retirement then proceeded in layers:

1. remote advertisements;
2. forwarding and NAT dependencies;
3. legacy DHCP and DNS behaviour;
4. host addresses one at a time;
5. router-facing legacy configuration;
6. final discovery and service acceptance.

### Selective restoration

During the later OpenWrt recovery, only missing ACME account state and custom trust material were restored from the safety archive. The working network configuration was not overwritten by a broad restore.

### Quorum safety

While the second cluster node was intentionally offline, quorum state was checked after every relevant cluster restart. No backup or further cluster change was allowed to continue while quorum was unhealthy.

## 7. Final migration result

At the end of the migration:

- all seven intended production zones existed with their defined roles;
- the server, trusted-client, and management migration stages were complete;
- the final service policy had replaced broad interim forwarding;
- the defined core service and isolation checks passed;
- remote access advertised only the selected networks;
- no legacy hosts were found in the final scan;
- no legacy production interface remained on the router, DNS host, active hypervisor, or managed switch;
- required physical parents and Layer-2 bridges remained in place.

This is the basis for calling the migration **complete and in production**. It is not a claim that every optional zone is currently populated or every possible future device has been accepted.

## 8. Explicit limitations and follow-ups

### Inactive-zone client tests

Some guest, camera, and DMZ live-client tests were deferred because those zones were not all actively serving clients at the time. Earlier guest and IoT faults had been fixed and tested, but final acceptance for every inactive zone was not inferred from that history.

### Wireless performance

The confirmed unbridged-SSID fault was corrected and household service was accepted. Separate low-rate, retry, shared-channel, driver, and roaming questions remain bounded rather than declared permanently resolved.

### Offline-device return

Offline or stopped devices need a safe-return review covering their intended zone, addressing, DNS, certificates, SSH, and—in the hypervisor's case—cluster configuration. Old configurations must not be restored blindly.

### Household-specific integrations

Device migrations that depend on a family account holder or external cloud behaviour remain separate operational tasks. They do not change the verified completion of the core network migration.

### Future services and labs

Future internet-facing services, dynamic VLAN assignment, additional wireless policy, and other experiments are intentionally outside the completed migration's scope.

## 9. Acceptance statement

The production network-segmentation migration is accepted because the intended architecture was implemented, required production services were tested, security boundaries were checked with negative tests, and the legacy network was retired with runtime and discovery evidence.

Remaining items are tracked as operational follow-ups with explicit scope. None is silently reclassified as complete, and no unresolved root cause is presented as proven.

## Related documents

- [Main case study](README.md)
- [Troubleshooting investigations](troubleshooting.md)
- [Homelab case-study index](../../README.md)
- [Portfolio project page](https://jasper.devsnull.net/projects/network-segmentation)
