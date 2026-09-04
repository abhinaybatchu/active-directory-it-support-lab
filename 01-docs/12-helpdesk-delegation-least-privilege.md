# Phase 12 — Help Desk Delegation and Least-Privilege Administration

## 1. Objective

The objective of this phase was to implement a realistic Help Desk administration model in the Abhinay Labs Active Directory environment.

Instead of granting the Help Desk account unrestricted administrative privileges, a dedicated security group was used to delegate only the permissions required for a common support task: resetting user passwords.

The implementation demonstrated:

- Role-based administrative access using an Active Directory security group
- Delegation of password-reset permissions to the Help Desk role
- Remote Active Directory administration from CLIENT01 using RSAT
- Password reset and forced password change at next logon
- Validation that the Help Desk account was not a Domain Administrator
- Least-privilege separation between routine Help Desk duties and unrestricted Active Directory administration

---

## 2. Lab Environment

| System         | Role                                                |
| -------------- | --------------------------------------------------- |
| DC01           | Primary domain controller, AD DS and DNS            |
| DC02           | Additional domain controller and DNS server         |
| CLIENT01       | Domain-joined Windows 11 administrative workstation |
| Domain         | `abhinaylabs.internal`                              |
| NetBIOS Domain | `ABHINAYLABS`                                       |

### Accounts Used

| Account                     | Purpose                                                                       |
| --------------------------- | ----------------------------------------------------------------------------- |
| `ABHINAYLABS\Administrator` | Domain Administrator used to configure delegation                             |
| `ABHINAYLABS\helpdesk1`     | Delegated Help Desk account                                                   |
| `ABHINAYLABS\dmiller`       | Synthetic HR user used for the password-reset test                            |
| `CLIENT01\labadmin`         | Local administrator used for workstation administration and RSAT installation |

### Security Group

```text
GG_IT_Helpdesk
```

The `helpdesk1` account was added to this group.

Administrative permissions were delegated to the **group**, rather than directly to the individual user account.

This provides a cleaner and more scalable role-based administration model:

```text
helpdesk1
    ↓
GG_IT_Helpdesk
    ↓
Delegated Active Directory permissions
    ↓
Corp\Users
```

---

## 3. Why Delegation Was Used

Granting Help Desk personnel membership in highly privileged groups such as `Domain Admins` would provide far more access than is required for routine user-support tasks.

Instead, Active Directory's **Delegation of Control** functionality was used.

Delegation allows administrators to assign specific administrative permissions over selected Active Directory objects without providing unrestricted control over the domain.

For this lab, the Help Desk role required the ability to:

- Reset user passwords
- Require users to change their password at the next logon

It did not require unrestricted domain administration or general user-creation privileges.

This follows the **principle of least privilege**:

> An account should receive only the permissions required to perform its assigned responsibilities.

---

## 4. Help Desk Security Group

The following security group had already been created:

```text
GG_IT_Helpdesk
```

The Help Desk account was assigned to this group:

```text
helpdesk1 → GG_IT_Helpdesk
```

Using a security group instead of assigning permissions directly to `helpdesk1` provides several benefits:

- Easier administration
- Consistent permissions for multiple Help Desk employees
- Simplified onboarding and offboarding
- Better visibility of administrative roles
- Reduced direct permission assignments
- More scalable enterprise access management

If another Help Desk employee were added later, the account could be added to `GG_IT_Helpdesk` and inherit the same delegated role.

---

## 5. Delegation Target

Delegation was applied to the custom enterprise Users OU:

```text
abhinaylabs.internal
└── Corp
    └── Users
        ├── Disabled-Users
        ├── Finance
        ├── HR
        ├── IT
        └── Sales
```

The delegation was applied at:

```text
Corp\Users
```

This allows the delegated permission to apply to appropriate descendant user objects within the organizational structure.

The built-in domain `Users` container was not used for this delegation.

---

## 6. Delegated Permission

The `GG_IT_Helpdesk` security group was delegated the ability to reset user passwords within the `Corp\Users` administrative scope.

The resulting Active Directory permission entry was validated through:

```text
Active Directory Users and Computers
→ View
→ Advanced Features
→ Corp
→ Users
→ Properties
→ Security
→ Advanced
```

The permission entry showed:

```text
Principal:
GG_IT_Helpdesk

Applies to:
Descendant User objects

Permission:
Reset password
```

This confirms that password-reset authority was assigned to the Help Desk security group without granting unrestricted control over the OU.

---

## 7. RSAT Installation on CLIENT01

To allow Help Desk personnel to administer Active Directory without signing directly into a domain controller, **Remote Server Administration Tools (RSAT)** were installed on CLIENT01.

The required Windows capability was:

```text
Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0
```

The capability was verified with:

```powershell
Get-WindowsCapability -Online -Name "Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0"
```

Final state:

```text
State : Installed
```

The Active Directory Users and Computers console was also verified to exist:

```powershell
Test-Path "$env:SystemRoot\System32\dsa.msc"
```

Result:

```text
True
```

Because launching `dsa.msc` directly through the Run dialog did not initially resolve correctly on CLIENT01, the console was successfully launched using its full path:

```powershell
& "$env:SystemRoot\System32\dsa.msc"
```

This provided Active Directory management functionality directly from the domain-joined workstation.

---

## 8. RSAT Installation Troubleshooting

The initial RSAT installation did not complete normally.

The capability remained:

```text
State : NotPresent
```

even after an attempted installation using:

```powershell
Add-WindowsCapability -Online -Name "Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0"
```

The installation appeared to remain at an operation-running state for an extended period.

Before making configuration changes, the workstation was validated.

### Windows Servicing Services

The following services were checked:

```powershell
Get-Service wuauserv,bits,cryptsvc,TrustedInstaller |
Select-Object Name,Status,StartType
```

Windows Update, BITS, and Cryptographic Services were available, while TrustedInstaller remained configured for manual startup.

### Windows Update Connectivity

HTTPS connectivity to Microsoft's update infrastructure was tested:

```powershell
Test-NetConnection download.windowsupdate.com -Port 443
```

Result:

```text
TcpTestSucceeded : True
```

DNS resolution was also validated:

```powershell
Resolve-DnsName download.windowsupdate.com
```

Resolution completed successfully.

### WSUS Policy Check

Windows Update policy locations were checked for WSUS restrictions:

```powershell
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" -ErrorAction SilentlyContinue |
Select-Object WUServer,WUStatusServer,DisableWindowsUpdateAccess
```

and:

```powershell
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" -ErrorAction SilentlyContinue |
Select-Object UseWUServer,NoAutoUpdate
```

No configured WSUS settings were returned.

### Component Store Health

The Windows component store was checked using:

```cmd
DISM /Online /Cleanup-Image /CheckHealth
```

The component store check completed successfully.

### Resolution

Rather than repeatedly executing the same PowerShell installation command, RSAT was installed through Windows **Optional Features**.

The capability was then verified again:

```powershell
Get-WindowsCapability -Online -Name "Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0"
```

Final result:

```text
State : Installed
```

This troubleshooting process demonstrated a structured support methodology:

```text
Identify symptom
      ↓
Verify current state
      ↓
Test network and DNS
      ↓
Check policy restrictions
      ↓
Validate Windows servicing health
      ↓
Use an alternate supported installation method
      ↓
Verify final state
```

---

## 9. Help Desk Administrative Session

The delegated administrative workflow was tested from CLIENT01 while signed in as:

```text
ABHINAYLABS\helpdesk1
```

The account identity was verified using:

```cmd
whoami
```

Result:

```text
abhinaylabs\helpdesk1
```

Group membership was inspected using:

```cmd
whoami /groups
```

The output confirmed membership in:

```text
ABHINAYLABS\GG_IT_Helpdesk
```

This established that the administrative session was running under the intended Help Desk identity rather than a Domain Administrator account.

---

## 10. Delegated Password Reset Test

A realistic Help Desk password-reset scenario was performed against the synthetic HR employee:

```text
David Miller
Username: dmiller
Department: HR
```

From CLIENT01, while operating as `ABHINAYLABS\helpdesk1`, Active Directory Users and Computers was opened and the following path was selected:

```text
abhinaylabs.internal
└── Corp
    └── Users
        └── HR
            └── David Miller
```

The Help Desk account successfully used:

```text
Reset Password
```

A temporary lab password was assigned.

The following option was enabled:

```text
User must change password at next logon
```

Active Directory confirmed:

```text
The password for David Miller has been changed.
```

This demonstrated that the delegated Help Desk account could perform the intended support operation without Domain Administrator privileges.

---

## 11. Forced Password Change Validation

After the Help Desk password reset, David Miller attempted to sign into CLIENT01 using the temporary password.

Windows displayed:

```text
The user's password must be changed before signing in.
```

This validated the complete account-recovery workflow:

```text
User requires password assistance
        ↓
Help Desk authenticates using helpdesk1
        ↓
Help Desk resets user's password
        ↓
Temporary password assigned
        ↓
"User must change password at next logon"
        ↓
User authenticates
        ↓
Windows requires a new password
```

This is representative of a common enterprise Help Desk identity-support workflow.

---

## 12. Least-Privilege Validation

The Help Desk account's group memberships were validated from DC01 using:

```powershell
Get-ADPrincipalGroupMembership "helpdesk1" |
Select-Object Name,GroupScope,GroupCategory |
Sort-Object Name
```

The result showed:

```text
Domain Users
GG_IT_Helpdesk
```

The account was not a member of privileged administrative groups such as:

```text
Domain Admins
Enterprise Admins
Schema Admins
```

This provides direct evidence that the password-reset operation succeeded through **delegated permissions**, rather than unrestricted domain-level administrative privileges.

Broader user-management functionality in ADUC also remained unavailable under the Help Desk context, reinforcing the restricted administrative role.

---

## 13. Security Model

The final administrative model implemented in this phase was:

```text
ABHINAYLABS\Administrator
        │
        │ configures delegation
        ▼
GG_IT_Helpdesk
        │
        │ delegated permission
        ▼
Corp\Users
        │
        └── Reset password for descendant user objects

helpdesk1
        │
        └── Member of GG_IT_Helpdesk
```

This is preferable to:

```text
helpdesk1
        ↓
Domain Admins
```

because Domain Admin membership would grant substantially more privilege than required for routine password-reset duties.

---

## 14. Enterprise Relevance

In enterprise environments, Help Desk personnel frequently perform identity-support tasks such as:

- Password resets
- Account unlocks
- User authentication troubleshooting
- MFA assistance
- Basic group-membership administration
- User onboarding and offboarding tasks

These activities should not require unrestricted domain administrative privileges.

Delegated administration allows organizations to separate responsibilities between teams such as:

```text
Help Desk
Identity / IAM
Server Administration
Active Directory Administration
Security Operations
```

This reduces administrative risk and supports separation of duties.

---

## 15. IT Support Relevance

This phase directly reflects common Service Desk and IT Support responsibilities.

A technician may receive a ticket such as:

> User cannot authenticate because they forgot their password.

A proper support workflow may include:

1. Verify the user's identity according to organizational procedures.
2. Locate the account in Active Directory.
3. Confirm the account status.
4. Reset the password using delegated privileges.
5. Require a password change at next logon.
6. Provide the temporary credential using an approved secure method.
7. Confirm that the user can authenticate.
8. Document the action in the ticket.

The lab demonstrated the technical Active Directory portion of this workflow.

---

## 16. Security and IAM Relevance

This phase also demonstrates several foundational security and IAM concepts.

### Authentication

Authentication determines whether a user can prove their identity.

The password-reset workflow directly affects authentication credentials.

### Authorization

Authorization determines what an authenticated account is permitted to do.

`helpdesk1` was authorized to reset passwords through delegated permissions but was not granted unrestricted Active Directory administration.

### Role-Based Access Control

The Help Desk permission was assigned through:

```text
GG_IT_Helpdesk
```

rather than directly to the individual account.

This represents a role-based access model.

### Least Privilege

The Help Desk role received only the administrative rights required for its support responsibilities.

### Separation of Duties

Routine Help Desk administration remained separate from unrestricted domain administration.

---

## 17. Evidence

### Figure 21 — Help Desk Delegation

`helpdesk1` is assigned to the `GG_IT_Helpdesk` security group, which is delegated the **Reset password** extended right for descendant user objects within the `Corp\Users` OU, implementing group-based least-privilege administration.

```text
02-screenshots/21-helpdesk-delegation.png
```

### Figure 22 — Delegated Password Reset Workflow

`ABHINAYLABS\helpdesk1` uses delegated Active Directory permissions from CLIENT01 to successfully reset David Miller's password and require a password change at the next sign-in, demonstrating a typical Help Desk account-recovery workflow.

```text
02-screenshots/22-helpdesk-password-reset.png
```

### Figure 23 — Least-Privilege Validation

Group-membership validation confirms `helpdesk1` belongs to `Domain Users` and `GG_IT_Helpdesk` without privileged administrative groups, while broader user-management functionality remains unavailable in ADUC.

```text
02-screenshots/23-helpdesk-least-privilege.png
```

---

## 18. Troubleshooting Summary

### RSAT Installation Appeared Stuck

**Symptom**

`Add-WindowsCapability` remained in an operation-running state for an extended period and the capability continued to report:

```text
State : NotPresent
```

**Investigation**

The following were validated:

- Windows Update connectivity
- DNS resolution
- BITS
- Windows Update service
- Cryptographic Services
- Windows servicing state
- WSUS policy configuration
- Windows component-store health
- Pending reboot state

**Resolution**

RSAT Active Directory tools were installed using Windows Optional Features and subsequently verified as:

```text
State : Installed
```

---

### `dsa.msc` Did Not Initially Launch Through Run

**Symptom**

The ADUC console file existed:

```powershell
Test-Path "$env:SystemRoot\System32\dsa.msc"
```

Result:

```text
True
```

but Windows initially reported that it could not find `dsa.msc` when launched directly through the Run dialog.

**Resolution**

ADUC was successfully launched using the full path:

```powershell
& "$env:SystemRoot\System32\dsa.msc"
```

The issue did not prevent completion of the delegated administration lab.

---

## 19. Commands Used

### Verify Current User

```cmd
whoami
```

### Inspect Current Security Groups

```cmd
whoami /groups
```

### Check RSAT Capability

```powershell
Get-WindowsCapability -Online -Name "Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0"
```

### Attempt RSAT Installation

```powershell
Add-WindowsCapability -Online -Name "Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0"
```

### Verify ADUC Console Exists

```powershell
Test-Path "$env:SystemRoot\System32\dsa.msc"
```

### Launch ADUC Using Full Path

```powershell
& "$env:SystemRoot\System32\dsa.msc"
```

### Verify Windows Update Connectivity

```powershell
Test-NetConnection download.windowsupdate.com -Port 443
```

### Verify DNS Resolution

```powershell
Resolve-DnsName download.windowsupdate.com
```

### Check Windows Servicing Services

```powershell
Get-Service wuauserv,bits,cryptsvc,TrustedInstaller |
Select-Object Name,Status,StartType
```

### Check Component Store

```cmd
DISM /Online /Cleanup-Image /CheckHealth
```

### Validate Help Desk AD Group Membership

```powershell
Get-ADPrincipalGroupMembership "helpdesk1" |
Select-Object Name,GroupScope,GroupCategory |
Sort-Object Name
```

---

## 20. Key Lessons Learned

1. Administrative permissions should normally be assigned to security groups rather than directly to individual users.
2. Help Desk personnel do not require Domain Administrator privileges for routine password resets.
3. Active Directory delegation allows granular administrative responsibilities to be assigned to specific organizational scopes.
4. RSAT allows Active Directory administration from a domain-joined workstation without requiring administrators to work directly on a domain controller.
5. `whoami` and `whoami /groups` are useful for validating the security context of an administrative session.
6. Successful password reset does not prove least privilege by itself; privileged-group membership and administrative scope should also be validated.
7. A forced password change after a Help Desk reset reduces the period during which the temporary credential remains valid.
8. Troubleshooting should verify the underlying state before making broad configuration changes.

---

## 21. Interview Explanation

A concise interview explanation of this phase is:

> I built a delegated Help Desk administration model in my Active Directory home lab. I created a dedicated Help Desk security group and used Active Directory delegation to allow the group to reset passwords for users within our corporate Users OU without granting Domain Admin privileges. I installed RSAT on a domain-joined Windows 11 workstation and tested the workflow using a dedicated Help Desk account. The account successfully reset a user's password and forced a password change at the next logon. I then verified that the Help Desk account belonged only to Domain Users and the Help Desk security group, demonstrating least-privilege administration.

---

## 22. Phase Result

Phase 12 successfully implemented and validated:

- Dedicated Help Desk administrative identity
- Group-based role assignment
- Password-reset delegation
- RSAT-based remote Active Directory administration
- Successful delegated password reset
- Forced password change at next logon
- Restricted administrative scope
- Absence of Domain Admin membership
- Least-privilege Help Desk administration

**Phase 12 Status: COMPLETE**
