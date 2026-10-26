# 01 — Infrastructure Overview

## Goal

Replace the legacy GLPI server with a fresh, secure, well-documented
Debian 12 deployment, as the first piece of a broader ITSM + Monitoring
stack for Mello Transportes.

## Topology

- **Hypervisor:** VMware ESXi 7.0 U2, host `192.0.2.10`
- **VM:** `SRV--GLPI`, provisioned from a fresh Debian 12 (Bookworm) net-install
- **Network interface:** `ens192`
- **Temporary IP:** `192.0.2.229/24` — used during build-out, while the
  legacy GLPI server (which currently holds `192.0.2.228`) is still
  running dashboards/tickets for end users
- **Production IP:** `192.0.2.228` — to be assigned once the legacy
  server is fully decommissioned
- **DNS / hostname:** `srv-glpi.example.com.br`,
  resolvers `192.0.2.2`, `192.0.2.1`

> Addresses and the domain above are placeholders — IPs use the
> `192.0.2.0/24` documentation range (RFC 5737), and the domain appears as
> `example.com.br` — not the real internal network. See the note in the
> root `README.md`.

## Why a staged cutover

Standing the new server up on a temporary address lets us install,
configure, and test GLPI 11 end-to-end without any risk to the
still-in-service legacy system. The final IP swap is a short, well
understood step (stop legacy service → reassign address → update DNS/
firmware references) rather than a live migration.

## Stack summary

| Layer | Choice | Why |
|---|---|---|
| OS | Debian 12 "Bookworm" | Free, stable, long support tail via Debian LTS (community-maintained, extends to ~2028 with no `sources.list` changes) |
| Web server | Apache 2.4 | Well documented, matches GLPI's official install guides |
| Database | MariaDB | Debian's default MySQL-compatible RDBMS, meets GLPI 11's ≥10.2 requirement |
| PHP | 8.2 (Debian's default for Bookworm) | Meets GLPI 11's ≥8.2 requirement, no third-party repo needed |
| ITSM application | GLPI 11 | Current stable major version (GLPI 10 is maintenance-only LTS); chosen deliberately over 10 for longer runway |

See `02-base-os-and-network.md` and `03-lamp-and-glpi-installation.md` for
the actual build steps.
