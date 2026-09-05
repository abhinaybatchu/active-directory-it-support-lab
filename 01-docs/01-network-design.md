# Phase 1 — Active Directory Lab Network Design

## 1. Objective

The objective of this phase was to design and validate the network foundation for the Abhinay Labs Active Directory IT Support Home Lab.

The environment was built on VMware Workstation using an isolated private NAT network. The final architecture contains two Windows Server domain controllers and one Windows 11 domain workstation.

The network design provides:

- Private communication between lab virtual machines
- Internet access through VMware NAT when required
- Static IPv4 addressing for infrastructure systems
- Active Directory-integrated DNS
- DNS redundancy through two domain controllers
- Domain authentication and domain-controller discovery
- Active Directory replication
- Group Policy communication
- SMB file-share connectivity
- A predictable environment for IT Support troubleshooting

The network was intentionally kept private and was not exposed directly to the public Internet.

---

## 2. Fictional Lab Environment

This project represents a fictional organization named:

```text
Abhinay Labs
```

Active Directory domain:

```text
abhinaylabs.internal
```

NetBIOS domain:

```text
ABHINAYLABS
```

All users, systems, organizational data, credentials, and business scenarios used in this project are synthetic and exist solely for the home-lab environment.

---

## 3. Final Network Architecture

The final Active Directory environment consists of three virtual machines:

| System   | Operating System                 | Role                                           | IPv4 Address     |
| -------- | -------------------------------- | ---------------------------------------------- | ---------------- |
| DC01     | Windows Server 2025              | Domain Controller, AD DS, DNS, file-share host | `192.168.170.10` |
| DC02     | Windows Server 2025              | Additional Domain Controller, AD DS, DNS       | `192.168.170.11` |
| CLIENT01 | Windows 11 Enterprise Evaluation | Domain-joined workstation                      | `192.168.170.20` |

The environment was initially built using DC01 and CLIENT01. DC02 was added later to provide:

- Active Directory replication
- DNS redundancy
- Additional domain-controller availability
- Domain authentication redundancy
- Controlled failover testing

The completed architecture is therefore:

```text
                         Internet
                            |
                            |
                     VMware NAT Gateway
                       192.168.170.2
                            |
                            |
              VMnet8 — 192.168.170.0/24
                            |
            +---------------+---------------+
            |                               |
            |                               |
          DC01                            DC02
     192.168.170.10                  192.168.170.11
     Windows Server                  Windows Server
            |                               |
            | AD DS / DNS                   | AD DS / DNS
            |                               |
            +---------- AD Replication -----+
                            |
                            |
                         CLIENT01
                     192.168.170.20
                       Windows 11
                            |
                            |
                    Domain Workstation
```

---

## 4. VMware Network

The lab uses the existing VMware Workstation:

```text
VMnet8
```

network in:

```text
NAT mode
```

The network configuration is:

| Setting        | Value                               |
| -------------- | ----------------------------------- |
| VMware Network | `VMnet8`                            |
| Network Type   | NAT                                 |
| Subnet         | `192.168.170.0/24`                  |
| Subnet Mask    | `255.255.255.0`                     |
| NAT Gateway    | `192.168.170.2`                     |
| DHCP Range     | `192.168.170.128 – 192.168.170.254` |

The infrastructure addresses selected for the lab are outside the VMware DHCP allocation range.

This prevents VMware DHCP from automatically assigning those addresses to another virtual machine.

---

## 5. Why NAT Was Used

VMware NAT provides a useful balance between connectivity and isolation for this home lab.

The virtual machines can:

- Communicate with each other
- Reach the host through the VMware virtual network where appropriate
- Reach the Internet through VMware NAT when required
- Download updates and supported Windows components
- Perform DNS forwarding through the Active Directory DNS infrastructure

At the same time, the virtual machines are not intentionally exposed as directly reachable services on the physical network or public Internet.

This is appropriate for a private Active Directory training environment.

---

## 6. Static IPv4 Addressing

Infrastructure systems in the final environment use manually assigned IPv4 addresses.

The addressing plan is:

| Device   | IPv4 Address     | Prefix | Default Gateway |
| -------- | ---------------- | ------ | --------------- |
| DC01     | `192.168.170.10` | `/24`  | `192.168.170.2` |
| DC02     | `192.168.170.11` | `/24`  | `192.168.170.2` |
| CLIENT01 | `192.168.170.20` | `/24`  | `192.168.170.2` |

Equivalent subnet mask:

```text
255.255.255.0
```

The selected addresses are outside the VMware DHCP pool:

```text
192.168.170.128 – 192.168.170.254
```

This provides predictable addresses for Active Directory and DNS services.

---

## 7. Why Static Addresses Matter

Infrastructure servers should have predictable network identities.

If a domain controller's address changed unexpectedly, clients and other servers could experience problems locating services such as:

- DNS
- Kerberos
- LDAP
- SMB
- SYSVOL
- NETLOGON
- Group Policy
- Active Directory replication

For this lab:

```text
DC01 = 192.168.170.10
DC02 = 192.168.170.11
```

remain predictable DNS and Active Directory infrastructure endpoints.

CLIENT01 also uses a static address in this controlled lab so that troubleshooting evidence remains predictable.

In a production enterprise, ordinary workstations commonly receive addresses through DHCP rather than being manually configured. CLIENT01's static address is a deliberate home-lab design choice rather than a claim that enterprise endpoints normally require static addressing.

---

# DNS Design

## 8. Why DNS Is Critical to Active Directory

Active Directory depends heavily on DNS.

Domain systems use DNS not only to resolve hostnames such as:

```text
dc01.abhinaylabs.internal
```

but also to locate Active Directory services through DNS service records.

Examples include locating:

- Domain controllers
- LDAP services
- Kerberos services
- Global Catalog services

A domain client therefore should use the organization's Active Directory-aware DNS infrastructure rather than an unrelated external DNS resolver as its primary domain DNS service.

---

## 9. Final Domain Controller DNS Design

The final environment contains two Active Directory-integrated DNS servers:

```text
DC01
192.168.170.10
```

and:

```text
DC02
192.168.170.11
```

The final domain-controller DNS client design is:

### DC01

```text
Preferred DNS: 192.168.170.11
Alternate DNS: 192.168.170.10
```

### DC02

```text
Preferred DNS: 192.168.170.10
Alternate DNS: 192.168.170.11
```

This allows each domain controller to use the partner domain controller as its preferred DNS server while retaining its own DNS service as an alternate.

Both domain controllers host Active Directory-integrated DNS for:

```text
abhinaylabs.internal
```

---

## 10. Final CLIENT01 DNS Design

CLIENT01 uses the Active Directory DNS servers hosted by the domain controllers:

```text
Preferred DNS: 192.168.170.10
Alternate DNS: 192.168.170.11
```

The relationship is:

```text
CLIENT01
192.168.170.20
      |
      +---- Preferred DNS ----> DC01
      |                         192.168.170.10
      |
      +---- Alternate DNS ----> DC02
                                192.168.170.11
```

This provides CLIENT01 with redundant access to the domain DNS infrastructure.

---

## 11. Why the VMware Gateway Is Not the Final Client DNS Server

VMware VMnet8 provides the NAT gateway:

```text
192.168.170.2
```

The gateway is used for routing traffic outside the lab subnet.

However, CLIENT01's final DNS configuration does not use the VMware gateway as its domain DNS server.

Instead:

```text
CLIENT01
   ↓
DC01 / DC02
   ↓
Active Directory-integrated DNS
```

is used.

This is important because Active Directory publishes DNS records that domain clients use to discover domain services.

Examples include SRV records associated with:

```text
_ldap
_kerberos
_tcp
_msdcs
```

Using the domain DNS infrastructure allows CLIENT01 to locate the services required for domain operations.

---

## 12. DNS Redundancy

The addition of DC02 changed the lab from a single-DNS-server design to a redundant DNS design.

The final model is:

```text
                abhinaylabs.internal
                         |
             +-----------+-----------+
             |                       |
           DC01                    DC02
      DNS: 192.168.170.10     DNS: 192.168.170.11
             |                       |
             +-----------+-----------+
                         |
                      CLIENT01
```

If one domain controller is temporarily unavailable, the client has another Active Directory DNS server configured.

This does not make every possible Active Directory dependency automatically fault tolerant, but it removes the original single-DNS-server dependency and supports the lab's domain-controller redundancy testing.

---

# Active Directory Network Services

## 13. Important Services and Ports

Several network services are important to this Active Directory environment.

| Port        | Protocol | Service                  | Lab Relevance                                      |
| ----------- | -------- | ------------------------ | -------------------------------------------------- |
| 53          | TCP/UDP  | DNS                      | Name resolution and AD service discovery           |
| 88          | TCP/UDP  | Kerberos                 | Domain authentication                              |
| 135         | TCP      | RPC Endpoint Mapper      | Windows/AD RPC communication                       |
| 389         | TCP/UDP  | LDAP                     | Active Directory directory access                  |
| 445         | TCP      | SMB                      | SYSVOL, NETLOGON and departmental file shares      |
| 464         | TCP/UDP  | Kerberos password change | Domain password operations                         |
| 636         | TCP      | LDAPS                    | LDAP over TLS when configured                      |
| 3268        | TCP      | Global Catalog           | Forest-wide directory searches                     |
| 3269        | TCP      | Global Catalog over TLS  | Secure Global Catalog communication                |
| Dynamic RPC | TCP      | RPC services             | Active Directory and Windows management operations |

Not every port listed above was individually configured or tested during Phase 1.

Later phases validated the services required for the workflows being implemented.

---

## 14. Domain Service Discovery

Active Directory clients use DNS records to discover domain services.

For example, LDAP domain-controller service records can be queried using:

```powershell
Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.abhinaylabs.internal
```

In the completed redundant environment, DNS service discovery can identify both domain controllers.

This connects DNS directly to Active Directory availability:

```text
CLIENT01
   ↓
DNS query
   ↓
AD SRV records
   ↓
Domain Controller discovery
   ↓
DC01 / DC02
```

---

# Domain Controller Architecture

## 15. DC01

DC01 was the first server built in the environment.

Final identity:

```text
Hostname: DC01
IPv4:    192.168.170.10
Domain:  abhinaylabs.internal
```

Major roles used in the project include:

- Active Directory Domain Services
- DNS
- Domain authentication
- Group Policy infrastructure
- SYSVOL
- NETLOGON
- Active Directory administration
- Security event investigation
- Departmental SMB file-share hosting for the lab

The departmental shares were hosted on DC01 to keep the home-lab footprint manageable.

In a production enterprise, file services would commonly be separated from domain controllers according to the organization's architecture and security requirements.

---

## 16. DC02

DC02 was later introduced as an additional domain controller.

Final identity:

```text
Hostname: DC02
IPv4:    192.168.170.11
Domain:  abhinaylabs.internal
```

Major roles include:

- Active Directory Domain Services
- DNS
- Global Catalog
- Active Directory replication
- Additional domain authentication capability
- DNS redundancy
- Controlled domain-controller failover support

DC02 is a writable additional domain controller rather than a Read-Only Domain Controller.

---

## 17. Active Directory Replication

DC01 and DC02 replicate Active Directory information.

Conceptually:

```text
              DC01
                ⇅
      Active Directory Replication
                ⇅
              DC02
```

Replication allows both domain controllers to maintain domain directory information required for Active Directory operations.

Replication health was validated later in the project using tools such as:

```cmd
repadmin /replsummary
```

and:

```cmd
repadmin /showrepl
```

as well as targeted:

```cmd
dcdiag
```

testing.

The detailed replication implementation and troubleshooting are documented separately in:

```text
01-docs/08-dc02-replication-redundancy.md
```

---

# CLIENT01 Architecture

## 18. CLIENT01

CLIENT01 is the Windows 11 workstation used for end-user and Help Desk testing.

Final identity:

```text
Hostname: CLIENT01
IPv4:    192.168.170.20
Domain:  abhinaylabs.internal
```

DNS:

```text
Preferred DNS: 192.168.170.10
Alternate DNS: 192.168.170.11
```

CLIENT01 was used throughout the project for:

- Domain joining
- Domain-user authentication
- Group Policy processing
- Departmental share access
- Mapped-drive testing
- RSAT administration
- Delegated Help Desk operations
- Account-lockout testing
- Windows Security Event investigation
- Employee onboarding validation
- Employee offboarding validation

---

# Traffic Flow

## 19. Domain Authentication Flow

A simplified authentication flow is:

```text
Domain User
    |
    v
CLIENT01
    |
    | DNS / domain discovery
    v
DC01 or DC02
    |
    | Active Directory / Kerberos
    v
Domain authentication
```

DNS must function correctly before many Active Directory operations can function reliably.

---

## 20. Group Policy Flow

A simplified Group Policy relationship is:

```text
Active Directory / GPO
        |
        v
DC01 / DC02
        |
        | DNS + SMB + domain services
        v
CLIENT01
        |
        v
Applied user/computer policy
```

The project later validated Group Policy using commands including:

```cmd
gpupdate /force
```

```cmd
gpresult /r
```

and:

```cmd
gpresult /r /scope computer
```

```cmd
gpresult /r /scope user
```

---

## 21. Departmental File-Share Flow

The lab also uses SMB departmental resources hosted on DC01.

Example:

```text
David Miller
ABHINAYLABS\dmiller
        |
        v
GG_HR_Users
        |
        v
DL_HR_Share_RW
        |
        v
SMB + NTFS permissions
        |
        v
\\DC01\HR
```

Network connectivity, DNS, authentication, group membership, share permissions, and NTFS permissions all contribute to successful access.

---

# Network Validation

## 22. Basic IP Validation

Useful Windows commands for inspecting network configuration include:

```cmd
ipconfig
```

and:

```cmd
ipconfig /all
```

These can show:

- IPv4 address
- Subnet mask
- Default gateway
- DNS servers
- Network adapter information
- DNS suffix information

---

## 23. DNS Validation

DNS resolution can be tested using:

```cmd
nslookup dc01.abhinaylabs.internal
```

and:

```cmd
nslookup dc02.abhinaylabs.internal
```

PowerShell can also be used:

```powershell
Resolve-DnsName dc01.abhinaylabs.internal
```

```powershell
Resolve-DnsName dc02.abhinaylabs.internal
```

Active Directory service records can be inspected using:

```powershell
Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.abhinaylabs.internal
```

---

## 24. Service Connectivity Validation

PowerShell can test whether a required TCP service is reachable.

Examples:

```powershell
Test-NetConnection dc01.abhinaylabs.internal -Port 53
Test-NetConnection dc01.abhinaylabs.internal -Port 88
Test-NetConnection dc01.abhinaylabs.internal -Port 389
Test-NetConnection dc01.abhinaylabs.internal -Port 445
```

Equivalent tests can be performed against DC02 where appropriate:

```powershell
Test-NetConnection dc02.abhinaylabs.internal -Port 53
Test-NetConnection dc02.abhinaylabs.internal -Port 88
Test-NetConnection dc02.abhinaylabs.internal -Port 389
```

A successful TCP test helps confirm network reachability to that service, but it does not by itself prove that the complete application or Active Directory workflow is healthy.

---

## 25. Domain Controller Discovery

Domain-controller discovery can be tested with:

```cmd
nltest /dsgetdc:abhinaylabs.internal
```

This helps verify that a domain client can locate a domain controller for the Active Directory domain.

---

## 26. Replication Validation

With multiple domain controllers, replication should also be validated.

Useful commands include:

```cmd
repadmin /replsummary
```

and:

```cmd
repadmin /showrepl
```

These commands were used later in the project to identify and validate Active Directory replication state.

---

# Troubleshooting Model

## 27. Recommended Troubleshooting Order

When a domain workstation cannot authenticate, locate a domain controller, receive Group Policy, or access a domain resource, troubleshooting should proceed systematically.

A useful sequence is:

```text
1. Check local IP configuration
        ↓
2. Check subnet and default gateway
        ↓
3. Check configured DNS servers
        ↓
4. Test DNS name resolution
        ↓
5. Test AD service connectivity
        ↓
6. Test domain-controller discovery
        ↓
7. Verify authentication/account state
        ↓
8. Verify Group Policy or resource-specific configuration
        ↓
9. Review relevant logs
```

This is preferable to immediately changing DNS, resetting accounts, recreating GPOs, or rebuilding domain configuration.

---

## 28. Common Network Problems

### Incorrect DNS Server

A domain workstation using an inappropriate DNS resolver may fail to locate Active Directory services correctly.

Check:

```cmd
ipconfig /all
```

Expected CLIENT01 DNS:

```text
192.168.170.10
192.168.170.11
```

---

### Wrong IPv4 Configuration

An incorrect subnet, gateway, or IP address can prevent communication.

Expected network:

```text
192.168.170.0/24
```

Gateway:

```text
192.168.170.2
```

---

### DNS Works but an AD Service Does Not

Name resolution alone does not prove that a required service is reachable.

Use:

```powershell
Test-NetConnection dc01.abhinaylabs.internal -Port 88
Test-NetConnection dc01.abhinaylabs.internal -Port 389
Test-NetConnection dc01.abhinaylabs.internal -Port 445
```

to narrow the problem.

---

### Domain Controller Is Reachable but Replication Fails

With multiple domain controllers, basic IP connectivity does not guarantee healthy Active Directory replication.

Validate:

```cmd
repadmin /replsummary
```

and:

```cmd
repadmin /showrepl
```

Then investigate the specific failing naming context or dependency rather than assuming the entire domain is unavailable.

---

# Enterprise Relevance

## 29. IT Support Relevance

IT Support and Service Desk technicians frequently troubleshoot problems involving:

- Incorrect IP configuration
- DNS failures
- Domain connectivity
- Domain sign-in problems
- Group Policy failures
- Mapped-drive problems
- SMB resource access
- Workstation domain membership
- Domain-controller reachability

Understanding the relationship:

```text
IP
↓
DNS
↓
Domain discovery
↓
Authentication
↓
Policy / resource access
```

helps a technician troubleshoot systematically rather than guessing.

---

## 30. NOC Relevance

A NOC analyst may monitor or troubleshoot:

- Server availability
- DNS availability
- Network reachability
- Latency
- Routing
- Service ports
- Infrastructure health
- Connectivity between systems

The same basic network-validation skills used in this lab are therefore relevant to entry-level NOC work.

---

## 31. Security Analyst / SOC Relevance

Security analysts need to understand normal infrastructure communication before identifying abnormal activity.

Relevant examples include:

- DNS queries
- Kerberos authentication
- LDAP communication
- SMB access
- Failed authentication
- Domain-controller communication
- Unexpected service exposure
- Network segmentation
- Source and destination systems

The Active Directory authentication investigation later in this project relies on this network foundation.

---

## 32. IAM Relevance

IAM depends on infrastructure services that allow users and systems to locate and communicate with identity platforms.

In this lab:

```text
CLIENT01
   ↓
DNS
   ↓
Domain Controller
   ↓
Active Directory
   ↓
Authentication / Authorization
```

A user account can be correctly configured in Active Directory and still experience authentication problems if the underlying DNS or network path is broken.

---

# Security Considerations

## 33. Private Lab Network

The Active Directory lab remains on the VMware private NAT network.

No requirement exists to expose:

- RDP
- LDAP
- SMB
- DNS
- Domain controllers

directly to the public Internet.

This reduces unnecessary exposure of the training environment.

---

## 34. Synthetic Information

The project uses:

- Fictional organization information
- Synthetic employee identities
- Private RFC1918 addresses
- Lab-only hostnames
- Lab-only domain information

Passwords, recovery credentials, product keys, VMware encryption information, and other secrets are not included in the public repository.

---

# Evidence

## 35. Figure 1 — VMware Active Directory Lab Network

File:

```text
02-screenshots/01-vmware-ad-lab-network.png
```

Description:

> VMware Workstation VMnet8 NAT configuration used as the private network foundation for the Active Directory lab. The environment uses the `192.168.170.0/24` subnet with VMware NAT gateway `192.168.170.2` and a DHCP range beginning at `192.168.170.128`, leaving the selected infrastructure addresses outside the automatic allocation pool.

The screenshot captures the underlying VMware network configuration established at the beginning of the project.

DC02 was added during a later phase, so the screenshot represents the network foundation rather than the complete final three-VM architecture.

---

# Final Network Design

## 36. Completed Architecture

The completed network design is:

```text
                           Internet
                              |
                              v
                    VMware NAT Gateway
                      192.168.170.2
                              |
                              v
                  VMnet8 192.168.170.0/24
                              |
              +---------------+---------------+
              |                               |
              v                               v
            DC01                            DC02
       192.168.170.10                  192.168.170.11
       AD DS + DNS                     AD DS + DNS
       File Shares                     Global Catalog
              |                               |
              +-------- Replication ----------+
                              |
                              |
                              v
                           CLIENT01
                       192.168.170.20
                    Windows 11 Workstation
                              |
                +-------------+-------------+
                |             |             |
                v             v             v
          Authentication     GPO       SMB Resources
```

Final addressing:

```text
Network:     192.168.170.0/24
Gateway:     192.168.170.2

DC01:        192.168.170.10
DC02:        192.168.170.11
CLIENT01:    192.168.170.20
```

Final CLIENT01 DNS:

```text
Preferred:   192.168.170.10
Alternate:   192.168.170.11
```

Domain:

```text
abhinaylabs.internal
```

NetBIOS:

```text
ABHINAYLABS
```

---

# Key Lessons Learned

1. Active Directory depends heavily on DNS.
2. Domain clients should use the Active Directory DNS infrastructure for domain service discovery.
3. Domain controllers should use predictable network addresses.
4. Static addressing is useful for lab infrastructure, while enterprise workstations commonly use DHCP.
5. A default gateway provides routing outside the local subnet, while DNS resolves names to addresses.
6. DNS resolution and network connectivity should be validated separately.
7. A successful ping does not prove that a required application service is working.
8. `Test-NetConnection` can validate connectivity to specific TCP services.
9. DNS SRV records help clients locate Active Directory services.
10. A second domain controller provides additional AD and DNS availability and enables replication/failover testing.
11. Multiple domain controllers require replication-health monitoring in addition to ordinary network testing.
12. Troubleshooting should progress from basic network configuration toward higher-level domain services rather than changing infrastructure randomly.
13. A private NAT network is appropriate for this home lab because the domain infrastructure does not need direct public exposure.

---

# Interview Explanation

A concise interview explanation of the network design is:

> I built my Active Directory home lab on a private VMware NAT network using the 192.168.170.0/24 subnet. I assigned predictable addresses to two Windows Server domain controllers and a Windows 11 domain workstation. DC01 uses 192.168.170.10, DC02 uses .11, and CLIENT01 uses .20. Both domain controllers run Active Directory-integrated DNS, and the client uses DC01 and DC02 as its preferred and alternate DNS servers rather than an external resolver. I added DC02 later in the project to provide AD replication, DNS redundancy and controlled failover testing. I validated the environment using tools such as ipconfig, nslookup, Resolve-DnsName, Test-NetConnection, nltest, repadmin and dcdiag.

---

# Phase Result

The final network architecture provides the foundation required for the complete Active Directory IT Support lab:

```text
Private VMware NAT network
        ↓
Static infrastructure addressing
        ↓
DC01 + DC02
        ↓
AD DS + redundant DNS
        ↓
Active Directory replication
        ↓
CLIENT01 domain connectivity
        ↓
Authentication
        ↓
Group Policy
        ↓
Help Desk administration
        ↓
Departmental resource access
        ↓
Authentication troubleshooting
        ↓
Identity lifecycle administration
```

**Phase 1 Status: COMPLETE**
