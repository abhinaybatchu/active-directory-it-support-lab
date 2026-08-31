# Active Directory Lab — Troubleshooting

## Issue 01 — Second VMware NAT Network Could Not Be Created

### Problem

The original network design planned to use VMnet2 as a dedicated NAT network for the Active Directory lab.

When VMnet2 was changed from Host-only networking to NAT, VMware Workstation displayed the following error:

> Cannot change network to NAT: Only one network can be set to NAT.

---

## Investigation

The VMware Virtual Network Editor showed that VMnet8 was already configured as the NAT network.

VMnet8 configuration was verified as:

- Network Type: NAT
- Network Address: 192.168.170.0/24
- NAT Gateway: 192.168.170.2
- VMware DHCP: Enabled
- DHCP Range: 192.168.170.128 - 192.168.170.254

---

## Root Cause

The installed VMware Workstation configuration permitted only one NAT virtual network.

VMnet8 was already using that NAT configuration, preventing VMnet2 from being configured as another NAT network.

---

## Resolution

The existing VMnet8 NAT network was retained and selected for the Active Directory lab.

VMware DHCP was left enabled.

Static IP addresses outside the DHCP allocation range were selected for the Active Directory systems:

- DC01: 192.168.170.10
- CLIENT01: 192.168.170.20

Because VMware DHCP begins at 192.168.170.128, these static addresses do not overlap with the automatic DHCP allocation pool.

---

## Result

The final network design provides:

- Internal communication between the Active Directory virtual machines
- NAT-based Internet connectivity
- Predictable static addressing for lab infrastructure
- No direct bridged exposure to the physical home network

---

## Lesson Learned

Existing infrastructure should be inspected and verified before making network design changes.

When the original design could not be implemented because of a VMware limitation, the existing NAT network was evaluated and reused rather than forcing an unnecessary configuration change.
