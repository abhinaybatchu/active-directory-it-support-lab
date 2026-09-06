# Phase 9 — Help Desk Account Administration

## 1. Objective

The objective of this phase was to practice common Active Directory account-administration tasks performed by IT Support and Service Desk technicians.

A dedicated Help Desk identity was created and placed into a Help Desk security group in preparation for least-privilege delegation later in the project.

The phase also practiced common user-account support operations including:

- Creating a dedicated Help Desk account
- Assigning Help Desk group membership
- Looking up Active Directory users
- Inspecting account status
- Inspecting group membership
- Resetting a user password
- Requiring password change at next logon
- Validating the password-reset workflow from a domain workstation
- Disabling and enabling a user account
- Adding and removing group memberships
- Checking account-lockout status
- Using both Active Directory Users and Computers (ADUC) and PowerShell
- Restoring temporary test changes after validation

---

## 2. Lab Environment

The following systems were used:

| System   | Role                                     | IPv4 Address     |
| -------- | ---------------------------------------- | ---------------- |
| DC01     | Active Directory Domain Controller / DNS | `192.168.170.10` |
| CLIENT01 | Domain-joined Windows 11 workstation     | `192.168.170.20` |

Domain:

```text
abhinaylabs.internal
```

NetBIOS domain:

```text
ABHINAYLABS
```

DC02 was not required for the normal account-administration exercises and remained powered off to conserve host resources.

---

## 3. Existing Active Directory Structure

The relevant organizational-unit structure was:

```text
abhinaylabs.internal
└── Corp
    ├── Groups
    └── Users
        ├── Disabled-Users
        ├── Finance
        ├── HR
        ├── IT
        └── Sales
```

Existing synthetic employee identities included:

| Department | Employee       | Username  |
| ---------- | -------------- | --------- |
| HR         | David Miller   | `dmiller` |
| Finance    | Elena Rivera   | `erivera` |
| Sales      | Joseph Daniel  | `jdaniel` |
| IT         | Pauline Hudson | `phudson` |

All identities used in this project are synthetic.

---

# Dedicated Help Desk Identity

## 4. Creating the Help Desk Account

A dedicated Help Desk user account was created in:

```text
Corp
└── Users
    └── IT
```

Account information:

```text
Display Name: Helpdesk 1
Username: helpdesk1
UPN: helpdesk1@abhinaylabs.internal
```

The account was created separately from the normal Domain Administrator identity.

This provides a foundation for implementing least-privilege Help Desk administration later in the project.

---

## 5. Help Desk Security Group

The following security group had previously been created:

```text
GG_IT_Helpdesk
```

The new account:

```text
helpdesk1
```

was added to this group.

The intended model is:

```text
helpdesk1
     ↓
GG_IT_Helpdesk
     ↓
Delegated Help Desk permissions
```

The account was intentionally **not** added to highly privileged groups such as:

```text
Domain Admins
Enterprise Admins
Administrators
Account Operators
```

Actual Help Desk permissions will be delegated later in the project.

---

## 6. Validating the Help Desk Account

The account was inspected using PowerShell:

```powershell
Get-ADUser helpdesk1 -Properties Enabled,PasswordLastSet |
Select-Object Name,SamAccountName,UserPrincipalName,Enabled,PasswordLastSet
```

The account was confirmed to be enabled.

Group membership was checked using:

```powershell
Get-ADGroupMember "GG_IT_Helpdesk"
```

The Help Desk account was returned as a member.

Membership was also checked from the account perspective:

```powershell
Get-ADPrincipalGroupMembership helpdesk1 |
Select-Object Name
```

The relevant memberships included:

```text
Domain Users
GG_IT_Helpdesk
```

This confirmed that the account existed and had the intended Help Desk group membership without unnecessary administrative privileges.

---

# User Account Administration

## 7. User Lookup and Inspection

David Miller was used as the primary synthetic user for account-administration exercises.

Account:

```text
ABHINAYLABS\dmiller
```

Department:

```text
HR
```

The account was inspected using Active Directory Users and Computers.

Relevant properties included:

- Account status
- Username
- Password-related settings
- Group membership
- Account expiration settings

The same information was inspected using PowerShell:

```powershell
Get-ADUser dmiller -Properties Enabled,LockedOut,PasswordLastSet,LastLogonDate |
Select-Object Name,SamAccountName,Enabled,LockedOut,PasswordLastSet,LastLogonDate
```

Group memberships were inspected using:

```powershell
Get-ADPrincipalGroupMembership dmiller |
Select-Object Name
```

The intended departmental membership was:

```text
GG_HR_Users
```

This demonstrated an important support workflow:

```text
Identify user
     ↓
Inspect account
     ↓
Determine current state
     ↓
Perform required action
     ↓
Validate result
```

Administrative changes should not be performed before the technician understands the current account state and the requested change.

---

# Password Administration

## 8. Password Reset Exercise

A simulated Help Desk scenario was performed in which David Miller had forgotten his domain password.

Using Active Directory Users and Computers:

```text
David Miller
→ Reset Password
```

A temporary lab password was assigned.

The following option was enabled:

```text
User must change password at next logon
```

This simulated a common enterprise Help Desk password-reset workflow.

---

## 9. Reset Password vs Change Password

A password **reset** and a password **change** are different operations.

### Password Reset

An authorized administrator or Help Desk technician assigns a new password to another user's account.

Example:

```text
Help Desk
    ↓
Reset user's password
    ↓
Temporary password assigned
```

### Password Change

The authenticated user replaces their own current or temporary password.

Example:

```text
User receives temporary password
          ↓
Signs in
          ↓
Required to change password
          ↓
Creates private password
```

Using **User must change password at next logon** prevents a Help Desk technician from permanently knowing the user's password.

---

## 10. Password Reset Validation from CLIENT01

The password-reset workflow was validated from the domain-joined workstation.

The test user:

```text
ABHINAYLABS\dmiller
```

attempted to sign into CLIENT01 using the temporary password.

Windows required the password to be changed before completing the sign-in.

After the password was changed, the user successfully authenticated.

The logged-on identity was verified using:

```cmd
whoami
```

Expected identity:

```text
abhinaylabs\dmiller
```

After the exercise, David Miller's password was restored to the standardized private lab-user credential for continued lab testing.

No passwords were recorded in the public project documentation.

---

# Account Status Administration

## 11. Disabling a User Account

A temporary account-disable scenario was performed against David Miller.

Using Active Directory Users and Computers:

```text
David Miller
→ Disable Account
```

The state was verified using:

```powershell
Get-ADUser dmiller |
Select-Object Name,Enabled
```

During the test:

```text
Enabled = False
```

This demonstrated how an account can be prevented from authenticating without immediately deleting the Active Directory object.

---

## 12. Re-Enabling the Account

After validation, David Miller was re-enabled.

The account was verified again:

```powershell
Get-ADUser dmiller |
Select-Object Name,Enabled
```

Final state:

```text
Enabled = True
```

The temporary administrative change was therefore reversed after testing.

---

## 13. Why Disable Instead of Immediately Delete?

Disabling an account prevents its use while preserving the Active Directory object.

This can preserve information such as:

- Username
- Group memberships
- Account attributes
- Organizational placement
- Administrative history
- Resource associations

For employee offboarding, disabling the account is commonly an early containment step before additional offboarding actions are completed.

A complete onboarding/offboarding workflow will be practiced later in this project.

---

# Group Membership Administration

## 14. Inspecting Group Membership

David Miller's group memberships were inspected using:

```powershell
Get-ADPrincipalGroupMembership dmiller |
Select-Object Name
```

His intended departmental security group was:

```text
GG_HR_Users
```

---

## 15. Temporary Group Membership Test

A simulated access-request exercise was performed by temporarily adding David Miller to:

```text
GG_Finance_Users
```

The change was validated using:

```powershell
Get-ADPrincipalGroupMembership dmiller |
Select-Object Name
```

During the test, the account belonged to both:

```text
GG_HR_Users
GG_Finance_Users
```

This demonstrated how security-group membership can be modified as part of an approved access request.

---

## 16. Removing Temporary Access

The temporary Finance membership was removed after testing.

David Miller's membership was checked again:

```powershell
Get-ADPrincipalGroupMembership dmiller |
Select-Object Name
```

The final departmental membership returned to:

```text
GG_HR_Users
```

This ensured that unnecessary access was not left in the environment after the exercise.

---

# Account Lockout Administration

## 17. Checking Locked Accounts

Active Directory can be searched for locked user accounts using:

```powershell
Search-ADAccount -LockedOut |
Select-Object Name,SamAccountName
```

David Miller's individual lockout state was checked using:

```powershell
Get-ADUser dmiller -Properties LockedOut |
Select-Object Name,LockedOut
```

The intended final state was:

```text
LockedOut = False
```

---

## 18. Unlocking an Account

An authorized administrator can unlock a locked Active Directory account using:

```powershell
Unlock-ADAccount -Identity dmiller
```

A deliberate authentication attack/lockout was not generated during this phase because authentication-event investigation and account-lockout troubleshooting are covered in a later dedicated phase.

That later exercise will correlate the lockout with Windows security events rather than duplicating the investigation here.

---

# Common PowerShell Commands

## 19. User Lookup

```powershell
Get-ADUser dmiller
```

---

## 20. Detailed Account Inspection

```powershell
Get-ADUser dmiller -Properties Enabled,LockedOut,PasswordLastSet,LastLogonDate
```

---

## 21. Group Membership

```powershell
Get-ADPrincipalGroupMembership dmiller |
Select-Object Name
```

---

## 22. Find Locked Accounts

```powershell
Search-ADAccount -LockedOut
```

---

## 23. Unlock an Account

```powershell
Unlock-ADAccount -Identity dmiller
```

---

## 24. Disable an Account

```powershell
Disable-ADAccount -Identity dmiller
```

---

## 25. Enable an Account

```powershell
Enable-ADAccount -Identity dmiller
```

---

## 26. Add a User to a Group

```powershell
Add-ADGroupMember -Identity "GG_HR_Users" -Members dmiller
```

---

## 27. Remove a User from a Group

```powershell
Remove-ADGroupMember -Identity "GG_HR_Users" -Members dmiller
```

---

# Least Privilege

## 28. Why `helpdesk1` Is Not a Domain Administrator

The Help Desk account was intentionally not granted Domain Admin membership.

Domain Administrators have extensive control over the Active Directory environment.

Routine Help Desk responsibilities generally do not require that level of privilege.

Examples of Help Desk responsibilities may include:

- Resetting user passwords
- Unlocking accounts
- Viewing user information
- Performing approved account-management operations

The project will later delegate specific permissions to:

```text
GG_IT_Helpdesk
```

rather than granting unrestricted domain-wide administrative privileges.

This demonstrates the principle of **least privilege**:

> Give an identity only the permissions required to perform its authorized responsibilities.

---

# Enterprise Relevance

## 29. IT Support / Service Desk Relevance

Active Directory account administration is a common responsibility in Windows enterprise support environments.

Typical tickets may include:

```text
"I forgot my password."

"My account is locked."

"I cannot sign in."

"I need access to an approved department resource."

"An employee's account needs to be disabled."

"A new support technician requires appropriate administrative access."
```

A technician should be able to:

1. Identify the correct user.
2. Verify the account state.
3. Determine the approved action.
4. Perform only the required change.
5. Validate the result.
6. Document the action.
7. Escalate when the request exceeds their authority.

---

## 30. IAM Relevance

The exercises also introduce Identity and Access Management concepts including:

- Identity lifecycle
- Authentication
- Account status
- Password administration
- Security-group membership
- Access assignment
- Least privilege
- Separation of administrative identities
- Removal of unnecessary access

These concepts will be expanded during the dedicated IAM portion of the broader cybersecurity roadmap.

---

## 31. Security Analyst / SOC Relevance

User-account changes can have security significance.

Examples include:

- Unexpected password resets
- Unauthorized group changes
- Disabled accounts being re-enabled
- Privileged group membership changes
- Repeated account lockouts
- Unauthorized account creation

Security analysts may investigate these activities using Windows Event Logs and SIEM platforms.

Later phases of this project will connect Active Directory administration with authentication and account-management events.

---

# Common Mistakes

## 32. Granting Domain Admin for Routine Support

A common lab shortcut is to make every administrative user a Domain Administrator.

This does not represent good enterprise security practice.

The `helpdesk1` identity was therefore intentionally kept outside privileged administrative groups.

---

## 33. Resetting Passwords Before Investigating

A login problem is not automatically a password problem.

The technician should first inspect:

- Account enabled/disabled state
- Lockout status
- Username/domain
- Password state
- Connectivity
- Domain-controller availability
- DNS where relevant

Only then should the appropriate corrective action be performed.

---

## 34. Leaving Temporary Group Memberships

Temporary testing or approved temporary access should not become permanent accidentally.

The Finance membership assigned to David Miller during this exercise was removed after validation.

---

## 35. Deleting Accounts Too Quickly

Deleting a user immediately can remove information that may still be required for administration, auditing or offboarding.

Disabling an account first is generally safer when immediate access must be blocked.

---

## 36. Using Shared Passwords in Production

The synthetic lab users may use standardized lab credentials to reduce unnecessary training overhead.

This is strictly a home-lab convenience.

Real enterprise users should have individual private passwords, and Help Desk personnel should never know or share users' permanent passwords.

---

# Evidence

## Figure 13 — Help Desk Account Created

File:

[Open Screenshot](../02-screenshots/13-helpdesk-account-created.png)

Description:

> Dedicated `helpdesk1` account created in the IT organizational unit for Help Desk administration exercises.

---

## Figure 14 — Help Desk Security Group Membership

File:

[Open Screenshot](../02-screenshots/14-helpdesk-security-group-membership.png)

Description:

> Dedicated `helpdesk1` identity assigned to the `GG_IT_Helpdesk` security group in preparation for least-privilege Help Desk delegation.

---

# Final State

The important final Active Directory state is:

```text
David Miller
Username: dmiller
Enabled: Yes
Locked Out: No
Department Group: GG_HR_Users

Elena Rivera
Username: erivera
Enabled: Yes
Department Group: GG_Finance_Users

Joseph Daniel
Username: jdaniel
Enabled: Yes
Department Group: GG_Sales_Users

Pauline Hudson
Username: phudson
Enabled: Yes
Department Group: GG_IT_Users

Helpdesk 1
Username: helpdesk1
Enabled: Yes
Help Desk Group: GG_IT_Helpdesk
```

`helpdesk1` was intentionally not granted Domain Administrator privileges.

Temporary password, account-state and cross-department group-membership changes performed during testing were restored after validation.

---

# Phase 9 Result

Phase 9 established the Help Desk identity-management foundation of the Abhinay Labs environment.

The phase demonstrated:

- Dedicated Help Desk identity creation
- Security-group assignment
- Active Directory user lookup
- Account-state inspection
- Password reset workflows
- Forced password change
- Domain-workstation validation
- Account disable/enable operations
- Group-membership administration
- Account-lockout inspection
- PowerShell-based account administration
- Least-privilege planning
- Restoration of temporary test changes

The environment is now ready for **Phase 10 — Group Policy**, where centralized Windows configuration and security policies will be introduced.
