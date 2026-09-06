# Phase 14 — Employee Onboarding & Offboarding

## 1. Objective

The objective of this phase was to simulate a complete employee identity lifecycle in the Abhinay Labs Active Directory environment.

A new synthetic Finance employee was onboarded, assigned departmental access through the existing Active Directory group structure, validated from a domain workstation, and later offboarded by disabling the account, removing business access, moving the identity to the Disabled-Users OU, and confirming that authentication was blocked.

The phase demonstrated:

- Active Directory user provisioning
- OU placement
- Temporary-password workflow
- Security-group assignment
- AGDLP-based access provisioning
- Domain authentication validation
- Authorized and denied resource access
- Account disablement
- Group-membership removal
- Disabled-user OU management
- Authentication revocation
- Joiner / Leaver identity lifecycle administration

---

## 2. Lab Environment

| System         | Role                                     |
| -------------- | ---------------------------------------- |
| DC01           | Active Directory Domain Controller / DNS |
| CLIENT01       | Domain-joined Windows 11 workstation     |
| Domain         | `abhinaylabs.internal`                   |
| NetBIOS Domain | `ABHINAYLABS`                            |

The new synthetic employee was:

```text
Name: Sophia Carter
Username: scarter
UPN: scarter@abhinaylabs.internal
Department: Finance
```

No real employee identity or business data was used.

---

# Employee Onboarding

## 3. Account Provisioning

Sophia Carter was created in:

```text
abhinaylabs.internal
└── Corp
    └── Users
        └── Finance
```

The account was created as:

```text
Sophia Carter
scarter
scarter@abhinaylabs.internal
```

The account was enabled and configured with:

```text
User must change password at next logon
```

This simulated the use of a temporary onboarding credential.

---

## 4. Department Group Assignment

Sophia was added to:

```text
GG_Finance_Users
```

The existing Finance access structure was:

```text
scarter
   ↓
GG_Finance_Users
   ↓
DL_Finance_Share_RW
   ↓
Finance resource permissions
```

Permissions were therefore not assigned directly to the user account.

This reused the AGDLP design established earlier in the project.

---

## 5. Active Directory Validation

The new account was validated using:

```powershell
Get-ADUser "scarter" -Properties Enabled,Department,PasswordLastSet |
Select-Object Name,SamAccountName,UserPrincipalName,Enabled,DistinguishedName
```

The account was confirmed as:

```text
Enabled: True
OU: Finance
```

Group membership was validated using:

```powershell
Get-ADPrincipalGroupMembership "scarter" |
Select-Object Name,GroupScope,GroupCategory |
Sort-Object Name
```

The relevant memberships included:

```text
Domain Users
GG_Finance_Users
```

---

## 6. Finance Authorization Chain

The departmental groups were also validated:

```powershell
Get-ADGroupMember "GG_Finance_Users"
```

Sophia Carter was present with the existing Finance employee.

The resource group was checked using:

```powershell
Get-ADGroupMember "DL_Finance_Share_RW"
```

which confirmed that:

```text
GG_Finance_Users
```

was nested into the Finance Domain Local resource group.

The resulting authorization path was:

```text
Sophia Carter
      ↓
GG_Finance_Users
      ↓
DL_Finance_Share_RW
      ↓
SMB / NTFS permissions
      ↓
\\DC01\Finance
```

---

## 7. First Domain Logon

CLIENT01 was used to validate the newly provisioned account.

The user authenticated as:

```text
ABHINAYLABS\scarter
```

Because a password change had been required at first logon, the temporary credential was replaced before the user entered the desktop.

The active identity was verified using:

```cmd
whoami
```

Result:

```text
abhinaylabs\scarter
```

The security token was inspected using:

```cmd
whoami /groups
```

The token included:

```text
ABHINAYLABS\GG_Finance_Users
```

and the effective nested Finance resource group.

This demonstrated that the user's current logon token contained the expected departmental authorization.

---

## 8. Authorized Finance Access

The Finance departmental share was accessed from CLIENT01:

```text
\\DC01\Finance
```

Sophia successfully opened the resource.

Write access was validated by creating:

```text
Sophia-Onboarding-Test.txt
```

inside the Finance share.

This demonstrated that the new employee received the intended departmental access through group membership rather than direct ACL assignment.

---

## 9. Unauthorized HR Access

The same user attempted to access:

```text
\\DC01\HR
```

Windows denied the request.

The result demonstrated:

```text
Finance → Allowed
HR      → Denied
```

and confirmed departmental separation.

It also reinforced the difference between:

```text
Authentication
```

and:

```text
Authorization
```

Sophia was successfully authenticated to the domain but was authorized only for the resources associated with her assigned role.

---

# Employee Offboarding

## 10. Offboarding Objective

The same synthetic identity was then used to simulate an employee departure.

The account was not immediately deleted.

Instead, the workflow was:

```text
Disable account
      ↓
Remove departmental access
      ↓
Move to Disabled-Users
      ↓
Retain identity object
      ↓
Verify authentication is blocked
```

This preserves the account object while immediately revoking active access.

---

## 11. Account Disablement

Sophia Carter was disabled using Active Directory Users and Computers.

The account state was later validated using:

```powershell
Get-ADUser "scarter" -Properties Enabled |
Select-Object Name,SamAccountName,Enabled,DistinguishedName
```

The resulting state was:

```text
Enabled: False
```

---

## 12. Department Access Removal

Sophia was removed from:

```text
GG_Finance_Users
```

The standard:

```text
Domain Users
```

membership was retained.

Final membership was validated using:

```powershell
Get-ADPrincipalGroupMembership "scarter" |
Select-Object Name,GroupScope,GroupCategory |
Sort-Object Name
```

The Finance departmental group was no longer present.

This revoked the business-resource authorization previously provided through the Finance AGDLP chain.

---

## 13. Disabled-Users OU

The disabled account was moved from:

```text
Corp
└── Users
    └── Finance
```

to:

```text
Corp
└── Users
    └── Disabled-Users
```

The final Distinguished Name confirmed the new location.

This provides an organized way to separate inactive identities from active employee accounts without immediately deleting them.

---

## 14. Authentication Revocation Test

CLIENT01 was used to perform one controlled sign-in attempt using the disabled domain account.

The correct account credentials were supplied for:

```text
ABHINAYLABS\scarter
```

Windows rejected the sign-in and displayed:

```text
Your account has been disabled.
Please see your system administrator.
```

This demonstrated that account disablement successfully prevented further domain authentication.

The failure was caused by account state rather than an incorrect password.

---

# Joiner / Leaver Workflow

## 15. Completed Identity Lifecycle

The completed lifecycle was:

```text
HR / Business Request
        ↓
Create AD account
        ↓
Place in Finance OU
        ↓
Assign GG_Finance_Users
        ↓
Require password change
        ↓
User authenticates
        ↓
Finance access validated
        ↓
HR access denied
        ↓
──────────────────────
Employee departure
        ↓
Disable account
        ↓
Remove GG_Finance_Users
        ↓
Move to Disabled-Users
        ↓
Authentication denied
```

---

## 16. Why the Account Was Not Deleted

Immediate deletion was avoided because disabling an identity:

- Immediately prevents authentication
- Preserves the user object
- Preserves useful account attributes
- Allows administrative review
- Supports staged offboarding processes
- Is reversible if the request was made in error

Permanent deletion can be performed later according to organizational retention and identity-lifecycle policies.

---

## 17. IT Support Relevance

Common onboarding responsibilities may include:

- Creating user accounts
- Assigning usernames
- Setting temporary passwords
- Requiring first-logon password changes
- Placing users in appropriate OUs
- Assigning approved security groups
- Validating resource access

Common offboarding responsibilities may include:

- Disabling user accounts
- Removing departmental access
- Removing unnecessary group memberships
- Moving inactive accounts
- Validating authentication revocation
- Documenting the completed request

These are common Service Desk and IT administration workflows.

---

## 18. IAM Relevance

This phase demonstrated several IAM lifecycle concepts.

### Joiner

A new identity was created and assigned the access required by the employee's role.

### Access Provisioning

Authorization was assigned through:

```text
GG_Finance_Users
```

rather than direct resource permissions.

### Leaver

The identity was disabled and business access was removed.

### Access Revocation

Removal from the Finance group broke the user's access path:

```text
scarter
    X
GG_Finance_Users
    ↓
DL_Finance_Share_RW
    ↓
Finance resource
```

### Least Privilege

The employee received Finance access while HR access remained unavailable.

---

## 19. Security Relevance

Inactive accounts that remain enabled can create unnecessary security exposure.

A proper offboarding process reduces risks associated with:

- Former employee access
- Orphaned identities
- Excessive permissions
- Unauthorized resource access
- Forgotten credentials
- Insider-threat scenarios

Account disablement and access-group removal are therefore important security controls.

---

# Evidence

## Figure 29 — Employee Onboarding and Access Provisioning

[Open Screenshot](../02-screenshots/29-employee-onboarding-account-provisioning.png)

> A new synthetic Finance employee, Sophia Carter (`scarter`), was provisioned in the Finance OU and assigned to `GG_Finance_Users`, integrating the account into the existing departmental AGDLP access model.

---

## Figure 30 — Onboarding Access Validation

[Open Screenshot](../02-screenshots/30-onboarding-access-validation.png)

> `ABHINAYLABS\scarter` successfully authenticated to CLIENT01, received Finance security-group membership in the user token, accessed the authorized `\\DC01\Finance` share with write capability, and was denied access to the HR departmental share.

---

## Figure 31 — Employee Offboarding and Access Revocation

[Open Screenshot](../02-screenshots/31-employee-offboarding-account-disabled.png)

> Sophia Carter's account was disabled, removed from `GG_Finance_Users`, and moved to the `Disabled-Users` OU. PowerShell validation confirms the account is disabled and retains only its standard `Domain Users` membership.

---

## Figure 32 — Disabled Account Authentication Denied

[Open Screenshot](../02-screenshots/32-disabled-account-logon-denied.png)

> A controlled CLIENT01 sign-in attempt confirmed that the offboarded `scarter` domain account could no longer authenticate after account disablement and departmental access revocation.

---

## 20. Commands Used

### Inspect New User

```powershell
Get-ADUser "scarter" -Properties Enabled,Department,PasswordLastSet |
Select-Object Name,SamAccountName,UserPrincipalName,Enabled,DistinguishedName
```

### Inspect Group Membership

```powershell
Get-ADPrincipalGroupMembership "scarter" |
Select-Object Name,GroupScope,GroupCategory |
Sort-Object Name
```

### Inspect Finance Users

```powershell
Get-ADGroupMember "GG_Finance_Users"
```

### Inspect Finance Resource Group

```powershell
Get-ADGroupMember "DL_Finance_Share_RW"
```

### Verify Current User

```cmd
whoami
```

### Inspect Security Token

```cmd
whoami /groups
```

### Validate Offboarded Account

```powershell
Get-ADUser "scarter" -Properties Enabled |
Select-Object Name,SamAccountName,Enabled,DistinguishedName
```

---

## 21. Key Lessons Learned

1. Employee access should be provisioned through security groups rather than direct user permissions.
2. OU placement provides administrative organization and policy scope.
3. New users should be functionally tested after provisioning.
4. Authentication success does not imply authorization to every resource.
5. Offboarding should revoke access promptly.
6. Disabling an account is safer than immediately deleting it when the identity must be retained.
7. Business-access groups should be removed during offboarding.
8. Moving disabled users into a dedicated OU improves administration.
9. Account disablement should be validated through an actual authentication test.
10. Joiner and Leaver workflows are central to both IT Support and IAM operations.

---

## 22. Interview Explanation

> I simulated a complete employee Joiner and Leaver workflow in my Active Directory home lab. I created a new Finance user in the appropriate OU, configured a temporary password with forced password change, and assigned the employee to the Finance Global Group. Because the existing environment used AGDLP, that membership provided access to the Finance resource through the Domain Local permissions group. I validated that the employee could access Finance but was denied HR access. For offboarding, I disabled the account, removed the Finance group membership, moved the identity to a Disabled-Users OU, and confirmed from the Windows client that the disabled account could no longer authenticate.

---

## 23. Phase Result

Phase 14 successfully demonstrated:

- User onboarding
- OU-based identity placement
- Temporary-password workflow
- Department security-group assignment
- AGDLP-based access provisioning
- Authorized and unauthorized access testing
- Account disablement
- Resource-access revocation
- Disabled-user lifecycle management
- Authentication revocation
- Joiner / Leaver administration

**Phase 14 Status: COMPLETE**
