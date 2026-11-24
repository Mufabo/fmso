---
title: "Joining GCP VMs to an On-Premises Active Directory Domain"
date: 2026-11-22
draft: true
description: "How to domain-join Compute Engine VMs (Windows and Linux) to domain controllers that live in your own data center: hybrid connectivity, Cloud DNS forwarding and the 35.199.192.0/19 range, firewall ports, AD sites, automating the join at boot, and the problems you will hit along the way."
summary: "Your VMs run in Google Cloud, your domain controllers are on-prem. Joining one to the other needs three things to work: a network path, DNS that can find the DCs, and a way to hand out join credentials safely. Here is each piece, with gcloud, Terraform and scripts."
tags: ["gcp", "active directory", "windows", "linux", "sssd", "dns", "hybrid cloud", "networking"]
categories: ["Cloud"]
---

A common first step when moving Windows workloads to Google Cloud: the VMs move, the domain controllers stay home. You want a fresh Compute Engine VM to come up, join `corp.example.com` like any server in the data center would, and accept logons from the same users and groups.

Domain join isn't magic. Under the hood it's a sequence of DNS lookups, Kerberos and LDAP calls and an RPC or two. If each of those can get from the VM to a DC and back, the join works. If any one is blocked, you get a vague "The specified domain either does not exist or could not be contacted".

This post covers:

1. **Architecture choices**: do you really want to rely on DCs on the other end of a VPN?
2. **Connectivity**: Cloud VPN / Interconnect and routing
3. **DNS**: Cloud DNS forwarding zones and the `35.199.192.0/19` gotcha
4. **Firewall ports**
5. **AD sites and subnets**
6. **Joining Windows VMs**, manually, automatically at first boot, and through a domain-join broker
7. **Joining Linux VMs** with realmd and SSSD
8. **Troubleshooting**

## The big picture

```text
        Google Cloud (VPC: prod-vpc)                         On-premises
 ┌─────────────────────────────────────────┐       ┌───────────────────────────────┐
 │                                         │       │                               │
 │  ┌───────────┐   DNS query for          │       │   ┌──────────┐  ┌──────────┐  │
 │  │ VM        │   corp.example.com       │       │   │ DC01     │  │ DC02     │  │
 │  │ 10.20.1.5 │──► 169.254.169.254       │       │   │ 10.0.0.10│  │ 10.0.0.11│  │
 │  └─────┬─────┘   (metadata server)      │       │   │ DNS + AD │  │ DNS + AD │  │
 │        │              │                 │       │   └────▲─────┘  └────▲─────┘  │
 │        │      Cloud DNS forwarding zone │       │        │             │        │
 │        │      corp.example.com → DC IPs │       │        │             │        │
 │        │              │ src 35.199.192.0/19      │        │             │        │
 │        │              ▼                 │       │        │             │        │
 │        │      ┌───────────────┐  HA VPN / Interconnect   │             │        │
 │        └─────►│ Cloud Router  │══════════════════════════╪═════════════┘        │
 │  Kerberos,    │ BGP: advertise│       │       │   on-prem router / firewall     │
 │  LDAP, SMB,   │ subnets +     │       │       │   (must allow + route back      │
 │  RPC          │ 35.199.192/19 │       │       │    to 35.199.192.0/19)          │
 │               └───────────────┘       │       │                               │
 └─────────────────────────────────────────┘       └───────────────────────────────┘
```

Two separate traffic flows matter:

- **DNS** from the VM goes to the metadata server, which hands it to Cloud DNS, which forwards to your DCs from a Google-owned source range.
- **Everything else** (Kerberos, LDAP, SMB, RPC) goes straight from the VM's own IP to the DC over the hybrid link.

Most failed setups get one of these right and not the other.

## Step 0: decide whether this is the right architecture

Joining cloud VMs to DCs that exist only on-prem is the simplest setup, but it makes every logon in the cloud depend on the VPN or Interconnect being up. Google's architecture guide describes the options:[^1]

| Pattern | What runs in GCP | Depends on on-prem being reachable? |
|---|---|---|
| **Join to on-prem DCs only** (this post) | Nothing AD-related, only member servers | Yes, for every logon, GPO refresh and ticket |
| **Extended domain** | Extra DCs of your existing domain, in their own AD site | No, local DCs keep serving |
| **Resource forest** | A separate forest (self-managed or Managed Microsoft AD) with a one-way trust to on-prem | Yes, for authenticating on-prem users |
| **Resource forest + extended domain** | Both of the above | No |

Google points out that the extended-domain pattern keeps cloud resources working when the on-prem DCs are unreachable, because the directory is replicated locally. The resource-forest pattern keeps a trust boundary between environments, but its availability depends on the on-prem DCs and the link.[^1]

A pragmatic path many teams take: start with "join to on-prem DCs" for a pilot, and add two DCs in GCP (in different zones) once production workloads arrive. Everything in this post still applies then; you just add DC IPs in GCP to the DNS forwarding targets and an extra AD site.

Whatever you choose, **redundant links** are a must. Google recommends redundant Dedicated Interconnect, Partner Interconnect or Cloud VPN connections for hybrid AD.[^1]

## Step 1: connectivity

You need private IP reachability in both directions between the VPC subnets and the DC subnet. That means HA VPN or Interconnect, with a Cloud Router running BGP.

Things to get right:

- **No overlapping ranges.** If your VPC subnet is `10.0.0.0/24` and so is the server VLAN in the data center, nothing else in this post will save you.
- **Advertise the VPC subnets** to on-prem (the Cloud Router default does this).
- **Advertise `35.199.192.0/19` as well.** This is the source range for Cloud DNS forwarding. Your on-prem network must have a route for it pointing back into the VPC, or DNS replies vanish.[^2] By default the Cloud Router only advertises subnet routes, so add it as a custom advertisement:

```bash
gcloud compute routers update onprem-router \
  --region=europe-west3 \
  --advertisement-mode=custom \
  --set-advertisement-groups=all_subnets \
  --set-advertisement-ranges=35.199.192.0/19
```

If you're using classic policy-based VPN tunnels instead of BGP, the range must be in the traffic selectors on both ends.[^3]

**Shared VPC:** if you have several VPCs that each forward DNS to on-prem, they'd all advertise `35.199.192.0/19`, and on-prem can only route replies to one of them. Google's networking blog walks through this problem; the usual fix is to forward from one hub VPC and use DNS peering from the others.[^4]

## Step 2: DNS

Domain join starts with DNS. The VM looks up SRV records like `_ldap._tcp.dc._msdcs.corp.example.com` to find a domain controller. Compute Engine VMs use the metadata server (`169.254.169.254`) as their resolver, which only knows about Google Cloud names and the public internet. So you need to teach Cloud DNS that `corp.example.com` lives on-prem.

### Option A: a Cloud DNS forwarding zone (recommended)

A forwarding zone is a private zone with no records, only a list of target name servers.[^5] Every VM in the authorized VPCs keeps using the metadata server, and queries for the AD domain get forwarded to your DCs:

```bash
gcloud dns managed-zones create corp-example-com \
  --description="Forward AD domain to on-prem DCs" \
  --dns-name="corp.example.com." \
  --networks=prod-vpc \
  --visibility=private \
  --private-forwarding-targets=10.0.0.10,10.0.0.11
```

Or in Terraform:

```hcl
resource "google_dns_managed_zone" "corp" {
  name        = "corp-example-com"
  dns_name    = "corp.example.com."
  description = "Forward AD domain to on-prem DCs"
  visibility  = "private"

  private_visibility_config {
    networks {
      network_url = google_compute_network.prod.id
    }
  }

  forwarding_config {
    target_name_servers {
      ipv4_address    = "10.0.0.10"
      forwarding_path = "private"
    }
    target_name_servers {
      ipv4_address    = "10.0.0.11"
      forwarding_path = "private"
    }
  }
}
```

`--private-forwarding-targets` (Terraform: `forwarding_path = "private"`) forces the query over the VPN or Interconnect even for targets Cloud DNS might otherwise try to reach over the internet.[^3] For RFC 1918 targets it's private anyway, but being explicit doesn't hurt.

Three things that bite people:

1. **The source address is not the VM.** Forwarded queries come from `35.199.192.0/19`, so the on-prem firewall must allow TCP and UDP 53 from that range to the DCs, across every tunnel or Interconnect attachment.[^2]
2. **The same server must answer.** Cloud DNS ignores a reply that comes from a different name server than the one it sent the query to.[^6] If your DCs relay queries to a separate DNS appliance that replies directly, it breaks.
3. **Forward the whole AD namespace.** If the forest root is `example.com` and the domain is `corp.example.com`, the DC locator also needs `_msdcs.example.com`. Forward every zone the DCs are authoritative for that clients need, and any reverse zones you care about (`0.10.in-addr.arpa.`).

To go the other direction (on-prem servers resolving names in Cloud DNS private zones), add an inbound server policy to the VPC and create a conditional forwarder on the DCs pointing to the inbound forwarder IPs. You don't need this for domain join.

### Option B: point the VMs straight at the DCs

You can also set the DC IPs as DNS servers inside each VM (or via DHCP options on Windows, or `systemd-resolved` on Linux). It works, but now every VM sends *all* DNS, including `metadata.google.internal` and `*.googleapis.com`, to on-prem, and you have to set it on every image. Unless you have a reason, prefer the forwarding zone.

## Step 3: firewall ports

Google Cloud allows all egress by default, so traffic from the VMs leaves the VPC unless you've added deny rules. The work is on the **on-prem firewall**, which must allow inbound from the VPC subnets to the DCs. Google's baseline list for GCP VMs that are members of an on-prem domain:[^7]

| Service | Port | Needed for |
|---|---|---|
| DNS | TCP/UDP 53 | Only from `35.199.192.0/19` if you use a forwarding zone; from VM subnets if VMs query DCs directly |
| Kerberos | TCP/UDP 88 | Authentication, tickets |
| NTP | UDP 123 | Time sync with the domain hierarchy |
| RPC endpoint mapper | TCP 135 | Locating RPC services (join, netlogon, GPO) |
| RPC dynamic | TCP 49152–65535 | The actual RPC traffic |
| LDAP | TCP/UDP 389 | Directory queries, DC locator (UDP "LDAP ping") |
| SMB | TCP 445 | SYSVOL / NETLOGON (Group Policy), join |
| Global Catalog | TCP 3268 | Forest-wide lookups |
| Kerberos password change | TCP/UDP 464 | Machine account password rotation (optional but you want it) |
| LDAPS / GC over TLS | TCP 636 / 3269 | If you enforce LDAP over TLS |

The dynamic RPC range is the one security teams push back on. You can [restrict the RPC port range on the DCs](https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/configure-rpc-dynamic-port-allocation-with-firewalls), but that's a change on the DCs, not something to do lightly.

To find what else you need, Google suggests enabling logging on a low-priority deny rule and reading the log entries. Turn logging off again afterwards, since it can be very noisy.[^7] The same trick works on the on-prem firewall: deny-and-log from the GCP subnets, try a join, read the drops.

## Step 4: AD sites and subnets

This one gets skipped, and it causes slow logons rather than failures, which is worse because nobody notices.

When a client looks for a DC, the DC locator first checks which AD *site* the client's IP belongs to, then prefers DCs in that site. If the client's subnet isn't mapped to any site, it can get *any* DC in the domain, including one in a branch office on the other side of the world.

In **Active Directory Sites and Services**:

- **Without DCs in GCP:** add the VPC subnets (e.g. `10.20.0.0/16`) to the site whose DCs are closest to where your VPN or Interconnect lands.
- **With DCs in GCP:** create a site per GCP region (e.g. `GCP-europe-west3`), put the regional subnets in it, and set up site links to on-prem with sensible costs. Google recommends separate sites for on-prem and GCP so clients use the nearest DCs.[^1]

Check from a VM with `nltest /dsgetsite` (Windows) or by looking at which DC SSSD picked (Linux, see below).

## Step 5: joining Windows VMs

### The prerequisites, checked from the VM

Before touching `Add-Computer`, check the plumbing from a test VM:

```powershell
# DNS: can we find a DC?
Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.corp.example.com

# Which DC and site would the locator pick?
nltest /dsgetdc:corp.example.com

# Ports
'dc01.corp.example.com' | ForEach-Object {
  foreach ($p in 88,135,389,445,464,3268) {
    Test-NetConnection $_ -Port $p | Select-Object ComputerName,RemotePort,TcpTestSucceeded
  }
}
```

If all of that's green, a join will almost certainly work.

### Hostnames: the 15-character trap

Instance names in Compute Engine can be up to 63 characters. NetBIOS computer names, which AD still uses for `sAMAccountName`, are limited to 15. A VM called `prod-sql-europe-west3-01` will be truncated, and two VMs that share their first 15 characters will fight over the same computer account. Name Windows VMs with 15 characters or less, or set the computer name explicitly in the join script.

### Manual join

RDP in with the local admin account (`gcloud compute reset-windows-password` creates one) and:

```powershell
Add-Computer -DomainName corp.example.com `
  -OUPath "OU=GCP,OU=Servers,DC=corp,DC=example,DC=com" `
  -Credential (Get-Credential CORP\svc-gcp-join) `
  -Restart
```

Fine for a test, useless at scale. You want the join to happen without a human.

### Automated join at first boot

Compute Engine runs scripts from metadata at different points of Windows setup. `sysprep-specialize-script-ps1` runs once, during the sysprep *specialize* pass on the very first boot, before the VM is usable. That's the right moment to join: the machine reboots at the end of specialize anyway, and comes up already domain-joined.[^8] (Google's own [domain-join solution](https://docs.cloud.google.com/architecture/configuring-active-directory-for-vms-to-automatically-join-the-domain) uses exactly this pass.[^9])

The join account's password must not be in the metadata, since anyone with `compute.instances.get` can read metadata. Put it in Secret Manager and give the VM's service account read access to that one secret:

```bash
# A dedicated service account for domain-joined VMs
gcloud iam service-accounts create win-domain-member

# The join password, stored once
printf '%s' 'S3cret!' | gcloud secrets create ad-join-password --data-file=-

gcloud secrets add-iam-policy-binding ad-join-password \
  --member="serviceAccount:win-domain-member@my-project.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

The specialize script (`join-domain.ps1`):

```powershell
$ErrorActionPreference = 'Stop'

$Domain = 'corp.example.com'
$OU     = 'OU=GCP,OU=Servers,DC=corp,DC=example,DC=com'
$User   = 'CORP\svc-gcp-join'

# Skip if already joined (e.g. image built from a joined machine by mistake)
if ((Get-CimInstance Win32_ComputerSystem).PartOfDomain) { return }

# Read the password from Secret Manager (Cloud SDK ships with GCE Windows images)
$Plain  = & gcloud secrets versions access latest --secret=ad-join-password
$Secure = ConvertTo-SecureString $Plain -AsPlainText -Force
$Cred   = New-Object System.Management.Automation.PSCredential($User, $Secure)
Remove-Variable Plain

# Wait for DNS to the DCs to work (the link may be slower than the boot)
$deadline = (Get-Date).AddMinutes(5)
while (-not (Resolve-DnsName -Type SRV "_ldap._tcp.dc._msdcs.$Domain" -ErrorAction SilentlyContinue)) {
  if ((Get-Date) -gt $deadline) { throw "Cannot locate a DC for $Domain" }
  Start-Sleep -Seconds 10
}

# No -Restart: Windows reboots after the specialize pass anyway
Add-Computer -DomainName $Domain -OUPath $OU -Credential $Cred -Force
```

Create the VM:

```bash
gcloud compute instances create app01 \
  --zone=europe-west3-a \
  --machine-type=e2-standard-4 \
  --image-family=windows-2025 --image-project=windows-cloud \
  --subnet=prod-subnet --no-address \
  --service-account=win-domain-member@my-project.iam.gserviceaccount.com \
  --scopes=cloud-platform \
  --metadata-from-file=sysprep-specialize-script-ps1=join-domain.ps1
```

Watch progress in the serial console (`gcloud compute instances get-serial-port-output app01`). The specialize script's output shows up there.

### Locking down the join account

The account used for joining should be able to do one thing: create computer objects in one OU.

- **Delegate, don't use Domain Admin.** In AD Users and Computers, right-click the `OU=GCP` OU → *Delegate Control* → give `svc-gcp-join` "Create, delete and manage computer accounts" on that OU only.
- **Mind the machine account quota.** By default, any authenticated user can join up to 10 computers to the domain (`ms-DS-MachineAccountQuota`).[^10] An account with delegated rights on an OU isn't limited by this, but a plain user account is, and will start failing at VM number 11. Many organizations set the quota to 0 so that only delegated accounts can join.
- **Account reuse is blocked.** Since the October 2022 hardening (KB5020276), a join can't reuse an existing computer account unless the joining user created that account or it was created by a domain admin.[^11] If you recreate a VM with the same name, either delete the old computer object first or join with the same account that created it.

The weakness of the Secret Manager approach is that any code running on any VM with that service account can read the join password. If that's not acceptable, there are two better options:

1. **Pre-staged accounts / offline join.** A trusted pipeline (Cloud Build, a CI runner with line of sight to the DCs) creates the computer account before the VM exists and passes a one-time join blob to the VM. On Windows that's `djoin.exe /provision` on the pipeline and `djoin /requestODJ` on the VM. The VM never sees a reusable credential.
2. **A domain-join broker.** A small trusted service does the join on the VM's behalf. This is worth its own section.

If you're using Managed Microsoft AD rather than your own DCs, it has a built-in automated join driven by metadata keys.[^12] That doesn't apply to on-prem DCs, though, which is the case here.

### A domain-join broker

Johannes Passing described this design in 2020,[^16] and Google publishes it as a reference solution with code and a deployment guide.[^9] It starts from two problems with the specialize-script approach above:

- **The join credential always leaks to someone.** The join user can join *any* computer. Wherever you keep its password (the script, Secret Manager, a custom image), it's exposed to someone: GCP users, Windows users on the VM, or anyone who can read the image.
- **Nothing cleans up.** Compute Engine has no termination hook that runs when a VM is deleted, so every deleted VM leaves an orphaned computer object in AD. With an autoscaled managed instance group creating and deleting VMs all day, the OU fills with junk.

The fix is a mediator: a service that is the *only* thing holding domain credentials. VMs never talk to AD with a user account; they ask the broker to join them, and prove who they are with their Google-issued identity.

```text
  New VM (specialize pass)        Broker (Cloud Run)                  Domain controller
  ────────────────────────        ──────────────────                  ─────────────────
  1. get ID token from
     metadata server
  2. POST /join  ──────────────►  3. verify token signature
     (ID token)                   4. read name, zone, project
                                  5. project allowed?  (OU mapping)
                                  6. VM really exists? (Compute API)
                                  7. create computer account  ──────►  LDAP
                                     with name/zone/project attrs
                                  8. set random password      ──────►  Kerberos kpasswd
  10. Add-Computer with      ◄──  9. return name + password
      pre-staged account           over TLS
  11. reboot (end of specialize)
                                  ⋯ every N minutes ⋯
                                  Cloud Scheduler → scan OU,
                                  delete accounts whose VM is gone  ──►  LDAP
```

The steps, as Google's guide describes them:[^9]

1. The VM gets an **ID token** from the metadata server. The token is signed by Google and contains the VM's name, zone and project, so the VM can't lie about who it is.
2. The broker validates the token and checks that the project is allowed to join. The mapping lives in AD itself: an OU per project, so VMs from project `app-prod` land in `OU=app-prod,OU=GCP,…`.
3. It checks through the Compute Engine API that the VM actually exists in that project.
4. It **pre-stages** the computer account and stores the VM's name, zone and instance ID as attributes on it.
5. It sets a random password on the account using the Kerberos set-password protocol and returns the name and password to the VM over TLS.
6. A PowerShell script in the specialize pass joins the domain with that pre-staged account, and the machine reboots.

The password the VM gets is good for one computer account and nothing else. Compromise a VM and you get that VM's own account, which you had anyway.

**Cleanup** runs as a Cloud Scheduler job that scans the OUs and deletes computer accounts whose VM no longer exists. The stored attributes are what make this reliable. Passing points out the two obvious alternatives don't work: last-logon timestamps are unreliable, and matching AD names to VM names is error-prone because the names can differ (hello, 15-character truncation).[^16]

#### Running it against on-prem DCs

The guide assumes DCs reachable from the VPC, but nothing in the design needs them to be *in* GCP. The broker is just another LDAP and Kerberos client. With on-prem DCs, a few extra things matter:

- **The broker's own network path.** The guide uses Serverless VPC Access, which needs an unused `/28` range in the VPC that can reach the DCs.[^9] That `/28` is the source address of the broker's traffic to AD. Add it to the Cloud Router's custom advertisements (next to `35.199.192.0/19`), and allow it through the on-prem firewall to the DCs. At minimum that's Kerberos (88), kpasswd (464) and LDAP (389); use the deny-and-log trick from step 3 to catch anything else.
- **DC discovery from the broker.** Make sure the broker can resolve the AD domain's SRV records through the VPC's Cloud DNS forwarding zone. Test this early; it's the least obvious hop.
- **Private Google Access** on the VM subnet, so VMs without external IPs can reach the Cloud Run endpoint.[^9]
- **The broker's service account in AD** needs delegated rights to create and delete computer accounts in the GCP OUs, and nothing more. Its password lives in Secret Manager, readable only by the broker, which is a far smaller blast radius than every VM.

The guide was originally written in 2020 and has been revised since. Check what the current code supports, for example Cloud Run's direct VPC egress instead of a Serverless VPC Access connector. The routing and firewall rules above apply either way.

#### Which approach when?

| | Secret Manager + delegated account | Pre-staged / `djoin` from a pipeline | Domain-join broker |
|---|---|---|---|
| Moving parts | One secret, one script | Pipeline needs line of sight to DCs | A Cloud Run service, a scheduler job, a VPC connector |
| Credential on the VM | Reusable join password | One-time blob | One-time password for its own account |
| Works with autoscaling (MIGs) | Yes | No, the pipeline doesn't create MIG VMs | Yes |
| Cleans up deleted VMs | No | Only if the pipeline also deletes | Yes |
| Good fit | A handful of long-lived servers | VMs created by Terraform or CI | Autoscaled or frequently recreated VMs, many projects |

For a dozen pet servers, the broker is more machinery than it's worth: a delegated account in Secret Manager plus an occasional manual cleanup is fine. Once VMs come and go on their own, the broker pays for itself through cleanup alone.

## Step 6: joining Linux VMs

Linux VMs join with **realmd**, which discovers the domain and configures **SSSD** for identity and authentication, plus Kerberos and PAM.[^13]

### Packages

```bash
# RHEL, Rocky, Alma
sudo dnf install -y realmd sssd adcli oddjob oddjob-mkhomedir samba-common-tools krb5-workstation

# Debian, Ubuntu
sudo apt-get install -y realmd sssd sssd-tools adcli libnss-sss libpam-sss \
  packagekit krb5-user
```

### Check, then join

```bash
# DNS: these must return your DCs
dig +short -t SRV _ldap._tcp.dc._msdcs.corp.example.com
dig +short -t SRV _kerberos._tcp.corp.example.com

# Discover the domain
realm discover corp.example.com

# Join
sudo realm join corp.example.com \
  -U svc-gcp-join \
  --computer-ou="OU=Linux,OU=GCP,OU=Servers,DC=corp,DC=example,DC=com"

# Verify
realm list
id alice@corp.example.com
```

Red Hat's knowledge base lists the same SRV lookups and `realm discover` as the pre-flight checks.[^14] After joining, SSSD by default maps AD SIDs to UIDs and GIDs algorithmically. If your AD already has POSIX attributes (`uidNumber`, `gidNumber`) that must match NFS shares on-prem, join with `--automatic-id-mapping=no` instead.[^15]

Linux hostnames have the same 15-character problem: `adcli` derives the computer account name from the short hostname and truncates it. Set `--computer-name` explicitly if your instance names are long.

### Automating it with a startup script

Linux startup scripts run on *every* boot, so guard the join:

```bash
#!/bin/bash
set -euo pipefail
DOMAIN=corp.example.com
OU="OU=Linux,OU=GCP,OU=Servers,DC=corp,DC=example,DC=com"

realm list | grep -q "^${DOMAIN}$" && exit 0

dnf install -y realmd sssd adcli oddjob oddjob-mkhomedir samba-common-tools krb5-workstation

# Wait for DC discovery to work
for i in $(seq 1 30); do
  dig +short -t SRV "_ldap._tcp.dc._msdcs.${DOMAIN}" | grep -q . && break
  sleep 10
done

gcloud secrets versions access latest --secret=ad-join-password \
  | realm join "$DOMAIN" -U svc-gcp-join --computer-ou="$OU"

# Allow only one AD group to log in, and use short usernames
realm permit -g linux-admins@corp.example.com
sed -i 's/^use_fully_qualified_names = True/use_fully_qualified_names = False/' /etc/sssd/sssd.conf
systemctl restart sssd
```

```bash
gcloud compute instances create web01 \
  --zone=europe-west3-a \
  --image-family=rhel-9 --image-project=rhel-cloud \
  --subnet=prod-subnet --no-address \
  --service-account=linux-domain-member@my-project.iam.gserviceaccount.com \
  --scopes=cloud-platform \
  --metadata-from-file=startup-script=join-domain.sh
```

The equivalent of offline join on Linux is pre-staging: create the account from a trusted host with `adcli preset-computer --one-time-password=…` and have the VM run `realm join --one-time-password=…`. No reusable credential ever touches the VM. The same idea would let a domain-join broker serve Linux VMs too, though Google's reference implementation targets Windows.

A note on OS Login: if you use OS Login for SSH, it's a separate mechanism from AD. Both can coexist, but decide which one controls SSH access, or you'll end up with two places to revoke someone's access.

## Troubleshooting

Work from the bottom up. Each layer depends on the one below.

| Symptom | Likely cause | Check |
|---|---|---|
| "The specified domain either does not exist or could not be contacted" | DNS: SRV records can't be resolved | `Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.<domain>` / `dig SRV` |
| SRV lookups time out, but work when querying a DC IP directly | Forwarding zone replies blocked or not routed | On-prem firewall allows `35.199.192.0/19` on 53? Route back advertised by Cloud Router? |
| DNS works, join fails with "network path not found" | SMB 445 or RPC blocked | `Test-NetConnection dc01 -Port 445`, then 135 |
| Join hangs then fails, small queries work | RPC dynamic range blocked, or MTU/fragmentation over the VPN | Deny-and-log on the firewall; test with large pings (`ping -f -l 1400`) |
| "Access denied" / "account already exists" | Computer account exists from a previous VM (KB5020276 hardening) | Delete the stale object, or join with the account that created it |
| Join works for 10 VMs, then fails | `ms-DS-MachineAccountQuota` reached for a non-delegated account | Delegate OU permissions to the join account |
| Broker returns errors, VMs never get a password | Broker's `/28` not routed or not allowed on-prem, or it can't resolve the domain | Cloud Router advertisements, on-prem firewall logs, Cloud Run logs |
| Joined, but logons are slow | Subnet not mapped to a site, client picks a far-away DC | `nltest /dsgetsite`, `nltest /dsgetdc:<domain>`; add the subnet in Sites and Services |
| Kerberos errors like `KRB_AP_ERR_SKEW` | Clock drift over 5 minutes | `w32tm /query /source` and `/status`; on Linux `chronyc sources` |
| Linux: `realm join` OK, `id user@domain` returns nothing | SSSD can't reach a DC or GC, or access not permitted | `sssctl domain-status corp.example.com`, `journalctl -u sssd` |
| Everything breaks when the VPN flaps | No local DCs, cached credentials only help interactive logons | Add DCs in GCP (extended domain) |

A word on **time**: Kerberos tolerates five minutes of clock skew by default. Compute Engine images sync time from the metadata server, which is accurate, so drift is rarely the problem. What matters is that members and DCs agree. If your on-prem DCs sync from a poor source, the GCP VMs will be correct and the DCs wrong, and tickets will fail. Fix the PDC emulator's time source rather than fighting it from the cloud side.

## Checklist

```text
Architecture   on-prem DCs only? → accept the link as a dependency, or add DCs in GCP
Connectivity   HA VPN / Interconnect, BGP, no overlapping ranges, redundant links
Routing        Cloud Router advertises VPC subnets + 35.199.192.0/19 (custom)
DNS            private forwarding zone for every AD zone → DC IPs
               on-prem FW: TCP/UDP 53 from 35.199.192.0/19 to DCs
Firewall       on-prem FW: VM subnets → DCs on 88, 123, 135, 389, 445, 464, 3268, 49152-65535
               GCP: check no custom egress deny rules
Sites          VPC subnets mapped to the nearest AD site (or a GCP site with its own DCs)
Join account   delegated "create computer objects" on one OU, nothing else
Secrets        few long-lived VMs: join password in Secret Manager, readable only by the member SA
               VMs from a pipeline: pre-staged accounts / djoin
               autoscaled VMs: domain-join broker (+ advertise and allow its /28)
Windows        hostname ≤ 15 chars, sysprep-specialize-script-ps1 → Add-Computer
Linux          realmd + SSSD, startup-script guarded by `realm list`
Cleanup        delete computer objects when VMs are deleted (broker does this; otherwise script it)
Verify         nltest /dsgetdc, nltest /dsgetsite, realm list, id user@domain
```

If you're running Kubernetes on GKE next to these VMs and want Windows containers to use AD identities too, the same plumbing (forwarding zone, firewall, join account) is the foundation for gMSA. That's a topic for another post.

## References

[^1]: [Patterns for using Active Directory in a hybrid environment](https://docs.cloud.google.com/architecture/patterns-for-using-active-directory-in-a-hybrid-environment), Google Cloud Architecture Center.
[^2]: [Resolve queries for non-Managed Microsoft AD objects](https://docs.cloud.google.com/managed-microsoft-ad/docs/resolve-non-managed-objects-vpc), Google Cloud documentation.
[^3]: [Create a forwarding zone](https://docs.cloud.google.com/dns/docs/zones/forwarding-zones), Cloud DNS documentation.
[^4]: [Cloud DNS forwarding, peering and zones](https://cloud.google.com/blog/products/networking/cloud-forwarding-peering-and-zones), Google Cloud blog.
[^5]: [DNS zones overview](https://docs.cloud.google.com/dns/docs/zones/zones-overview), Cloud DNS documentation.
[^6]: [Criar uma zona de encaminhamento](https://docs.cloud.google.com/dns/docs/zones/forwarding-zones?hl=pt-br) (forwarding zone requirements, "return route" section), Cloud DNS documentation.
[^7]: [Firewall configurations for Active Directory](https://docs.cloud.google.com/managed-microsoft-ad/docs/firewalls), Google Cloud documentation.
[^8]: [Using startup scripts on Windows VMs](https://cloud.google.com/compute/docs/instances/startup-scripts/windows), Compute Engine documentation.
[^9]: [Configure Active Directory for VMs to automatically join a domain](https://docs.cloud.google.com/architecture/configuring-active-directory-for-vms-to-automatically-join-the-domain), Google Cloud Architecture Center.
[^10]: [Domain Join and Basic troubleshooting](https://techcommunity.microsoft.com/blog/askds/-/4405860), Microsoft Ask the Directory Services Team blog.
[^11]: [Active Directory domain join troubleshooting guidance](https://learn.microsoft.com/en-us/troubleshoot/windows-server/identity/active-directory-domain-join-troubleshooting-guidance), Microsoft Learn.
[^12]: [Join a Windows VM automatically to a domain](https://docs.cloud.google.com/managed-microsoft-ad/docs/seamless-domain-join-gce), Managed Service for Microsoft Active Directory documentation.
[^13]: [Integrating RHEL systems directly with Windows Active Directory](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/integrating_rhel_systems_directly_with_windows_active_directory/integrating_rhel_systems_directly_with_windows_active_directory), Red Hat Enterprise Linux 10 documentation.
[^14]: [How to join RHEL to Active Directory using realmd](https://access.redhat.com/node/2980151), Red Hat Knowledgebase.
[^15]: [Connecting RHEL systems directly to AD using SSSD](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/integrating_rhel_systems_directly_with_windows_active_directory/connecting-rhel-systems-directly-to-ad-using-sssd), Red Hat Enterprise Linux 10 documentation.
[^16]: [Automatically joining a VM to Active Directory on Google Cloud](https://jpassing.com/2020/03/08/automatically-joining-a-vm-to-active-directory-on-google-cloud/), Johannes Passing.
