# 02 — Base OS, Access, and Network

## Base install

Debian 12 "Bookworm" was installed on the `SRV--GLPI` VM via the ESXi
console using the standard netinst flow. Only a `root` account was
created (no separate unprivileged user was needed for this scope).

## SSH access

Debian ships `sshd` with a secure-by-default setting:

```
PermitRootLogin prohibit-password
```

This **silently rejects password authentication for `root` over SSH**,
even with the correct password — it only accepts key-based auth. This
produced confusing `Permission denied (publickey,password)` errors
early in the build, before SSH keys were in place, and is worth knowing
if you hit the same error on a fresh Debian install.

The actual SSH access policy in place for this server is tracked in a
private checklist rather than published here — see the note in
`04-security-hardening.md`.

## Static network configuration

`/etc/network/interfaces` (classic `ifupdown`, Debian's default):

```
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

allow-hotplug ens192
iface ens192 inet static
    address 192.0.2.229/24
    gateway 192.0.2.1
    dns-nameservers 192.0.2.2 192.0.2.1
    dns-search example.com.br
```

> Addresses and the domain above are placeholders — IPs use the
> `192.0.2.0/24` documentation range (RFC 5737), and the domain appears as
> `example.com.br` — see the note in the root `README.md`.

Applied with:

```bash
systemctl restart networking
```

> Note: this drops any SSH session connected on the old (DHCP) address —
> reconnect using the new static IP.

## Long-term support

Debian 12's regular security support window ends ~July 2026; the system
then rolls automatically into community-maintained **Debian LTS**
(no `sources.list` changes needed, same `deb.debian.org` /
`security.debian.org` repositories), extending coverage to ~2028. Paid
**Freexian Extended LTS** exists as a separate option beyond that if
ever needed — not configured here.
