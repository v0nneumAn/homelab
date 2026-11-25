# Internal PKI and DNS: Incident Investigations

These investigations support the [main case study](README.md). They focus on failures where DNS, ACME, routing, container permissions, and service lifecycle assumptions interacted.

Commands and identifiers are illustrative. Real internal addresses, hostnames, account state, private keys, and credential material are omitted.

## Incident 1: Certificate renewal took down the resolver that ACME needed

### Symptom

A certificate renewal for the DNS service turned into a broader name-resolution and ACME failure:

- the DNS container was left stopped;
- the CA could not resolve the names it needed for HTTP-01 validation;
- other certificate renewals began to look like independent CA failures;
- internal services could not resolve the management names their browsers and tools expected.

This was a dependency loop, not simply an expired certificate.

### Evidence

The renewal procedure used standalone HTTP-01. That required the DNS service to stop so the standalone ACME listener could use the validation port.

The script also used `set -e`. Its intended sequence was conceptually:

```text
stop DNS service
renew certificate
start DNS service
```

If renewal failed, the final start command was never reached.

At the same time, the CA used the DNS service for name resolution. Stopping the resolver therefore removed the very dependency ACME needed to resolve and validate the certificate request.

### Root cause

There were two interacting faults:

1. **Failure-unsafe lifecycle:** `set -e` allowed a renewal error to bypass service restoration.
2. **Circular dependency:** standalone validation stopped the resolver that the CA required for validation.

Restarting the CA or repeatedly forcing renewal would not address either cause.

### Minimal correction

The recovery combined lifecycle and dependency fixes:

- Restore the DNS service before treating the CA as the root problem.
- Move renewals into a reusable wrapper with an `EXIT` trap and an `ALWAYS_HOOK`, so service restoration runs on success, failure, and early exit.
- Treat the ACME client's “not due” result as a successful no-op while still running cleanup.
- Give the CA a small, controlled set of static validation-name records and a fallback resolver for resolver-down conditions.
- Preserve those records through the CA's provisioning/configuration path rather than relying on an undocumented manual edit.

The wrapper became the common lifecycle for DNS, web, monitoring, routing, and other service certificates. Host configs retained only their service-specific hooks and deployment requirements.

### Verification

- ACME directory access worked through the trusted private chain.
- A real renewal completed against the CA.
- The DNS service was restored by the always hook even when renewal was not due.
- Internal and external DNS resolution recovered.
- TLS and application checks were performed separately so an HTTP application response was not misread as a TLS failure.

### Reusable lessons

1. **A pre-hook needs an always-run recovery path.** A successful renewal is not the only path that must restore service.
2. **Map lifecycle dependencies before stopping a service.** The service being renewed may also be required by the validator.
3. **Do not mistake a CA symptom for a CA root cause.** The failed component can be downstream of DNS or service lifecycle state.

---

## Incident 2: One-way packet loss looked like a CA or TLS failure

### Symptom

The CA could reach some infrastructure peers but not the DNS host. From the DNS host, a TCP connection to the CA could begin, yet TLS negotiation stalled. The behaviour suggested a firewall or certificate problem, but the packet loss was asymmetric.

### Evidence

Layered checks showed:

- the DNS host and CA did not have reliable bidirectional IP reachability;
- replies from the DNS host to the CA were being sent to the router's MAC address instead of directly to the CA;
- two network managers were active on the DNS host's Ethernet interface;
- the normal directly connected route for the local subnet was absent.

The last two facts explained the routing behaviour. The interface had been configured to forward traffic through the router in a way that suppressed the ordinary connected route. Replies therefore took an indirect path that the router did not reliably hairpin back to the destination.

### Root cause

`systemd-networkd` and `dhcpcd` were both managing the same interface. The resulting route state removed the directly connected route and produced asymmetric forwarding between the DNS host and the CA.

The CA and ACME were not the original fault. TLS could not complete reliably over a broken return path.

### Minimal correction

- Disable the duplicate network manager.
- Restore an explicit directly connected `scope link` route in the DNS host's persistent network configuration.
- Keep the corrected configuration in the provisioning path so it survives reboot.
- Re-test IP reachability and the full ACME/TLS path before changing certificate settings again.

### Verification

- The DNS host and CA could exchange packets in both directions.
- TCP and TLS to the CA completed normally.
- ACME validation and renewal worked after the route and DNS lifecycle issues were corrected.
- The route persisted through the intended network configuration rather than as a temporary command-line fix.

### Reusable lessons

1. **Check both directions of a network path.** A successful outbound SYN does not prove replies take a valid route.
2. **TLS timeouts can be routing symptoms.** Do not begin by replacing certificates.
3. **Two network managers can produce non-obvious route state.** Inspect the actual routing table and interface ownership, not only assigned addresses.

---

## Incident 3: Unbound fought the container model and DNS bootstrap order

### Symptom

The original DNS design used Unbound for recursive resolution and encrypted upstream transport. In the intended container environment, the design encountered several independent constraints:

- the Unbound entrypoint's privilege-dropping path conflicted with `no-new-privileges`;
- remapped container users caused ownership problems;
- Pi-hole could not resolve the DoH endpoint while its own resolver was still starting.

Adding more configuration around those issues made the design harder to operate without improving the actual requirement.

### Evidence and reframing

The requirement was not “run a full validating recursive resolver.” It was:

- answer internal names from controlled records;
- forward other queries to an encrypted upstream;
- preserve the existing rootless/remapped Docker security model;
- start without a circular DNS dependency.

A small forwarding proxy satisfied all four. It did not need to perform local recursion or own the upstream DNSSEC trust decision.

### Minimal correction

I replaced Unbound with an 81-line standard-library Python proxy that:

- accepts DNS over UDP and TCP;
- forwards the raw DNS message over HTTPS;
- uses the platform TLS trust context;
- returns the raw DNS response;
- runs with no added capabilities, a read-only filesystem, and `no-new-privileges`.

Container configuration binds the public DoH hostname to its public resolver address, so startup does not require a working resolver before the proxy starts. The proxy has no public port and shares a small internal container network with Pi-hole.

### Verification

- Pi-hole forwarded both UDP and TCP queries through the proxy.
- Internal split-horizon records continued to resolve.
- External queries returned through the encrypted upstream path.
- The proxy ran read-only with no added capabilities and no public listener outside the intended container network.
- Current runtime inspection shows Pi-hole healthy and the DoH proxy running.

### Reusable lessons

1. **Choose the simplest architecture that meets the requirement.** Do not preserve local recursion merely because it is a common DNS design.
2. **Bootstrap dependencies belong in the design.** A resolver cannot depend on itself to discover its upstream.
3. **Record the trade-off.** Upstream DNSSEC validation is delegated to Cloudflare; this is not local validating recursion.

---

## Incident 4: Renewal reused old identities after the network changed

### Symptom

During network migration, some certificate renewals did not produce certificates with the new names or addresses. An ACME account that worked at the CA's old directory was then rejected when used at the new directory.

### Evidence

Three distinct identity issues appeared:

1. `acme.sh --renew` continued using the identifiers stored in the existing certificate configuration rather than adopting a newly supplied SAN list.
2. The CA rejected the old account at the new directory with an account-location mismatch (`kid does not match stored account location`).
3. Proxmox's native ACME path validated names but failed CSR finalization for IP identifiers because the CSR identifiers did not match exactly.

Treating any of these as a generic TLS or network error would have led to the wrong fix.

### Minimal correction

- Use fresh `--issue` when SANs change rather than relying on `--renew` to infer a new identity set.
- Register a new ACME account at the current CA directory instead of trying to make an old account's directory identity match by editing contact details.
- Use `acme.sh` for the affected IP-SAN workflow rather than assuming the Proxmox-native path supports every identifier type.
- Verify the actual service certificate paths and live served certificate after deployment.

### Verification

- The newly issued certificates contained the intended old and new names during staged migration.
- The current CA directory accepted the new account and issuance path.
- `openssl s_client` verified the live service chain and SANs rather than only checking files on disk.
- ACME account files and state remained private throughout recovery and publication.

### Reusable lessons

1. **Renewal is not reissuance.** A changed SAN set is a new certificate identity, not just a new address on an old service.
2. **ACME account location is part of account identity.** A working account at one directory does not automatically transfer to another.
3. **Verify what the service actually serves.** Configuration success and a file's existence are not enough.

---

## Incident 5: HTTP-01 exposed listener, privilege, and container-permission gaps

### Symptom

After CA and DNS issues were corrected, some HTTP-01 validations still failed. Additional symptoms included:

- an unprivileged ACME client could not bind the validation port;
- the target host firewall blocked validation traffic;
- a container could not read its assembled TLS material even though the files appeared present on the host.

### Evidence and root cause

HTTP-01 requires a complete path, not just an ACME client command:

```text
CA → routed network → host firewall → validation listener → challenge files
```

Separate failures existed at different points:

- an unprivileged standalone process could not bind the privileged validation port;
- a host firewall rejected the CA's validation traffic;
- remapped container users meant host-side ownership assumptions did not prove readability inside the container.

### Minimal correction

- Keep the ACME client unprivileged and use a separate root-owned temporary HTTP listener with `--webroot`, rather than running the entire ACME client with elevated privileges.
- Add only the validation access required for the CA and service path.
- Assemble the certificate and key with the intended service ownership and permissions.
- Verify readability directly as the container user.

Illustrative check:

```bash
docker exec --user <container-uid> <container-name> cat <mounted-tls-path>
```

### Verification

- The CA reached the validation listener and completed HTTP-01.
- The renewed service restarted through the always hook.
- The container could read the mounted TLS material as its runtime user.
- Live TLS and application responses were checked independently. An application `403` or redirect was treated as evidence that the application path had been reached, not automatically as a certificate error.

### Reusable lessons

1. **ACME validation crosses multiple security boundaries.** Check the whole path before changing the CA.
2. **Use the smallest privilege split that works.** A root-owned temporary listener can solve port binding without making the ACME client fully privileged.
3. **Host ownership is not container readability.** Test as the runtime user inside the container.

## Cross-incident method

The reusable sequence is:

1. Identify which dependency actually failed: DNS, route, validation, identity, privilege, or service lifecycle.
2. Inspect live state at that layer instead of assuming a CA or application fault.
3. Change the smallest supported variable.
4. Verify the protocol path and application recovery independently.
5. Preserve the lesson in a reusable script or acceptance check rather than relying on memory.

## Related documents

- [Main case study](README.md)
- [Verification and limitations](verification.md)
- [Selected implementation artifacts](https://github.com/v0nneumAn/dns-ca-stack)
- [Homelab case-study index](../../README.md)
