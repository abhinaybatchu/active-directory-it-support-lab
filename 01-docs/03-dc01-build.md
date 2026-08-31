# DC01 — Windows Server Build

## Overview

DC01 is the Windows Server system that will become the Domain Controller and DNS server for the simulated Abhinay Labs enterprise environment.

This phase created the virtual machine and installed the base Windows Server operating system. Active Directory Domain Services is installed and configured in a later phase.

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

## Current Network State

During the base installation, the server receives its IPv4 configuration from VMware DHCP.

The final static network configuration will be applied during the server configuration phase.

Planned configuration:

| Setting                 | Value                |
| ----------------------- | -------------------- |
| Computer Name           | DC01                 |
| Static IPv4             | 192.168.170.10       |
| Subnet Mask             | 255.255.255.0        |
| Default Gateway         | 192.168.170.2        |
| Active Directory Domain | abhinaylabs.internal |

## Base Validation

The base Windows Server installation was validated by checking:

- Windows Server version
- Current computer hostname
- IPv4 configuration
- VMware NAT gateway connectivity
- DNS name resolution
- Outbound HTTPS connectivity

## VMware Tools

VMware Tools was installed to provide appropriate guest drivers and improved integration between Windows Server and VMware Workstation.

## Security Notes

The local Administrator account uses a unique lab-only password.

Passwords, credentials, product keys, and other sensitive values are not stored in the public repository.

## Evidence

The configured DC01 server state is captured after hostname and static network configuration in the following phase.
