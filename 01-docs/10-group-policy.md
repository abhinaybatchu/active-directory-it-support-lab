# Phase 10 — Group Policy Configuration and Validation

## 1. Objective

The objective of this phase was to implement and validate centralized Windows configuration using Active Directory Group Policy.

The phase configured domain-wide password and account-lockout controls, created a computer-targeted workstation security policy, created a user-targeted HR policy, and validated that the policies were actually processed and enforced on the domain-joined CLIENT01 workstation.

The phase demonstrated:

- Group Policy Objects (GPOs)
- Group Policy Management Console (GPMC)
- Domain-level account policy
- Password policy
- Account-lockout policy
- Computer Configuration
- User Configuration
- GPO linking
- Organizational Unit targeting
- Group Policy processing
- Group Policy inheritance
- `gpupdate`
- `gpresult`
- Resultant Set of Policy (RSoP)
- SYSVOL and NETLOGON validation
- Functional endpoint testing
- Group Policy troubleshooting

---

## 2. Lab Environment

The primary systems used during this phase were:

| System   | Role                                                                   | IPv4 Address     |
| -------- | ---------------------------------------------------------------------- | ---------------- |
| DC01     | Active Directory Domain Controller / DNS / Group Policy administration | `192.168.170.10` |
| CLIENT01 | Domain-joined Windows 11 workstation                                   | `192.168.170.20` |

Domain:

```text
abhinaylabs.internal
```

NetBIOS domain:

```text
ABHINAYLABS
```

DC02 was not required for normal Group Policy testing and remained powered off to conserve host resources.

---

# Group Policy Fundamentals

## 3. What Is Group Policy?

Group Policy is an Active Directory technology used to centrally configure and enforce settings for domain users and computers.

Instead of manually configuring every workstation individually, administrators can define settings centrally and apply them to appropriate Active Directory objects.

Conceptually:

```text
Administrator
      ↓
Creates/configures GPO
      ↓
Links GPO to appropriate AD location
      ↓
Target users/computers process policy
      ↓
Configuration is centrally applied
```

---

## 4. Group Policy Object

A **Group Policy Object (GPO)** is a collection of centrally managed Windows configuration settings.

A GPO can contain settings for:

```text
Computer Configuration
```

and:

```text
User Configuration
```

Creating a GPO alone does not normally make it affect users or computers.

The GPO must also be appropriately linked and targeted.

---

## 5. Group Policy Management Console

The **Group Policy Management Console (GPMC)** provides centralized administration of Group Policy.

It was accessed on DC01 through:

```text
Server Manager
→ Tools
→ Group Policy Management
```

GPMC was used to:

- View existing GPOs
- Create new GPOs
- Link GPOs to OUs
- Inspect inheritance
- Edit policy settings
- Validate GPO status and links

---

## 6. Computer Configuration vs User Configuration

### Computer Configuration

Computer Configuration contains settings targeted at computer objects.

These settings generally apply to the computer regardless of which user signs in.

Example from this lab:

```text
CLIENT01
    ↓
Corp\Workstations
    ↓
GPO-Workstation-Security
    ↓
Computer Configuration
    ↓
Interactive logon notice
```

### User Configuration

User Configuration contains settings targeted at user objects.

These settings generally follow the user based on the user's Active Directory location and applicable policies.

Example:

```text
David Miller
    ↓
Corp\Users\HR
    ↓
GPO-HR-User-Policy
    ↓
User Configuration
    ↓
Control Panel restriction
```

This distinction was validated practically during the phase.

---

# Domain Password Policy

## 7. Initial Domain Policy

Before making changes, the effective domain password policy was inspected using:

```powershell
Get-ADDefaultDomainPasswordPolicy |
Select-Object ComplexityEnabled,
MinPasswordLength,
PasswordHistoryCount,
MinPasswordAge,
MaxPasswordAge,
LockoutThreshold,
LockoutDuration,
LockoutObservationWindow
```

The original environment included:

```text
Password complexity:        Enabled
Minimum password length:    7
Password history:           24
Minimum password age:       1 day
Maximum password age:       42 days
Account lockout threshold:  0
```

A lockout threshold of `0` meant that account lockout was not enabled.

The existing configuration was inspected before modification rather than blindly replacing the domain policy.

---

## 8. Final Password Policy

The domain password policy was configured through:

```text
Default Domain Policy
→ Computer Configuration
→ Policies
→ Windows Settings
→ Security Settings
→ Account Policies
→ Password Policy
```

The final lab configuration was:

| Setting                                     | Value                   |
| ------------------------------------------- | ----------------------- |
| Enforce password history                    | 24 passwords remembered |
| Maximum password age                        | 42 days                 |
| Minimum password age                        | 1 day                   |
| Minimum password length                     | 12 characters           |
| Password complexity                         | Enabled                 |
| Store passwords using reversible encryption | Disabled                |

The 12-character minimum strengthened the original seven-character lab configuration while retaining the existing password-history and password-age settings.

These values represent the configuration selected for this training environment and are not presented as universal values for every enterprise.

---

# Account Lockout Policy

## 9. Final Account Lockout Configuration

Account lockout was configured through:

```text
Default Domain Policy
→ Computer Configuration
→ Policies
→ Windows Settings
→ Security Settings
→ Account Policies
→ Account Lockout Policy
```

The final configuration was:

| Setting                             | Value                    |
| ----------------------------------- | ------------------------ |
| Account lockout threshold           | 5 invalid logon attempts |
| Account lockout duration            | 15 minutes               |
| Reset account lockout counter after | 15 minutes               |
| Allow Administrator account lockout | Enabled                  |

The lockout threshold provides a controlled mechanism for limiting repeated invalid authentication attempts.

It will also support a later dedicated account-lockout investigation in which failed authentication will be generated, investigated using Windows Event Logs, and resolved through the Help Desk workflow.

---

## 10. Effective Domain Policy Validation

After configuration, Group Policy processing was refreshed and the effective Active Directory domain policy was validated using:

```powershell
Get-ADDefaultDomainPasswordPolicy |
Select-Object ComplexityEnabled,
MinPasswordLength,
PasswordHistoryCount,
MinPasswordAge,
MaxPasswordAge,
LockoutThreshold,
LockoutDuration,
LockoutObservationWindow
```

The resulting values were:

```text
ComplexityEnabled         : True
MinPasswordLength         : 12
PasswordHistoryCount      : 24
MinPasswordAge            : 1.00:00:00
MaxPasswordAge            : 42.00:00:00
LockoutThreshold          : 5
LockoutDuration           : 00:15:00
LockoutObservationWindow  : 00:15:00
```

This validated the effective domain configuration rather than relying only on the Group Policy Editor.

---

# Workstation Security GPO

## 11. Creating the Workstation GPO

A custom Group Policy Object was created:

```text
GPO-Workstation-Security
```

It was linked to:

```text
Corp
└── Workstations
```

CLIENT01 was already located in this OU.

The relationship was therefore:

```text
GPO-Workstation-Security
           ↓
Corp\Workstations
           ↓
CLIENT01
```

This separated workstation-specific configuration from the Default Domain Policy.

---

## 12. Interactive Logon Policy

The following settings were configured:

```text
Computer Configuration
→ Policies
→ Windows Settings
→ Security Settings
→ Local Policies
→ Security Options
```

### Message Title

Policy:

```text
Interactive logon: Message title for users attempting to log on
```

Configured value:

```text
Abhinay Labs Authorized Access
```

### Message Text

Policy:

```text
Interactive logon: Message text for users attempting to log on
```

Configured value:

```text
This system is for authorized lab users only. Activity may be monitored for security and support purposes.
```

This demonstrated centralized configuration of an interactive logon notice.

In a production environment, organizational logon-notice wording should follow the organization's approved security, policy and legal requirements.

---

# Workstation Policy Validation

## 13. Updating Group Policy

On CLIENT01, computer policy processing was refreshed using:

```cmd
gpupdate /force
```

Computer policy completed successfully.

Because the initial validation session used the local CLIENT01 administrative account, user-side Group Policy processing produced an error during one refresh attempt.

The computer-side policy was therefore validated independently rather than assuming that the user-side error meant the workstation GPO had failed.

---

## 14. Validating Applied Computer GPOs

The effective computer-side policy was checked using:

```cmd
gpresult /r /scope computer
```

Under:

```text
Applied Group Policy Objects
```

CLIENT01 showed:

```text
GPO-Workstation-Security
Default Domain Policy
```

The output also confirmed that Group Policy had been received from:

```text
DC01.abhinaylabs.internal
```

This proved that the workstation GPO was not merely configured in GPMC—it was actually processed by CLIENT01.

---

## 15. Functional Workstation Validation

CLIENT01 was restarted after policy processing.

The workstation displayed the centrally configured interactive logon notice:

```text
Abhinay Labs Authorized Access
```

along with the configured authorization/monitoring message.

This provided functional evidence of the complete policy path:

```text
GPO created
    ↓
GPO configured
    ↓
GPO linked to Workstations OU
    ↓
CLIENT01 processes policy
    ↓
gpresult confirms application
    ↓
Configured setting appears on endpoint
```

---

# HR User Policy

## 16. Creating the HR User GPO

A second custom Group Policy Object was created:

```text
GPO-HR-User-Policy
```

It was linked to the HR organizational unit containing David Miller.

The relevant Active Directory path was:

```text
Corp
└── Users
    └── HR
        └── David Miller
```

The policy relationship was:

```text
GPO-HR-User-Policy
        ↓
HR user OU
        ↓
ABHINAYLABS\dmiller
```

---

## 17. Configuring the User Restriction

The following policy path was configured:

```text
User Configuration
→ Policies
→ Administrative Templates
→ Control Panel
```

Policy:

```text
Prohibit access to Control Panel and PC settings
```

State:

```text
Enabled
```

This provided a clear functional demonstration of a user-targeted Group Policy setting.

---

# HR Policy Validation

## 18. Processing User Policy

CLIENT01 was signed into using the synthetic HR domain user:

```text
ABHINAYLABS\dmiller
```

Group Policy was refreshed using:

```cmd
gpupdate /force
```

Both computer and user policy processing completed successfully in the domain-user session.

---

## 19. Validating Applied User Policy

The Resultant Set of Policy was inspected using:

```cmd
gpresult /r
```

The output identified:

```text
ABHINAYLABS\dmiller
```

and showed:

```text
GPO-HR-User-Policy
```

under:

```text
Applied Group Policy Objects
```

This confirmed that the HR policy was processed for the intended domain user.

---

## 20. Functional HR Restriction Test

While signed in as David Miller, Control Panel was launched using:

```text
Win + R
```

followed by:

```text
control
```

Windows blocked the operation and displayed a restriction message indicating that the operation had been cancelled due to restrictions in effect on the computer.

This demonstrated actual endpoint enforcement of the user-side GPO.

The validation path was:

```text
GPO-HR-User-Policy
        ↓
Linked to HR user OU
        ↓
David Miller signs into CLIENT01
        ↓
User policy processes successfully
        ↓
gpresult confirms GPO
        ↓
Control Panel access blocked
```

---

# Group Policy Reporting

## 21. `gpresult`

`gpresult` displays Resultant Set of Policy information showing which policies were applied to a user or computer.

Useful commands include:

### Computer Policy

```cmd
gpresult /r /scope computer
```

### User Policy

```cmd
gpresult /r /scope user
```

### General Result

```cmd
gpresult /r
```

These commands are useful when a technician needs to determine whether a GPO actually reached the target system.

---

## 22. HTML Group Policy Report

A more detailed Group Policy report can be generated.

In PowerShell:

```powershell
$Desktop = [Environment]::GetFolderPath("Desktop")
gpresult /h "$Desktop\gpresult-dmiller.html"
```

The generated HTML report can provide detailed information about:

- Applied GPOs
- User policy
- Computer policy
- Security filtering
- Policy settings
- Resultant Set of Policy information

PowerShell uses environment variables differently from Command Prompt.

For example:

```powershell
$env:USERPROFILE
```

is PowerShell syntax, while:

```cmd
%USERPROFILE%
```

is Command Prompt syntax.

---

# Group Policy Processing and Inheritance

## 23. LSDOU

A useful basic model for Group Policy processing order is:

```text
Local
  ↓
Site
  ↓
Domain
  ↓
Organizational Unit
```

This is commonly remembered as:

```text
LSDOU
```

When nested OUs exist, applicable parent OU policies are generally processed before applicable child OU policies.

Later-applied settings can normally take precedence when conflicting settings exist, subject to additional Group Policy mechanisms.

---

## 24. Additional Group Policy Controls

Group Policy behavior can also be affected by mechanisms such as:

- Enforced links
- Block Inheritance
- Security Filtering
- WMI Filtering
- Loopback processing

These mechanisms were not required for the current lab configuration but are important considerations in larger enterprise environments.

---

# Group Policy Infrastructure

## 25. SYSVOL

**SYSVOL** is a domain-controller shared folder that stores domain-wide files required by Active Directory, including Group Policy files and logon scripts.

During troubleshooting, SYSVOL accessibility was validated using:

```powershell
Test-Path "\\dc01.abhinaylabs.internal\SYSVOL"
```

and:

```powershell
Test-Path "\\abhinaylabs.internal\SYSVOL"
```

Both returned:

```text
True
```

---

## 26. NETLOGON

**NETLOGON** is a domain-controller share used for domain logon-related resources and scripts.

DC01's published shares were inspected using:

```cmd
net view \\dc01
```

The output confirmed:

```text
NETLOGON
SYSVOL
```

The shares were also confirmed locally on DC01 using:

```cmd
net share
```

This helped verify that the Group Policy infrastructure was available during troubleshooting.

---

# Troubleshooting

## 27. User Policy Refresh Error

During CLIENT01 workstation-policy validation, `gpupdate /force` reported:

```text
Computer Policy update has completed successfully.
```

while user policy did not complete successfully during the local-administrator session.

The failure was not treated as evidence that the computer GPO was broken.

The environment was validated systematically.

---

## 28. Domain Connectivity Validation

CLIENT01 connectivity to DC01 was checked using DNS and required Active Directory services.

Tests included:

```powershell
nslookup dc01.abhinaylabs.internal
```

and:

```powershell
Test-NetConnection dc01.abhinaylabs.internal -Port 53
Test-NetConnection dc01.abhinaylabs.internal -Port 88
Test-NetConnection dc01.abhinaylabs.internal -Port 389
Test-NetConnection dc01.abhinaylabs.internal -Port 445
```

The tests succeeded.

These ports represented important services including:

```text
53   DNS
88   Kerberos
389  LDAP
445  SMB
```

---

## 29. Domain Controller Discovery

Domain-controller discovery was checked using:

```cmd
nltest /dsgetdc:abhinaylabs.internal /force
```

DC01 was successfully located.

---

## 30. SYSVOL and NETLOGON Validation

SYSVOL accessibility was validated using fully qualified and domain-based UNC paths.

DC01 also advertised both:

```text
SYSVOL
NETLOGON
```

The successful tests demonstrated that no SYSVOL repair, DNS redesign or GPO recreation was required.

The workstation computer policy was subsequently confirmed with `gpresult`, and the configured logon notice was successfully enforced.

---

## 31. PowerShell vs Command Prompt Syntax

During report generation, a Command Prompt-style environment variable was initially used from PowerShell.

Command Prompt syntax:

```cmd
%USERPROFILE%
```

PowerShell syntax:

```powershell
$env:USERPROFILE
```

A reliable PowerShell method for locating the user's Desktop is:

```powershell
$Desktop = [Environment]::GetFolderPath("Desktop")
```

This distinction is useful when troubleshooting Windows administration commands across different shells.

---

# Group Policy Troubleshooting Workflow

## 32. Recommended Workflow

When a Group Policy setting does not appear to work, a structured troubleshooting process should be followed:

```text
1. Confirm the user/computer is in the intended OU.
2. Confirm the GPO exists.
3. Confirm the GPO is linked to the correct location.
4. Confirm the link and GPO are enabled.
5. Confirm the correct Computer/User Configuration section is configured.
6. Verify DNS and domain-controller connectivity.
7. Verify SYSVOL accessibility when appropriate.
8. Run gpupdate.
9. Use gpresult to inspect actual policy processing.
10. Check Applied and Denied GPOs.
11. Investigate inheritance/filtering only if required.
12. Functionally test the configured setting.
```

This prevents unnecessary changes to working infrastructure.

---

# Enterprise Relevance

## 33. Why Enterprises Use Group Policy

Group Policy allows organizations to manage large numbers of Windows systems consistently.

Examples include:

- Security configuration
- Windows settings
- Authentication-related controls
- Desktop restrictions
- Firewall configuration
- Administrative templates
- Logon configuration
- Scripts
- Software-related settings
- User-environment configuration

Centralized administration reduces inconsistent manual configuration across endpoints.

---

## 34. IT Support / Service Desk Relevance

IT Support technicians frequently troubleshoot problems caused by or related to Group Policy.

Examples include:

```text
"A setting is missing."

"My Control Panel is blocked."

"My mapped drive did not appear."

"A workstation is not receiving company settings."

"A policy works for one user but not another."
```

Useful troubleshooting tools include:

```cmd
gpupdate /force
gpresult /r
gpresult /r /scope computer
gpresult /r /scope user
```

A technician should understand that a GPO existing in Active Directory does not prove that it successfully applied to a particular user or computer.

---

## 35. Security Analyst Relevance

Group Policy can centrally enforce security controls across Windows endpoints.

Unexpected GPO modifications can also represent security risk because a malicious or unauthorized policy change could affect many systems.

Security teams may therefore monitor:

- GPO modifications
- Privileged administrative activity
- Authentication-policy changes
- Security-setting changes
- Endpoint configuration drift

---

## 36. IAM Relevance

The domain password and account-lockout controls directly affect identity authentication.

Relevant concepts include:

- Password policy
- Authentication controls
- Account lockout
- Credential protection
- Domain identity management
- Centralized policy enforcement

These controls complement the identity and least-privilege work performed elsewhere in the project.

---

# Common Mistakes

## 37. Assuming a Created GPO Automatically Applies

A GPO must be appropriately linked and targeted.

A useful mental model is:

```text
Create
  ≠
Apply
```

Instead:

```text
Create
  ↓
Configure
  ↓
Link
  ↓
Target
  ↓
Process
  ↓
Validate
```

---

## 38. Confusing Computer and User Policy

Computer Configuration targets computers.

User Configuration targets users.

Troubleshooting the wrong policy scope can waste significant time.

---

## 39. Using `gpupdate` as the Only Validation

A successful `gpupdate` does not by itself prove that the intended GPO applied.

`gpresult` should be used to inspect actual resultant policy.

Functional validation should then confirm that the intended setting took effect.

---

## 40. Rebuilding GPOs Before Diagnosing the Problem

Deleting and recreating a GPO should not be the first troubleshooting action.

The administrator should first validate:

- Object location
- GPO link
- Policy scope
- DNS
- Domain-controller connectivity
- SYSVOL
- Resultant policy
- Filtering and inheritance

---

## 41. Weakening Security to Resolve a Lab Problem

Working domain security controls should not be disabled simply to make troubleshooting easier.

The lab used validation and targeted diagnostics instead of disabling security features unnecessarily.

---

# Evidence

## Figure 15 — Domain Password and Account Lockout Policy

File:

```text
02-screenshots/15-domain-password-lockout-policy.png
```

Description:

> Domain password and account-lockout policies configured through the Default Domain Policy and validated with PowerShell. The domain enforces password complexity, a 12-character minimum password length, 24-password history, and a five-attempt account-lockout threshold with 15-minute lockout and observation periods.

---

## Figure 16 — Workstation Group Policy Configuration and Enforcement

File:

```text
02-screenshots/16-workstation-gpo-logon-banner.png
```

Description:

> `GPO-Workstation-Security` linked to the Workstations OU, verified as applied to CLIENT01 using `gpresult`, and functionally validated through the centrally configured Abhinay Labs interactive logon notice.

---

## Figure 17 — HR User Group Policy Configuration and Enforcement

File:

```text
02-screenshots/17-hr-user-gpo-restriction.png
```

Description:

> `GPO-HR-User-Policy` linked to the HR user OU, verified as applied to `ABHINAYLABS\dmiller` using `gpresult`, and functionally validated by the enforced Control Panel and Windows Settings restriction on CLIENT01.

---

# Final Configuration

The completed Group Policy design is:

```text
abhinaylabs.internal
│
├── Default Domain Policy
│   │
│   ├── Password Policy
│   │   ├── Complexity: Enabled
│   │   ├── Minimum length: 12
│   │   ├── Password history: 24
│   │   ├── Minimum age: 1 day
│   │   └── Maximum age: 42 days
│   │
│   └── Account Lockout Policy
│       ├── Threshold: 5 invalid attempts
│       ├── Duration: 15 minutes
│       └── Observation window: 15 minutes
│
└── Corp
    │
    ├── Workstations
    │   └── GPO-Workstation-Security
    │       └── Interactive logon notice
    │
    └── Users
        └── HR
            └── GPO-HR-User-Policy
                └── Control Panel/Settings restriction
```

---

# Phase 10 Result

Phase 10 successfully implemented and validated centralized Group Policy administration in the Abhinay Labs Active Directory environment.

The lab demonstrated the complete administrative lifecycle:

```text
Inspect existing configuration
        ↓
Configure policy
        ↓
Link policy
        ↓
Target appropriate AD objects
        ↓
Process policy
        ↓
Validate using gpresult
        ↓
Functionally verify endpoint behavior
        ↓
Troubleshoot systematically when required
```

The environment is now ready for **Phase 11 — File Shares, NTFS/Share Permissions and AGDLP**, where the security groups created earlier in the project will finally be connected to actual resource permissions.
