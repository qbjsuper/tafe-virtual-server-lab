# Lab Logbook

## 2026-05-11

Started the TAFE virtual server lab documentation repository.

Confirmed the current lab topology:

| Site | Hyper-V Host | Internal Network | Gateway | Server |
|---|---|---|---|---|
| Big Site | Big Hyper-V host | 172.16.60.0/24 | pfSense | Windows Server |
| Small Site | Small Hyper-V host | 172.16.50.0/24 | pfSense | Windows Server |

Current plan:

- use pfSense as the gateway for each site
- configure one Windows Server at each site
- build one Active Directory domain: bojieanzac.com
- use the Big Site server as the first domain controller
- use the Small Site server as the second domain controller
- document all build steps and testing evidence

## 2026-05-11

Decision: rebuild the current lab VMs from start.

Reason:

The lab is still early, and only four VMs are required at this stage. Rebuilding gives cleaner VM names, cleaner screenshots, cleaner VHDX/storage names, and easier documentation.

Final core VM names:

| VM Name | Site | Role | Network |
|---|---|---|---|
| BAA-BIG-PFS1 | Big Site | pfSense gateway | 172.16.60.0/24 |
| BAA-BIG-DC1 | Big Site | Windows Server domain controller | 172.16.60.0/24 |
| BAA-SML-PFS1 | Small Site | pfSense gateway | 172.16.50.0/24 |
| BAA-SML-DC1 | Small Site | Windows Server domain controller | 172.16.50.0/24 |

Next action:

Recreate the four VMs with the final naming standard before starting Active Directory configuration.

## 2026-05-11

Foundation VM rebuild completed.

The four foundation VMs are now up and running:

| VM Name | Site | Role | Network |
|---|---|---|---|
| BAA-BIG-PFS1 | Big Site | pfSense gateway | 172.16.60.0/24 |
| BAA-BIG-DC1 | Big Site | Windows Server domain controller candidate | 172.16.60.0/24 |
| BAA-SML-PFS1 | Small Site | pfSense gateway | 172.16.50.0/24 |
| BAA-SML-DC1 | Small Site | Windows Server domain controller candidate | 172.16.50.0/24 |

Next steps:

- configure pfSense LAN interfaces
- configure static IP addresses on both Windows Servers
- test local gateway connectivity
- configure inter-site routing/VPN
- start Active Directory only after network connectivity is confirmed

## 2026-05-14

Completed the first stage of the multi-site Active Directory foundation.

### Current lab status

The four foundation VMs are running:

| VM Name | Site | Role | Network |
|---|---|---|---|
| BAA-BIG-PFS1 | Big Site | pfSense gateway | 172.16.60.0/24 |
| BAA-BIG-DC1 | Big Site | First domain controller | 172.16.60.0/24 |
| BAA-SML-PFS1 | Small Site | pfSense gateway | 172.16.50.0/24 |
| BAA-SML-DC1 | Small Site | Second domain controller | 172.16.50.0/24 |

### Big Site domain controller

BAA-BIG-DC1 has been configured as the first domain controller for:

bojieanzac.com

Roles configured:

- Active Directory Domain Services
- DNS Server
- DHCP Server

Confirmed checks:

- AD DS service is running
- DNS service is running
- Netlogon service is running
- Kerberos KDC service is running
- SYSVOL and NETLOGON shares are present
- internal DNS resolution works
- AD SRV records resolve correctly
- external DNS forwarding works
- DHCP server is authorised in Active Directory
- DHCP scope BAA-BIG-LAN is active

Big Site DHCP scope:

| Setting | Value |
|---|---|
| Scope name | BAA-BIG-LAN |
| Network | 172.16.60.0/24 |
| Range | 172.16.60.100 - 172.16.60.199 |
| Gateway | 172.16.60.1 |
| DNS server | 172.16.60.10 |
| DNS domain | bojieanzac.com |

### Site-to-site pfSense connection

Configured IPsec site-to-site VPN between:

| Side | pfSense | LAN |
|---|---|---|
| Big Site | BAA-BIG-PFS1 | 172.16.60.0/24 |
| Small Site | BAA-SML-PFS1 | 172.16.50.0/24 |

Ping test across the tunnel was successful.

Result:

172.16.50.0/24 can reach 172.16.60.10.

This confirms that the Small Site network can reach the Big Site domain controller through the IPsec VPN.

### Small Site domain controller

BAA-SML-DC1 was renamed correctly before promotion.

BAA-SML-DC1 was joined to the existing domain and promoted as the second domain controller for:

bojieanzac.com

Roles configured:

- Active Directory Domain Services
- DNS Server
- Global Catalog

Confirmed checks:

- BAA-SML-DC1 can resolve bojieanzac.com
- BAA-SML-DC1 can resolve BAA-BIG-DC1.bojieanzac.com
- BAA-SML-DC1 can resolve BAA-SML-DC1.bojieanzac.com
- SYSVOL share is present
- AD replication summary shows 0 failures

Replication check:

repadmin /replsummary showed 0 failures between BAA-BIG-DC1 and BAA-SML-DC1.

### Notes

dcdiag /q showed DFSREvent and SystemLog warnings on BAA-SML-DC1.

These appear to be recent event log warnings after promotion and SYSVOL sharing. AD replication itself is healthy because repadmin /replsummary shows 0 failures.

The next actions are:

- create proper AD sites and subnets
- map 172.16.60.0/24 to BAA-BIG-SITE
- map 172.16.50.0/24 to BAA-SML-SITE
- configure DHCP on BAA-SML-DC1
- verify client DHCP and domain join

# Checkpoint — BAA-CLUSTER1 Stretched Cluster and Storage Replica Preparation

Date: 2026-05-19

## Objective

Build a two-node stretched Windows Failover Cluster across two routed subnets in the bojieanzac.com lab, using File Share Witness for quorum and preparing local D: and L: volumes for a future Storage Replica partnership.

## Cluster identity

- Cluster name: BAA-CLUSTER1
- Domain: bojieanzac.com
- Node 1: BAA-BIG-HA1
- Node 2: BAA-SML-HA1
- BIG subnet: 172.16.60.0/24
- SMALL subnet: 172.16.50.0/24
- BIG cluster IP: 172.16.60.110
- SMALL cluster IP: 172.16.50.110
- Witness: \\BAA-BIG-DC1\ClusterWitness

## Storage preparation completed

Each cluster node was prepared with local storage:

- D: volume for data
- L: volume for log
- File system: ReFS

During disk preparation, Windows had assigned or reserved the D: drive letter incorrectly. The DVD drive was moved to Z:, and a leftover ghost partition using D: was removed. After that, Disk 1 was initialized and formatted as D:, and Disk 2 was initialized and formatted as L:.

## Network validation completed

Before creating the cluster, inter-site connectivity was tested across the pfSense IPsec tunnel.

Validated items:

- ICMP between nodes
- DNS name resolution
- TCP 445 / SMB between nodes
- Access to BAA-BIG-DC1 over TCP 445

This confirmed that the routed/IPsec path was good enough for cluster creation.

## Cluster validation completed

Cluster validation was run against:

- BAA-BIG-HA1
- BAA-SML-HA1

Selected validation categories:

- Network
- System Configuration
- Inventory

Storage validation was intentionally not used as a traditional shared-storage test because the design target is Storage Replica, not shared SAN/iSCSI storage.

Result:

- Validation completed with warnings
- No major blocking failure identified
- Warnings were expected because the design is multi-subnet and currently has only one main network path

## Cluster creation completed

The cluster was created with:

- Cluster name: BAA-CLUSTER1
- Nodes: BAA-BIG-HA1 and BAA-SML-HA1
- Static addresses: 172.16.60.110 and 172.16.50.110
- No automatic storage ingestion

The use of -NoStorage was intentional because this is not a traditional shared-disk cluster.

## Quorum configured

A File Share Witness was created on BAA-BIG-DC1:

- Path: C:\ClusterWitness
- Share path: \\BAA-BIG-DC1\ClusterWitness
- Cluster object: BAA-CLUSTER1$

The cluster quorum was configured as Node and File Share Majority.

This provides a third vote for the two-node cluster and reduces split-brain risk.

## Site topology configured

Cluster fault domains were created:

- BAA-BIG-SITE
- BAA-SML-SITE

Nodes were assigned to their corresponding sites:

- BAA-BIG-HA1 → BAA-BIG-SITE
- BAA-SML-HA1 → BAA-SML-SITE

This gives the cluster site awareness for the stretched design.

## Current storage interpretation

Local disks may have been added to the cluster inventory using:

Get-ClusterAvailableDisk -All | Add-ClusterDisk

However, this does not complete Storage Replica.

Current storage state:

- Local D: and L: volumes exist on both nodes
- Traditional shared storage is not the design goal
- Storage Replica partnership still needs to be created and validated

## Current project state

Completed:

- Domain membership
- Inter-site network connectivity
- pfSense/IPsec path validation
- Failover cluster feature installation
- Storage Replica feature installation
- Cluster validation
- Multi-subnet cluster creation
- File Share Witness quorum
- Fault-domain/site awareness
- Local D: and L: volume preparation

Not completed yet:

- Storage Replica topology test
- Storage Replica partnership creation
- Replication health validation
- Planned failover test
- Unplanned failover behaviour test
- Final documentation of active/passive storage ownership

## Key caution

Do not describe the lab as a completed Storage Replica cluster yet.

The accurate status is:

The stretched cluster foundation is complete, and the environment is prepared for Storage Replica. The Storage Replica partnership is the next major build step.

# Checkpoint — Storage Replica Investigation Result

The two-node multi-subnet failover cluster `BAA-CLUSTER1` was successfully built with nodes `BAA-BIG-HA1` and `BAA-SML-HA1`. The cluster uses a File Share Witness hosted at `\\BAA-BIG-DC1\ClusterWitness`.

Local ReFS volumes were prepared on both nodes:
- D: Data, approximately 40 GB
- L: Log, approximately 10 GB

The real Windows file-system view was verified with `Win32_LogicalDisk` and `Get-PSDrive`. Each node showed only one usable D: and one usable L: volume, and write tests to both volumes succeeded.

Cluster disk resource checks showed that no disk resources were currently added to the cluster. `Get-ClusterAvailableDisk -All` returned no available disks.

A Storage Replica topology test was attempted from `BAA-BIG-HA1` to `BAA-SML-HA1`. The test failed because both servers are members of `BAA-CLUSTER1`, and the D: data volume is not a clustered disk.

Conclusion:
The cluster foundation is working, and the local D:/L: disk layout is clean. However, Storage Replica does not support replicating non-clustered local disks on nodes that are already part of a failover cluster. The planned asymmetric local-disk stretched cluster design is therefore not supported in this form.

Next decision:
Keep `BAA-CLUSTER1` as the failover cluster proof, and either build a separate server-to-server Storage Replica proof-of-concept or redesign the cluster storage layer using a supported clustered-storage architecture.

# Update the DHCP dependency of both DCs.