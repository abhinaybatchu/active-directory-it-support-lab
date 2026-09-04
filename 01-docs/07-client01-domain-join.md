# CLIENT01 Domain Join and Domain Authentication

## Overview

A Windows 11 Enterprise Evaluation workstation named `CLIENT01` was deployed and joined to the `abhinaylabs.internal` Active Directory domain.

The workstation was configured to use DC01 as its DNS server, validated against core Active Directory services, organized within the enterprise Workstations OU, and tested using a synthetic domain user account.

---

## CLIENT01 Configuration

| Setting                        | Configuration                    |
| ------------------------------ | -------------------------------- |
| Computer Name                  | `CLIENT01`                       |
| Operating System               | Windows 11 Enterprise Evaluation |
| Network                        | VMware VMnet8                    |
| IPv4 Address                   | `192.168.170.20`                 |
| Subnet Mask                    | `255.255.255.0`                  |
| Default Gateway                | `192.168.170.2`                  |
| Preferred DNS                  | `192.168.170.10`                 |
| Active Directory Domain        | `abhinaylabs.internal`           |
| Domain Controller / DNS Server | `DC01` / `192.168.170.10`        |

CLIENT01 was assigned a static IPv4 address outside the VMware DHCP allocation range used by the lab.

---

## Active Directory DNS Configuration

CLIENT01 was configured to use:

```text
192.168.170.10
```

as its preferred DNS server.

This address belongs to DC01, which hosts DNS for the `abhinaylabs.internal` Active Directory domain.

Active Directory depends on DNS for domain-controller and service discovery. Using the domain DNS server allows CLIENT01 to locate services such as LDAP and Kerberos within the private domain.

Public DNS servers were not configured directly on CLIENT01 because they do not contain the private DNS records for `abhinaylabs.internal`.

---

## Network and DNS Validation

The workstation network configuration was verified with:

```powershell
ipconfig /all
```

The expected configuration was confirmed:

```text
IPv4 Address:     192.168.170.20
Subnet Mask:      255.255.255.0
Default Gateway:  192.168.170.2
DNS Server:       192.168.170.10
```

DC01 name resolution was tested with:

```powershell
nslookup dc01.abhinaylabs.internal
```

Active Directory domain-controller service discovery was validated with:

```powershell
nslookup -type=SRV _ldap._tcp.dc._msdcs.abhinaylabs.internal
```

The SRV lookup confirmed that the workstation could use DNS to locate the domain-controller LDAP service.

---

## Active Directory Service Connectivity

Connectivity to important services on DC01 was tested from CLIENT01.

### DNS

```powershell
Test-NetConnection dc01.abhinaylabs.internal -Port 53
```

Port `53` is used for DNS.

### Kerberos

```powershell
Test-NetConnection dc01.abhinaylabs.internal -Port 88
```

Port `88` is used for Kerberos authentication.

### LDAP

```powershell
Test-NetConnection dc01.abhinaylabs.internal -Port 389
```

Port `389` is used for LDAP communication with Active Directory.

Successful connectivity confirmed that CLIENT01 could communicate with the core services required for the domain-join and authentication process.

---

## Domain Join

CLIENT01 was joined to:

```text
abhinaylabs.internal
```

using the Windows System Properties domain-join interface.

Administrative domain credentials were used only to authorize the domain join and were not stored in the repository.

After the join completed successfully, CLIENT01 was restarted.

The domain join created a computer account for CLIENT01 in Active Directory.

---

## Workstation OU Placement

After joining the domain, the CLIENT01 computer object initially appeared in the default Active Directory `Computers` container.

The computer object was moved to:

```text
Corp
└── Workstations
    └── CLIENT01
```

This separates managed employee workstations from default Active Directory objects and prepares CLIENT01 for workstation-specific Group Policy configuration later in the project.

---

## Domain User Authentication

Domain authentication was tested using the synthetic HR account:

```text
David Miller
Username: dmiller
```

David signed into CLIENT01 using the Abhinay Labs domain identity.

Because the account had been configured to require a password change at the next logon, the initial authentication workflow required the temporary lab password to be replaced.

No credentials are stored in the public repository.

---

## Authentication Validation

The authenticated identity was verified with:

```cmd
whoami
```

The result confirmed:

```text
abhinaylabs\dmiller
```

The domain controller responsible for the logon session was identified with:

```cmd
echo %LOGONSERVER%
```

The result was:

```text
\\DC01
```

CLIENT01 domain membership was verified with:

```cmd
systeminfo | findstr /B /C:"Domain"
```

The result confirmed:

```text
Domain: abhinaylabs.internal
```

---

## Domain Controller Discovery

CLIENT01's ability to locate an Active Directory domain controller was validated with:

```cmd
nltest /dsgetdc:abhinaylabs.internal
```

The command successfully identified DC01 as a domain controller for the `abhinaylabs.internal` domain.

This provides an additional validation that CLIENT01 can discover the domain infrastructure through Active Directory and DNS.

---

## Security Group Validation

The security context of the authenticated domain user was inspected with:

```cmd
whoami /groups
```

This displayed the security groups represented in the user's Windows access token.

Group membership becomes important when Windows evaluates authorization to domain resources, including the AGDLP-based file-share permissions that will be implemented later in the project.

---

## IT Support Relevance

This phase demonstrates a common enterprise workstation deployment and troubleshooting workflow:

1. Configure workstation networking.
2. Configure the organization's Active Directory DNS server.
3. Validate DNS and domain-controller discovery.
4. Verify connectivity to required Active Directory services.
5. Join the workstation to the domain.
6. Organize the computer account within the appropriate OU.
7. Authenticate using a domain user account.
8. Verify the authenticated identity and logon server.

When a workstation cannot locate or join an Active Directory domain, DNS configuration is a critical troubleshooting checkpoint because Active Directory relies on DNS service records to locate domain controllers and authentication services.

---

## Evidence

### CLIENT01 Computer Object

`02-screenshots/08-client01-domain-joined.png`

Shows CLIENT01 successfully joined to the `abhinaylabs.internal` domain and organized within the `Corp\Workstations` OU.

### Domain User Authentication

`02-screenshots/09-domain-user-authentication.png`

Shows David Miller authenticated as `abhinaylabs\dmiller`, CLIENT01 joined to `abhinaylabs.internal`, and DC01 identified as the user's logon server.

---

## Phase Result

The Abhinay Labs environment now contains a functional domain-joined Windows workstation with:

- Windows 11 Enterprise Evaluation
- Static IPv4 configuration
- Active Directory-integrated DNS configuration
- DNS and Active Directory service discovery
- Connectivity to DNS, Kerberos, and LDAP services
- Membership in the `abhinaylabs.internal` domain
- Proper placement within the `Corp\Workstations` OU
- Successful synthetic domain-user authentication
- DC01 domain-controller discovery
- Verified domain-user security context

CLIENT01 is now ready for Group Policy, delegated Help Desk administration, file-share authorization, and authentication troubleshooting in later phases.
