# 05 — Active Directory / LDAP Integration

> Addresses and the domain in this page are placeholders
> (`192.0.2.0/24` per RFC 5737, domain as `example.com`) — see the
> note in the root `README.md`. Login names of real, active directory
> objects are described rather than quoted verbatim, since this is a
> live production directory, and the entry counts below are illustrative
> approximations rather than the exact production numbers.

## Why integrate with Active Directory

Up to this point, GLPI relied only on its four local accounts. That does
not scale: every employee would need a separate GLPI-only password to
remember, and offboarding would require remembering to disable two
accounts instead of one. The company already runs a corporate
Active Directory (domain `example.com`) for Windows login and
Microsoft 365 — pointing GLPI at the same directory means one identity,
one password, and one place to disable a departing employee's access.

## A dedicated, least-privilege service account

GLPI needs a "bind" account to query the directory — enough privilege to
search and read user attributes, nothing more. Two accounts already
existed that could technically do this: the old GLPI server's service
account, and the account used to sync the domain with Microsoft 365/Entra
ID. Both were rejected on purpose:

- Reusing the **old GLPI's account** would carry over unknown history and
  permissions from a system being retired.
- Reusing the **Microsoft 365 sync account** would mean a failure or
  password rotation on GLPI's side could risk the cloud identity sync —
  two unrelated systems should not share credentials.

Instead, a new AD user, `svc-helpdesk`, was created specifically for this
purpose: a non-expiring, non-interactive-login-friendly password, and no
elevated group membership. If it's ever compromised, the blast radius is
"can read directory attributes" — not "can log into a workstation" or
"can break Microsoft 365 sync."

## LDAP directory configuration

Configured under **Configuração > Autenticação > Diretórios LDAP**:

| Field | Value | Why |
|---|---|---|
| Servidor | `192.0.2.2` | The domain controller, `SRV-AD` |
| Porta LDAP | `389` | Standard unencrypted LDAP port (internal network only) |
| BaseDN | `DC=example,DC=com` | Root of the domain's directory tree |
| Usar vinculação (bind) | Sim | Anonymous bind is disabled on this AD, as it should be |
| RootDN | `svc-helpdesk@example.com` | **UPN format**, not a full `CN=...` distinguished name — both work against AD, UPN is shorter and easier to get right |
| Campo de Login | `samaccountname` | GLPI's default here is `uid`, which is an OpenLDAP convention — Active Directory doesn't populate `uid` by default, so authentication silently fails until this is changed |
| Campo de sincronização | `objectguid` | A stable, AD-generated identifier that survives a username change — safer than tracking users by their login name |
| Servidor padrão | Sim | Only directory configured; must be the default for login to use it |
| Ativo | Sim | An inactive directory is invisible to both login and import |

GLPI ships a built-in **"Testar"** button on this form that validates, in
order: TCP connectivity, the Base DN, the LDAP URI, the bind
authentication, and a sample search. All five passed before moving on to
the actual import — worth doing before troubleshooting anything else,
since it isolates *which* layer is broken (network vs. credentials vs.
search scope).

## The user import problem: it's not just people

GLPI's bulk import tool (**Administração > Usuários > Importação em massa
de usuários de um diretório LDAP > Importar novos usuários**) runs a
search against the directory and lists every match with a checkbox, ready
to select and import.

Running it with no filter returned **roughly 600 entries** — but a scroll
through the list showed it wasn't 600 employees. Active Directory stores several
kinds of objects under the same tree GLPI was searching:

- **Security groups** — the built-in "Administrators", "Enterprise Key
  Admins", "Schema Admins" groups, and dozens of other built-in AD
  groups, all with the same "person-shaped" listing as a real user.
- **Azure AD Connect service accounts** — the automatic sync accounts
  created by the Microsoft 365 directory sync tool (named along the
  lines of `ADSyncAdmins`, `ADSyncBrowse`, `ADSyncOperators`).

Importing all of them as-is would have filled GLPI's user list with security
groups and sync robots labeled as "users" — confusing at best, and
actively wrong for anything that reports on headcount or assigns
tickets by user.

## The fix: an LDAP search filter

The directory's **"Filtro da conexão"** field accepts a standard LDAP
filter expression applied to every search GLPI runs against it —
including the import search. Setting it to:

```
(&(objectClass=user)(objectCategory=person))
```

restricts every search to objects that are both class `user` and
category `person` — which excludes `group` objects and Azure AD
Connect's managed service accounts (a different, non-`person` object
category) in one step. Re-running the import search afterward cut the
result down to **roughly 400** — confirmation the filter was doing its
job.

## Remaining manual cleanup

A `person`-category filter still doesn't guarantee *human employee*.
Reviewing the remaining entries turned up two more categories worth
excluding by hand before importing:

- **Built-in AD system accounts**: the domain Administrator account, the
  `krbtgt` account (an internal Kerberos account present on every AD
  domain by design — never a real login), the built-in Guest account, the
  default local admin account, and a Microsoft 365 directory-sync account
  (Microsoft always names these with an `MSOL_` prefix).
- **Functional / shared accounts**: a handful of AD user objects that
  turned out to represent a printer, a meeting room, a department, and a
  shared mailbox — not individual employees. Exact login names are
  omitted here since these are live accounts on a real production
  directory.

A handful were unchecked manually before running the import, leaving
**roughly 390 real user accounts** imported successfully.

> Some department/process-style logins (e.g. names tied to internal
> workflows) likely still slipped through this pass, since they follow
> no consistent naming pattern that a filter alone can catch. These are
> tracked for a manual review pass inside GLPI's user list — cheap to
> fix later (deactivate, don't delete) versus expensive to catch
> perfectly up front.

## Default profile and verification

Imported users are automatically assigned the **Self-Service** profile
via GLPI's dynamic LDAP sync (marked `(D)` in their authorizations) —
appropriate for the majority of employees, who only need to open and
track their own tickets through the self-service portal, not the full
technician interface.

Finally, a real employee logged in using their **Active Directory
credentials** directly against the GLPI login form — confirming
authentication itself works end-to-end, not just that the directory
search finds the right entries.
