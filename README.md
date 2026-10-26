# Acme Logistics — ITSM & Monitoring Infrastructure

Infrastructure-as-documentation repository for Acme Logistics' internal
IT Service Management and Monitoring stack, built on Debian 12 (Bookworm).

> **A note on the company name:** "Acme Logistics" is a fictional stand-in.
> This is a real build log from a real company's infrastructure, but the
> company's identity is not published here.

This repo tracks the design decisions, configuration, and step-by-step
build log for the project — not the deployed application code itself
(GLPI, Zabbix, etc. are third-party software installed on the server).

> **A note on addresses:** all IPs, hostnames, and the internal Active
> Directory domain shown in this repository are placeholders — IPs use
> the `192.0.2.0/24` block, reserved by [RFC 5737](https://www.rfc-editor.org/rfc/rfc5737)
> specifically for documentation and examples, and the domain appears as
> `example.com`. Every command and configuration shown was actually run
> in production; only these identifiers were swapped out before
> publishing.

## Project Phases

### Phase 1 — ITSM Core (in progress)
- [x] Base OS install & hardening (Debian 12)
- [x] Static network configuration
- [x] LAMP stack (Apache, MariaDB, PHP 8.2)
- [x] GLPI 11 installation & secure directory layout
- [x] Default credentials rotated, demo data disabled
- [x] Automatic actions verified running in CLI mode via cron
- [x] Active Directory / LDAP integration
- [x] Group Policy (GPO) rollout of GLPI Agent
- [x] Self-Service portal UX configuration

### Phase 2 — Monitoring & Communications (in progress)
- [x] Zabbix server + Grafana dashboards
- [x] Telegram bot integration (GLPI ticket alerts + Zabbix network alerts, zero recurring cost)

## Infrastructure Overview

| Component | Value |
|---|---|
| Hypervisor | VMware ESXi 7.0 U2 (`192.0.2.10`) |
| VM name | `SRV--GLPI` |
| OS | Debian 12 "Bookworm" |
| Web stack | Apache 2.4 + MariaDB + PHP 8.2 |
| ITSM application | GLPI 11 |
| Current IP (temporary) | `192.0.2.229` |
| Production IP (planned) | `192.0.2.228` |
| Monitoring stack | Zabbix + Grafana, `192.0.2.230` |

See [`docs/`](./docs) for the detailed build log and rationale behind each
configuration decision.

## Repository Structure

```
itsm-infra/
├── README.md
└── docs/
    ├── 01-infrastructure-overview.md
    ├── 02-base-os-and-network.md
    ├── 03-lamp-and-glpi-installation.md
    ├── 04-security-hardening.md
    ├── 05-ldap-active-directory-integration.md
    └── 06-monitoring-zabbix-grafana.md
```

## Conventions

- Commit messages follow [Conventional Commits](https://www.conventionalcommits.org/)
  (`feat:`, `fix:`, `docs:`, `chore:`, ...).
- Documentation is written in English; the deployed applications
  (GLPI, etc.) remain in Portuguese for end users at Acme Logistics.
- Credentials are **never** committed to this repository. See
  `.gitignore` and keep secrets in a password manager instead.
- Live security posture (current SSH policy, open hardening items) is
  tracked privately, not published — see `docs/04-security-hardening.md`.

## Author

Maintained by TI — Acme Logistics.
