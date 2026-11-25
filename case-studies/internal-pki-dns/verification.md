# Internal PKI and DNS: Verification and Limitations

This document supports the [main case study](README.md). It records what was checked to establish trust, issuance, DNS, renewal, and recovery behaviour, and which limitations remain explicit.

Examples use roles and placeholders rather than real internal identifiers.

## 1. Acceptance principles

The project used four evidence classes:

1. **Architecture invariants:** properties such as an offline root with no network interfaces or a proxy with no added capabilities.
2. **Protocol evidence:** real ACME issuance, TLS chain verification, UDP/TCP DNS responses, and valid NTP/DHCP/HTTP behaviour where relevant.
3. **Runtime evidence:** current service state, container health, live certificate chains, and application responses.
4. **Negative checks:** private key material is not exported, proxy exposure is limited, management remains restricted, and public artifacts contain no operational secrets.

A successful configuration command was not accepted as proof by itself. The relevant service path also had to work.

## 2. Trust architecture verification

| Control | Evidence | Result |
|---|---|---|
| Offline root isolation | Root VM has no network interfaces | Confirmed |
| Root key protection | Root private key is passphrase-encrypted and remains in the offline VM | Confirmed |
| Public root distribution | Only the public root certificate is used for client trust | Confirmed |
| Intermediate signing | Intermediate CSR was signed by the offline root | Confirmed |
| CA privilege boundary | `step-ca` runs as an unprivileged user with constrained capability and filesystem access | Confirmed |
| ACME provisioner | ACME directory was reached through the trusted private chain | Confirmed |
| End-to-end test issuance | A test certificate was issued through ACME and verified against the root certificate | Passed |
| Client trust | Relevant system, browser, and device trust stores were configured deliberately | Completed for the intended clients; not assumed universal |
| Live service chains | Served certificates were inspected with tools such as `openssl s_client` | Verified for onboarded roles |

### Negative trust checks

- The root private key is not included in backups or public artifacts shown here.
- ACME account state is not included in the public artifact repository.
- Public example configs contain no production certificates, private keys, or account credentials.
- The artifact repository is not a recovery source for the CA itself.

## 3. DNS verification

| Path or property | Evidence | Result |
|---|---|---|
| UDP DNS | Raw UDP queries through Pi-hole and the DoH proxy return responses | Passed |
| TCP DNS | TCP queries and DNS-length-prefixed responses complete | Passed |
| Internal names | Split-horizon records resolve only through the intended internal path | Passed |
| External names | Non-local queries return through the encrypted DoH upstream | Passed |
| Bootstrap independence | Proxy startup does not require Pi-hole to resolve the public DoH endpoint first | Confirmed by configuration and successful startup |
| Proxy privilege | All capabilities dropped, `no-new-privileges`, read-only root filesystem and script mount | Confirmed in inspected configuration |
| Proxy exposure | No public proxy port; only the intended container network path is used | Confirmed |
| Executable temporary files | Proxy and service temporary mounts are constrained with no-exec options where configured | Confirmed |
| Current runtime | Pi-hole reports healthy and the DoH proxy is running | Verified by read-only inspection |

The wider network also performed UDP and TCP DNS tests from multiple zones during segmentation. Those results are documented in the [network-segmentation verification record](../network-segmentation/verification.md).

### DNS design boundary

The DoH proxy is not a validating recursive resolver. Upstream DNSSEC validation is delegated to Cloudflare. A successful response proves the forwarding path, not local recursive validation.

## 4. Renewal lifecycle verification

| Behaviour | Evidence | Result |
|---|---|---|
| Standard renewal | ACME renewal completes and deploys the intended certificate, chain, and key | Passed for onboarded services |
| Service pre-hook | Service is stopped only where standalone validation requires it | Confirmed by host-specific config |
| Always-run cleanup | The always hook restores the service on success, failure, and early exit | Implemented in the wrapper and exercised during recovery |
| Not-due result | ACME exit code 2 is treated as a successful no-op; cleanup still runs | Verified in a real not-due renewal |
| Certificate permissions | Leaf, chain, key, and assembled TLS material are installed with intended modes/ownership | Verified per service |
| Container readability | Mounted TLS material is read directly as the container user | Passed after correcting ownership assumptions |
| Application recovery | Live TLS and application responses are checked separately from renewal exit status | Passed |

### Why application checks are separate

A successful ACME command does not prove the service adopted the certificate or recovered its application path. Conversely, an HTTP `403` or redirect can prove the TLS/application path was reached and should not automatically be diagnosed as a certificate problem.

## 5. Migration and identity verification

### SAN changes

Changing the SAN list required fresh `--issue` rather than relying on `--renew` to adopt new identifiers. Newly issued certificates were checked for the intended names during staged migration.

### ACME account location

An account registered at the old directory was rejected at the new directory with an account-location mismatch. Registering at the current directory restored the expected identity path. Account contents remained private.

### Proxmox and service-specific behaviour

Where the Proxmox-native ACME path could not finalize the affected IP-SAN CSR, `acme.sh` provided the alternative issuance path. The live Proxmox certificate and key paths were verified rather than assumed from documentation.

### HTTP-01 path

Validation was tested across:

```text
CA → routed path → host firewall → validation listener → challenge files
```

The working design used an unprivileged ACME client, a separate root-owned temporary listener with `--webroot`, and only the required firewall access. Container readability was checked as the runtime user rather than inferred from host ownership.

## 6. Recovery verification

### DNS/ACME dependency recovery

After the unsafe renewal and asymmetric-route incidents:

- the DNS service was restored;
- duplicate network management was removed;
- the connected route was restored persistently;
- CA validation-name fallback records were added through a persistent configuration path;
- renewal moved to the always-hook wrapper;
- ACME and TLS were re-tested successfully.

This established that the CA symptom was downstream of resolver and routing failures rather than an unexplained CA defect.

### OpenWrt upgrade recovery

A later attended router upgrade was recovered using validated private backups and selective restoration:

- the required ACME implementation and dependencies were confirmed in the package manifest and installed system;
- only missing ACME account state and custom CA trust material were restored;
- the known-good network configuration was left intact;
- the renewal script returned the correct not-due result;
- the always hook restarted the web service;
- ACME directory TLS and router HTTPS passed.

The exact WAN DHCP trigger during that incident remains unproven and is not presented as permanently fixed. The full incident is recorded in the [network-segmentation troubleshooting record](../network-segmentation/troubleshooting.md#investigation-5-openwrt-upgrade-recovery-and-selective-acme-restoration).

## 7. Evidence-quality rules

| Observation | What it proves | What it does not prove |
|---|---|---|
| ACME directory responds | The directory is reachable through a TLS path | Every account, SAN, validation, or client trust path works |
| Certificate file exists | A file is present | The service loaded it or serves it |
| Renewal exit status is zero | The ACME command succeeded or was not due | Application recovery and live TLS are correct |
| Container is running | A container process exists | DNS responses, upstream health, or metrics coverage is complete |
| Host file has intended ownership | Host-side metadata looks correct | A remapped container user can read the file |
| Browser trusts one store | One browser/system store has the root | Every device and browser trusts the CA |
| Root VM has no NICs | Root has no ordinary network path | The physical host cannot affect root and intermediate together |
| Example Compose validates | YAML and substitution are structurally valid | The example is a reviewed production deployment or backup |

## 8. Explicit limitations and follow-ups

### Physical-host consolidation

The offline root and online intermediate currently share a physical hypervisor. The root's no-NIC boundary remains meaningful for network attacks, but physical-host failure or compromise can affect both tiers. This is an accepted current trade-off, not a claim of independent physical failure domains.

### DoH health and metrics

The proxy still lacks a dedicated health endpoint and Prometheus metrics. Current container status is useful evidence, but it is not a complete query-latency, error-rate, or upstream-health observability design.

### Delegated DNSSEC

Upstream validation is delegated to Cloudflare. Migrating to local validating recursion would be a different architecture with different bootstrap and privilege requirements.

### Trust distribution

Root trust installation remains a deliberate per-client responsibility where applicable. Public artifacts do not attempt to automate trust on every device.

### Artifact scope

The public `dns-ca-stack` repository contains selected source and illustrative examples. It is not:

- a backup of production;
- a CA recovery image;
- a source of private keys or ACME account state;
- a newly reviewed, version-pinned deployment baseline;
- proof that the example Compose stack was started in this publication workflow.

### Upgrade and migration uncertainty

Where evidence was insufficient—such as the later WAN DHCP trigger—the case study records the uncertainty rather than inventing a root cause. Deferred health/metrics and physical-placement work remain open rather than being silently marked complete.

## 9. Acceptance statement

The Internal PKI and DNS implementation is accepted as production because the trust architecture, ACME issuance, encrypted DNS forwarding, renewal cleanup, and service recovery were exercised through real protocol and application paths.

The remaining work is explicitly bounded: physical-host consolidation, DoH health/metrics, delegated DNSSEC, deliberate trust distribution, and the difference between public examples and restorable production state.

## Related documents

- [Main case study](README.md)
- [Incident investigations](incidents.md)
- [Selected implementation artifacts](https://github.com/v0nneumAn/dns-ca-stack)
- [Homelab case-study index](../../README.md)
- [Portfolio project](https://jasper.devsnull.net/projects/internal-pki)
