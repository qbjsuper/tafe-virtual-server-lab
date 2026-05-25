# Nested Hyper-V Cluster Checkpoint — 2026-05-25

## 1. Current Lab Context

This checkpoint records the work completed today for the nested Hyper-V failover cluster lab.

The lab is part of the `bojieanzac.com` TAFE virtual server / cluster environment.

```text
Domain:        bojieanzac.com
Cluster:       BAA-NEST-CL1
Cluster nodes: BAA-BIG-Nest1
               BAA-SML-Nest1
Storage host:  BAA-STOR1
Storage IP:    172.16.50.101
CSV path:      C:\ClusterStorage\Volume1
Test VM:       BAA-TEST-VM01
```

The main goal is to prove that a nested Hyper-V failover cluster can run a highly available VM from shared iSCSI/CSV storage and later live migrate that VM between the two nested Hyper-V nodes.

---

## 2. Nested Hyper-V Cluster Nodes

Two new nested Hyper-V nodes were used:

```text
BAA-BIG-Nest1
BAA-SML-Nest1
```

Purpose:

```text
Physical Hyper-V host
└── Nested Windows Server Hyper-V nodes
    └── Clustered nested test VM
```

Both nodes are domain-joined to:

```text
bojieanzac.com
```

Cluster name:

```text
BAA-NEST-CL1
```

Cluster node health was confirmed with:

```powershell
Get-ClusterNode
```

Confirmed result:

```text
Name           State  Type
----           -----  ----
BAA-BIG-Nest1  Up     Node
BAA-SML-Nest1  Up     Node
```

Result:

```text
Both cluster nodes are up and participating in the cluster.
```

---

## 3. Shared Storage and CSV

Storage server:

```text
BAA-STOR1
172.16.50.101
```

Storage design:

```text
BAA-STOR1 provides iSCSI storage.
Both BAA-BIG-Nest1 and BAA-SML-Nest1 connect to the iSCSI target.
The shared disk is added to the failover cluster.
The shared disk is converted to Cluster Shared Volume.
```

Current CSV path:

```text
C:\ClusterStorage\Volume1
```

CSV check command:

```powershell
Get-ClusterSharedVolume | Select-Object Name, State, OwnerNode
```

Confirmed result:

```text
Name            State   OwnerNode
----            -----   ---------
Cluster Disk 1  Online  BAA-BIG-Nest1
```

Interpretation:

```text
The Cluster Shared Volume is online and currently coordinated by BAA-BIG-Nest1.
```

---

## 4. iSCSI Connectivity

The iSCSI session was checked with:

```powershell
Get-IscsiSession
```

Confirmed important fields:

```text
IsConnected:   True
IsPersistent:  True
Target:        iqn.1991-05.com.microsoft:baa-stor1-baa-nest-csv-tgt01-target
```

TCP connectivity to the iSCSI target was checked with:

```powershell
Test-NetConnection 172.16.50.101 -Port 3260
```

Confirmed result:

```text
ComputerName     : 172.16.50.101
RemotePort       : 3260
InterfaceAlias   : vEthernet (BAA-NEST-EXT-SW)
SourceAddress    : 172.16.60.100
TcpTestSucceeded : True
```

Interpretation:

```text
BAA-BIG-Nest1 can reach BAA-STOR1 over iSCSI TCP port 3260.
Storage connectivity is currently working.
```

Note:

```text
The storage path from BAA-BIG-Nest1 to BAA-STOR1 crosses from the BIG-side network to the SML-side storage network.
This works, but may be slower than running heavy storage operations from BAA-SML-Nest1.
```

---

## 5. Test VM Creation

Test VM name:

```text
BAA-TEST-VM01
```

Storage location:

```text
C:\ClusterStorage\Volume1\VMs\BAA-TEST-VM01
```

Confirmed VM files:

```text
C:\ClusterStorage\Volume1\VMs\BAA-TEST-VM01
C:\ClusterStorage\Volume1\VMs\BAA-TEST-VM01\Virtual Hard Disks
C:\ClusterStorage\Volume1\VMs\BAA-TEST-VM01\Virtual Machines
C:\ClusterStorage\Volume1\VMs\BAA-TEST-VM01\Virtual Hard Disks\BAA-TEST-VM01.vhdx
C:\ClusterStorage\Volume1\VMs\BAA-TEST-VM01\Virtual Machines\905379D1-98EA-49C8-BA75-B6E84EA2DFC5.vmcx
C:\ClusterStorage\Volume1\VMs\BAA-TEST-VM01\Virtual Machines\905379D1-98EA-49C8-BA75-B6E84EA2DFC5.vmgs
C:\ClusterStorage\Volume1\VMs\BAA-TEST-VM01\Virtual Machines\905379D1-98EA-49C8-BA75-B6E84EA2DFC5.VMRS
```

Current VM design:

```text
Name:             BAA-TEST-VM01
Generation:       2
Memory:           2048 MB currently
Storage:          VHDX on CSV
Clustered:        Yes
Guest OS:         Ubuntu Server test VM
Purpose:          Clustered VM / live migration test
```

---

## 6. Clustered VM Role

The test VM was added as a clustered role with:

```powershell
Add-ClusterVirtualMachineRole -VMName "BAA-TEST-VM01"
```

The current cluster role check shows:

```powershell
Get-ClusterGroup -Name "BAA-TEST-VM01"
```

Current confirmed result:

```text
Name           OwnerNode      State
----           ---------      -----
BAA-TEST-VM01  BAA-BIG-Nest1  Offline
```

Clustered VM resources were checked with:

```powershell
Get-ClusterResource | Where-Object OwnerGroup -eq "BAA-TEST-VM01" | Format-Table Name, ResourceType, State, OwnerNode, OwnerGroup -AutoSize
```

Current confirmed result:

```text
Name                                         ResourceType                   State    OwnerNode      OwnerGroup
----                                         ------------                   -----    ---------      ----------
Virtual Machine BAA-TEST-VM01                Virtual Machine                Offline  BAA-BIG-Nest1  BAA-TEST-VM01
Virtual Machine Configuration BAA-TEST-VM01  Virtual Machine Configuration  Online   BAA-BIG-Nest1  BAA-TEST-VM01
```

Interpretation:

```text
The VM itself is powered off, so the Virtual Machine resource is Offline.
The VM configuration resource is Online, which is normal for a clustered VM that is currently off.
```

---

## 7. Offline Move vs Live Migration

A key troubleshooting point was discovered.

This command failed while the VM role was offline:

```powershell
Move-ClusterVirtualMachineRole -Name "BAA-TEST-VM01" -Node "BAA-SML-Nest1"
```

Error:

```text
The virtual machine resource is not in an appropriate state for this operation.
```

Reason:

```text
Move-ClusterVirtualMachineRole is better suited for moving an online/running clustered VM role, especially when using live migration.
```

For an offline VM role, this command worked:

```powershell
Move-ClusterGroup -Name "BAA-TEST-VM01" -Node "BAA-SML-Nest1"
```

Result:

```text
Offline clustered VM role movement works.
```

Important rule:

```text
Offline clustered role movement:
Use Move-ClusterGroup.

Running VM live migration:
Use Move-ClusterVirtualMachineRole with -MigrationType Live.
```

---

## 8. Ubuntu ISO Handling

Ubuntu ISO filename:

```text
ubuntu-26.04-live-server-amd64.iso
```

ISO location:

```text
C:\ClusterStorage\Volume1\ISO\ubuntu-26.04-live-server-amd64.iso
```

Reason for storing the ISO on CSV:

```text
Both cluster nodes can access the same path.
This avoids migration or startup problems caused by local-only ISO paths.
```

The ISO was attached from the CSV path.

Important next cleanup step:

```powershell
Set-VMDvdDrive -ComputerName "BAA-BIG-Nest1" -VMName "BAA-TEST-VM01" -Path $null
```

Purpose:

```text
Detach the Ubuntu installer ISO after installation.
Prevent the VM from booting back into the installer.
Remove an unnecessary dependency before live migration.
```

---

## 9. Ubuntu Installation and Storage Warning

The Ubuntu installer started successfully.

The installer log showed normal disk activity such as:

```text
/dev/sda
sgdisk
udevadm
partitioning
SUCCESS configuring partition
```

However, during installation, the VM temporarily entered:

```text
State:  PausedCritical
Status: Disk(s) encountered critical IO errors
```

Interpretation:

```text
Hyper-V temporarily paused the VM because the VHDX storage path encountered critical I/O delay or error.
```

Likely cause:

```text
Nested Hyper-V + CSV + iSCSI + routed storage path created a temporary storage stall during heavy disk writes.
```

The VM later recovered to:

```text
State:    Running
CPUUsage: 5
Status:   Operating normally
```

Conclusion:

```text
This was a real warning, but not a fatal failure.
The test VM recovered and later became usable.
```

Recommendation:

```text
Avoid heavy install/update work from the node farther away from storage.
For heavy disk operations, prefer BAA-SML-Nest1 because BAA-STOR1 is in the SML-side network.
```

---

## 10. Ubuntu Test VM Status

The test VM was confirmed as up and running after the Ubuntu install process.

Later, the VM was shut down cleanly/offline.

Current Hyper-V state check:

```powershell
Get-VM -ComputerName "BAA-BIG-Nest1" -Name "BAA-TEST-VM01" | Select-Object Name, State, CPUUsage, MemoryAssigned, Uptime, Status
```

Current confirmed result:

```text
Name           : BAA-TEST-VM01
State          : Off
CPUUsage       : 0
MemoryAssigned : 0
Uptime         : 00:00:00
Status         : Operating normally
```

Interpretation:

```text
The VM is currently powered off and healthy.
```

---

## 11. Domain Join Decision

Question considered:

```text
Should BAA-TEST-VM01 join bojieanzac.com now?
```

Decision:

```text
Not yet.
```

Reason:

```text
The current purpose is to test Hyper-V cluster, CSV, VM startup, and live migration.
Joining Ubuntu to AD would add extra variables such as DNS, Kerberos, SSSD, time sync, and domain login.
```

Recommended sequence:

```text
1. Confirm Ubuntu boots normally.
2. Confirm network/time sync works.
3. Detach ISO.
4. Reboot once from disk.
5. Test live migration.
6. Confirm VM stays running after migration.
7. Join to bojieanzac.com later as a separate milestone.
```

---

## 12. Ubuntu Time Sync Plan

Recommended time sync tool:

```text
chrony
```

Basic Ubuntu commands:

```bash
sudo apt update
sudo apt install -y chrony
sudo systemctl enable --now chrony
systemctl status chrony
chronyc sources -v
chronyc tracking
```

For later AD/domain integration, point Ubuntu time sync to a domain controller:

```text
server BAA-BIG-DC1.bojieanzac.com iburst
```

or use the DC IP address if DNS is not ready:

```text
server <DC-IP-ADDRESS> iburst
```

Reason:

```text
Kerberos and domain authentication depend on accurate time sync.
```

---

## 13. Nested VM Network / External Switch

The Ubuntu test VM initially did not have external network access.

Design requirement for nested Hyper-V internet access:

```text
Physical Hyper-V host
└── Nest1 VM network adapter with MAC spoofing enabled
    └── Nested Hyper-V External vSwitch
        └── BAA-TEST-VM01
```

Nested switch name involved:

```text
BAA-NEST-EXT-SW
```

Important live migration rule:

```text
The same vSwitch name must exist on both cluster nodes.
```

So `BAA-NEST-EXT-SW` should exist on both:

```text
BAA-BIG-Nest1
BAA-SML-Nest1
```

Outer physical Hyper-V host requirement:

```powershell
Set-VMNetworkAdapter -VMName "BAA-BIG-Nest1" -MacAddressSpoofing On
Set-VMNetworkAdapter -VMName "BAA-SML-Nest1" -MacAddressSpoofing On
```

Purpose:

```text
Allow traffic from nested VMs behind the Nest1 Hyper-V hosts to pass through the outer Hyper-V switch.
```

---

## 14. SR-IOV Decision

For the nested External vSwitch, SR-IOV should remain disabled.

Correct setting:

```text
External network:                                                Yes
Allow management operating system to share this network adapter: Yes
Enable single-root I/O virtualization (SR-IOV):                  No
```

Reason:

```text
SR-IOV is not needed for this nested Hyper-V cluster lab.
It depends on physical NIC, firmware, driver, and host support.
It can reduce flexibility and is not required for live migration testing.
```

---

## 15. Current Confirmed State

Latest confirmed cluster state:

```text
Cluster nodes:
BAA-BIG-Nest1    Up
BAA-SML-Nest1    Up

Cluster groups:
Available Storage    Offline
BAA-TEST-VM01        Offline
Cluster Group        Online

CSV:
Cluster Disk 1       Online
OwnerNode:           BAA-BIG-Nest1

Test VM:
BAA-TEST-VM01        Off
Status:              Operating normally

Test VM cluster resources:
Virtual Machine resource:                Offline
Virtual Machine Configuration resource:  Online

iSCSI:
Connected:           True
Port 3260:           True
```

Interpretation:

```text
The cluster is healthy.
The CSV is online.
The test VM is cleanly powered off.
The VM configuration is managed by the cluster.
Storage connectivity is working.
```

`Available Storage` being offline is acceptable because the shared disk has already been converted to CSV.

The important resource is:

```text
Cluster Disk 1    Online
```

---

## 16. Next Recommended Steps

### Step 1 — Detach the Ubuntu ISO

```powershell
Set-VMDvdDrive -ComputerName "BAA-BIG-Nest1" -VMName "BAA-TEST-VM01" -Path $null
```

This detaches the Ubuntu installer ISO from the VM DVD drive.

Verify:

```powershell
Get-VMDvdDrive -ComputerName "BAA-BIG-Nest1" -VMName "BAA-TEST-VM01" | Select-Object VMName, Path
```

Expected result:

```text
Path should be blank.
```

### Step 2 — Start VM from disk

```powershell
Start-ClusterGroup -Name "BAA-TEST-VM01"
```

This starts the clustered VM role through Failover Clustering.

Verify:

```powershell
Get-ClusterGroup -Name "BAA-TEST-VM01"
Get-VM -ComputerName "BAA-BIG-Nest1" -Name "BAA-TEST-VM01" | Select-Object Name, State, CPUUsage, MemoryAssigned, Uptime, Status
```

Expected result:

```text
Cluster role: Online
VM state:     Running
Status:       Operating normally
```

### Step 3 — Test live migration

After the VM is running normally:

```powershell
Move-ClusterVirtualMachineRole -Name "BAA-TEST-VM01" -Node "BAA-SML-Nest1" -MigrationType Live
```

This performs the real live migration test from `BAA-BIG-Nest1` to `BAA-SML-Nest1`.

Verify after migration:

```powershell
Get-ClusterGroup -Name "BAA-TEST-VM01"
Get-VM -ComputerName "BAA-SML-Nest1" -Name "BAA-TEST-VM01" | Select-Object Name, State, Status, Uptime
```

Expected result:

```text
OwnerNode changes to BAA-SML-Nest1.
VM stays Running.
Uptime should continue.
```

---

## 17. Current Milestone Status

```text
Cluster foundation:        Working
Both Nest1 nodes:          Working
Domain membership:         Working
BAA-STOR1 iSCSI:           Working
CSV:                       Working
Clustered test VM role:    Working
Offline role move:         Working
Ubuntu boot/install:       Mostly working, with one temporary storage I/O warning
Nested external switch:    Being configured
Live migration:            Next major test
Domain join for Ubuntu:    Later milestone
```

