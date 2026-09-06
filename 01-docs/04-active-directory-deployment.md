# Active Directory Domain Services Deployment

## Overview

DC01 was promoted from a standalone Windows Server system to the first Domain Controller for the simulated Abhinay Labs enterprise environment.

The deployment created a new Active Directory forest and domain and installed DNS services required for Active Directory service discovery.

---

## Active Directory Configuration

| Component                   | Configuration        |
| --------------------------- | -------------------- |
| Domain Controller           | DC01                 |
| Active Directory Domain     | abhinaylabs.internal |
| NetBIOS Domain              | ABHINAYLABS          |
| Forest                      | abhinaylabs.internal |
| DNS Server                  | DC01                 |
| DNS Server IPv4             | 192.168.170.10       |
| Global Catalog              | Enabled              |
| Read-Only Domain Controller | No                   |

---

## New Forest Deployment

A new Active Directory forest was created because the Abhinay Labs environment did not have an existing Active Directory infrastructure.

The forest contains the root domain:

`abhinaylabs.internal`

This lab currently uses one forest, one domain, and one Domain Controller.

A production environment may use multiple Domain Controllers to provide redundancy and resilience.

---

## Active Directory Domain Services

Active Directory Domain Services provides centralized directory and identity services for Windows domain environments.

The domain will be used to centrally manage:

- User accounts
- Computer accounts
- Security groups
- Organizational Units
- Authentication
- Authorization
- Group Policy
- Access to enterprise resources

---

## DNS Integration

DNS was installed with Active Directory Domain Services.

Active Directory clients rely on DNS to locate Domain Controllers and services including LDAP and Kerberos.

After DNS installation, DC01 was configured to use its own Active Directory-aware DNS service:

`192.168.170.10`

Domain clients will later use DC01 as their preferred DNS server.

Public DNS resolvers are not configured directly on domain clients because they do not contain the private DNS records required to locate the `abhinaylabs.internal` Active Directory services.

---

## Domain Controller Services

The deployment was validated by checking important Domain Controller services including:

- Active Directory Domain Services (NTDS)
- DNS Server
- Netlogon
- Kerberos Key Distribution Center (KDC)

---

## Domain Validation

The deployment was validated using:

`hostname`

`(Get-CimInstance Win32_ComputerSystem).Domain`

`nltest /dsgetdc:abhinaylabs.internal`

`Get-ADDomain`

`Get-ADForest`

`Get-Service NTDS,DNS,Netlogon,Kdc`

Validation confirmed that DC01 was operating as a Domain Controller for the `abhinaylabs.internal` domain.

---

## DNS Validation

Internal Active Directory DNS resolution was tested using:

`nslookup abhinaylabs.internal`

`nslookup DC01.abhinaylabs.internal`

Domain Controller service records were tested using:

`nslookup -type=SRV _ldap._tcp.dc._msdcs.abhinaylabs.internal`

External DNS resolution and HTTPS connectivity were also tested to confirm that the server could resolve both internal Active Directory resources and external Internet resources.

---

## Directory Services Restore Mode

A Directory Services Restore Mode (DSRM) password was configured during Domain Controller promotion.

DSRM provides a recovery environment for specific Active Directory maintenance and recovery scenarios.

The DSRM credential is not stored in the public project repository.

---

## Security Considerations

- DC01 uses a static IPv4 address.
- Microsoft Defender Firewall remains enabled.
- Microsoft Defender Antivirus remains enabled.
- Credentials are excluded from project documentation.
- The Active Directory environment is isolated within the VMware lab network.
- The Domain Controller is not exposed directly to the public Internet.

---

## Evidence

[Open Screenshot](../02-screenshots/03-dc01-ad-ds-dns-roles.png)

DC01 operating with Active Directory Domain Services and DNS roles.

[Open Screenshot](../02-screenshots/04-abhinaylabs-domain-created.png)

The `abhinaylabs.internal` domain and DC01 Domain Controller object displayed in Active Directory Users and Computers.
