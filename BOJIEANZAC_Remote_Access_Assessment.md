# BOJIEANZAC Home Lab — Remote Access Infrastructure Build
## Final Infrastructure Lab Assessment
**TAFE Coomera | Diploma of ICT**
**Student:** Bojie Qiao
**Repository:** `bojieanzac-remote-access` (private) · `qbjsuper/tafe-virtual-server-lab` (public)
**Date:** June 2026

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Lab Environment Overview](#2-lab-environment-overview)
3. [Remote Access Architecture](#3-remote-access-architecture)
4. [Tailscale Configuration](#4-tailscale-configuration)
5. [Build Phases and Implementation](#5-build-phases-and-implementation)
6. [Security Model](#6-security-model)
7. [Verification and Testing](#7-verification-and-testing)
8. [Key Learnings and Troubleshooting](#8-key-learnings-and-troubleshooting)
9. [Engineering Methodology](#9-engineering-methodology)
10. [Documentation Structure](#10-documentation-structure)
11. [Future Direction](#11-future-direction)
12. [Appendix: Repository Structure](#12-appendix-repository-structure)

---

## 1. Executive Summary

This document describes the design, implementation, and verification of a secure remote access infrastructure for the **BOJIEANZAC home lab** — a multi-site, enterprise-grade Windows Server and Hyper-V environment running on two mini PCs at home, built for TAFE Diploma of ICT coursework.

The project (tracked as **BOJIEANZAC-REMOTE-ACCESS**) was delivered in four phases:

- **Phase 0** — Pre-checks and metal layer baseline
- **Phase 1** — pfSense Tailscale subnet routing on both lab sites
- **Phase 2** — Tailscale ACL tag policy and least-privilege access control
- **Phase 3** — MFA, key expiry, and device approval (identity hardening)
- **Phase 4** — Host firewall hardening and network boundary verification

All four phases are complete. The end state is a fully verified, remotely accessible lab reachable from TAFE Coomera (or any location) via Tailscale, with MFA-enforced identity, least-privilege ACLs, pfSense as the authoritative security boundary, hardened host firewalls, and a PiKVM out-of-band rescue path independent of domain state. Six connectivity paths were verified from both home LAN and a mobile hotspot (simulating the TAFE network environment).

The design follows a **defense-in-depth** model: every access path passes through multiple independent security controls, and no lab service is exposed directly to the public internet.

---

## 2. Lab Environment Overview

### 2.1 Purpose and Context

The BOJIEANZAC lab simulates a two-site corporate Active Directory environment. It demonstrates enterprise networking and server administration skills across domain services, failover clustering, nested virtualisation, iSCSI storage, and a Linux application platform. All workloads run as Hyper-V virtual machines on two consumer mini PCs connected to a home LAN managed by a UniFi Cloud Gateway Max.

- **Domain:** `bojieanzac.com`
- **NetBIOS name:** `BOJIEANZAC`
- **Lab VM prefix:** `BAA`
- **AD model:** One forest · one domain · two sites
- **Forest root DC:** `BAA-BIG-DC1`

### 2.2 Physical Infrastructure

#### Physical Hosts

Both hosts are in a **workgroup — not domain-joined**. (See architecture decision in Section 3.4.)

| Host | Role | Home LAN IP | Lab Site | Notes |
|------|------|-------------|----------|-------|
| Bojiemini2 | Hyper-V host | 192.168.1.252 (static) | BIG — 172.16.60.0/24 | Win11 Pro, workgroup |
| Bojie-Mini | Hyper-V host | 192.168.1.135 (static) | SML — 172.16.50.0/24 | Win11 Pro, workgroup |
| QIAO-AU | Home LAN host / Tailscale subnet router | 192.168.1.x (DHCP) | Home LAN | Win11, Hyper-V, advertises 192.168.1.0/24 |
| PiKVM | Out-of-band KVM | 192.168.1.x (DHCP) / 100.118.254.20 (Tailscale) | Home LAN | Arch Linux ARM, OOB rescue + Wake-on-LAN |
| UniFi CGM | Router / firewall / DHCP | 192.168.1.1 | Home LAN | No WAN port-forwarding |

#### Bojiemini2 — NIC Configuration

| NIC Name | Description | MAC | IP | Role |
|----------|-------------|-----|----|------|
| NIC-MGMT-TAILSCALE | Realtek 2.5GbE #3 | 38-05-25-38-50-5D | 192.168.1.252 | Host management + Tailscale |
| NIC-LAB-HYPERV-PHYSICAL | Realtek 2.5GbE #4 | 38-05-25-38-50-5C | (vSwitch only) | BOJIEANZAC-External vSwitch (pfSense WAN) |
| vEthernet (BOJIEANZAC-Internal) | Hyper-V Internal vSwitch | — | 172.16.60.2 (static) | Host lab leg |

`AllowManagementOS`: **disabled** on BOJIEANZAC-External. Host IP routing: **disabled** (`IPEnableRouter = No`).

#### Bojie-Mini — NIC Configuration

| NIC Name | Description | MAC | IP | Role |
|----------|-------------|-----|----|------|
| NIC-MGMT-TAILSCALE | Realtek 2.5GbE | 38-05-25-34-06-20 | 192.168.1.135 | Host management + Tailscale |
| NIC-LAB-HYPERV-PHYSICAL | Intel I226-V 1GbE | 38-05-25-34-06-21 | (vSwitch only) | BAA external switch (pfSense WAN) |
| vEthernet (BAA internal switch) | Hyper-V Internal vSwitch | — | 172.16.50.2 (static) | Host lab leg |
| Ethernet 3 | Intel X710 10GbE | 38-05-25-34-06-22 | disconnected | Future: iSCSI / heartbeat |
| Ethernet 4 | Intel X710 10GbE #2 | 38-05-25-34-06-23 | disconnected | Future: iSCSI / heartbeat |

`AllowManagementOS`: **disabled** on BAA external switch. Host IP routing: **disabled**.

### 2.3 Lab Sites

#### BIG Site — 172.16.60.0/24 (hosted on Bojiemini2)

| VM | Role | IP | Notes |
|----|------|----|-------|
| BAA-BIG-PFS1 | pfSense gateway / firewall / VPN | 172.16.60.1 | Site default gateway, IPsec VPN endpoint, Tailscale subnet router |
| BAA-BIG-DC1 | AD DS / DNS / DHCP | 172.16.60.10 | Forest root DC, PDC emulator |
| BAA-BIG-HA1 | Failover cluster node | 172.16.60.21 | BAA-CLUSTER1 member |
| BAA-BIG-Nest1 | Nested Hyper-V node | 172.16.60.x | BAA-NEST-CL1 member |
| BAA-BIG-LX1 | Linux domain member | DHCP | Ubuntu, domain-joined |
| BAA-BIG-LX2 | Linux test VM | DHCP | Imported VM |
| BAA-BIG-WS | Windows workstation | DHCP | Domain-joined |
| Bojiemini2 host leg | Physical host on lab segment | 172.16.60.2 | Static, infra range |

DHCP scope: `172.16.60.100 – 172.16.60.199`
DNS: `172.16.60.10` (primary) · `172.16.50.10` (secondary)

#### SML Site — 172.16.50.0/24 (hosted on Bojie-Mini)

| VM | Role | IP | Notes |
|----|------|----|-------|
| BAA-SML-PFS1 | pfSense gateway / firewall / VPN | 172.16.50.1 | Site default gateway, IPsec VPN endpoint, Tailscale subnet router |
| BAA-SML-DC1 | AD DS / DNS | 172.16.50.10 | Additional DC, Global Catalog |
| BAA-SML-HA1 | Failover cluster node | 172.16.50.21 | BAA-CLUSTER1 member |
| BAA-STOR1 | iSCSI storage server | 172.16.50.101 | Serves CSV to nested cluster |
| BAA-SML-NETH1 | Rocky Linux 9 / NethServer 8 | 172.16.50.60 | NS8 leader, 10.250.0.0/24 internal |
| BAA-SML-Nest1 | Nested Hyper-V node | 172.16.50.x | BAA-NEST-CL1 member |
| Bojie-Mini host leg | Physical host on lab segment | 172.16.50.2 | Static, infra range |

DHCP scope: `172.16.50.100 – 172.16.50.199`
DNS: `172.16.50.10` (primary) · `172.16.60.10` (secondary)

### 2.4 Site-to-Site Connectivity

BIG and SML are connected via an IPsec VPN tunnel between the two pfSense gateways:

| Side | pfSense | Local LAN | Remote LAN | Method |
|------|---------|-----------|------------|--------|
| BIG | BAA-BIG-PFS1 | 172.16.60.0/24 | 172.16.50.0/24 | IPsec VPN |
| SML | BAA-SML-PFS1 | 172.16.50.0/24 | 172.16.60.0/24 | IPsec VPN |

Status: **confirmed working** as of 2026-06-01.

This tunnel carries:
- Active Directory replication between BAA-BIG-DC1 and BAA-SML-DC1
- BAA-NEST-CL1 cluster heartbeat between nested Hyper-V nodes
- iSCSI storage traffic from BAA-BIG-Nest1 to BAA-STOR1 (same-site BAA-SML-Nest1 preferred for heavy disk operations to avoid cross-tunnel iSCSI load)

### 2.5 Cluster Infrastructure

#### BAA-NEST-CL1 — Nested Hyper-V Failover Cluster

| Item | Detail |
|------|--------|
| Node 1 | BAA-BIG-Nest1 (on Bojiemini2) |
| Node 2 | BAA-SML-Nest1 (on Bojie-Mini) |
| Shared storage | BAA-STOR1 iSCSI → Cluster Disk 1 |
| CSV path | `C:\ClusterStorage\Volume1` |
| Test VM | BAA-TEST-VM01 / Ubibi (Ubuntu 26.04) |
| Status | **Working — live migration proven** |

Live migration finding: the BIG node presents the guest on `172.16.60.0/24`, the SML node on `172.16.50.0/24`. A systemd timer inside Ubibi runs `networkctl reconfigure eth0` every 30 seconds to adapt automatically after migration.

#### BAA-CLUSTER1 — Stretched Windows Failover Cluster

| Item | Detail |
|------|--------|
| Node 1 | BAA-BIG-HA1 · 172.16.60.21 |
| Node 2 | BAA-SML-HA1 · 172.16.50.21 |
| Cluster IP (BIG online) | 172.16.60.110 |
| Cluster IP (SML online) | 172.16.50.110 |
| Quorum | File Share Witness on BAA-BIG-DC1 |
| Fault domains | BAA-BIG-SITE · BAA-SML-SITE |
| Status | Stretched cluster foundation working |
| Storage Replica | Not supported on local disks in a cluster; kept as failover proof |

### 2.6 Topology Diagram

```
TAFE Coomera
  qlaptop
    |
    | Tailscale (WireGuard · MFA · ACLs · DERP fallback via Sydney)
    |
Home network — 192.168.1.0/24
  UniFi Cloud Gateway Max (no WAN port-forwarding)
    |
    +-- Bojiemini2 (Win11 Pro · Hyper-V) ── NIC-MGMT: 192.168.1.252
    |     BIG site — 172.16.60.0/24          Lab leg:  172.16.60.2
    |     BAA-BIG-PFS1 (.1)  ← pfSense · IPsec · Tailscale subnet router
    |     BAA-BIG-DC1  (.10) ← AD DS · DNS · DHCP · Forest root
    |     BAA-BIG-HA1  (.21) ← BAA-CLUSTER1 node
    |     BAA-BIG-Nest1      ← BAA-NEST-CL1 node (nested Hyper-V)
    |     BAA-BIG-LX1/LX2/WS
    |
    +-- Bojie-Mini (Win11 Pro · Hyper-V) ── NIC-MGMT: 192.168.1.135
    |     SML site — 172.16.50.0/24           Lab leg:  172.16.50.2
    |     BAA-SML-PFS1  (.1)   ← pfSense · IPsec · Tailscale subnet router
    |     BAA-SML-DC1   (.10)  ← AD DS · DNS · GC
    |     BAA-SML-HA1   (.21)  ← BAA-CLUSTER1 node
    |     BAA-STOR1     (.101) ← iSCSI → BAA-NEST-CL1 CSV
    |     BAA-SML-NETH1 (.60)  ← NethServer 8 · Rocky Linux 9
    |     BAA-SML-Nest1        ← BAA-NEST-CL1 node (nested Hyper-V)
    |
    +-- QIAO-AU (Win11 · Hyper-V)
    |     Home-LAN host · Tailscale tag:lab-router
    |     Advertises 192.168.1.0/24 to tailnet
    |
    +-- PiKVM (Arch Linux ARM)
          tag:rescue · 100.118.254.20 (Tailscale)
          OOB console + Wake-on-LAN for Bojie-Mini and Bojiemini2

BIG ↔ SML: IPsec VPN tunnel (BAA-BIG-PFS1 ↔ BAA-SML-PFS1)
AD replication: BAA-BIG-DC1 ↔ BAA-SML-DC1
NEST-CL1 live migration: BAA-BIG-Nest1 ↔ BAA-SML-Nest1
iSCSI: BAA-STOR1 → both cluster nodes (BAA-SML-Nest1 preferred, cross-tunnel for BAA-BIG-Nest1)
```

---

## 3. Remote Access Architecture

### 3.1 Core Design Principle: Defense in Depth

The overarching design principle is **defense in depth**. Every access path from qlaptop to the lab passes through multiple independent security controls. No single point of failure exposes the lab, and no service is published directly to the public internet.

Access flows through these layers, outside to inside:

| Layer | Control | Description |
|-------|---------|-------------|
| 1 | qlaptop at TAFE | The only external entry point |
| 2 | Tailscale identity | WireGuard transport · MFA · key expiry · device approval |
| 3 | Tailscale ACLs | Least-privilege; qlaptop reaches only required management ports |
| 4 | pfSense subnet routers | Each pfSense advertises its own lab subnet; single security boundary |
| 5 | Network + host hardening | No WAN forwarding · Windows Firewall scoped to Tailscale interface |
| 6 | Lab resources | AD · Hyper-V · BAA-NEST-CL1 · iSCSI · NethServer |

The **PiKVM rescue path** still passes through layers 2–3 (Tailscale identity and ACL) but bypasses layers 4–5 because it is out-of-band hardware access to physical host consoles, independent of the domain or pfSense state.

### 3.2 Access Matrix

| Scenario | Access Path |
|----------|-------------|
| Daily lab work on a VM | Tailscale → RDP to VM at 172.16.x.x directly |
| Manage a Hyper-V host | Tailscale → RDP to mini host mgmt NIC → Hyper-V Manager |
| pfSense web UI | Tailscale → HTTPS to 172.16.x.1 |
| NethServer dashboard | Tailscale → HTTPS to 172.16.50.60 |
| Cluster admin (NEST-CL1) | Tailscale → routes to both subnets → Failover Cluster Manager |
| Host unbootable / BIOS | Tailscale → PiKVM (100.118.254.20 : 443) → console + Wake-on-LAN |
| TAFE network (UDP blocked) | Tailscale DERP relay over HTTPS port 443 — automatic, no user action |
| Emergency (no Tailscale) | Mobile hotspot on qlaptop as alternative network path |

### 3.3 Subnet Routing Model

Routes into each lab subnet are advertised via pfSense at each site gateway. Windows IP routing on the physical hosts is explicitly disabled.

| Router | Subnet Advertised | Method |
|--------|-------------------|--------|
| BAA-BIG-PFS1 | 172.16.60.0/24 | pfSense Tailscale package (`VPN → Tailscale`) |
| BAA-SML-PFS1 | 172.16.50.0/24 | pfSense Tailscale package (`VPN → Tailscale`) |
| QIAO-AU | 192.168.1.0/24 | Tailscale Windows client (`tailscale up --advertise-routes`) |

This means qlaptop's traffic destined for `172.16.60.10` travels over Tailscale's WireGuard tunnel to BAA-BIG-PFS1, which then routes it into the BIG site LAN — passing through pfSense firewall rules on the way in and out.

### 3.4 Key Architecture Decisions

Three decisions shape the entire design. Full reasoning is in `docs/02-remote-access-design.md`.

---

**Decision 1: Physical Hosts Stay Workgroup — Not Domain-Joined**

All domain controllers run as Hyper-V guest VMs on the two mini hosts. Joining the physical hosts to `bojieanzac.com` would create a **circular boot dependency**: the host needs AD to authenticate, but AD only exists once the host boots and starts the DC virtual machines.

Keeping hosts in a workgroup with local administrator accounts, combined with the PiKVM as an out-of-band console path, means the management plane is completely independent of domain state. If all domain services fail or the DC VMs won't start, the hosts are still accessible for recovery.

*Revisit if a physical or external DC is added — that would break the circular dependency and make domain-joining hosts safe.*

---

**Decision 2: Subnet Routing via pfSense, Not Windows Hosts**

Windows IP routing (`IPEnableRouter`) is explicitly **disabled** on both Bojiemini2 and Bojie-Mini.

Each mini has a leg on both the home LAN (192.168.1.0/24 via the management NIC) and the lab subnet (172.16.x.0/24 via the internal vSwitch). Enabling IP routing on the host would create a home-to-lab path that **bypasses pfSense entirely**, undermining the firewall boundary that is central to the security model.

By running the Tailscale package inside pfSense VMs instead, pfSense remains the single clean boundary. All inter-network traffic passes through pfSense firewall rules.

---

**Decision 3: AllowManagementOS Disabled on External vSwitches**

Both Hyper-V hosts have `AllowManagementOS` disabled on their external vSwitch (the switch that carries pfSense's WAN/upstream traffic).

Before this change, the host management OS was bridged onto the same physical segment as pfSense's WAN interface — meaning the host had an implicit presence on that network segment. After disabling it, each host has exactly two addresses:
- Management NIC: `192.168.1.x` (home LAN, Tailscale)
- Lab leg: `172.16.x.2` (static, internal vSwitch only)

The external physical NIC carries only pfSense WAN/VM traffic and is invisible to the host OS.

---

## 4. Tailscale Configuration

### 4.1 Tailnet Device Inventory

All five devices are enrolled in the same Tailscale tailnet under a single account. Devices are tagged to enable ACL policy enforcement.

| Device | Tag | Tailscale IP | Advertises | Role |
|--------|-----|-------------|------------|------|
| qlaptop | `tag:client` | 100.x.x.x | — | External entry point; TAFE laptop |
| pikvm | `tag:rescue` | 100.118.254.20 | — | OOB rescue; accessible on port 443 only |
| qiao-au | `tag:lab-router` | 100.x.x.x | 192.168.1.0/24 | Home LAN subnet router |
| bojie-mini (BAA-SML-PFS1) | `tag:lab-router` | 100.x.x.x | 172.16.50.0/24 | SML site subnet router |
| bojiemini2 (BAA-BIG-PFS1) | `tag:lab-router` | 100.x.x.x | 172.16.60.0/24 | BIG site subnet router |

### 4.2 ACL Policy (HuJSON)

The full ACL policy is stored in `tailscale/acl-policy.hujson` and is pasted directly into Tailscale Admin → Access Controls. It enforces least-privilege access: `tag:client` can only reach specific management ports on each lab subnet, and can only reach the PiKVM on HTTPS.

```hujson
{
  "tagOwners": {
    "tag:client":     ["autogroup:admin"],
    "tag:lab-router": ["autogroup:admin"],
    "tag:rescue":     ["autogroup:admin"],
  },

  "acls": [
    {
      "action": "accept",
      "src":    ["tag:client"],
      "dst": [
        "172.16.60.0/24:3389,5985,5986,443,22",  // BIG site
        "172.16.50.0/24:3389,5985,5986,443,22",  // SML site
        "192.168.1.0/24:3389,443,22",             // home LAN hosts
        "tag:rescue:443",                         // PiKVM web UI only
      ],
    },
  ],

  "autoApprovers": {
    "routes": {
      "172.16.60.0/24": ["tag:lab-router"],
      "172.16.50.0/24": ["tag:lab-router"],
      "192.168.1.0/24": ["tag:lab-router"],
    },
  },
}
```

Permitted ports per destination:
- **BIG site (172.16.60.0/24):** RDP (3389), WinRM (5985, 5986), HTTPS (443), SSH (22)
- **SML site (172.16.50.0/24):** RDP (3389), WinRM (5985, 5986), HTTPS (443), SSH (22)
- **Home LAN (192.168.1.0/24):** RDP (3389), HTTPS (443), SSH (22)
- **PiKVM (tag:rescue):** HTTPS (443) only

The `autoApprovers` block allows each `tag:lab-router` device to self-approve the route it advertises, removing the need for manual approval on every restart.

All other traffic — including lab-to-lab traffic over Tailscale — is implicitly denied. Lab nodes communicate with each other via their site-local networks and the existing IPsec tunnel, not via Tailscale.

### 4.3 Identity Hardening

Three controls are configured on the Tailscale admin console beyond the ACL policy:

- **MFA enforcement** — required for all account logins; second factor through the identity provider
- **Key expiry** — Tailscale keys expire on a set schedule; devices must re-authenticate periodically
- **Device approval** — new devices must be explicitly approved by an admin before joining the tailnet

These controls layer on top of WireGuard's built-in encryption. Even if an adversary obtained a valid Tailscale key from a stolen device, they cannot access the tailnet without the MFA second factor, and the key expires within the configured window.

### 4.4 TAFE Network Fallback — DERP Relay

Tailscale uses WireGuard over **UDP 41641** by default for direct peer-to-peer connections. TAFE Coomera's network blocks outbound UDP on non-standard ports.

When Tailscale cannot establish a direct WireGuard connection (because UDP 41641 is blocked), it automatically falls back to **DERP relay servers over HTTPS (port 443)**, which all networks permit. The fallback is entirely transparent — no user configuration, no reconnect required.

For Australian users, the **Sydney DERP relay** is used when relay mode is forced, keeping latency manageable. Testing confirmed that in hotspot mode (simulating the TAFE environment), Tailscale fell back to DERP automatically and all six verification paths passed.

If both UDP and DERP were blocked (extremely unlikely), the contingency is to use qlaptop's phone hotspot as an alternate network path.

---

## 5. Build Phases and Implementation

The project was structured into five phases, each building on the previous. A personal **8-step engineering methodology** was applied throughout every phase (see Section 9).

### Phase 0 — Pre-Checks and Metal Layer (Complete)

Established a verified baseline before any remote access components were added. Goal: confirm the physical and virtual infrastructure is in a clean, known state.

**Tasks completed:**

- All five devices confirmed enrolled in Tailscale admin: `qlaptop · pikvm · qiao-au · bojie-mini · bojiemini2`
- NIC bindings verified by MAC address on both hosts — confirmed the correct physical NIC is bound to each Hyper-V external vSwitch:
  - Bojiemini2: `NIC-LAB-HYPERV-PHYSICAL` (MAC `…50-5C`) → BOJIEANZAC-External vSwitch ✓
  - Bojie-Mini: `NIC-LAB-HYPERV-PHYSICAL` (MAC `…06-21`) → BAA external switch ✓
- `AllowManagementOS` disabled on the external vSwitch on both hosts — host management OS no longer bridged onto the pfSense WAN segment
- Static lab leg IPs set on both hosts:
  - Bojiemini2: `172.16.60.2`
  - Bojie-Mini: `172.16.50.2`
- DHCP service dependency fix applied on both DCs to prevent Event 1059 on slow VM boot (startup race condition):
  ```
  sc.exe config DHCPServer depend= NTDS/Tcpip/Afd
  ```
- BIG ↔ SML IPsec tunnel confirmed working
- Tailscale re-login completed on both minis — back on tailnet with valid `100.x` addresses

### Phase 1 — pfSense Subnet Routing (Complete)

Installed and configured the Tailscale package on each pfSense gateway VM to advertise its lab subnet into the tailnet. Configured QIAO-AU to advertise the home LAN subnet.

**pfSense setup sequence (repeated for each gateway):**

1. Log in to pfSense web UI (`https://172.16.50.1` or `https://172.16.60.1`)
2. Navigate to **System → Package Manager → Available Packages**, search for `Tailscale`, click Install
3. Navigate to **VPN → Tailscale**
4. Enable Tailscale; enter a one-time auth key generated from Tailscale Admin → Settings → Keys
5. Set advertised route: `172.16.50.0/24` (SML) or `172.16.60.0/24` (BIG)
6. Save and Apply

**QIAO-AU home LAN route (PowerShell, run elevated):**

```powershell
# scripts/setup-subnet-router-lan.ps1
tailscale up --advertise-routes=192.168.1.0/24 --accept-dns=false
```

**Route approval in Tailscale admin:**

For each device: Machines → device → Edit route settings → approve the advertised subnet.
(With `autoApprovers` in the ACL policy, this self-approves on subsequent restarts.)

SML was completed first, then BIG, as SML is the simpler site to test against.

### Phase 2 — Tailscale ACL Policy (Complete)

Tagged all devices in Tailscale admin, then applied the ACL policy.

**Tagging order:**

| Device | Tag Applied |
|--------|-------------|
| qlaptop | `tag:client` |
| BAA-BIG-PFS1 (bojiemini2) | `tag:lab-router` |
| BAA-SML-PFS1 (bojie-mini) | `tag:lab-router` |
| qiao-au | `tag:lab-router` |
| pikvm | `tag:rescue` |

The ACL HuJSON (see Section 4.2) was pasted into **Tailscale Admin → Access Controls**. Post-ACL verification confirmed qlaptop still reached both DCs on port 3389 and that non-allowed ports were blocked.

### Phase 3 — Identity Hardening (Complete)

Configured on the Tailscale admin console:

- **MFA** enabled via the identity provider
- **Key expiry** enabled — devices re-authenticate on schedule
- **Device approval** enabled — new devices cannot join without admin approval

These controls are independent of the ACL. An adversary who bypasses one (e.g., obtains a key) still hits the others (MFA, expiry).

### Phase 4 — Network and Host Hardening (Complete)

Tightened Windows Firewall rules on both mini hosts and both domain controllers using PowerShell scripts committed to the repo.

**Host firewall hardening (`harden-host-firewall.ps1`):**

- RDP (3389) restricted to inbound from Tailscale interface (`100.x/8`) and lab subnets (`172.16.0.0/16`) only
- WinRM (5985/5986) same restriction
- Access from the open home LAN (`192.168.1.0/24`) blocked for these ports

**DC firewall hardening (`harden-dc-firewall.ps1`):**

- Same RDP/WinRM restrictions applied to both BAA-BIG-DC1 and BAA-SML-DC1
- Verified service was listening on port 3389 with `Test-NetConnection -Port 3389` before and after

**Other hardening verified:**

- No WAN port-forwarding rules on the UniFi Cloud Gateway Max for any lab management ports (confirmed in UniFi dashboard)
- AllowManagementOS confirmed disabled on both hosts' external vSwitches (completed in Phase 0, re-verified)

---

## 6. Security Model

### 6.1 Principles

**Least Privilege**

The Tailscale ACL grants `qlaptop` access to only the ports required for lab management. If qlaptop is lost or compromised, an attacker with access to the tailnet cannot reach unexpected services, cannot pivot to non-management ports, and cannot reach resources outside the defined subnets. pfSense also applies its own firewall rules on the Tailscale interface, providing a second independent filtering layer.

**Separation of Contexts**

- Hosts are workgroup — domain state cannot affect host access or authentication
- `AllowManagementOS` disabled — hosts are not bridged onto the pfSense WAN segment
- Windows IP routing disabled — hosts cannot route between home LAN and lab subnets; pfSense is the enforced boundary
- External vSwitch carries only pfSense WAN/VM traffic — host OS is invisible on that segment

**Defense in Depth**

Identity (MFA), transport (WireGuard), authorisation (ACL), routing (pfSense), and host hardening are all independent layers. Compromising one layer does not defeat the others. A valid Tailscale identity without ACL authorisation cannot reach management ports. Valid ACL authorisation without a route approved on the pfSense gateway cannot reach lab VMs. A route without passing pfSense firewall rules still hits host-level Windows Firewall.

### 6.2 Non-Negotiables

- **No public internet exposure** — no WAN port-forwarding on UniFi for RDP, AD, Hyper-V, or any lab service
- **Windows Firewall** restricts RDP (3389) and WinRM (5985/5986) to accept connections only from the Tailscale interface (`100.x`) and lab subnets — not from the open home LAN
- **MFA + key expiry + device approval** all active on the tailnet
- **Repo stays private** — secrets never committed; `.gitignore` covers `*.key`, `*.pem`, `*.pfx`, `*.env`, `tskey-*`
- **Physical hosts never domain-joined** while all DCs are guest VMs (circular boot dependency)
- **Windows IP routing disabled** on both hosts permanently — pfSense is the routing and security boundary
- **PiKVM** accessible on port 443 only, tagged `tag:rescue`

---

## 7. Verification and Testing

### 7.1 Six-Path Verification

Full end-to-end verification was performed from two distinct network contexts:

- **Home LAN** — qlaptop on 192.168.1.0/24, direct WireGuard path to Tailscale peers
- **Mobile hotspot** — qlaptop isolated on cellular data, simulating the TAFE Coomera environment where UDP 41641 is blocked and Tailscale falls back to DERP relay over HTTPS

Six paths were tested in each context:

| # | Destination | Port | Via |
|---|-------------|------|-----|
| 1 | BAA-BIG-DC1 (172.16.60.10) | 3389 RDP | BIG site pfSense subnet router |
| 2 | BAA-SML-DC1 (172.16.50.10) | 3389 RDP | SML site pfSense subnet router |
| 3 | BAA-BIG-PFS1 (172.16.60.1) | 443 HTTPS | BIG pfSense web UI |
| 4 | BAA-SML-PFS1 (172.16.50.1) | 443 HTTPS | SML pfSense web UI |
| 5 | NethServer (172.16.50.60) | 443 HTTPS | SML site, NethServer 8 dashboard |
| 6 | PiKVM (100.118.254.20) | 443 HTTPS | Direct Tailscale peer, tag:rescue |

**Result: all six paths passed in both network contexts.** In hotspot mode, Tailscale correctly transitioned to the Sydney DERP relay without user intervention and all paths remained accessible.

### 7.2 Diagnostic Methodology — Layer-by-Layer

A consistent layer-by-layer diagnostic approach was applied to every connectivity issue during the build. Skipping layers leads to misdiagnosed root causes.

| Layer | Command / Tool | What It Confirms |
|-------|---------------|-----------------|
| 1. Routing | `tailscale status` | Tailscale peer visibility and advertised route table |
| 2. TCP port | `Test-NetConnection 172.16.x.x -Port 3389` | Firewall open and service listening — **ICMP ping is not sufficient** |
| 3. Service state | `Get-Service TermService` | RDP service is running on the destination |
| 4. Auth / credentials | `mstsc` / Windows App | NLA, credential format, Microsoft account delegation issues |

**Critical rule:** ICMP ping (`ping 172.16.x.x`) blocked by Windows Firewall does not mean the host is unreachable. Always test the specific port with `Test-NetConnection -Port 3389`. This was the pivotal diagnostic lesson from an early incident where both DCs appeared down but were fully accessible on port 3389.

### 7.3 Verify-Routing.ps1 Script

A PowerShell verification script runs on qlaptop after any routing change and tests all three subnets:

```powershell
# scripts/verify-routing.ps1 — run on qlaptop after route approval

Write-Host "== Tailscale status ==" -ForegroundColor Cyan
tailscale status

Write-Host "`n== Testing BIG site DC (172.16.60.10 : RDP 3389) ==" -ForegroundColor Cyan
Test-NetConnection 172.16.60.10 -Port 3389

Write-Host "`n== Testing SML site DC (172.16.50.10 : RDP 3389) ==" -ForegroundColor Cyan
Test-NetConnection 172.16.50.10 -Port 3389

Write-Host "`n== Testing NethServer (172.16.50.60 : HTTPS 443) ==" -ForegroundColor Cyan
Test-NetConnection 172.16.50.60 -Port 443

Write-Host "`nTcpTestSucceeded = True on all = routing + ACLs working." -ForegroundColor Green
```

`TcpTestSucceeded = True` on all three confirms the full stack: Tailscale identity → ACL → pfSense routing → host firewall.

---

## 8. Key Learnings and Troubleshooting

All troubleshooting entries are committed to a private `troubleshooting` GitHub repository using a flat-entry, multi-tag taxonomy. Problems are not filed under a single category because real issues span multiple domains simultaneously (e.g., an RDP failure touches networking, Windows service state, and authentication at once).

### 8.1 Ping Does Not Equal Service Availability

**What happened:** During the build, both DCs appeared unreachable because `ping 172.16.x.10` failed. The initial assumption was a routing or firewall problem blocking all traffic.

**Actual cause:** Windows Firewall on server SKUs blocks ICMP echo requests by default. RDP on port 3389 was listening and fully accessible throughout.

**Lesson:** `Test-NetConnection -Port 3389` against the destination is the correct diagnostic for RDP reachability. In one related incident, `fDenyTSConnections` was set to `1` in the registry on a DC (RDP disabled at the service level). TCP test to port 3389 would have confirmed the port was not listening — ICMP would have looked identical (blocked in both cases).

**Rule now applied:** Never use ping as a diagnostic for service availability. Always test the specific TCP port.

### 8.2 Microsoft Account Delegation Removes Local SAM Authority

**What happened:** qlaptop could not RDP to Bojiemini2 (192.168.1.252) despite correct routing. All credential formats were rejected — `hostname\username`, `.\username`, `username@domain`. `net user` and `Set-LocalUser` commands also failed. Error code: **8646** — the local SAM has no authority over this account's password.

**Root cause:** The local account `qiaob` on Bojiemini2 was linked to a Microsoft account. When a Windows local account is linked to a Microsoft account, the **local SAM database loses authority** over the account's password. The password is managed by Microsoft's identity service, not the local machine. This makes every local credential tool and format fail.

**Failed workarounds:** All credential formats via mstsc and Windows App, NLA disabled, creating a separate `labrdp` local account (rejected — MS-account-linked apps on Bojiemini2 would be inaccessible from a different profile).

**Correct fix:** On Bojiemini2: **Settings → Accounts → Your info → Sign in with a local account instead**. This converts `qiaob` to a standalone local account without touching the profile or installed applications. Local SAM authority is restored and standard RDP credential handling works.

**Current workaround:** RDP via QIAO-AU → Bojiemini2 over home LAN, which bypasses the credential issue. Permanent fix (account conversion) is the next step.

### 8.3 Subnet Routing Boundary — pfSense, Not Windows

**What was considered:** Running Tailscale on the Windows hosts directly instead of on the pfSense VMs, and enabling IP routing on the hosts to pass traffic between the home LAN and lab subnets.

**Why it was rejected:** A Windows host with a leg on both `192.168.1.0/24` (management NIC) and `172.16.x.0/24` (internal vSwitch) with IP routing enabled creates a **home-to-lab bypass that completely circumvents pfSense**. Traffic would flow: home LAN → host NIC → Windows IP routing → lab subnet, never passing through pfSense firewall rules.

**Decision:** Tailscale terminates on pfSense at each site. pfSense is the single, clean boundary between home and lab. Windows IP routing stays permanently disabled on both hosts.

### 8.4 DHCP Service Startup Race Condition

**What happened:** Both DCs logged **Event 1059** on boot: DHCP server could not start because a required dependency was not yet ready. The issue appeared intermittently on slower VM boot sequences.

**Root cause:** Windows starts the DHCP Server service before `NTDS` (the AD DS database) and the TCP/IP stack are fully initialised.

**Fix:**
```
sc.exe config DHCPServer depend= NTDS/Tcpip/Afd
```

This adds an explicit service dependency chain, ensuring DHCP Server only starts after NTDS, TCP/IP, and the AFD (Windows Sockets) service are ready. Applied to both BAA-BIG-DC1 and BAA-SML-DC1 and documented in CHECKLIST.md.

### 8.5 RDP Credential Complexity

RDP authentication involves multiple overlapping subsystems that interact unexpectedly:

- **NLA (Network Level Authentication)** — authenticates before the remote session starts; can block connection attempts when credential format doesn't match
- **Microsoft account delegation** — removes local SAM authority (see 8.2)
- **Credential Manager** — can cache stale credentials and silently override what is typed
- **AllowManagementOS** — if left enabled, can create unexpected routing paths that make authentication behave differently depending on which interface traffic arrives on

**Approach:** Layer-by-layer elimination. Confirm routing first (can you reach the IP?), then TCP port (is RDP listening?), then service state (is TermService running?), then authentication (does the credential format match the account type?). Skipping layers leads to misdiagnosis.

---

## 9. Engineering Methodology

All work in this project was guided by a personal **8-step engineering methodology** applied consistently to each phase and each problem encountered:

| Step | Stage | What Happens |
|------|-------|-------------|
| 1 | **Requirements** | Define what needs to be achieved and why — scope, constraints, and success criteria |
| 2 | **Evaluate** | Research options, understand trade-offs, consider alternatives |
| 3 | **Decide** | Choose an approach; record the decision and reasoning |
| 4 | **Implement** | Execute the change in a controlled, documented way |
| 5 | **Verify** | Test against the success criteria defined in step 1 |
| 6 | **Troubleshoot** | If verification fails, diagnose layer by layer, log findings |
| 7 | **Log** | Commit decisions, commands, outcomes, and lessons to GitHub immediately |
| 8 | **Understand** | Extract the general principle from the specific problem for future use |

**Why this matters in practice:**

Steps 3 and 7 are the most important for building lasting knowledge. Documenting the *decision and its reasoning* (not just the command that was run) means future-self (or a future employer) understands *why* the design is the way it is, not just *what* was configured. Step 8 converts a one-off fix into a transferable skill.

---

## 10. Documentation Structure

### GitHub Repositories

| Repository | Visibility | Purpose |
|-----------|-----------|---------|
| `qbjsuper/tafe-virtual-server-lab` | Public | Lab build log — VMs, AD, cluster, NethServer |
| `bojieanzac-remote-access` | Private | Remote access design — this project |
| `troubleshooting` | Private | Flat-entry personal troubleshooting knowledge base |

### bojieanzac-remote-access Repo Structure

```
bojieanzac-remote-access/
├── README.md                        ← Project overview and topology
├── .gitignore                       ← Excludes *.key, *.pem, tskey-*, etc.
├── CHECKLIST.md                     ← Ordered build steps; start here to resume
├── docs/
│   ├── 01-topology.md               ← Full physical + logical topology tables
│   ├── 02-remote-access-design.md   ← Access design and architecture decisions
│   ├── 03-security-model.md         ← Security principles and non-negotiables
│   ├── 04-pfsense-subnet-router.md  ← pfSense Tailscale setup steps
│   └── 05-vision-and-roadmap.md     ← Current state and future direction
├── tailscale/
│   └── acl-policy.hujson            ← Paste into Tailscale Admin → Access Controls
└── scripts/
    ├── setup-subnet-router-lan.ps1  ← Run on QIAO-AU to advertise home LAN route
    └── verify-routing.ps1           ← Run on qlaptop to test all three subnets
```

### Troubleshooting Repo Design

The troubleshooting repo uses **flat entries with multi-tag classification** rather than a hierarchical folder taxonomy. The design decision:

- Real problems span multiple domains — a single RDP failure touches networking, authentication, and Windows service state simultaneously
- Hierarchical folders force a problem into one category, hiding its other dimensions
- Flat entries tagged with multiple labels (e.g., `rdp, credentials, windows, ms-account, hyper-v`) are searchable across all relevant dimensions without duplication

The repo is designed as a **career-spanning personal knowledge base** — each entry is written to be useful in five years, not just in the moment.

---

## 11. Future Direction

### Immediate (Outstanding)

- **Bojiemini2 RDP permanent fix** — convert `qiaob` to a local account via Settings → Accounts → Sign in with a local account instead; restores SAM authority, enables standard RDP
- **Bojie-Mini Tailscale tag cleanup** — appears in Tailscale admin as untagged under a personal Gmail account rather than within the project tag structure; needs re-tagging as `tag:lab-router`

### Near-Term

| Item | Notes |
|------|-------|
| UniFi VLAN creation | Lab / management / home LAN separation; deferred pending a managed switch for meaningful wired VLAN segmentation (pfSense already provides the meaningful isolation boundary) |
| NethServer first app | DokuWiki — internal documentation platform |
| NethServer second app | Nextcloud — file sharing and collaboration |
| BAA-CLUSTER1 storage redesign | Storage Replica on local disks not supported in a failover cluster; redesign to a supported clustered-storage architecture |
| Windows Firewall tightening | Scope RDP/WinRM further once VLAN IDs are stable |

### Medium-Term / Future

| Item | Notes |
|------|-------|
| 10GbE on Bojie-Mini | Two Intel X710 ports dark — cable one for iSCSI/heartbeat path |
| Physical / external DC | Would break the host/domain boot dependency and allow domain-joining the physical hosts safely |
| EDGE site | 172.16.70.0/24 referenced in NS8 CIDR planning |
| Samba file sharing | Planned in lab requirements, not yet built |
| Mail server | After NethServer platform is proven — complex (MX/SPF/DKIM) |
| Storage Replica POC | Server-to-server SR between two non-clustered servers |
| Backup and restore | Planned in lab requirements, not yet built |
| Claude agent build | Operationalise the 8-step engineering methodology as an automated workflow |

---

## 12. Appendix: Repository Structure

### File Reference

| File | Purpose |
|------|---------|
| `README.md` | Project overview, key decisions, build status table |
| `.gitignore` | Excludes `*.key`, `*.pem`, `*.pfx`, `*.env`, `*-secret*`, `tskey-*`, `credentials*`, `.DS_Store`, `Thumbs.db` |
| `CHECKLIST.md` | Ordered build checklist; tracks phase completion with `[x]` / `[ ]` items |
| `docs/01-topology.md` | Full physical and logical topology tables; last verified 2026-06-01 |
| `docs/02-remote-access-design.md` | Access design, subnet routing model, architecture decisions with reasoning |
| `docs/03-security-model.md` | Three principles (least privilege, separation of contexts, defense in depth) and non-negotiables |
| `docs/04-pfsense-subnet-router.md` | Step-by-step pfSense Tailscale package install, config, and route approval |
| `docs/05-vision-and-roadmap.md` | Current state table, near/medium/future roadmap, architecture decisions summary |
| `tailscale/acl-policy.hujson` | Complete Tailscale ACL policy; paste directly into Tailscale Admin → Access Controls |
| `scripts/setup-subnet-router-lan.ps1` | Run elevated on QIAO-AU; executes `tailscale up --advertise-routes=192.168.1.0/24` |
| `scripts/verify-routing.ps1` | Run on qlaptop after setup; tests RDP to both DCs and HTTPS to NethServer |

### Key IP Reference

| Resource | IP / Address | Port(s) |
|----------|-------------|---------|
| BAA-BIG-DC1 | 172.16.60.10 | 3389 (RDP), 5985/5986 (WinRM) |
| BAA-SML-DC1 | 172.16.50.10 | 3389 (RDP), 5985/5986 (WinRM) |
| BAA-BIG-PFS1 | 172.16.60.1 | 443 (web UI) |
| BAA-SML-PFS1 | 172.16.50.1 | 443 (web UI) |
| BAA-SML-NETH1 | 172.16.50.60 | 443 (NethServer dashboard) |
| PiKVM | 100.118.254.20 (Tailscale) | 443 (HTTPS only) |
| Bojiemini2 host | 192.168.1.252 | 3389 (RDP — pending account conversion fix) |
| Bojie-Mini host | 192.168.1.135 | 3389 (RDP) |
| QIAO-AU | 192.168.1.x | — |

---

*Document generated from `bojieanzac-remote-access` repo contents and build logs. All design decisions, implementation steps, scripts, and troubleshooting entries are committed to GitHub. This document represents the complete picture of the BOJIEANZAC-REMOTE-ACCESS project as of June 2026.*
