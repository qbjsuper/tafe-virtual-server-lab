# Troubleshooting record template

Copy this template into `logbook.md` every time you encounter a problem
during a verify step. One record per issue.

---

## BLANK TEMPLATE

```markdown
## [DATE] — [SHORT DESCRIPTION OF PROBLEM]

### Symptom
<!-- What exactly is wrong? Be specific.
     Include error codes, event IDs, command output. -->

### Hypothesis
<!-- What do you think is causing it?
     One hypothesis at a time. -->

### Test
<!-- What command or action tests that hypothesis? -->

### Result
<!-- What did the test show?
     Paste relevant output. -->

### Fix
<!-- What change resolved the issue?
     Include the exact command or config change. -->

### Root cause
<!-- Why did it actually fail?
     Not just what fixed it — why it broke in the first place. -->

### Prevention
<!-- What stops this happening again?
     Could be a config change, a dependency, a design decision. -->

### Re-verify
<!-- Run the original verify step again.
     Confirm it now passes. Paste evidence. -->
```

---

## FILLED EXAMPLE — DHCP event 1059 on BAA-BIG-DC1 (2026-06-01)

### Symptom
Bojiemini2 vEthernet (BOJIEANZAC-Internal) showing APIPA address
`169.254.214.238` after BIG site VMs powered on.
No `172.16.60.x` lease issued. Event ID 1059 in DHCP Server log:
`The DHCP service failed to see a directory server for authorization.`

### Hypothesis
DHCP service started before AD DS was ready to authorize it.
On a single-box DC + DHCP design, all services start simultaneously
and DHCP can win the race against AD DS.

### Test
```powershell
Restart-Service DHCPServer
Get-DhcpServerInDC
Get-Service NTDS, DNS, DHCPServer
nltest /dsgetdc:bojieanzac.com
Get-DnsClientServerAddress -AddressFamily IPv4
```

### Result
After restart: DHCP service running and authorized.
`Get-DhcpServerInDC` listed `baa-big-dc1.bojieanzac.com` at `172.16.60.10`.
All services Running. `nltest` located DC successfully.
DNS pointing at `172.16.60.10` (itself) — correct.

### Fix
```powershell
Restart-Service DHCPServer
```
Immediate fix for this boot.

### Root cause
Startup race condition. The DHCP service started before AD DS and DNS
had fully initialized. DHCP's mandatory authorization check ran against
a directory service that was not yet answering LDAP queries.
Nondeterministic — does not fail every boot, only when AD DS is slow
enough that DHCP wins the startup race.

### Prevention
```powershell
sc.exe config DHCPServer depend= NTDS/Tcpip/Afd
```
Applied to both BAA-BIG-DC1 and BAA-SML-DC1.
Windows Service Control Manager will not start DHCPServer until
NTDS (AD DS), Tcpip, and Afd are all running.

### Re-verify
```
ipconfig on Bojiemini2 confirmed:
vEthernet (BOJIEANZAC-Internal)
  IPv4 Address: 172.16.60.106
  Lease from:   172.16.60.10 (BAA-BIG-DC1)
  Gateway:      172.16.60.1 ✓
```