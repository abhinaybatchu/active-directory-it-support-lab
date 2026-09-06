# Phase 8 — DC02, Active Directory Replication & Domain Controller Redundancy

## 1. Objective

The objective of this phase was to introduce a second domain controller into the `abhinaylabs.internal` Active Directory environment and validate that core directory and DNS services could continue operating if the primary domain controller became unavailable.

The following tasks were completed:

- Built a second Windows Server virtual machine named `DC02`
- Configured static IPv4 networking
- Joined DC02 to the existing `abhinaylabs.internal` domain
- Installed Active Directory Domain Services (AD DS) and DNS Server
- Promoted DC02 as an additional domain controller
- Configured DC02 as a Global Catalog server
- Validated AD-integrated DNS
- Validated bidirectional Active Directory replication
- Tested replication using a temporary Active Directory object
- Configured CLIENT01 with redundant DNS servers
- Performed a controlled DC01 outage
- Verified CLIENT01 could locate and use DC02
- Restored DC01 and confirmed replication returned to a healthy state

---

## 2. Lab Environment

| System   | Role                                   | IPv4 Address     |
| -------- | -------------------------------------- | ---------------- |
| DC01     | Domain Controller, DNS, Global Catalog | `192.168.170.10` |
| DC02     | Domain Controller, DNS, Global Catalog | `192.168.170.11` |
| CLIENT01 | Domain-joined Windows 11 workstation   | `192.168.170.20` |

Domain:

```text
abhinaylabs.internal
```

NetBIOS domain name:

```text
ABHINAYLABS
```

VMware network:

```text
VMnet8 NAT
192.168.170.0/24
```

VMware NAT gateway:

```text
192.168.170.2
```

---

## 3. Why a Second Domain Controller Was Added

A single-domain-controller environment creates a single point of failure.

If DC01 were the only domain controller and became unavailable, domain clients could lose access to services such as:

- Active Directory authentication
- Domain controller discovery
- Active Directory-integrated DNS
- LDAP directory services
- Kerberos authentication
- Group Policy processing that requires a domain controller

Enterprise Active Directory environments therefore commonly deploy multiple domain controllers.

For this lab, DC02 provides redundancy for the `abhinaylabs.internal` domain and allows the environment to demonstrate Active Directory replication and domain-controller failover.

---

## 4. DC02 Virtual Machine Configuration

DC02 was created as a separate Windows Server 2025 virtual machine in VMware Workstation.

Configuration:

```text
Computer Name: DC02
Operating System: Windows Server 2025 Standard Evaluation
Installation: Desktop Experience
Processors: 2 vCPU
Memory: 4 GB
Virtual Disk: 60 GB
Network: VMnet8 NAT
```

The virtual machine was manually installed rather than using VMware Easy Install.

After Windows Server installation, the server was renamed:

```text
DC02
```

---

## 5. DC02 Static Network Configuration

DC02 was configured with a static IPv4 address.

```text
IPv4 Address: 192.168.170.11
Subnet Mask: 255.255.255.0
Default Gateway: 192.168.170.2
```

Before promotion, DC02 used DC01 as its DNS server:

```text
Preferred DNS: 192.168.170.10
```

This allowed DC02 to resolve the existing Active Directory domain and locate DC01.

Connectivity and DNS resolution were validated before joining the domain.

Examples:

```powershell
nslookup dc01.abhinaylabs.internal
```

```powershell
Test-NetConnection dc01.abhinaylabs.internal -Port 53
```

```powershell
Test-NetConnection dc01.abhinaylabs.internal -Port 88
```

```powershell
Test-NetConnection dc01.abhinaylabs.internal -Port 389
```

These tests validated connectivity to:

- TCP 53 — DNS
- TCP 88 — Kerberos
- TCP 389 — LDAP

---

## 6. Joining DC02 to the Existing Domain

DC02 was joined to the existing domain:

```text
abhinaylabs.internal
```

Domain Administrator credentials were used to authorize the domain join.

DC02 was then restarted.

After joining the domain, the server was promoted as an **additional domain controller for the existing domain**.

A new forest was not created.

---

## 7. Installing Active Directory Domain Services and DNS

The following server roles were installed on DC02:

- Active Directory Domain Services
- DNS Server

DC02 was then promoted into the existing `abhinaylabs.internal` domain.

During promotion:

```text
DNS Server: Enabled
Global Catalog: Enabled
Read Only Domain Controller: Disabled
```

A private Directory Services Restore Mode (DSRM) password was configured and was not included in project documentation or screenshots.

After promotion, DC02 restarted and became a domain controller.

---

## 8. Domain Controller Placement

Both domain controllers are located in the built-in:

```text
Domain Controllers
```

organizational unit.

The environment therefore contains:

```text
abhinaylabs.internal
└── Domain Controllers
    ├── DC01
    └── DC02
```

Domain controllers were intentionally not moved into:

```text
Corp\Servers
```

because the built-in Domain Controllers OU provides the appropriate location for domain-controller-specific Group Policy and administration.

---

## 9. DNS Configuration After Promotion

After DC02 was promoted, its DNS client configuration initially contained:

```text
192.168.170.10
127.0.0.1
```

The final lab configuration was changed to:

```text
Preferred DNS: 192.168.170.10
Alternate DNS: 192.168.170.11
```

This allows DC02 to query DC01 while also using its own DNS service when necessary.

After the change, DNS registration was refreshed:

```powershell
ipconfig /flushdns
```

```powershell
ipconfig /registerdns
```

```powershell
Restart-Service Netlogon
```

DNS resolution was then validated.

Example:

```powershell
nslookup dc01.abhinaylabs.internal
```

The Active Directory LDAP SRV records were also checked:

```powershell
nslookup -type=SRV _ldap._tcp.dc._msdcs.abhinaylabs.internal
```

Both DC01 and DC02 were returned as domain controllers.

---

## 10. AD-Integrated DNS

The following Active Directory-integrated DNS zones were present on DC02:

```text
_msdcs.abhinaylabs.internal
abhinaylabs.internal
```

Because the DNS zones are Active Directory-integrated, DNS information can replicate through Active Directory rather than relying on a traditional standalone primary/secondary DNS-zone design.

This allows both domain controllers to provide DNS services for the domain.

---

## 11. DNS Health Validation

DNS health was tested on both domain controllers using:

```cmd
dcdiag /test:dns /v
```

The important DNS tests passed on both DC01 and DC02.

The results included successful checks for:

```text
Authentication
Basic DNS
Forwarders
Delegation
Dynamic Update
Record Registration
```

This confirmed that both domain controllers were correctly participating in Active Directory DNS.

---

## 12. Active Directory Replication Validation

Active Directory replication was checked using:

```cmd
repadmin /showrepl
```

and:

```cmd
repadmin /replsummary
```

Healthy replication showed successful replication of the Active Directory naming contexts between DC01 and DC02.

The replication summary eventually showed:

```text
0 / 5 failures
0%
```

for both domain controllers.

This confirmed healthy bidirectional replication.

---

## 13. Core Domain Controller Health Validation

Additional targeted domain-controller tests were performed using:

```cmd
dcdiag /test:Advertising /test:Services /test:Replications /test:DNS
```

The tests passed on both DC01 and DC02.

These checks validated:

- Domain controller advertising
- Required Active Directory services
- Directory replication
- DNS functionality

Together with `repadmin`, these results provided stronger validation than relying on a single health command.

---

## 14. Active Directory Object Replication Test

A temporary Active Directory security group was created on DC01:

```powershell
New-ADGroup `
-Name "GG_Replication_Test" `
-SamAccountName "GG_Replication_Test" `
-GroupCategory Security `
-GroupScope Global `
-Path "OU=Groups,OU=Corp,DC=abhinaylabs,DC=internal"
```

The object was verified on DC01:

```powershell
Get-ADGroup "GG_Replication_Test"
```

The same object was then queried from DC02:

```powershell
Get-ADGroup "GG_Replication_Test"
```

DC02 successfully located the group.

This demonstrated:

```text
DC01
  ↓
Active Directory replication
  ↓
DC02
```

The temporary group was then removed from DC02:

```powershell
Remove-ADGroup "GG_Replication_Test"
```

After replication occurred, the object was no longer found on DC01.

This demonstrated replication in the reverse direction:

```text
DC02
  ↓
Active Directory replication
  ↓
DC01
```

The temporary test object was therefore removed from the final Active Directory environment.

---

## 15. CLIENT01 Redundant DNS Configuration

CLIENT01 was configured to use both Active Directory DNS servers.

```text
Preferred DNS: 192.168.170.10
Alternate DNS: 192.168.170.11
```

The DNS cache was cleared:

```cmd
ipconfig /flushdns
```

The configuration was checked using:

```cmd
ipconfig /all
```

DC02 name resolution was tested:

```cmd
nslookup dc02.abhinaylabs.internal
```

Expected address:

```text
192.168.170.11
```

CLIENT01 connectivity to DC02 was also validated.

```powershell
Test-NetConnection dc02.abhinaylabs.internal -Port 53
Test-NetConnection dc02.abhinaylabs.internal -Port 88
Test-NetConnection dc02.abhinaylabs.internal -Port 389
```

These tests validated access to DNS, Kerberos and LDAP services on DC02.

---

## 16. Controlled Domain Controller Failover Test

A controlled outage was performed to verify that the domain could continue operating when DC01 became unavailable.

Before the test:

```text
DC01      ON
DC02      ON
CLIENT01  ON
```

Replication health was confirmed before beginning the outage.

DC01 was then properly shut down from Windows.

The environment became:

```text
DC01      OFF
DC02      ON
CLIENT01  ON
```

CLIENT01's DNS cache was cleared:

```cmd
ipconfig /flushdns
```

Domain controller discovery was forced:

```cmd
nltest /dsgetdc:abhinaylabs.internal /force
```

CLIENT01 successfully located:

```text
DC02
```

This demonstrated that the workstation could discover the secondary domain controller while DC01 was unavailable.

---

## 17. Domain User Authentication During Failover

While DC01 remained unavailable, CLIENT01 was used to validate domain functionality through DC02.

The synthetic HR user was used:

```text
David Miller
ABHINAYLABS\dmiller
```

The session was validated using:

```cmd
whoami
```

Expected identity:

```text
abhinaylabs\dmiller
```

The logon server was checked:

```cmd
echo %LOGONSERVER%
```

DC02 was identified as the available logon server during the failover test.

Domain controller discovery was also validated:

```cmd
nltest /dsgetdc:abhinaylabs.internal /force
```

DC02 was returned as the available domain controller.

This provided evidence that the domain remained functional while DC01 was intentionally unavailable.

---

## 18. Restoring DC01

After completing the failover test, DC01 was powered back on.

The domain controllers were allowed time to reconnect and resume replication.

Replication health was checked again:

```cmd
repadmin /replsummary
```

The final healthy state returned to:

```text
0 failures
```

This demonstrated that the domain controllers could resume normal replication after the temporary DC01 outage.

---

# Troubleshooting

## 19. Replication Error 8524 After DC02 Promotion

During initial replication validation, `repadmin` reported:

```text
8524 (0x214c)
The DSA operation is unable to proceed because of a DNS lookup failure.
```

Replication initially showed a failure between DC01 and DC02.

Because Active Directory replication depends heavily on DNS, DNS was investigated before making major configuration changes.

### DNS Client Configuration

The DNS configuration was inspected using:

```powershell
Get-DnsClientServerAddress -AddressFamily IPv4
```

DC01 used:

```text
192.168.170.10
```

DC02 initially contained:

```text
192.168.170.10
127.0.0.1
```

DC02 was changed to:

```text
Preferred DNS: 192.168.170.10
Alternate DNS: 192.168.170.11
```

### DNS Resolution Tests

DC01 successfully resolved DC02:

```powershell
nslookup dc02.abhinaylabs.internal
```

Result:

```text
192.168.170.11
```

DC02 successfully resolved DC01:

```powershell
nslookup dc01.abhinaylabs.internal
```

Result:

```text
192.168.170.10
```

LDAP SRV records were checked:

```powershell
nslookup -type=SRV _ldap._tcp.dc._msdcs.abhinaylabs.internal
```

Both DC01 and DC02 were registered.

DNS health was then validated:

```cmd
dcdiag /test:dns /v
```

DNS tests passed on both servers.

After DNS registration and Netlogon were refreshed, replication was checked again:

```cmd
repadmin /showrepl
repadmin /replsummary
```

Replication returned to:

```text
0 failures
```

### Lesson Learned

Active Directory replication failures should not immediately lead to rebuilding or re-promoting a domain controller.

DNS should be one of the first areas investigated because Active Directory relies heavily on DNS service records and name resolution to locate domain controllers and replication partners.

---

## 20. DCDIAG SystemLog / DCOM Error

During additional health testing:

```cmd
dcdiag /q
```

reported a SystemLog failure associated with a DistributedCOM event.

The event indicated that DCOM could not communicate with:

```text
192.168.170.2
```

The event was inspected using:

```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; Id=10028} -MaxEvents 5 |
Format-List TimeCreated,ProviderName,Id,LevelDisplayName,Message
```

The result showed:

```text
Provider: Microsoft-Windows-DistributedCOM
Event ID: 10028
Destination: 192.168.170.2
```

`192.168.170.2` is the VMware VMnet8 NAT gateway, not DC01 or DC02.

Instead of changing Active Directory, DCOM or firewall configuration solely to eliminate the event, core domain-controller functionality was tested directly:

```cmd
dcdiag /test:Advertising /test:Services /test:Replications /test:DNS
```

All targeted tests passed.

Replication also remained healthy:

```cmd
repadmin /replsummary
```

### Conclusion

The DCOM event was determined to be unrelated to Active Directory replication between DC01 and DC02.

No unnecessary DCOM, firewall or VMware NAT changes were made.

### Lesson Learned

A health-check command can report errors that are present in the Windows event logs without those errors necessarily representing a failure of the Active Directory service being investigated.

Errors should therefore be interpreted in context and validated with targeted diagnostic tests before configuration changes are made.

---

## 21. Expected Replication Failures During the Controlled Outage

During the intentional DC01 shutdown, `repadmin /replsummary` reported replication failures including:

```text
8524
The DSA operation is unable to proceed because of a DNS lookup failure.
```

This occurred while DC01 was deliberately offline.

Because DC02 could not communicate with an offline replication partner, replication between the two domain controllers could not remain healthy during the outage.

This result was therefore treated differently from the earlier 8524 error that occurred while both domain controllers were online.

After DC01 was restored, replication was allowed to recover and was checked again.

The final replication state returned to:

```text
0 failures
```

### Lesson Learned

The context in which an error occurs matters.

A replication failure while both domain controllers should be available requires investigation. A replication failure caused by an intentionally powered-off replication partner is expected during a controlled failover test.

---

# Enterprise Relevance

## 22. Why Enterprises Use Multiple Domain Controllers

Multiple domain controllers improve Active Directory availability.

If one domain controller becomes unavailable, another domain controller can continue providing services such as:

- User authentication
- Kerberos authentication
- LDAP directory access
- Active Directory DNS
- Domain controller discovery
- Directory queries

Domain controllers also replicate Active Directory data so that directory objects and changes are available across multiple servers.

---

## 23. DNS and Active Directory

DNS is a critical dependency for Active Directory.

Domain clients use DNS records to locate services such as:

- Domain controllers
- Kerberos authentication
- LDAP
- Global Catalog servers

For example:

```text
_ldap._tcp.dc._msdcs.abhinaylabs.internal
```

contains service records that help clients locate domain controllers.

Incorrect DNS configuration can therefore cause symptoms that appear to be Active Directory failures even when the underlying directory services are functioning.

---

## 24. IT Support Relevance

An IT Support or Service Desk technician may encounter issues such as:

- User cannot sign into the domain
- Workstation cannot locate a domain controller
- Domain resources are unavailable
- DNS resolution fails
- Group Policy cannot locate domain services
- Authentication becomes slow or inconsistent
- A domain controller is unavailable

Useful troubleshooting commands demonstrated in this phase include:

```cmd
ipconfig /all
```

```cmd
ipconfig /flushdns
```

```cmd
nslookup dc02.abhinaylabs.internal
```

```cmd
nltest /dsgetdc:abhinaylabs.internal /force
```

```powershell
Test-NetConnection dc02.abhinaylabs.internal -Port 53
```

```powershell
Test-NetConnection dc02.abhinaylabs.internal -Port 88
```

```powershell
Test-NetConnection dc02.abhinaylabs.internal -Port 389
```

Understanding these dependencies helps support technicians distinguish between DNS, connectivity, authentication and directory-service problems.

---

## 25. Security / SOC Relevance

Active Directory is a major identity and authentication platform in Windows enterprise environments.

Security analysts should understand:

- Domain controller roles
- DNS dependencies
- Kerberos
- LDAP
- Active Directory replication
- Authentication infrastructure
- Domain controller availability

Domain controllers also generate important authentication and account-management events that can later be investigated through Windows Event Logs and SIEM platforms.

---

# Evidence

## Figure 10 — Domain Controllers

File:

[Open Screenshot](../02-screenshots/10-domain-controllers-dc01-dc02.png)

Description:

> DC01 and DC02 operating as domain controllers for the `abhinaylabs.internal` Active Directory domain.

---

## Figure 11 — Active Directory Replication Health

File:

[Open Screenshot](../02-screenshots/11-ad-replication-health.png)

Description:

> Active Directory replication health validated between DC01 and DC02 with no replication failures.

---

## Figure 12 — DC02 Domain Failover

File:

[Open Screenshot](../02-screenshots/12-dc02-domain-failover.png)

Description:

> CLIENT01 successfully locating and using DC02 while DC01 is unavailable, validating domain-controller redundancy.

---

# Phase 8 Result

Phase 8 successfully expanded the lab from a single-domain-controller environment to a redundant Active Directory architecture.

The final environment contains:

```text
                  abhinaylabs.internal
                          |
              +-----------+-----------+
              |                       |
            DC01                    DC02
      192.168.170.10          192.168.170.11
       AD DS + DNS             AD DS + DNS
      Global Catalog          Global Catalog
              |                       |
              +-----------+-----------+
                          |
                      CLIENT01
                   192.168.170.20
                          |
              DNS: DC01 + DC02
```

The phase demonstrated:

- Deployment of an additional domain controller
- Active Directory-integrated DNS
- Bidirectional directory replication
- Replication health validation
- DNS troubleshooting
- Domain controller service validation
- Client DNS redundancy
- Controlled domain-controller failover
- Restoration and replication recovery

The environment is now ready for the next phase of the project: **Help Desk Account Administration**.
