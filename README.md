# Homelab Engineering Case Studies

Engineering records from the self-managed production infrastructure behind [Jasper Hale's portfolio](https://jasper.devsnull.net).

These projects support real household services on owned hardware. The case studies focus on architecture, operational constraints, design trade-offs, failure investigation, verification, and recovery—not just the tools used.

## Selected work

| Case study | Status | What it demonstrates |
|---|---|---|
| [Production Network Segmentation](case-studies/network-segmentation/README.md) | Production | Seven-zone architecture, staged migration, firewall service contracts, remote access, incident diagnosis, rollback, and legacy retirement |
| [Internal PKI and DNS Infrastructure](case-studies/internal-pki-dns/README.md) | Production | Offline-root PKI, ACME lifecycle automation, failure-safe renewal, encrypted DNS, dependency-loop diagnosis, and selective recovery |

## How I approach infrastructure

- **Design around failure:** staged changes, current backups, explicit rollback paths, and acceptance checks are part of the implementation.
- **Verify runtime behaviour:** configuration intent is not enough; inspect rendered rules, live service state, and real client paths.
- **Test both directions:** prove required services work and unintended access remains denied.
- **Prefer evidence over assumptions:** follow a request across layers instead of treating every failure as a DNS, firewall, or application problem.
- **Record uncertainty:** separate confirmed root causes from unresolved contributing factors and deferred tests.
- **Keep recovery practical:** restore tests matter more than merely producing backup files.

## Reading the case studies

Each project combines a main engineering narrative with supporting material where useful:

- problem, constraints, and success criteria;
- architecture and important decisions;
- implementation and migration sequence;
- troubleshooting and post-incident analysis;
- positive and negative verification;
- outcomes, limitations, and future lessons.

## Publication scope

Public documents are deliberately sanitised. Real internal addresses, hostnames, network identifiers, credentials, private keys, and sensitive operational details are omitted or replaced with role-based examples. Diagrams and command excerpts explain the engineering method; they are not a turnkey copy of the production environment.

Private project notes remain the source of operational truth. This repository contains reviewed public case studies derived from that work.

## Elsewhere

- [Portfolio and project overview](https://jasper.devsnull.net/projects)
- [GitHub profile](https://github.com/v0nneumAn)
