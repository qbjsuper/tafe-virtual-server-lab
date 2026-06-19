Goal

Configure BAA-NEST-CL1 so that BAA-TEST-VM01 automatically fails over to the
surviving node when a host reboots or fails — with zero manual intervention.
Prove it with a real unplanned node reboot.


Starting symptom

Failover Cluster Manager showed Automatic Refresh failing with:

The WS-Management service cannot process the request. The computed response
packet size (547138) exceeds the maximum envelope size that is allowed (512000).

Both cluster nodes (BAA-BIG-Nest1 and BAA-SML-Nest1) reported this error.


Diagnosis — what the WinRM error actually was

Initial hypothesis: a WinRM configuration problem. Disproved immediately.

Get-ClusterSharedVolume showed:

Name           State   Node
Cluster Disk 1 Offline BAA-BIG-Nest1

Get-ClusterGroup showed:

Name              OwnerNode     State
Available Storage BAA-BIG-Nest1 Offline
BAA-TEST-VM01     BAA-BIG-Nest1 Pending
Cluster Group     BAA-BIG-Nest1 Online

Get-ChildItem C:\ClusterStorage\Volume1 hung indefinitely.

The WinRM envelope error was a symptom, not the cause. FCM's refresh was
serialising all the failed-resource state into a response that exceeded the
512KB ceiling. Fix the cluster state, and the envelope error goes away on its own.


Root cause — the morning reboot

(Get-CimInstance Win32_OperatingSystem).LastBootUpTime
→ Friday, 19 June 2026 11:12:21 AM

BAA-BIG-Nest1 rebooted at 11:12. Windows never auto-onlines iSCSI or shared
disks after a reboot — by design, to protect shared storage in a cluster. The
iSCSI session reconnected (path was healthy), but the disk stayed Offline at the
OS layer. The cluster came up, found no storage, and could not start the VM.

Verified the storage path was healthy (not the cause):

powershellGet-IscsiSession | Format-Table IsConnected, TargetNodeAddress -AutoSize
# IsConnected: True  Target: iqn.1991-05.com.microsoft:baa-stor1-baa-nest-csv-tgt01-target

Test-NetConnection 172.16.50.101 -Port 3260
# TcpTestSucceeded: True  SourceAddress: 172.16.60.100

powershellGet-Disk | Select-Object Number, FriendlyName, OperationalStatus, HealthStatus, IsOffline, BusType
# Disk 1 (iSCSI): HealthStatus Healthy, IsOffline True

Disk was healthy but offline at the OS layer. Not corruption — a state problem.
The cluster had hit its failover threshold and stopped retrying.

Event log showed recurring 1069 errors approximately every 18 minutes —
the cluster retry loop firing against an offline CSV:

19/06/2026 11:27 AM  1069  Cluster resource 'Virtual Machine Configuration BAA-TEST-VM01' failed
19/06/2026 11:45 AM  1069  (same)
19/06/2026 12:03 PM  1069  (same)
19/06/2026 12:20 PM  1069  (same)
19/06/2026 12:38 PM  1069  (same)

The CSV recovered on its own during a pause in the session (the cluster's retry
loop eventually succeeded). But the VM role did not auto-start — which surfaced
the real problem.


Real goal identified — HA does not work

With the VM sitting Offline after a node reboot with no automatic recovery, the
requirements were reset:

REQUIREMENTS
Goal: A 2-node cluster that automatically keeps BAA-TEST-VM01 available and
      relocates it to the surviving node when a host fails or reboots — with
      zero manual intervention.
Success: Reboot BAA-BIG-Nest1 → BAA-TEST-VM01 auto-starts on BAA-SML-Nest1,
         Online and reachable, no manual Start-ClusterGroup.


Three independent faults identified

Fault 1 — No quorum witness (critical — blocks all automatic failover)

powershellGet-ClusterQuorum | Format-List *
# QuorumResource :
# QuorumType     : Majority

QuorumResource was blank. The cluster was running Node Majority with 2 nodes
= 2 votes. When BAA-BIG-Nest1 rebooted, BAA-SML-Nest1 held 1 of 2 votes — no
majority. A cluster without majority stops its resources rather than risk
split-brain. Automatic failover was structurally impossible.

This is the root of roots — the reason the VM did not fail over this morning.

Fault 2 — VM possible-owners list was empty

powershellGet-ClusterOwnerNode -Group "BAA-TEST-VM01"
# OwnerNodes: {}

The VM role had no permitted nodes to run on. Even with quorum fixed, it had
nowhere to fail over to.

Both VM resources were also checked:

powershellGet-ClusterResource | Where-Object OwnerGroup -eq "BAA-TEST-VM01" | ForEach-Object {
  [PSCustomObject]@{ Resource=$_.Name; Owners=(Get-ClusterOwnerNode -Resource $_.Name).OwnerNodes -join ', ' }
}
# Virtual Machine BAA-TEST-VM01               : {}
# Virtual Machine Configuration BAA-TEST-VM01 : {}

Empty at the resource level too.

Fault 3 — Cluster Network 3 partitioned

powershellGet-ClusterNetwork | Format-Table Name, State, Role, Address -AutoSize
# Cluster Network 3  Partitioned  Cluster  (no address)

The BAA-NEST-VM-IntenalSW (Internal-type Hyper-V switch) was registered as a
cluster heartbeat network. An Internal switch only connects VMs and the host OS
on the same physical machine — it cannot carry traffic between BAA-BIG-Nest1
(on Bojiemini2) and BAA-SML-Nest1 (on Bojie-Mini). It was permanently
partitioned and had no address. The cluster was trying to use a heartbeat path
that physically cannot exist.

Additionally, the cluster NIC DHCP lease renewal was found to have caused the
1126/1129 partition events at 6:50 PM:

6:50:25  BAA-BIG-Nest1 NIC re-leased 172.16.60.100
6:50:36  BAA-SML-Nest1 NIC re-leased 172.16.50.100
6:50:54  Event 1126 (interface unreachable) + 1129 (network partitioned)


Fixes implemented

Fix 1 — File Share Witness for BAA-NEST-CL1

Existing witness on BAA-BIG-DC1 (C:\ClusterWitness) belongs to BAA-CLUSTER1:

powershellGet-SmbShareAccess -Name ClusterWitness
# BOJIEANZAC\BAA-CLUSTER1$  Full
# (GUID subfolder present — live, in-use witness)

Two clusters must never share one witness folder. A new dedicated share was
created for BAA-NEST-CL1.

On BAA-BIG-DC1:

powershellNew-Item -Path C:\ClusterWitness-NESTCL1 -ItemType Directory

New-SmbShare -Name "ClusterWitness-NESTCL1" `
             -Path "C:\ClusterWitness-NESTCL1" `
             -FullAccess "BOJIEANZAC\BAA-NEST-CL1$"

$acl = Get-Acl C:\ClusterWitness-NESTCL1
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "BOJIEANZAC\BAA-NEST-CL1$","FullControl","ContainerInherit,ObjectInherit","None","Allow")
$acl.SetAccessRule($rule)
Set-Acl C:\ClusterWitness-NESTCL1 $acl

Verified share access:

BOJIEANZAC\BAA-NEST-CL1$  Full  (no Everyone)

On BAA-BIG-Nest1:

powershellSet-ClusterQuorum -NodeAndFileShareMajority "\\BAA-BIG-DC1\ClusterWitness-NESTCL1"
# Cluster: BAA-NEST-CL1  QuorumResource: File Share Witness

Get-ClusterResource | Where-Object ResourceType -eq "File Share Witness" | Format-Table Name, State, OwnerNode
# File Share Witness  Online  BAA-BIG-Nest1

Result: 3 votes total. Losing one node → 2/3 = majority → automatic failover
can proceed.

Fix 2 — VM possible-owners set on group and resources

powershellSet-ClusterOwnerNode -Group "BAA-TEST-VM01" -Owners BAA-BIG-Nest1,BAA-SML-Nest1

Get-ClusterOwnerNode -Group "BAA-TEST-VM01"
# OwnerNodes: {BAA-BIG-Nest1, BAA-SML-Nest1}

Resource-level owners verified clean (both nodes, both resources).

Fix 3 — Cluster Network 3 role set to None

powershell$net = Get-ClusterNetwork "Cluster Network 3"
$net.Role = 0
$net.Role
# None

Get-ClusterNetwork | Format-Table Name, State, Role, Address -AutoSize
# Cluster Network 3  Partitioned  None

Cluster stops trying to use the Internal switch for heartbeat. Heartbeat
runs on Networks 1 and 2 only (172.16.60.0/24 and 172.16.50.0/24 — the
BIG↔SML IPsec tunnel).

Side effect: after the NIC static conversion (below), Network 3 healed to
Up/None — the static addresses eliminated the DHCP churn that was keeping
it partitioned.


Addressing hardening — static IPs on cluster infra VMs

Why: cluster NIC DHCP churn caused the 6:50 PM partition events. Reserved
DHCP guarantees the address but not the lease-renewal window — during a
renewal, the NIC briefly re-initialises and the cluster can lose the node's
network identity. BAA-STOR1 additionally has a cross-site DHCP dependency
(DHCP server is on BIG, STOR1 is on SML — if the IPsec tunnel is down when
STOR1 boots, it cannot get its address, falls back to APIPA, and both cluster
nodes lose the CSV). Static removes both risks.

BAA-SML-Nest1 (Hyper-V console, ncpa.cpl):

Interface:  vEthernet (BAA-NEST-EXT-SW)
IP:         172.16.50.100
Mask:       255.255.255.0
Gateway:    172.16.50.1
DNS:        172.16.50.10 / 172.16.60.10

Verified:

powershellGet-NetIPInterface -InterfaceAlias "vEthernet (BAA-NEST-EXT-SW)" -AddressFamily IPv4
# Dhcp: Disabled

BAA-STOR1 (Hyper-V console, ncpa.cpl):

Interface:  Ethernet
IP:         172.16.50.101
Mask:       255.255.255.0
Gateway:    172.16.50.1
DNS:        172.16.50.10 / 172.16.60.10

Verified:

powershellGet-NetIPInterface -InterfaceAlias "Ethernet" -AddressFamily IPv4
# Dhcp: Disabled

BAA-BIG-Nest1 (Hyper-V console, ncpa.cpl):

Interface:  vEthernet (BAA-NEST-EXT-SW)
IP:         172.16.60.100
Mask:       255.255.255.0
Gateway:    172.16.60.1
DNS:        172.16.60.10 / 172.16.50.10

Verified:

powershellGet-NetIPInterface -InterfaceAlias "vEthernet (BAA-NEST-EXT-SW)" -AddressFamily IPv4
# Dhcp: Disabled

BAA-TEST-VM01 (nested test VM) — left on pure DHCP, no reservation.
Required — it live-migrates between BIG (172.16.60.0/24) and SML (172.16.50.0/24)
and must re-request an address from whichever site it lands on. The systemd
timer (networkctl reconfigure eth0 every 30s) handles the per-site address
acquisition. A static IP would break migration.


DHCP scope cleanup

Context: reservations were removed before static config was applied — correct
order is static first, then remove reservation. Static was applied urgently to
prevent address loss on lease expiry.

Exclusion ranges added to keep infra addresses outside the dynamic pool:

SML scope (on BAA-SML-DC1 DHCP):

Exclusion: 172.16.50.100 – 172.16.50.101
Dynamic range remains: 172.16.50.102 – 172.16.50.199

BIG scope (on BAA-BIG-DC1 DHCP):

Exclusion: 172.16.60.100 – 172.16.60.100
Dynamic range remains: 172.16.60.101 – 172.16.60.199

Result: infra static addresses sit below/outside the dynamic pool. DHCP no
longer touches BAA-SML-Nest1, BAA-STOR1, or BAA-BIG-Nest1.


Pre-test cluster health check

powershellGet-ClusterNode | Format-Table Name, State -AutoSize
# BAA-BIG-Nest1  Up
# BAA-SML-Nest1  Up

Get-ClusterNetwork | Format-Table Name, State, Role -AutoSize
# Cluster Network 1  Up  ClusterAndClient
# Cluster Network 2  Up  ClusterAndClient
# Cluster Network 3  Up  None             ← healed after static NIC change

Get-ClusterSharedVolume | Format-Table Name, State, Node -AutoSize
# Cluster Disk 1  Online

Get-ClusterGroup | Format-Table Name, OwnerNode, State -AutoSize
# BAA-TEST-VM01  BAA-BIG-Nest1  Online

Get-VM -ComputerName BAA-BIG-Nest1 -Name BAA-TEST-VM01 | Format-Table Name, State, Status -AutoSize
# BAA-TEST-VM01  Running  Operating normally

All green. Baseline confirmed.


Failover test

Pre-test state: BAA-TEST-VM01 Online, Running on BAA-BIG-Nest1.

Action — on BAA-BIG-Nest1:

powershellRestart-Computer -Force

Watch loop on BAA-SML-Nest1 (no manual intervention):

powershellwhile ($true) {
  Get-ClusterGroup | Format-Table Name, OwnerNode, State -AutoSize
  Start-Sleep 5
}

Observed sequence:

BAA-TEST-VM01  BAA-SML-Nest1  Pending   ← cluster detected node loss
BAA-TEST-VM01  BAA-SML-Nest1  Pending   ← starting on survivor
BAA-TEST-VM01  BAA-SML-Nest1  Online    ← automatic, zero commands issued

No Start-ClusterGroup was run. The cluster detected BIG-Nest1 going down,
held quorum via the File Share Witness (2 of 3 votes), and automatically
started the VM on the surviving node.

Post-reboot final state (BIG-Nest1 rejoined):

powershellGet-ClusterNode | Format-Table Name, State -AutoSize
# BAA-BIG-Nest1  Up
# BAA-SML-Nest1  Up

Get-ClusterGroup | Format-Table Name, OwnerNode, State -AutoSize
# BAA-TEST-VM01  BAA-SML-Nest1  Online
# Cluster Group  BAA-SML-Nest1  Online

Get-ClusterSharedVolume | Format-Table Name, State, Node -AutoSize
# Cluster Disk 1  Online


Verify

Positive test: reboot BAA-BIG-Nest1 → VM auto-starts on BAA-SML-Nest1
Result:        PASS

Negative test: no manual Start-ClusterGroup issued
Result:        PASS (Pending → Online with zero intervention)


Troubleshoot records

#SymptomRoot causeFixPrevented by1WinRM envelope error 547KB > 512KBFCM serialising failed-resource state from offline CSVFix the cluster state (not WinRM)Keeping cluster healthy2CSV offline after rebootWindows never auto-onlines iSCSI/shared disks by design; cluster had no quorum to online itCSV recovered via cluster retry loop; quorum fixed to prevent recurrenceFSW gives majority on node loss3VM did not fail over after rebootNo quorum witness — 2-node cluster could not hold majority with 1 node downFSW on BAA-BIG-DC1 (dedicated folder, separate from BAA-CLUSTER1)Always configure a witness on 2-node clusters4VM had nowhere to fail over toPossible-owners list empty at group and resource levelSet-ClusterOwnerNode — both nodes on group and resourcesCheck owner list when adding clustered roles5Cluster Network 3 partitionedInternal-type Hyper-V switch cannot carry traffic between separate physical hostsSet role to NoneNever configure an Internal switch as a cluster heartbeat network61126/1129 partition events at 6:50DHCP lease renewal on cluster NICs caused brief NIC re-init; cluster lost node identityStatic IPs on all cluster infra VMsStatic addressing on all cluster-facing NICs


Final verified state

BAA-BIG-Nest1          Up
BAA-SML-Nest1          Up
BAA-TEST-VM01          Online  OwnerNode: BAA-SML-Nest1  (post-failover)
Cluster Group          Online  OwnerNode: BAA-SML-Nest1
Cluster Disk 1         Online
File Share Witness     Online
Cluster Network 1      Up      ClusterAndClient  172.16.60.0
Cluster Network 2      Up      ClusterAndClient  172.16.50.0
Cluster Network 3      Up      None
BAA-SML-Nest1 NIC      Static  172.16.50.100
BAA-STOR1 NIC          Static  172.16.50.101
BAA-BIG-Nest1 NIC      Static  172.16.60.100
BAA-TEST-VM01          DHCP    (no reservation — per-site address on migration)


Key findings for future reference


Windows never auto-onlines iSCSI/shared disks after reboot. By design —
to protect shared storage. Post-reboot CSV offline is expected; the cluster
onlines it, but only if quorum is held.
A 2-node cluster without a witness cannot hold majority when one node is
lost. 1 of 2 votes is not a majority. Automatic failover is structurally
impossible. Every 2-node cluster needs a third vote — FSW, Cloud Witness,
or Disk Witness.
Two clusters on the same DC must have separate, dedicated witness folders.
Pointing two clusters at one witness folder breaks quorum for both.
An Internal-type Hyper-V switch cannot carry traffic between two separate
physical hosts. Never register it as a cluster heartbeat network.
DHCP-reserved cluster NICs carry a lease-renewal timing risk. The
reservation guarantees the address; it does not eliminate the re-init window
during lease operations. Static is the correct end-state for cluster nodes
and storage servers.
BAA-STOR1 has a cross-site DHCP dependency. Its DHCP server (BAA-BIG-DC1)
is on the BIG site; STOR1 is on SML. If the IPsec tunnel is down when STOR1
boots, it cannot get its address → APIPA → both nodes lose the CSV. Static
removes this dependency entirely.
BAA-BIG-Nest1's iSCSI path to BAA-STOR1 crosses the BIG↔SML IPsec tunnel.
If the tunnel drops, BIG-Nest1 loses storage. Heavy disk operations should
prefer BAA-SML-Nest1 (same-site as BAA-STOR1).
BAA-TEST-VM01 must stay on pure DHCP. It migrates between two different
subnets. The systemd timer (networkctl reconfigure eth0 every 30s) handles
per-site address acquisition. A static IP breaks migration.



Understand

Concept: Windows Failover Cluster Quorum — voting model.

A Windows failover cluster requires a mathematical majority of votes to run
resources. This prevents split-brain: if two sides of a partitioned cluster
each tried to run the same VM, you get data corruption. By requiring majority,
only one side can ever act.

A File Share Witness holds one vote, stored as a file on a network share.
When a node goes down, the surviving node reaches the witness share, claims
that vote, achieves 2 of 3 — majority — and failover proceeds. The witness
does not store cluster state or make decisions; it contributes one vote.

This applies beyond this lab to any HA cluster design. 2-node clusters always
need a witness. Even large clusters use witness design to handle even-number
node loss scenarios where natural majority cannot be held.