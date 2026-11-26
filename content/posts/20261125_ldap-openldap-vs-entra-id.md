---
title: "LDAP: Why It Still Matters, How to Run OpenLDAP, and How It Compares to Entra ID"
date: 2026-11-25
draft: true
description: "What LDAP is and why a 1990s protocol still sits under so much infrastructure, a hands-on OpenLDAP setup (DIT, users and groups, TLS, ACLs, indexes, replication, SSSD), and an honest comparison with Microsoft Entra ID, including Entra Domain Services and hybrid setups."
summary: "LDAP is the protocol your Linux servers, network gear and self-hosted apps still use to ask 'who is this user and what groups are they in?'. Here is how the protocol works, how to run OpenLDAP properly, and where Microsoft Entra ID, which doesn't speak LDAP at all, fits in."
tags: ["ldap", "openldap", "entra id", "active directory", "identity", "sssd", "linux", "security"]
---

Every organisation past a handful of people runs into the same problem: who are our users, what are their passwords, and which groups are they in? Without an answer in one place, you end up with a local account on every server, a separate user table in every app, and an offboarding process that consists of hoping someone remembers all of them.

The **Lightweight Directory Access Protocol** (LDAP) is the oldest widely used answer to that problem that is still in production everywhere. Your Linux servers can log users in against it, your Wi-Fi can authenticate against it, and nearly every self-hosted application (GitLab, Grafana, Jenkins, Nextcloud, Keycloak, Harbor, ...) has an "LDAP" tab in its settings. Active Directory, which runs most corporate networks, is at its core an LDAP directory with Kerberos bolted on.

At the same time, the identity world has moved to the cloud, and Microsoft's cloud directory, **Entra ID** (formerly Azure AD), does not speak LDAP at all. This post covers why LDAP exists, how the protocol and data model work, how to run **OpenLDAP** properly, and how all of that compares to Entra ID.

## What LDAP is

LDAP is a **protocol** for reading and writing a **directory**: a hierarchical database optimised for many reads and few writes. It was designed in the early 1990s as a "lightweight" way to access X.500 directories over TCP/IP, without the full OSI stack X.500 needed.[^1] Today's version, LDAPv3, is defined in a family of RFCs, with RFC 4511 describing the protocol itself.[^2]

Two things are worth separating early:

- **LDAP the protocol**: the wire format and operations (bind, search, modify, ...). It runs on TCP port 389, or 636 for LDAP over TLS (LDAPS).
- **The directory server**: the software that stores the data and answers LDAP requests. OpenLDAP's `slapd`, 389 Directory Server, Active Directory, and Samba AD are all directory servers that speak LDAP.

So "we use LDAP" usually means "we have a directory server, and our systems query it over LDAP".

### The data model

An LDAP directory is a tree, called the **Directory Information Tree** (DIT). Every node in it is an **entry**, and every entry has a globally unique name, its **Distinguished Name** (DN), which spells out the path from the entry up to the root:

```text
                    dc=example,dc=org
                   /                  \
          ou=people                    ou=groups
          /       \                    /        \
     uid=ada    uid=alan     cn=developers    cn=grafana-admins

DN of Ada:  uid=ada,ou=people,dc=example,dc=org
```

The leftmost component (`uid=ada`) is the entry's **Relative Distinguished Name** (RDN), the part that is unique among its siblings. The abbreviations come from X.500: `dc` is domain component, `ou` organisational unit, `cn` common name, `uid` user ID.

An entry is a bag of **attributes**, each with one or more values. Which attributes an entry may or must have is decided by its **object classes**, and the object classes and attribute types together form the **schema**.[^3] Here is what Ada looks like, in the text format LDAP tools use for import and export, **LDIF**:

```ldif
dn: uid=ada,ou=people,dc=example,dc=org
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: ada
cn: Ada Lovelace
givenName: Ada
sn: Lovelace
mail: ada@example.org
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/ada
loginShell: /bin/bash
```

`inetOrgPerson` gives her a name and email address, `posixAccount` (from the RFC 2307 "NIS" schema[^4]) gives her the UID, GID, home directory and shell a Linux machine needs to log her in. Mixing object classes like this is normal: an entry is whatever combination of classes you need.

### The operations

The protocol is small. The operations you will use are:

| Operation | What it does |
| --- | --- |
| **Bind** | Authenticate the connection, either *simple* (DN + password) or via SASL (Kerberos, client certificates, ...). An unauthenticated connection is an *anonymous* bind. |
| **Search** | Find entries under a **base DN**, within a **scope** (`base`, `one` level, or the whole `sub`tree), matching a **filter**, returning selected attributes. |
| **Compare** | Ask "does this entry have attribute X with value Y?" without reading it. |
| **Add / Delete / Modify** | Create, remove or change an entry. |
| **ModifyDN** | Rename or move an entry within the tree. |
| **Extended** | Everything else, e.g. **StartTLS** (upgrade a port 389 connection to TLS), Password Modify, WhoAmI. |

Search filters use a prefix notation that looks odd at first but is easy to read once you have seen a few:

```text
(uid=ada)                                   exact match
(mail=*@example.org)                        wildcard
(&(objectClass=posixAccount)(uidNumber>=10000))   AND
(|(cn=Ada*)(cn=Alan*))                      OR
(!(loginShell=/usr/sbin/nologin))           NOT
(memberOf=cn=grafana-admins,ou=groups,dc=example,dc=org)
```

## Why LDAP, and why still?

LDAP's job is to be the **single source of truth for identities** that many systems can query with one standard protocol. Concretely, that buys you:

- **One account per person.** Create a user once, and they can log into servers, apps and VPN. Disable them once, and they are locked out everywhere.
- **Group-based authorisation.** Apps don't keep their own permission lists; they ask the directory "is this user in `grafana-admins`?".
- **Linux and Unix logins.** Through SSSD or `nslcd`, a Linux host resolves users, groups and passwords from LDAP as if they were in `/etc/passwd`.
- **A universal integration point.** Network devices, printers, mail servers, RADIUS servers and decades' worth of enterprise software speak LDAP. Very few of them speak OIDC or the Microsoft Graph API.
- **Read performance.** Directories are built for the access pattern identity has: millions of lookups, a few changes a day.

The reason it hasn't been replaced is mostly that last-but-one point. Modern web apps have moved to **OIDC and SAML**, where the app never sees a password and the user gets single sign-on and MFA. But "the thing that answers `getent passwd ada` on a server" and "the thing an appliance from 2012 can authenticate against" are still LDAP jobs. In practice, LDAP is increasingly the *backend* directory, and something like Keycloak or Entra ID sits in front of it for the web.

## How: running OpenLDAP

**OpenLDAP** is the reference open source LDAP implementation, maintained since 1998. The pieces are:

- **`slapd`**, the server daemon.
- **Backends**, which store the data. Use `mdb`, built on the LMDB memory-mapped database; it is the primary backend and the only one you should pick for new deployments.[^5]
- **Overlays**, plug-ins that sit between the frontend and the backend and add behaviour: replication (`syncprov`), password policies (`ppolicy`), referential integrity (`refint`), audit logs (`accesslog`), dynamic groups and `memberOf` (`dynlist`), and more.[^6]
- **`cn=config`**, the configuration, which is itself stored as an LDAP tree and modified with LDAP operations at runtime. The old flat `slapd.conf` file still works but is deprecated.
- **Client tools**: `ldapsearch`, `ldapadd`, `ldapmodify`, `ldapdelete`, `ldappasswd`, plus offline tools like `slapcat` and `slapadd`.

The examples below use Debian/Ubuntu and OpenLDAP 2.5/2.6, with a base DN of `dc=example,dc=org`.

### Install

```sh
sudo apt install slapd ldap-utils
sudo dpkg-reconfigure slapd   # domain: example.org, org name, admin password
```

This creates the database `olcDatabase={1}mdb,cn=config` with the suffix `dc=example,dc=org` and an admin entry `cn=admin,dc=example,dc=org`. It also gives the local root user full access to `cn=config` over the Unix socket via SASL EXTERNAL, which is how you change configuration:

```sh
sudo ldapsearch -Q -Y EXTERNAL -H ldapi:/// -b cn=config -LLL dn
```

There are now two different administrators, which trips people up: **root over `ldapi:///`** manages the server configuration, and **`cn=admin,dc=example,dc=org`** manages the data.

### Build the tree

Create the organisational units, a user, and two kinds of group in one LDIF file:

```ldif
# base.ldif
dn: ou=people,dc=example,dc=org
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=example,dc=org
objectClass: organizationalUnit
ou: groups

dn: ou=services,dc=example,dc=org
objectClass: organizationalUnit
ou: services

dn: uid=ada,ou=people,dc=example,dc=org
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: ada
cn: Ada Lovelace
givenName: Ada
sn: Lovelace
mail: ada@example.org
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/ada
loginShell: /bin/bash

# A Unix group, for Linux hosts (members listed by uid)
dn: cn=developers,ou=groups,dc=example,dc=org
objectClass: posixGroup
cn: developers
gidNumber: 20001
memberUid: ada

# An application group (members listed by DN)
dn: cn=grafana-admins,ou=groups,dc=example,dc=org
objectClass: groupOfNames
cn: grafana-admins
member: uid=ada,ou=people,dc=example,dc=org
```

```sh
ldapadd -x -H ldap://localhost -D cn=admin,dc=example,dc=org -W -f base.ldif
```

Why two kinds of group? The classic NIS schema OpenLDAP ships makes `posixGroup` a *structural* class, so an entry can't be both a `posixGroup` and a `groupOfNames`. Linux tooling wants the former, most applications want the latter. The alternative `rfc2307bis` schema fixes this by making `posixGroup` auxiliary, but it isn't shipped by default and switching later is painful, so decide this before you have data.

Set Ada's password with the Password Modify extended operation, so the server does the hashing:

```sh
ldappasswd -x -H ldap://localhost -D cn=admin,dc=example,dc=org -W \
  -S uid=ada,ou=people,dc=example,dc=org
```

By default `slapd` stores passwords as salted SHA-1 (`{SSHA}`). That is a fast hash, which is the wrong property for passwords. OpenLDAP 2.5+ ships an Argon2 module; load it and make it the default:[^7]

```ldif
# argon2.ldif  (apply with: sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f argon2.ldif)
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: argon2

dn: olcDatabase={-1}frontend,cn=config
changetype: modify
replace: olcPasswordHash
olcPasswordHash: {ARGON2}
```

### Search it

```sh
# Who is Ada? (simple bind as herself)
ldapsearch -x -H ldap://localhost -D uid=ada,ou=people,dc=example,dc=org -W \
  -b dc=example,dc=org -LLL "(uid=ada)" cn mail uidNumber

# Which groups is she in?
ldapsearch -x -H ldap://localhost -D cn=admin,dc=example,dc=org -W \
  -b ou=groups,dc=example,dc=org -LLL \
  "(|(memberUid=ada)(member=uid=ada,ou=people,dc=example,dc=org))" cn
```

### memberOf

Many applications want to ask the reverse question: given a user, list their groups, via a `memberOf` attribute on the user entry. OpenLDAP used to provide this with the `memberof` overlay, which is deprecated since 2.5. The replacement is the `dynlist` overlay, which computes `memberOf` at search time:[^8]

```ldif
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: dynlist

dn: olcOverlay=dynlist,olcDatabase={1}mdb,cn=config
objectClass: olcOverlayConfig
objectClass: olcDynListConfig
olcOverlay: dynlist
olcDynListAttrSet: groupOfURLs memberURL member+memberOf@groupOfNames
```

After that, `ldapsearch ... "(uid=ada)" memberOf` returns `cn=grafana-admins,ou=groups,dc=example,dc=org`, and filters like `(memberOf=cn=grafana-admins,...)` work.

### Turn on TLS

A simple bind sends the password in clear text, so an LDAP server without TLS is a password broadcast service. Point `slapd` at a certificate and key (on Debian, `slapd` runs as the `openldap` user and is built against GnuTLS, so the key must be readable by that user):

```ldif
dn: cn=config
changetype: modify
replace: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/ldap/tls/ca.crt
-
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/ldap/tls/ldap1.example.org.crt
-
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/ldap/tls/ldap1.example.org.key
-
replace: olcSecurity
olcSecurity: tls=128
```

`olcSecurity: tls=128` makes the server refuse any operation on a connection that isn't encrypted. Clients then use either StartTLS on port 389 (`ldapsearch -ZZ ...`) or LDAPS on 636 (add `ldaps:///` to `SLAPD_SERVICES` in `/etc/default/slapd`). Both are fine; pick one and enforce it.

### Access control

OpenLDAP's ACLs are an ordered list of `to <what> by <who> <access>` rules, evaluated top to bottom, where the first matching `to` clause wins and its `by` clauses are checked in order.[^9] A sensible starting point:

```ldif
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to attrs=userPassword
  by self write
  by anonymous auth
  by * none
olcAccess: {1}to attrs=shadowLastChange
  by self write
  by * read
olcAccess: {2}to dn.subtree="dc=example,dc=org"
  by dn.exact="cn=svc-sssd,ou=services,dc=example,dc=org" read
  by users read
  by * none
```

Rule 0 says nobody can read password hashes, users may change their own password, and anonymous connections may use the attribute only to *authenticate* (a bind). Rule 2 says only authenticated connections can read the directory at all, which closes the anonymous enumeration that many default setups leave open. The admin DN bypasses ACLs entirely, which is another reason not to hand it to applications: create a read-only service account per application instead.

### Indexes

Without an index, a search filter on an attribute means a scan of every entry. Index the attributes your clients filter on:

```ldif
dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcDbIndex
olcDbIndex: uid,mail eq,sub
olcDbIndex: cn eq,sub
olcDbIndex: uidNumber,gidNumber,memberUid,member eq
olcDbIndex: entryCSN,entryUUID eq
```

The last line is needed for replication. If `slapd` logs `<= mdb_equality_candidates: (foo) not indexed`, that's a client asking for an index you haven't built.

### Replication

A single directory server is a single point of failure for every login in the company. OpenLDAP replicates with **syncrepl**, a pull-based protocol defined in RFC 4533:[^10] a consumer connects to a provider, gets the current state, and then keeps the connection open to receive changes. The provider side is the `syncprov` overlay.

The common high availability setup is two **multi-provider** servers (OpenLDAP 2.4 called this "mirror mode"), each replicating from the other, behind a load balancer or listed as two URIs in every client. On *both* servers, apply:

```ldif
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: syncprov

dn: cn=config
changetype: modify
replace: olcServerID
olcServerID: 1 ldap://ldap1.example.org
olcServerID: 2 ldap://ldap2.example.org

dn: olcOverlay=syncprov,olcDatabase={1}mdb,cn=config
objectClass: olcOverlayConfig
objectClass: olcSyncProvConfig
olcOverlay: syncprov
olcSpCheckpoint: 100 10

dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcSyncRepl
olcSyncRepl: rid=001 provider=ldap://ldap1.example.org
  bindmethod=simple binddn="cn=replicator,ou=services,dc=example,dc=org"
  credentials=CHANGE-ME searchbase="dc=example,dc=org"
  type=refreshAndPersist retry="5 5 300 +" timeout=1
  starttls=critical tls_reqcert=demand
olcSyncRepl: rid=002 provider=ldap://ldap2.example.org
  bindmethod=simple binddn="cn=replicator,ou=services,dc=example,dc=org"
  credentials=CHANGE-ME searchbase="dc=example,dc=org"
  type=refreshAndPersist retry="5 5 300 +" timeout=1
  starttls=critical tls_reqcert=demand
-
add: olcMultiProvider
olcMultiProvider: TRUE
```

The `cn=replicator` account needs read access to everything, including `userPassword`, so add a `by dn.exact="cn=replicator,..." read` line to the ACLs above. For scaling reads, add any number of read-only consumers that only have the `olcSyncRepl` part and point at the providers.

### Log Linux hosts in with SSSD

On each client, SSSD resolves users and groups from the directory, checks passwords with an LDAP bind, and caches credentials so logins keep working when the directory is briefly unreachable:

```ini
# /etc/sssd/sssd.conf  (chmod 600)
[sssd]
services = nss, pam
domains = example

[domain/example]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldaps://ldap1.example.org, ldaps://ldap2.example.org
ldap_search_base = dc=example,dc=org
ldap_schema = rfc2307
ldap_default_bind_dn = cn=svc-sssd,ou=services,dc=example,dc=org
ldap_default_authtok = CHANGE-ME
ldap_tls_reqcert = demand
ldap_tls_cacert = /etc/ssl/certs/example-ca.pem
cache_credentials = true

access_provider = simple
simple_allow_groups = developers
```

```sh
sudo pam-auth-update --enable mkhomedir
sudo systemctl restart sssd
getent passwd ada      # ada:*:10001:10001:Ada Lovelace:/home/ada:/bin/bash
id ada                 # ... groups=10001,20001(developers)
```

`simple_allow_groups` restricts logins on this host to members of `developers`. If you want public SSH keys stored in the directory too, add the `openssh-lpk` schema, `ldap_user_ssh_public_key = sshPublicKey`, and point `AuthorizedKeysCommand` in `sshd_config` at `/usr/bin/sss_ssh_authorizedkeys`.

### Connect applications

Almost every application's LDAP integration follows the same **search-then-bind** pattern:

1. Bind as the application's service account (e.g. `cn=svc-grafana,ou=services,dc=example,dc=org`).
2. Search under `ou=people` with a filter like `(uid=%s)` to turn the username typed into the login form into a DN.
3. Bind again as that DN with the password the user typed. Success means the password is correct.
4. Read the user's `memberOf` (or search `ou=groups` for `(member=<DN>)`) to map groups to roles.

So the settings you will be asked for are always the same: server URI, bind DN and password, user search base and filter, group search base and filter, and which attribute holds the username, display name and email.

### Operations

- **Backups**: `slapcat -n 0` dumps the configuration, `slapcat -n 1` the data, both as LDIF, and both can run while `slapd` is up. Restore with `slapadd` while it is stopped. Test the restore.
- **LMDB map size**: `olcDbMaxSize` caps the database size. When you hit it, writes fail. Set it generously; it is a reservation of address space, not disk.
- **Schema**: adding schema is easy (`ldapadd` an LDIF into `cn=schema,cn=config`); removing or changing schema that entries use is not. Think before loading.
- **Logging**: `olcLogLevel: stats` logs every connection, bind and search, which is what you want when debugging an application's filter.

That is the whole story: OpenLDAP is small, fast and very stable, but everything beyond the protocol (a web UI, self-service password reset, MFA, single sign-on) is something you add yourself.

## Microsoft Entra ID

**Entra ID** is Microsoft's cloud identity service, renamed from Azure Active Directory in 2023.[^11] It is the directory behind every Microsoft 365 tenant, and despite the old name, it is **not** Active Directory in the cloud. There is no LDAP endpoint, no Kerberos, no domain controllers, no OUs and no Group Policy.

Instead, Entra ID is built for the web:

- **Protocols**: OpenID Connect, OAuth 2.0, SAML 2.0 and WS-Federation for sign-in, and SCIM for provisioning users into SaaS apps.
- **API**: the directory is read and written through **Microsoft Graph**, a REST API, using OAuth access tokens.[^12]
- **Data model**: a flat list of objects (users, groups, devices, applications, service principals) identified by GUIDs, with no tree. Administrative units can scope admin permissions, but they are not containers.
- **Security features**: MFA, passwordless sign-in, Conditional Access (allow or block a sign-in based on user, device compliance, location and risk), Identity Protection, Privileged Identity Management.
- **Hosting**: fully managed by Microsoft, highly available across regions, priced per user per month for the premium features.

The same question as earlier, "who is Ada and which groups is she in?", looks like this against Graph:

```sh
TOKEN=$(az account get-access-token --resource-type ms-graph --query accessToken -o tsv)

curl -s -H "Authorization: Bearer $TOKEN" \
  "https://graph.microsoft.com/v1.0/users?\$filter=startswith(displayName,'Ada')&\$select=id,displayName,userPrincipalName,mail"

curl -s -H "Authorization: Bearer $TOKEN" \
  "https://graph.microsoft.com/v1.0/users/ada@example.org/memberOf?\$select=displayName"
```

Same idea, different universe: JSON over HTTPS with OData filters instead of BER-encoded LDAP with prefix filters, and an OAuth token instead of a bind.

### Getting LDAP out of the Microsoft world

Because Entra ID doesn't speak LDAP, organisations that need it have three options:

1. **Keep on-premises Active Directory** as the LDAP and Kerberos directory, and synchronise it to Entra ID with **Entra Connect** or **Entra Cloud Sync**.[^13] This is the standard hybrid setup: AD remains the source of truth for users, servers join the AD domain (see my post on [joining GCP VMs to an on-premises AD domain]({{< relref "20261122_ad-join-gcp-vms-to-on-prem-domain-controllers.md" >}})), and Entra ID handles cloud sign-in and MFA.
2. **Entra Domain Services**: a managed, Microsoft-run pair of domain controllers in an Azure virtual network that synchronises one way *from* Entra ID.[^14] It offers LDAP, LDAPS, Kerberos, NTLM and Group Policy, so legacy apps and VMs in Azure can bind to it. The catches: it is read-only for synchronised users (you can't change a user over LDAP), you don't get Domain Admin rights, schema extensions are limited, and cloud-only users must change their password once after it is enabled so that the NTLM and Kerberos hashes it needs get generated. Secure LDAP access from outside the VNet has to be explicitly enabled with your own certificate.[^15]
3. **Provision from Entra ID into your own LDAP directory**: Entra ID's provisioning service can push users into an on-premises LDAP server such as OpenLDAP through the provisioning agent's ECMA connector.[^16] Entra ID stays the source of truth, and OpenLDAP becomes a downstream copy for Linux hosts and legacy apps. Passwords are the weak point here: Entra ID doesn't hand out password hashes, so users either get a separately managed LDAP password, or the apps move to SSO.

For Linux specifically, Entra ID supports SSH login to **Azure VMs and Azure Arc-enabled servers** with Entra accounts, via a VM extension and short-lived OpenSSH certificates (`az ssh vm`).[^17] That covers Azure-hosted fleets well; it is not a general-purpose replacement for `id_provider = ldap` on arbitrary hosts.

## OpenLDAP vs Entra ID

| | OpenLDAP | Entra ID |
| --- | --- | --- |
| **What it is** | A directory server you run | A cloud identity provider Microsoft runs |
| **Protocol** | LDAPv3 (+ SASL, StartTLS/LDAPS) | OIDC, OAuth 2.0, SAML, WS-Fed, SCIM; Microsoft Graph REST API |
| **Data model** | Hierarchical tree of entries, DNs | Flat objects with GUIDs, no containers |
| **Schema** | Fully extensible, any standard or custom schema | Fixed core schema, plus directory/open extensions and custom security attributes |
| **Authentication** | Password bind; MFA only via add-ons | Passwords, MFA, passkeys, Windows Hello, Conditional Access built in |
| **Single sign-on for web apps** | None by itself; put Keycloak or similar in front | Native, with a large app gallery |
| **Linux host login** | Native, via SSSD/nslcd on any host | Azure VMs and Arc servers only; otherwise via Entra Domain Services or AD |
| **Legacy apps and appliances** | Excellent, it is what they expect | Needs Entra Domain Services or a synced LDAP directory |
| **Groups** | Static groups, dynamic groups via `dynlist` | Security and M365 groups, dynamic membership rules |
| **High availability** | You build it (multi-provider syncrepl) | Included |
| **Operations** | Patching, backups, TLS, monitoring are yours | None for the directory itself |
| **Cost** | Free software, your infrastructure and time | Free tier; per-user licences (P1/P2) for Conditional Access and more |
| **Data location and control** | Entirely yours, works offline and air-gapped | Microsoft's cloud, requires internet connectivity |

The short version: **OpenLDAP is a component, Entra ID is a product**. OpenLDAP gives you a fast, standards-based, fully controllable directory and nothing else. Entra ID gives you a complete identity platform with MFA, SSO, risk detection and lifecycle management, but only for things that speak modern web protocols.

## Which one should you use?

- **You run Microsoft 365**: you already have Entra ID. Use it for every web app that supports OIDC or SAML, and don't build a parallel identity system for those.
- **You have Linux servers, network gear or legacy apps that need LDAP**: you need an LDAP directory somewhere. If you already have on-prem AD, that is it. If your workloads live in Azure, Entra Domain Services is the low-maintenance option. Otherwise, OpenLDAP fed by Entra ID provisioning is a reasonable pattern.
- **You have no Microsoft footprint** (a homelab, a startup on Google Workspace, an air-gapped environment, a university cluster): OpenLDAP, ideally with Keycloak in front of it for OIDC, SAML and MFA, gives you most of what Entra ID offers for the cost of running two services.
- **You are starting from scratch in 2026**: design for OIDC first, and treat LDAP as a compatibility layer for the systems that can't do anything else, not as the place your web apps authenticate.

LDAP is old, and its filter syntax will never win a beauty contest, but it is also one of the most successful protocols in infrastructure. Even in an Entra-first company, there is a good chance that somewhere a server is still asking a directory over port 636 whether Ada is allowed to log in.

[^1]: [X.500 Lightweight Directory Access Protocol](https://www.rfc-editor.org/rfc/rfc1487), RFC 1487, IETF, 1993.
[^2]: [Lightweight Directory Access Protocol (LDAP): The Protocol](https://www.rfc-editor.org/rfc/rfc4511), RFC 4511, IETF, 2006.
[^3]: [LDAP: Directory Information Models](https://www.rfc-editor.org/rfc/rfc4512), RFC 4512, IETF, 2006.
[^4]: [An Approach for Using LDAP as a Network Information Service](https://www.rfc-editor.org/rfc/rfc2307), RFC 2307, IETF, 1998.
[^5]: [Backends](https://www.openldap.org/doc/admin26/backends.html), OpenLDAP Software 2.6 Administrator's Guide.
[^6]: [Overlays](https://www.openldap.org/doc/admin26/overlays.html), OpenLDAP Software 2.6 Administrator's Guide.
[^7]: [Security Considerations: Password Storage](https://www.openldap.org/doc/admin26/security.html), OpenLDAP Software 2.6 Administrator's Guide.
[^8]: [slapo-dynlist(5)](https://www.openldap.org/software/man.cgi?query=slapo-dynlist), OpenLDAP manual pages.
[^9]: [Access Control](https://www.openldap.org/doc/admin26/access-control.html), OpenLDAP Software 2.6 Administrator's Guide.
[^10]: [The LDAP Content Synchronization Operation](https://www.rfc-editor.org/rfc/rfc4533), RFC 4533, IETF, 2006. See also [Replication](https://www.openldap.org/doc/admin26/replication.html), OpenLDAP Software 2.6 Administrator's Guide.
[^11]: [What is Microsoft Entra ID?](https://learn.microsoft.com/en-us/entra/fundamentals/whatis), Microsoft Learn.
[^12]: [Overview of Microsoft Graph](https://learn.microsoft.com/en-us/graph/overview), Microsoft Learn.
[^13]: [What is Microsoft Entra Connect?](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/whatis-azure-ad-connect), Microsoft Learn; [What is Microsoft Entra Cloud Sync?](https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/what-is-cloud-sync), Microsoft Learn.
[^14]: [What is Microsoft Entra Domain Services?](https://learn.microsoft.com/en-us/entra/identity/domain-services/overview), Microsoft Learn.
[^15]: [Configure secure LDAP for a Microsoft Entra Domain Services managed domain](https://learn.microsoft.com/en-us/entra/identity/domain-services/tutorial-configure-ldaps), Microsoft Learn.
[^16]: [Configure Microsoft Entra ID to provision users into an LDAP directory](https://learn.microsoft.com/en-us/entra/identity/app-provisioning/on-premises-ldap-connector-configure), Microsoft Learn.
[^17]: [Sign in to a Linux virtual machine in Azure by using Microsoft Entra ID and OpenSSH](https://learn.microsoft.com/en-us/entra/identity/devices/howto-vm-sign-in-azure-ad-linux), Microsoft Learn.
