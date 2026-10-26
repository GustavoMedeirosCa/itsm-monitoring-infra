# 06 — Monitoring Stack: Zabbix + Grafana

## Goal

Phase 2's first milestone: stand up Zabbix for metrics collection and
Grafana for visualization, on a single VM alongside each other, and build
dashboards covering both the ITSM server itself (`SRV--GLPI`) and the
network edge (pfSense WAN links) — plus a first pass at service-level
metrics pulled directly from GLPI (open vs. closed ticket counts).

## Topology

- **VM:** Zabbix server + Grafana, co-located on one Debian 12 host,
  `192.0.2.230`
- **Grafana:** `http://192.0.2.230:3000/`
- **Zabbix frontend:** `http://192.0.2.230/zabbix/`
- **Monitored targets:**
  - `SRV--GLPI` (`192.0.2.229`) — OS-level metrics via Zabbix agent
  - `pfsense` — WAN link status and throughput via SNMP
  - GLPI itself — ticket counts, pulled live from its REST API (not
    through Zabbix)

> Addresses use the `192.0.2.0/24` documentation range (RFC 5737) — see
> the note in the root `README.md`.

## Dashboards built

| Dashboard | Panels | Data source |
|---|---|---|
| **GLPi** | CPU, free memory, disk usage (`/`), server uptime, chamados abertos vs. fechados | Zabbix (agent) + GLPI REST API |
| **pfSense — Links WAN** | Per-link status (`WAN_ALGAR`, `WAN_BLINK`), WAN traffic graph | Zabbix (SNMP) |

Both dashboards are shared internally via Grafana's **"Share internally"**
short links, with auto-refresh enabled, so they can be left open on a wall
monitor or TV and stay live without any manual interaction.

## Pulling ticket metrics from GLPI's REST API

GLPI's legacy REST API uses a **two-tier auth model** that isn't obvious
from the datasource setup alone:

1. The `App-Token` + `Authorization: user_token <token>` headers (set once
   at the Grafana datasource level) are enough to call `/initSession` and
   get back a `session_token`.
2. Those same headers are **not** enough for actual data endpoints like
   `/Ticket` — those additionally require a `Session-Token: <token>`
   header carrying the value from step 1. Without it, the API returns a
   generic `400 Bad Request` with no further detail visible through
   Grafana's own error surface.

The fix: a hidden Grafana **dashboard variable**, `session_token`, backed
by a query against `/initSession`, referenced as `${session_token}` in
the `/Ticket` panel's `Session-Token` header. This keeps the session
handshake out of every individual panel.

### Keeping the session alive across refreshes

The `session_token` variable's refresh setting matters more than it looks:

- **"On dashboard load"** (the obvious default) only calls `/initSession`
  once, when the page is first opened. Panels keep auto-refreshing every
  minute using that *same* token — which eventually goes stale once
  GLPI's session expires, and the ticket panel silently drops to **"No
  data"** with no visible error.
- **"On time range change"** re-evaluates the variable every time the
  dashboard's time window shifts — which a relative range (`now-6h` to
  `now`) does automatically on every auto-refresh tick. This re-runs
  `/initSession` on the same cadence as the panel refresh, so the token
  never has a chance to expire.

Switching to the latter resolved a recurring "it was working yesterday"
symptom.

### Access control on the API client

The dedicated GLPI API client used by Grafana ("Grafana — read-only
dashboard") is restricted to a single-IP allowlist (the Grafana server's
own address) under **Configuração > API > Clients d'API**. Verified
working as intended: a direct request from outside that one IP is
rejected with `ERROR_NOT_ALLOWED_IP` before it ever reaches ticket data.

### Known limitation

The `/Ticket` query requests `range=0-999` (up to 1000 tickets per call,
GLPI's REST API default page size is much smaller). Fine for the current
ticket volume; if the total ever exceeds 1000, the open/closed counts
will silently undercount until the query is changed to paginate or filter
by date.

## pfSense WAN link status: a false "DOWN"

The **WAN_BLINK** panel (a PPPoE link) initially showed **DOWN** in deep
red, while pfSense's own gateway monitor reported it `Online` with normal
latency and zero packet loss. Root cause: the panel reads the interface's
SNMP `ifOperStatus` value and maps it to UP/DOWN, and the original mapping
treated every value from `2` to `7` as "DOWN". Per the SNMP `IF-MIB`
standard:

| Value | Meaning |
|---|---|
| 1 | up |
| 2 | down |
| 3 | testing |
| 4 | unknown |
| 5 | **dormant** |
| 6 | notPresent |
| 7 | lowerLayerDown |

PPPoE interfaces commonly report `5` (**dormant**) while fully operational
and passing traffic — an artifact of how the PPP daemon signals interface
state over SNMP, not an actual fault. The value mapping was corrected to
treat `1` and `5` as UP, and only `[2-4]` / `[6-7]` as DOWN.

## Making shared dashboard links actually work off-host

Grafana's dashboard "Share" links (and the shortened `/goto/...` form)
embed a destination built from the server's own `root_url` setting. Left
at its out-of-the-box default (`http://localhost:3000/`), every generated
link resolved to **the viewer's own machine**, not the Grafana server —
links worked only when opened on the Grafana host itself, and failed
everywhere else with `ERR_CONNECTION_REFUSED`.

Fixed in `/etc/grafana/grafana.ini`, under `[server]`:

```ini
domain = 192.0.2.230
root_url = http://192.0.2.230:3000/
```

(both lines existed in the file already, but commented out) followed by
`systemctl restart grafana-server`. New share links generated after the
fix resolve correctly from any device on the network.

## Dashboard auto-refresh

Both dashboards ship with auto-refresh enabled and saved as their default
(`1m` for the GLPI dashboard, `30s` for the WAN link dashboard) so they
stay current without anyone touching them — relevant for the wall-monitor
use case, and the reason the `session_token` refresh fix above mattered.

## Still open

- [ ] Mail gateway
- [ ] Telegram bot integration (GLPI ticket alerts + Zabbix network
      alerts, zero recurring cost)
- [ ] Additional GLPI ticket metrics (by technician/group, by category,
      average resolution time) — scoped but not built yet
- [ ] Paginate or date-filter the `/Ticket` query ahead of the 1000-ticket
      cap, if volume grows
