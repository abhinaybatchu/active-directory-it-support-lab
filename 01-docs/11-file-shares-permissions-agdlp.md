# Phase 11 — File Shares, NTFS Permissions and AGDLP

## 1. Objective

The objective of this phase was to implement departmental file-share access using Active Directory security groups, SMB share permissions, NTFS permissions, and the AGDLP access-control model.

The phase connected the users and security groups created earlier in the project to actual enterprise-style resources.

The completed implementation demonstrated:

- SMB departmental file shares
- NTFS permissions
- Share permissions
- Global security groups
- Domain Local security groups
- AGDLP
- Group-based authorization
- Authorized and unauthorized access testing
- Windows integrated authentication
- Group Policy Preferences
- Automatic mapped drives
- Group Policy validation
- Least-privilege resource access

---

## 2. Lab Environment

The primary systems used were:

| System   | Role                                                                   |
| -------- | ---------------------------------------------------------------------- |
| DC01     | Active Directory Domain Controller, DNS server and lab file-share host |
| CLIENT01 | Domain-joined Windows 11 workstation                                   |

Domain:

```text
abhinaylabs.internal
```

NetBIOS domain:

```text
ABHINAYLABS
```

The departmental resources were hosted on DC01 to minimize VM resource consumption in the home-lab environment.

In a production enterprise environment, departmental file services would normally be hosted on dedicated file-server infrastructure rather than directly on a domain controller.

---

# Departmental Access Design

## 3. Resources

Two synthetic departmental resources were created:

```text
C:\LabShares\HR
C:\LabShares\Finance
```

They were published as SMB shares:

```text
\\DC01\HR
\\DC01\Finance
```

Synthetic files were created to provide safe test data:

```text
HR-Welcome.txt
Finance-Welcome.txt
```

No real organizational or personal data was used.

---

## 4. Users

The access tests used the following synthetic Active Directory users:

| Department | User         | Username  |
| ---------- | ------------ | --------- |
| HR         | David Miller | `dmiller` |
| Finance    | Elena Rivera | `erivera` |

---

## 5. Security Groups

The following Global Groups represented departmental identities:

```text
GG_HR_Users
GG_Finance_Users
```

The following Domain Local Groups represented resource permissions:

```text
DL_HR_Share_RW
DL_Finance_Share_RW
```

Existing membership relationships were:

```text
David Miller
    ↓
GG_HR_Users
```

and:

```text
Elena Rivera
    ↓
GG_Finance_Users
```

The Global Groups were nested into the corresponding Domain Local Groups:

```text
GG_HR_Users
    ↓
DL_HR_Share_RW
```

and:

```text
GG_Finance_Users
    ↓
DL_Finance_Share_RW
```

---

# AGDLP

## 6. What Is AGDLP?

AGDLP is an Active Directory group-based permission model:

```text
A  → Accounts
G  → Global Groups
DL → Domain Local Groups
P  → Permissions
```

The model separates user membership from resource permissions.

The HR implementation was:

```text
David Miller
      ↓
GG_HR_Users
      ↓
DL_HR_Share_RW
      ↓
HR resource permissions
```

The Finance implementation was:

```text
Elena Rivera
      ↓
GG_Finance_Users
      ↓
DL_Finance_Share_RW
      ↓
Finance resource permissions
```

This completed the Permissions component of the AGDLP structure created earlier in the project.

---

## 7. Why Group-Based Permissions Were Used

Permissions were not assigned directly to individual users.

For example, the lab avoided:

```text
David Miller
    ↓
Direct permission on HR folder
```

Instead:

```text
David Miller
    ↓
GG_HR_Users
    ↓
DL_HR_Share_RW
    ↓
HR resource
```

This approach is easier to administer and scales better.

If another HR employee is onboarded, the administrator can add the employee to:

```text
GG_HR_Users
```

rather than repeatedly modifying the HR resource ACL.

---

# NTFS Permissions

## 8. What Are NTFS Permissions?

NTFS permissions control access to files and folders stored on NTFS-formatted Windows volumes.

Common permissions include:

```text
Full Control
Modify
Read & Execute
Read
Write
```

NTFS permissions apply to the underlying filesystem resource.

---

## 9. HR NTFS Configuration

The HR folder was created at:

```text
C:\LabShares\HR
```

Permission inheritance was disabled.

Existing inherited permissions were converted to explicit permissions before the ACL was cleaned.

Broad `Users` entries and unnecessary `CREATOR OWNER` access were removed from the departmental folder.

The intended final resource-access model retained administrative/system control and added:

```text
ABHINAYLABS\DL_HR_Share_RW
→ Modify
```

The Domain Local resource group therefore controlled departmental modification access rather than individual user accounts.

---

## 10. Finance NTFS Configuration

The Finance folder was created at:

```text
C:\LabShares\Finance
```

The same controlled permission model was implemented.

The Finance Domain Local group received:

```text
ABHINAYLABS\DL_Finance_Share_RW
→ Modify
```

This provided equivalent departmental access while keeping HR and Finance authorization separated.

---

## 11. Why Modify Instead of Full Control?

Departmental users require the ability to:

- Read files
- Create files
- Modify files
- Create folders
- Delete files and folders

They do not require administrative control over the resource's security configuration.

Therefore:

```text
Modify
```

was used instead of:

```text
Full Control
```

This follows the principle of least privilege.

---

# SMB File Shares

## 12. What Is SMB?

SMB, or Server Message Block, is a network file-sharing protocol used by Windows systems to access shared files, folders and other network resources.

The lab published:

```text
\\DC01\HR
```

and:

```text
\\DC01\Finance
```

as SMB resources.

---

## 13. HR Share

The HR resource was configured as:

```text
Share name:
HR

Local path:
C:\LabShares\HR

UNC path:
\\DC01\HR
```

Description:

```text
Abhinay Labs HR departmental share
```

The HR resource group received:

```text
DL_HR_Share_RW
→ Change
```

on the SMB share.

The Windows permission interface also includes Read as part of the required Change access.

Broad `Everyone → Full Control` access was not retained.

---

## 14. Finance Share

The Finance resource was configured as:

```text
Share name:
Finance

Local path:
C:\LabShares\Finance

UNC path:
\\DC01\Finance
```

Description:

```text
Abhinay Labs Finance departmental share
```

The Finance resource group received:

```text
DL_Finance_Share_RW
→ Change
```

on the SMB share.

---

# Share Permissions vs NTFS Permissions

## 15. Difference

Share permissions control access to a resource when it is accessed through an SMB share.

NTFS permissions control access to the underlying filesystem objects.

For network access, both layers participate:

```text
SMB Share Permissions
        +
NTFS Permissions
        ↓
Effective Network Access
```

The lab deliberately configured both layers.

For HR:

```text
DL_HR_Share_RW

Share:
Change

NTFS:
Modify
```

For Finance:

```text
DL_Finance_Share_RW

Share:
Change

NTFS:
Modify
```

A user must have sufficient effective authorization through the applicable permission layers to perform an operation.

---

# Validation

## 16. SMB Share Validation

The departmental shares were inspected using:

```powershell
Get-SmbShare |
Where-Object Name -in "HR","Finance" |
Select-Object Name,Path,Description
```

The configured resources were confirmed as:

```text
Finance   C:\LabShares\Finance
HR        C:\LabShares\HR
```

The displayed capitalization of a Windows filesystem path does not affect access because Windows filesystem paths are normally case-insensitive.

---

## 17. Share Permission Validation

SMB permissions were inspected using:

```powershell
Get-SmbShareAccess -Name "HR"
```

and:

```powershell
Get-SmbShareAccess -Name "Finance"
```

The intended resource groups were assigned Change access.

---

## 18. NTFS Permission Validation

NTFS access was inspected using:

```powershell
(Get-Acl "C:\LabShares\HR").Access |
Format-Table IdentityReference,FileSystemRights,AccessControlType,IsInherited -AutoSize
```

and:

```powershell
(Get-Acl "C:\LabShares\Finance").Access |
Format-Table IdentityReference,FileSystemRights,AccessControlType,IsInherited -AutoSize
```

The appropriate Domain Local resource groups were confirmed on the corresponding departmental resources.

---

## 19. Active Directory Group Validation

Active Directory group membership was inspected using:

```powershell
Get-ADGroupMember "GG_HR_Users"
```

```powershell
Get-ADGroupMember "DL_HR_Share_RW"
```

```powershell
Get-ADGroupMember "GG_Finance_Users"
```

```powershell
Get-ADGroupMember "DL_Finance_Share_RW"
```

These checks validated the identity and resource-group relationships before endpoint authorization testing.

---

# Authorized and Unauthorized Access Testing

## 20. HR User Test

CLIENT01 was signed into using:

```text
ABHINAYLABS\dmiller
```

The HR share was accessed using:

```text
\\DC01\HR
```

David Miller successfully accessed the resource and could view the synthetic HR test file.

Write/modify capability was also tested by creating and manipulating a temporary test file.

This validated authorized HR access.

---

## 21. HR User Denied Finance Access

While operating under the same HR user context, the Finance resource was requested:

```text
\\DC01\Finance
```

Windows denied access because David Miller did not receive Finance resource authorization.

The test demonstrated an important distinction:

```text
Authentication
≠
Authorization
```

David could successfully authenticate to the Active Directory domain while still being denied access to a resource for which his security groups did not provide authorization.

---

## 22. Finance User Test

CLIENT01 was also tested using:

```text
ABHINAYLABS\erivera
```

The Finance user successfully accessed:

```text
\\DC01\Finance
```

and the intended modification capability was validated.

The Finance user was not authorized for the HR departmental resource.

This confirmed that the access-control model functioned in both departmental directions.

---

# Windows Security Tokens

## 23. Group Membership and Logon

Windows creates a user's security access token during authentication.

The token contains security information used when Windows evaluates authorization, including relevant group memberships.

Because of this, newly changed group membership may require a user to sign out and sign back in before the new authorization is represented in the user's session.

This is an important troubleshooting consideration for Active Directory access problems.

---

# Group Policy Drive Mapping

## 24. Objective

The HR share was also automatically presented to HR users through Group Policy Preferences.

Instead of requiring a user to manually enter:

```text
\\DC01\HR
```

the resource was mapped as:

```text
H:
```

with the label:

```text
HR Department
```

---

## 25. HR GPO Scope

During configuration review, the existing:

```text
GPO-HR-User-Policy
```

link was verified and corrected so that the policy was linked specifically to:

```text
Corp
└── Users
    └── HR
```

rather than the parent Users OU.

This ensured that HR-specific settings were scoped to the intended departmental OU.

The correction also aligned the implemented environment with the documented Group Policy design.

---

## 26. Drive Map Configuration

The drive map was configured under:

```text
GPO-HR-User-Policy
→ User Configuration
→ Preferences
→ Windows Settings
→ Drive Maps
```

Configuration:

```text
Action:
Update

Location:
\\DC01\HR

Label:
HR Department

Drive letter:
H:
```

No alternate credentials were stored in the Group Policy Preference item.

---

## 27. Why Alternate Credentials Were Not Used

The mapped drive should access the resource using the signed-in domain user's Windows security context.

The intended process is:

```text
David authenticates to Active Directory
        ↓
Windows creates David's security token
        ↓
Group Policy maps \\DC01\HR
        ↓
SMB evaluates the connection
        ↓
NTFS evaluates filesystem access
        ↓
AGDLP group membership authorizes David
```

Embedding an administrator username or password into the mapping would defeat this access-control model and introduce unnecessary credential risk.

---

# Drive Mapping Validation

## 28. Group Policy Refresh

On CLIENT01, while signed in as:

```text
ABHINAYLABS\dmiller
```

Group Policy was refreshed using:

```cmd
gpupdate /force
```

Both computer and user policy processing completed successfully.

---

## 29. Endpoint Validation

CLIENT01 displayed:

```text
HR Department (H:)
```

under the user's network locations.

The mapping was also validated using:

```cmd
net use
```

which showed the relationship:

```text
H:
→ \\DC01\HR
```

---

## 30. GPO Validation

The user-side Resultant Set of Policy was inspected using:

```cmd
gpresult /r /scope user
```

The applied policies included:

```text
GPO-HR-User-Policy
```

This demonstrated that the mapped drive was delivered through the intended HR Group Policy.

---

# Troubleshooting Lessons

## 31. GPO Scope Review

During the phase, the HR GPO link was reviewed and found to be attached to the parent:

```text
Users
```

OU rather than specifically to:

```text
HR
```

Although the HR test user still received the policy because HR was a child OU, this made the policy scope broader than intended.

The incorrect link was removed and the existing GPO was linked directly to the HR OU.

This is an important Group Policy troubleshooting lesson:

> A policy working for the intended user does not automatically mean that its scope is correct.

Administrators must also verify that unintended users are not included in the policy scope.

---

## 32. Share Name vs Folder Path Capitalization

The HR SMB share was correctly named:

```text
HR
```

while Windows displayed the underlying local path with lowercase capitalization in one PowerShell view.

Windows filesystem paths are normally case-insensitive, so this did not represent a separate folder or configuration failure.

The user-facing SMB path remained:

```text
\\DC01\HR
```

No unnecessary corrective work was performed.

---

## 33. Command Context

Network-drive validation commands were executed from an existing Command Prompt or PowerShell session.

For example:

```cmd
net use
```

Launching a command-line utility directly through the Windows Run dialog may briefly open and close a console after the command executes.

This behavior does not indicate that the mapped drive failed.

---

# IT Support Relevance

## 34. Common File-Share Tickets

An IT Support technician may receive tickets such as:

```text
"I can't access the HR drive."

"My departmental drive disappeared."

"I get Access Denied."

"My coworker can open this folder but I can't."

"I was moved to another department and still have the old drive."

"I was added to the group but access still doesn't work."
```

A structured investigation can include:

```text
1. Confirm the user's identity.
2. Confirm the correct AD group membership.
3. Confirm Global → Domain Local nesting.
4. Confirm SMB share permissions.
5. Confirm NTFS permissions.
6. Confirm the user has refreshed their security token after membership changes.
7. Confirm network/DNS connectivity to the file server.
8. Confirm the UNC path.
9. Confirm mapped-drive Group Policy processing.
10. Use gpresult when the mapping is delivered through Group Policy.
```

---

# IAM Relevance

## 35. Identity-Based Authorization

This phase demonstrates the distinction between:

```text
Authentication
```

and:

```text
Authorization
```

Authentication answers:

> Who is the user?

Authorization answers:

> What is that authenticated user allowed to access?

Active Directory group membership was used to translate organizational identity into resource authorization.

---

## 36. Least Privilege

Users received only the departmental resource access required by their assigned role.

For example:

```text
David Miller
HR access       → Allowed
Finance access  → Denied
```

The model avoids unnecessary cross-department access.

---

# Security Analyst Relevance

## 37. Resource Access Controls

Improper file-share permissions can expose sensitive organizational data.

Security analysts may investigate:

- Excessive group membership
- Unauthorized share access
- Broad SMB permissions
- Permission changes
- Privileged group changes
- Unexpected file access
- Lateral movement involving SMB

Understanding normal access-control design helps distinguish expected access from suspicious activity.

---

# Common Mistakes

## 38. Assigning Permissions Directly to Users

Avoid repeatedly configuring:

```text
User
→ Resource
```

when group-based authorization is appropriate.

Prefer:

```text
Account
→ Global Group
→ Domain Local Group
→ Permission
```

---

## 39. Giving Department Users Full Control

Department users generally do not need permission to modify the security configuration of the resource.

For this lab:

```text
NTFS → Modify
SMB  → Change
```

provided the required working access without granting unnecessary administrative control.

---

## 40. Leaving Broad Access

Broad entries can undermine departmental separation.

The departmental folders and shares were configured so that access was provided through the intended Domain Local resource groups rather than relying on broad general-user access.

---

## 41. Troubleshooting Only the Share Permission

Network resource access can involve both:

```text
Share permissions
```

and:

```text
NTFS permissions
```

Checking only one layer can lead to incorrect conclusions.

---

## 42. Forgetting the User Security Token

After changing group membership, the existing user's logon session may still contain the old security token.

A clean sign-out and sign-in is an important troubleshooting step.

---

## 43. Mapping Drives with Administrative Credentials

A departmental mapped drive should normally operate using the user's own authenticated security context.

Hard-coding privileged credentials is unnecessary and introduces security risk.

---

# Evidence

## Figure 18 — AGDLP-Based HR Resource Permissions

File:

[Open Screenshot](../02-screenshots/18-agdlp-hr-resource-permissions.png)

Description:

> HR departmental access implemented using Active Directory security groups and resource-level permissions. The `DL_HR_Share_RW` domain-local group is assigned NTFS access to the HR folder and SMB Change permission to the HR share, completing the permission layer of the AGDLP access-control model.

---

## Figure 19 — Authorized and Unauthorized Departmental Access

File:

[Open Screenshot](../02-screenshots/19-hr-authorized-finance-denied.png)

Description:

> Synthetic HR user David Miller successfully accessing the authorized `\\DC01\HR` departmental share while being denied access to `\\DC01\Finance`, validating role-based separation of departmental resources.

---

## Figure 20 — Group Policy HR Drive Mapping

File:

[Open Screenshot](../02-screenshots/20-hr-gpo-drive-mapping.png)

Description:

> `GPO-HR-User-Policy` correctly scoped to the HR OU and configured to map `\\DC01\HR` as the `H:` drive. CLIENT01 validation shows successful Group Policy processing, the mapped HR Department drive, the active SMB connection, and the applicable HR user GPO.

---

# Final Architecture

The completed HR authorization path is:

```text
David Miller
ABHINAYLABS\dmiller
        │
        ▼
GG_HR_Users
        │
        ▼
DL_HR_Share_RW
        │
        ├── SMB: Change
        │
        └── NTFS: Modify
        │
        ▼
\\DC01\HR
        │
        ▼
HR Department (H:)
```

The completed Finance authorization path is:

```text
Elena Rivera
ABHINAYLABS\erivera
        │
        ▼
GG_Finance_Users
        │
        ▼
DL_Finance_Share_RW
        │
        ├── SMB: Change
        │
        └── NTFS: Modify
        │
        ▼
\\DC01\Finance
```

Departmental separation was validated:

```text
David Miller

HR       → Allowed
Finance  → Denied
```

and:

```text
Elena Rivera

Finance  → Allowed
HR       → Denied
```

---

# Phase 11 Result

Phase 11 successfully connected Active Directory identity management to practical Windows resource authorization.

The completed workflow was:

```text
Create departmental resources
        ↓
Configure NTFS permissions
        ↓
Publish SMB shares
        ↓
Configure share permissions
        ↓
Connect permissions to Domain Local groups
        ↓
Use Global Groups for departmental membership
        ↓
Validate authorized access
        ↓
Validate denied access
        ↓
Map resource using Group Policy
        ↓
Validate policy and endpoint behavior
```

This completed the practical AGDLP implementation:

```text
Accounts
   ↓
Global Groups
   ↓
Domain Local Groups
   ↓
Permissions
```

The environment is now ready for **Phase 12 — Help Desk Delegation and Least-Privilege Administration**.
