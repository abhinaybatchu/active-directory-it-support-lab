# DC01 — Windows Server Build

## Overview

DC01 is the Windows Server system that will become the Domain Controller and DNS server for the simulated Abhinay Labs enterprise environment.

The server was initially built as a standalone Windows Server system. Active Directory Domain Services and DNS will be installed and configured in the next phase.

---

## Virtual Machine Configuration

| Component              | Configuration                           |
| ---------------------- | --------------------------------------- |
| Virtual Machine        | DC01                                    |
| Hypervisor             | VMware Workstation                      |
| Guest Operating System | Windows Server 2025 Standard Evaluation |
| Installation Type      | Desktop Experience                      |
| Firmware               | UEFI                                    |
| Virtual CPUs           | 2                                       |
| Memory                 | 4 GB                                    |
| Virtual Disk           | 60 GB                                   |
| Network                | VMnet8 NAT                              |

---

## Server Identity Configuration

The Windows-generated computer name was replaced with the enterprise-style hostname:

`DC01`

The naming convention identifies the system's intended role as the first Domain Controller in the Abhinay Labs environment.

The server remained in the default `WORKGROUP` during this phase because the Active Directory domain had not yet been created.

---

## Static IPv4 Configuration

DC01 was changed from VMware DHCP addressing to a manually configured static IPv4 address.

| Setting              | Value          |
| -------------------- | -------------- |
| IPv4 Address         | 192.168.170.10 |
| Subnet Mask          | 255.255.255.0  |
| Default Gateway      | 192.168.170.2  |
| Temporary Pre-AD DNS | 192.168.170.2  |

The static address was selected outside the VMware DHCP allocation range of `192.168.170.128 - 192.168.170.254`.

This provides predictable addressing for the server before it begins hosting Active Directory Domain Services and DNS.

---

## Pre-AD DNS Configuration

During initial server configuration, DC01 temporarily uses the VMware network's DNS path through `192.168.170.2`.

This is a transitional configuration only.

After the DNS Server role and Active Directory Domain Services are installed, DNS configuration will be changed to support the `abhinaylabs.internal` Active Directory domain.

Domain workstations will later use DC01 (`192.168.170.10`) as their preferred DNS server.

---

## VMware Tools

VMware Tools was installed to provide appropriate guest drivers and improved integration between Windows Server and VMware Workstation.

---

## Base Validation

The base Windows Server installation and subsequent server configuration were validated using:

`hostname`

`ipconfig /all`

`Get-NetIPConfiguration`

`Get-DnsClientServerAddress -AddressFamily IPv4`

`ping 192.168.170.2`

`nslookup microsoft.com`

`Test-NetConnection microsoft.com -Port 443`

Validation confirmed:

- Windows Server 2025 Standard Evaluation was installed successfully
- Hostname changed successfully to `DC01`
- Static IPv4 address `192.168.170.10` was applied successfully
- VMware NAT gateway `192.168.170.2` was reachable
- External DNS resolution was working
- Outbound HTTPS connectivity over TCP port 443 was working
- Static network configuration survived the server restart

A failed external ICMP ping alone was not treated as an Internet connectivity failure because ICMP responses may be filtered while DNS and TCP/HTTPS connectivity remain functional.

---

## Time Configuration

The server was configured to use Eastern Time.

Accurate system time is particularly important in Active Directory environments because Kerberos authentication depends on time synchronization between systems.

---

## Windows Updates

Windows security, Microsoft Defender, .NET, and other applicable system updates were installed before adding Active Directory infrastructure roles.

This established an updated base server before Domain Controller promotion.

---

## Security Notes

The local Administrator account uses a unique lab-only password.

Passwords, credentials, product keys, and other sensitive values are not stored in the public repository.

Microsoft Defender Antivirus real-time protection and Microsoft Defender Firewall remained enabled.

---

## Evidence

`02-screenshots/02-dc01-server-identity.png`

The screenshot documents DC01 after server identity, static IPv4 configuration, time-zone configuration, and Windows updates were completed.
