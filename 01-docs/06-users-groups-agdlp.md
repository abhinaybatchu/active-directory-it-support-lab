# Active Directory Users, Security Groups, and AGDLP

## Overview

Synthetic employee accounts and security groups were created in the `abhinaylabs.internal` domain to simulate enterprise identity administration and group-based access control.

All identities used in this lab are fictional.

---

## Synthetic Users

| Department | User           | Username  | UPN                            |
| ---------- | -------------- | --------- | ------------------------------ |
| HR         | David Miller   | `dmiller` | `dmiller@abhinaylabs.internal` |
| Finance    | Elena Rivera   | `erivera` | `erivera@abhinaylabs.internal` |
| Sales      | Joseph Daniel  | `jdaniel` | `jdaniel@abhinaylabs.internal` |
| IT         | Pauline Hudson | `phudson` | `phudson@abhinaylabs.internal` |

Each user was created in the appropriate departmental Organizational Unit under:

`Corp > Users`

A consistent first-initial-plus-last-name username convention was used.

Each account was configured with a temporary lab-only password and required to change the password at the next applicable logon.

Credentials are not stored in the public repository.

---

## Global Security Groups

The following Global security groups were created:

| Group              | Purpose                                 |
| ------------------ | --------------------------------------- |
| `GG_HR_Users`      | HR departmental membership              |
| `GG_Finance_Users` | Finance departmental membership         |
| `GG_Sales_Users`   | Sales departmental membership           |
| `GG_IT_Users`      | General IT departmental membership      |
| `GG_IT_Helpdesk`   | Dedicated Help Desk administrative role |

The `GG_` prefix identifies Global groups within the lab naming convention.

Departmental users were assigned as follows:

| User           | Global Group       |
| -------------- | ------------------ |
| David Miller   | `GG_HR_Users`      |
| Elena Rivera   | `GG_Finance_Users` |
| Joseph Daniel  | `GG_Sales_Users`   |
| Pauline Hudson | `GG_IT_Users`      |

`GG_IT_Helpdesk` was intentionally kept separate from `GG_IT_Users`.

General IT membership does not automatically provide Help Desk administrative privileges. Specific Help Desk permissions will be delegated later according to least-privilege principles.

---

## Domain Local Security Groups

The following Domain Local security groups were created:

| Group                 | Intended Resource Access                  |
| --------------------- | ----------------------------------------- |
| `DL_HR_Share_RW`      | HR shared-resource read/write access      |
| `DL_Finance_Share_RW` | Finance shared-resource read/write access |

The `DL_` prefix identifies Domain Local groups within the lab naming convention.

These groups represent access to resources rather than departmental membership.

---

## AGDLP Design

The lab uses the AGDLP access-control model:

`Accounts -> Global Groups -> Domain Local Groups -> Permissions`

### HR

`David Miller -> GG_HR_Users -> DL_HR_Share_RW -> HR resource permissions`

### Finance

`Elena Rivera -> GG_Finance_Users -> DL_Finance_Share_RW -> Finance resource permissions`

The following group nesting was configured:

- `GG_HR_Users` was added to `DL_HR_Share_RW`
- `GG_Finance_Users` was added to `DL_Finance_Share_RW`

Actual file-share and NTFS permissions will be assigned during the file-sharing phase.

This design avoids assigning resource permissions directly to individual user accounts.

---

## Why Group-Based Access Is Used

Global groups represent business or departmental membership.

Domain Local groups represent access to specific resources.

Separating these responsibilities makes access easier to administer when users join, change roles, or leave the organization.

For example:

`David Miller -> GG_HR_Users -> DL_HR_Share_RW`

David's resource access can therefore be managed through group membership rather than by assigning permissions directly to his individual account.

---

## PowerShell Validation

### Verify Synthetic User Accounts

The departmental user accounts were validated with:

```powershell
Get-ADUser -Filter * -SearchBase "OU=Users,OU=Corp,DC=abhinaylabs,DC=internal" |
Select-Object Name,SamAccountName,UserPrincipalName,Enabled |
Sort-Object Name
```

This confirmed that the four departmental accounts were created with the intended usernames, UPNs, and enabled status.

### Verify Departmental Global Group Membership

```powershell
Get-ADGroupMember "GG_HR_Users"
Get-ADGroupMember "GG_Finance_Users"
Get-ADGroupMember "GG_Sales_Users"
Get-ADGroupMember "GG_IT_Users"
```

The expected memberships were:

- David Miller -> `GG_HR_Users`
- Elena Rivera -> `GG_Finance_Users`
- Joseph Daniel -> `GG_Sales_Users`
- Pauline Hudson -> `GG_IT_Users`

### Verify Direct AGDLP Group Nesting

```powershell
Get-ADGroupMember "DL_HR_Share_RW"
Get-ADGroupMember "DL_Finance_Share_RW"
```

The expected direct group relationships were:

- `GG_HR_Users` -> `DL_HR_Share_RW`
- `GG_Finance_Users` -> `DL_Finance_Share_RW`

### Verify Recursive Membership

```powershell
Get-ADGroupMember "DL_HR_Share_RW" -Recursive
Get-ADGroupMember "DL_Finance_Share_RW" -Recursive
```

Recursive validation confirmed:

- David Miller is reached through `GG_HR_Users` -> `DL_HR_Share_RW`
- Elena Rivera is reached through `GG_Finance_Users` -> `DL_Finance_Share_RW`

This validates the **A -> G -> DL** portion of the AGDLP model.

The final **P (Permissions)** component will be implemented and tested when the HR and Finance shared resources are configured later in the project.

---

## Security Design

The configuration demonstrates:

- Group-based access control
- Separation of business roles and resource permissions
- Nested security-group membership
- Least-privilege administration
- Separation of normal IT users from Help Desk administrative roles
- Avoidance of direct user-to-resource permission assignments

---

## Evidence

### Synthetic Users

[Open Screenshot](../02-screenshots/06-ad-synthetic-users.png)

Shows the four synthetic departmental user accounts created in the Abhinay Labs Active Directory environment.

### Security Groups and AGDLP Structure

[Open Screenshot](../02-screenshots/07-ad-security-groups-agdlp.png)

Shows the Global and Domain Local security groups created for departmental membership, Help Desk delegation, and AGDLP-based resource authorization.

---

## Phase Result

The Abhinay Labs Active Directory environment now contains:

- Four departmental user accounts
- Departmental Global security groups
- A dedicated Help Desk role group
- HR and Finance Domain Local resource groups
- Departmental group memberships
- Global-to-Domain-Local group nesting
- PowerShell-validated direct and recursive memberships
- An AGDLP structure ready for file-share permission testing
