# Abhinay Labs — Network Design

## Overview

This project uses VMware Workstation to simulate the internal network of a small enterprise environment.

Abhinay Labs is a fictional organization created solely for this Active Directory IT Support home lab. All users, systems, identities, and data used in the environment are synthetic.

The Active Directory lab uses VMware's existing VMnet8 NAT network. This provides communication between the virtual machines and Internet connectivity while keeping the lab logically separated from the physical home network.

---

## Network Configuration

| Component               | Configuration        |
| ----------------------- | -------------------- |
| VMware Network          | VMnet8               |
| Network Type            | NAT                  |
| Network Address         | 192.168.170.0/24     |
| Subnet Mask             | 255.255.255.0        |
| NAT Gateway             | 192.168.170.2        |
| VMware DHCP             | Enabled              |
| DHCP Start Address      | 192.168.170.128      |
| DHCP End Address        | 192.168.170.254      |
| Active Directory Domain | abhinaylabs.internal |
| NetBIOS Domain          | ABHINAYLABS          |

---

## Planned Systems

| System   | Role                                            | IPv4 Address   |
| -------- | ----------------------------------------------- | -------------- |
| DC01     | Windows Server Domain Controller and DNS Server | 192.168.170.10 |
| CLIENT01 | Windows 11 Domain Workstation                   | 192.168.170.20 |

Both lab systems will use manually assigned IPv4 addresses outside VMware's DHCP allocation range.

---

## IP Addressing Design

VMware automatically distributes addresses between:

192.168.170.128 - 192.168.170.254

The Active Directory systems therefore use static addresses outside this range:

DC01:

192.168.170.10

CLIENT01:

192.168.170.20

This prevents VMware DHCP from automatically assigning either address to another virtual machine.

---

## Planned DNS Design

DC01 will host Active Directory-integrated DNS for the abhinaylabs.internal domain.

After Active Directory and DNS are installed, CLIENT01 will use DC01 as its preferred DNS server.

Planned configuration:

DC01:

IPv4 Address: 192.168.170.10

CLIENT01:

IPv4 Address: 192.168.170.20

Preferred DNS Server: 192.168.170.10

This allows CLIENT01 to locate Active Directory domain services through DNS.

---

## Network Architecture

Internet
    |
    |
VMware NAT
VMnet8
192.168.170.2
    |
    |
192.168.170.0/24
    |
    +-----------------------------+
    |                             |
    |                             |
   DC01                        CLIENT01
192.168.170.10              192.168.170.20
Windows Server              Windows 11
AD DS + DNS                 Domain Workstation

Active Directory Domain:

abhinaylabs.internal

NetBIOS Domain:

ABHINAYLABS

---

## Design Decisions

### VMware NAT

VMware's existing VMnet8 NAT network was selected for the lab.

NAT allows the virtual machines to communicate with each other and access external resources while avoiding direct bridged exposure of the Active Directory environment to the physical home network.

### Static Infrastructure Addressing

DC01 and CLIENT01 use manually assigned IPv4 addresses.

DC01 requires predictable addressing because it will provide important infrastructure services including Active Directory Domain Services and DNS.

### VMware DHCP

VMware DHCP remains enabled on VMnet8 because the network may also be used by other virtual machines.

The Active Directory lab avoids DHCP conflicts by assigning static addresses outside the configured DHCP range.

### DNS

CLIENT01 will use DC01 as its DNS server after Active Directory DNS is configured.

This is required because Active Directory clients use DNS to locate domain controllers and services such as Kerberos and LDAP.

---

## Portfolio Evidence

Screenshot:

`02-screenshots/01-vmware-ad-lab-network.png`

The screenshot demonstrates the VMware NAT network used by the simulated Abhinay Labs enterprise environment.
