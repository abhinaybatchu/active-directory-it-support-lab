# Active Directory Organizational Unit Design

## Overview

The `abhinaylabs.internal` Active Directory domain was organized using an enterprise-style Organizational Unit structure.

The design separates users, groups, workstations, servers, and departmental identities to support centralized administration, Group Policy targeting, delegation, and future access-control exercises.

---

## OU Structure

The following structure was created:

    abhinaylabs.internal
    |
    +-- Corp
        |
        +-- Groups
        |
        +-- Servers
        |
        +-- Users
        |   |
        |   +-- Disabled-Users
        |   +-- Finance
        |   +-- HR
        |   +-- IT
        |   +-- Sales
        |
        +-- Workstations

The built-in `Domain Controllers` OU remains outside the custom `Corp` hierarchy.

DC01 remains in the `Domain Controllers` OU.

---

## Design Purpose

### Corp

`Corp` provides a top-level administrative structure for custom Abhinay Labs directory objects.

### Users

The `Users` OU separates organization-managed user accounts from the default Active Directory Users container.

Departmental child OUs provide logical organization and future Group Policy and delegation targets.

### Disabled-Users

The `Disabled-Users` OU will support the simulated employee offboarding workflow.

Accounts can be disabled and moved into this OU rather than immediately deleted.

Actual retention and deletion decisions in a production environment would follow organizational policy.

### Groups

The `Groups` OU will contain organization-managed security groups used for role and resource access.

### Workstations

The `Workstations` OU will contain domain-joined client computer objects such as CLIENT01.

Separating workstations provides an administrative and Group Policy targeting boundary for endpoint configurations.

### Servers

The `Servers` OU is reserved for member servers.

Domain Controllers are not moved into this OU because Active Directory maintains a dedicated `Domain Controllers` OU with Domain Controller-specific policy and administrative considerations.

---

## Organizational Units vs Security Groups

Organizational Units and security groups serve different purposes.

Organizational Units are primarily used for:

- Directory organization
- Group Policy targeting
- Administrative delegation

Security groups are primarily used to:

- Group identities
- Assign permissions
- Control access to resources

A user's departmental OU does not by itself grant that user access to departmental resources.

Resource authorization will later be implemented through security-group membership and permissions.

---

## Distinguished Names

Active Directory objects can be identified using Distinguished Names.

For example:

`OU=HR,OU=Users,OU=Corp,DC=abhinaylabs,DC=internal`

represents the HR Organizational Unit located inside:

`Corp > Users > HR`

In the Distinguished Name:

- `OU` represents an Organizational Unit
- `DC` represents a domain component

---

## Validation

The OU structure was validated through Active Directory Users and Computers and PowerShell.

PowerShell validation included:

`Get-ADOrganizationalUnit -Filter *`

The resulting directory structure confirmed that the custom organizational units were successfully created in the intended hierarchy.

---

## Security and Administration

Protection from accidental deletion was enabled when creating the custom Organizational Units.

This provides an additional safeguard against accidental OU deletion.

The OU design also prepares the environment for later least-privilege Help Desk delegation.

---

## Evidence

`02-screenshots/05-ad-ou-structure.png`

The screenshot documents the custom Abhinay Labs Organizational Unit hierarchy in Active Directory Users and Computers.
