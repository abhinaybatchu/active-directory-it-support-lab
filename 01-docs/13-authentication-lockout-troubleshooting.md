# Phase 13 — Authentication, Account Lockout & Event Log Troubleshooting

## 1. Objective

This phase simulated and investigated a realistic Active Directory authentication incident in the Abhinay Labs environment.

A synthetic HR employee, David Miller (`dmiller`), intentionally entered an incorrect password repeatedly on CLIENT01 until the configured domain account-lockout threshold was reached.

The incident was then investigated using Active Directory and Windows Security logs, remediated using the delegated Help Desk account created in Phase 12, and validated through successful post-recovery domain authentication.

The workflow demonstrated:

- Domain account-lockout policy enforcement
- Controlled generation of failed domain logons
- Active Directory account-state investigation
- Windows Security Event 4625 analysis
- Active Directory lockout Event 4740 analysis
- Correlation between endpoint and domain-controller evidence
- Delegated Help Desk account recovery
- Post-recovery authentication validation

---

## 2. Lab Environment

| System         | Role                                                                |
| -------------- | ------------------------------------------------------------------- |
| DC01           | Domain controller, AD DS, DNS and Security log investigation        |
| DC02           | Additional domain controller; not required for this controlled test |
| CLIENT01       | Domain-joined Windows 11 workstation                                |
| Domain         | `abhinaylabs.internal`                                              |
| NetBIOS Domain | `ABHINAYLABS`                                                       |

### Accounts Used

| Account                     | Role                                                       |
| --------------------------- | ---------------------------------------------------------- |
| `ABHINAYLABS\Administrator` | Domain Administrator used for investigation and validation |
| `ABHINAYLABS\helpdesk1`     | Delegated Help Desk account used for account recovery      |
| `ABHINAYLABS\dmiller`       | Synthetic HR employee used for the lockout scenario        |
| `CLIENT01\labadmin`         | Local administrator used for CLIENT01 log investigation    |

No real user accounts or credentials were used.

---

## 3. Scenario

The simulated support incident was:

> David Miller from HR cannot sign in to his domain workstation.

Rather than immediately resetting the user's password, the incident was investigated to determine why authentication was failing.

The intended troubleshooting workflow was:

```text
User reports authentication failure
            ↓
Verify account state
            ↓
Review endpoint authentication evidence
            ↓
Review domain-controller lockout evidence
            ↓
Correlate account, workstation and timestamps
            ↓
Perform Help Desk recovery
            ↓
Validate successful authentication
```

---

## 4. Domain Account-Lockout Policy

The effective Active Directory domain policy was verified on DC01 using:

```powershell
Get-ADDefaultDomainPasswordPolicy |
Select-Object LockoutThreshold,LockoutDuration,LockoutObservationWindow
```

The configured values were:

```text
LockoutThreshold         : 5
LockoutDuration          : 00:15:00
LockoutObservationWindow : 00:15:00
```

This means:

- Five invalid authentication attempts trigger an account lockout.
- The lockout duration is 15 minutes.
- The bad-password observation window is 15 minutes.

The policy provides protection against repeated password-guessing attempts while maintaining a manageable recovery process for legitimate users.

---

## 5. Pre-Test Account Validation

Before generating the incident, the state of David Miller's account was checked using:

```powershell
Get-ADUser "dmiller" -Properties LockedOut,Enabled,BadPwdCount,LastBadPasswordAttempt |
Select-Object SamAccountName,Enabled,LockedOut,BadPwdCount,LastBadPasswordAttempt
```

The account was verified as enabled and available for authentication before the controlled test.

A successful domain logon was also verified before deliberately generating authentication failures.

This established a known-good baseline and ensured that the later failure was caused by the controlled lockout scenario.

---

## 6. Controlled Failed-Authentication Test

On CLIENT01, five intentionally incorrect passwords were entered for:

```text
ABHINAYLABS\dmiller
```

The test was stopped immediately after the fifth failed attempt.

No additional failed attempts were generated.

This matched the configured domain threshold:

```text
5 invalid attempts
        ↓
Account lockout
```

The activity was performed only against a synthetic account in the isolated home-lab environment.

---

## 7. Active Directory Lockout Validation

After the controlled failed-logon attempts, the account state was inspected on DC01:

```powershell
Get-ADUser "dmiller" -Properties LockedOut,BadPwdCount,LastBadPasswordAttempt |
Select-Object SamAccountName,LockedOut,BadPwdCount,LastBadPasswordAttempt
```

The result confirmed:

```text
SamAccountName : dmiller
LockedOut      : True
BadPwdCount    : 5
```

The last bad-password attempt was recorded at approximately:

```text
2026-09-04 8:33:18 PM
```

Locked accounts were also queried using:

```powershell
Search-ADAccount -LockedOut |
Select-Object Name,SamAccountName,Enabled
```

The result included:

```text
David Miller
dmiller
Enabled = True
```

This established that the user's problem was an **account lockout**, not an account-disablement condition.

---

## 8. Windows Security Event 4625

The CLIENT01 Security log was investigated using Event Viewer:

```text
Event Viewer
→ Windows Logs
→ Security
```

The log was filtered for:

```text
Event ID 4625
```

Event 4625 means:

> An account failed to log on.

Five Event 4625 audit failures were observed around the time of the controlled test, corresponding with the five failed authentication attempts.

A selected event contained:

```text
Account Name:          dmiller
Account Domain:        ABHINAYLABS
Logon Type:            2
Failure Reason:        Unknown user name or bad password
Status:                0xC000006D
Sub Status:            0xC000006A
Workstation Name:      CLIENT01
Source Network Address: 127.0.0.1
Authentication Package: Negotiate
```

### Logon Type 2

Logon Type `2` represents an **interactive logon**.

This matched the test because the incorrect password was entered directly at the CLIENT01 Windows sign-in screen.

### Status `0xC000006D`

`0xC000006D` represents a general logon failure caused by incorrect authentication information.

### Sub Status `0xC000006A`

`0xC000006A` indicates that the specified account exists but an incorrect password was supplied.

This provided stronger evidence than the generic failure message alone.

---

## 9. Active Directory Security Event 4740

DC01's Security log was then investigated for:

```text
Event ID 4740
```

Event 4740 means:

> A user account was locked out.

The matching event was recorded at:

```text
2026-09-04 8:33:18 PM
```

The event identified:

```text
Account That Was Locked Out:
Account Name: dmiller

Caller Computer Name:
CLIENT01
```

The event was recorded on:

```text
DC01.abhinaylabs.internal
```

This provided domain-controller evidence that David Miller's account had entered the locked state and associated the lockout with CLIENT01.

---

## 10. Event Correlation

The endpoint and domain-controller evidence were correlated rather than investigated independently.

### CLIENT01

Five Event 4625 failures showed:

```text
dmiller
ABHINAYLABS
Logon Type 2
Incorrect password
CLIENT01
```

### DC01

Event 4740 showed:

```text
dmiller
Account locked
Caller Computer = CLIENT01
```

### Active Directory

The account state showed:

```text
LockedOut   = True
BadPwdCount = 5
```

Together, the evidence established:

```text
CLIENT01
Five incorrect interactive passwords
        │
        ▼
Security Event 4625 × 5
        │
        ▼
Domain lockout threshold reached
        │
        ▼
DC01 Security Event 4740
Caller Computer = CLIENT01
        │
        ▼
dmiller
LockedOut = True
BadPwdCount = 5
```

This correlation is more useful than relying on a single event ID because it connects the user symptom, endpoint activity, domain policy and Active Directory account state.

---

## 11. Kerberos Event 4771 Observation

DC01 was also checked for a matching:

```text
Event ID 4771
```

which can record Kerberos pre-authentication failures.

No matching Event 4771 for `dmiller` was observed for this controlled test.

No additional authentication failures were generated merely to force a specific event ID.

The investigation therefore documented the events actually produced by the environment:

```text
CLIENT01 → Event 4625
DC01     → Event 4740
```

This preserves the integrity of the lab evidence rather than assuming that every authentication scenario must produce the same set of Windows events.

---

## 12. Help Desk Recovery

After the cause of the authentication problem was identified, recovery was performed from CLIENT01 using:

```text
ABHINAYLABS\helpdesk1
```

This account had previously been granted delegated password-reset authority through:

```text
GG_IT_Helpdesk
```

The Help Desk account was not a member of Domain Admins.

Using Active Directory Users and Computers, the synthetic HR account was located at:

```text
abhinaylabs.internal
└── Corp
    └── Users
        └── HR
            └── David Miller
```

A temporary lab password was assigned using:

```text
Reset Password
```

and:

```text
User must change password at next logon
```

was enabled.

Active Directory confirmed that David Miller's password had been changed.

The recovery therefore used the delegated Help Desk permissions implemented in Phase 12 instead of unrestricted Domain Administrator privileges.

---

## 13. Forced Password Change

David Miller then authenticated to CLIENT01 using the temporary password.

Because the Help Desk reset required a password change at next logon, Windows required the user to replace the temporary password before continuing.

The user completed the password change and successfully entered a normal domain session.

This demonstrates an appropriate account-recovery workflow:

```text
Help Desk assigns temporary credential
            ↓
User authenticates
            ↓
User must change password
            ↓
Temporary password replaced
            ↓
Normal user authentication restored
```

---

## 14. Post-Recovery Authentication Validation

After recovery, CLIENT01 was accessed as:

```text
ABHINAYLABS\dmiller
```

The security context was verified:

```cmd
whoami
```

Result:

```text
abhinaylabs\dmiller
```

The logon server was checked:

```cmd
echo %LOGONSERVER%
```

Result:

```text
\\DC01
```

Domain membership was confirmed:

```cmd
systeminfo | findstr /B /C:"Domain"
```

Result:

```text
Domain: abhinaylabs.internal
```

These results confirmed that David Miller was operating in a domain-authenticated session on CLIENT01.

---

## 15. Post-Recovery Active Directory State

DC01 was used to validate the account after successful recovery:

```powershell
Get-ADUser "dmiller" -Properties LockedOut,BadPwdCount,LastLogonDate |
Select-Object SamAccountName,Enabled,LockedOut,BadPwdCount,LastLogonDate
```

The relevant result was:

```text
SamAccountName : dmiller
Enabled        : True
LockedOut      : False
BadPwdCount    : 0
```

A final locked-account search was performed:

```powershell
Search-ADAccount -LockedOut |
Select-Object Name,SamAccountName
```

David Miller was no longer returned as a locked account.

This completed the recovery validation.

### LastLogonDate Note

`LastLogonDate` was not used as the authoritative real-time timestamp for the successful authentication test.

Active Directory's `LastLogonDate` PowerShell property is based on replicated logon-timestamp information and should not be treated as a precise record of the latest authentication event.

For this lab, successful recovery was validated using the active domain-user session, logon server, account-lockout state and bad-password count.

---

## 16. Evidence

### Figure 24 — Account Lockout Validation

Domain account-lockout policy requires five invalid authentication attempts before lockout, and Active Directory confirms that the synthetic HR account `dmiller` became locked after five failed sign-in attempts.

```text
02-screenshots/24-account-lockout-validation.png
```

### Figure 25 — Active Directory Account Lockout Event

DC01 Security Event 4740 records the lockout of the synthetic HR account `dmiller` and identifies `CLIENT01` as the caller computer, correlating the account lockout with the workstation used for the controlled failed-logon test.

```text
02-screenshots/25-account-lockout-event-4740.png
```

### Figure 26 — Failed Interactive Logon Investigation

CLIENT01 Security Event 4625 records repeated failed interactive logons for `ABHINAYLABS\dmiller`. Five audit failures correspond with the configured five-attempt domain lockout threshold, while the selected event identifies Logon Type 2, an incorrect-password failure (`0xC000006A`), and CLIENT01 as the affected workstation.

```text
02-screenshots/26-failed-logon-event-4625.png
```

### Figure 27 — Help Desk Account Recovery

Following investigation of the repeated authentication failures and account lockout, the delegated `helpdesk1` account performed password recovery for the synthetic HR user David Miller from CLIENT01 without requiring Domain Administrator privileges.

```text
02-screenshots/27-helpdesk-account-recovery.png
```

### Figure 28 — Post-Recovery Authentication Validation

Following Help Desk account recovery, `ABHINAYLABS\dmiller` successfully authenticated to CLIENT01 using DC01 as the logon server, while Active Directory validation confirmed that the account was enabled, no longer locked, and had a bad-password count of zero.

```text
02-screenshots/28-post-recovery-authentication.png
```

---

## 17. Troubleshooting Methodology

The incident followed a structured troubleshooting process:

```text
1. Establish baseline
        ↓
2. Reproduce controlled authentication failure
        ↓
3. Verify account state
        ↓
4. Investigate endpoint Security events
        ↓
5. Investigate domain-controller Security events
        ↓
6. Correlate account + workstation + timestamps
        ↓
7. Identify root cause
        ↓
8. Recover using least privilege
        ↓
9. Validate successful authentication
```

### Root Cause

The synthetic user entered an incorrect password five times.

This reached the configured Active Directory account-lockout threshold.

### Evidence

The root cause was supported by:

- Five CLIENT01 Event 4625 failures
- Incorrect-password sub-status `0xC000006A`
- Active Directory `BadPwdCount = 5`
- Active Directory `LockedOut = True`
- DC01 Event 4740
- Caller Computer Name `CLIENT01`

### Resolution

The delegated Help Desk account reset the user's password and required a password change at next logon.

Successful domain authentication and the recovered Active Directory account state were then validated.

---

## 18. IT Support Relevance

This scenario represents a common Service Desk incident:

> "My password isn't working and I can't log in."

A technician should not immediately assume that the password simply needs to be reset.

Useful checks include:

- Is the account enabled?
- Is the account locked?
- How many bad-password attempts occurred?
- Which workstation generated the failures?
- Does the endpoint show failed-logon events?
- Does the domain controller show an account lockout?
- Is the user entering an old or incorrect password?
- Can the Help Desk account perform the required recovery action?
- Does authentication succeed after remediation?

This approach reduces guesswork and creates a defensible troubleshooting process.

---

## 19. Security Analyst / SOC Relevance

Repeated authentication failures can represent either a legitimate user problem or suspicious activity.

For example:

```text
Five failures from a known user's workstation
→ may indicate legitimate password mistyping

Hundreds of failures across many accounts
→ possible password spraying

Many passwords attempted against one account
→ possible brute-force activity

Repeated failures followed by successful authentication
→ potentially important investigation sequence
```

Security analysts therefore correlate:

- Username
- Source workstation
- Source IP where relevant
- Event IDs
- Failure codes
- Timestamps
- Authentication type
- Account state
- Subsequent successful authentication

The same Windows authentication events used by IT Support can therefore also contribute to security monitoring and incident investigation.

---

## 20. IAM Relevance

This phase also demonstrated several IAM concepts.

### Authentication

David Miller had to prove his identity using domain credentials.

### Account State

Authentication can fail even when credentials are otherwise valid if the account is locked or disabled.

### Password Policy

The domain controls password and lockout behavior centrally.

### Least Privilege

Recovery was performed by the delegated Help Desk role rather than Domain Admin.

### Credential Lifecycle

The temporary Help Desk password was replaced by the user during the next authentication.

---

## 21. Important Event IDs

### Event 4625 — Failed Logon

Meaning:

```text
An account failed to log on.
```

In this lab:

```text
System: CLIENT01
Account: dmiller
Logon Type: 2
Sub Status: 0xC000006A
```

### Event 4740 — Account Locked Out

Meaning:

```text
A user account was locked out.
```

In this lab:

```text
System: DC01
Account: dmiller
Caller Computer: CLIENT01
```

### Event 4771 — Kerberos Pre-Authentication Failure

This event can be useful during Kerberos authentication investigations.

No matching `dmiller` Event 4771 was observed during this specific controlled test, so it was not used as evidence.

---

## 22. Commands Used

### Domain Lockout Policy

```powershell
Get-ADDefaultDomainPasswordPolicy |
Select-Object LockoutThreshold,LockoutDuration,LockoutObservationWindow
```

### Investigate User Account State

```powershell
Get-ADUser "dmiller" -Properties LockedOut,Enabled,BadPwdCount,LastBadPasswordAttempt |
Select-Object SamAccountName,Enabled,LockedOut,BadPwdCount,LastBadPasswordAttempt
```

### Search for Locked Accounts

```powershell
Search-ADAccount -LockedOut |
Select-Object Name,SamAccountName,Enabled
```

### Unlock Account When Required

```powershell
Unlock-ADAccount -Identity "dmiller"
```

### Verify User Security Context

```cmd
whoami
```

### Verify Logon Server

```cmd
echo %LOGONSERVER%
```

### Verify Domain

```cmd
systeminfo | findstr /B /C:"Domain"
```

### Post-Recovery Validation

```powershell
Get-ADUser "dmiller" -Properties LockedOut,BadPwdCount,LastLogonDate |
Select-Object SamAccountName,Enabled,LockedOut,BadPwdCount,LastLogonDate
```

---

## 23. Key Lessons Learned

1. Account lockout is different from account disablement.
2. The effective domain lockout policy should be verified before diagnosing lockout behavior.
3. Event 4625 records failed logons and provides details such as logon type and failure status.
4. Logon Type 2 represents an interactive Windows logon.
5. Sub-status `0xC000006A` indicates an incorrect password for an existing account.
6. Event 4740 identifies an account lockout and can identify the caller computer associated with the lockout.
7. Endpoint and domain-controller logs should be correlated rather than investigated independently.
8. Not every authentication test produces every possible Windows authentication event.
9. Evidence should reflect what the environment actually logged rather than forcing expected event IDs.
10. Routine account recovery can be performed through delegated Help Desk permissions without Domain Admin access.
11. Recovery should always be followed by validation.
12. `LastLogonDate` should not be treated as an exact real-time authentication timestamp.

---

## 24. Interview Explanation

A concise interview explanation is:

> I simulated an Active Directory account-lockout incident using a synthetic domain user. I first verified that the domain policy locked accounts after five invalid attempts, then generated five controlled failed interactive logons from a Windows 11 domain workstation. CLIENT01 recorded five Event 4625 failures, including Logon Type 2 and the incorrect-password sub-status `0xC000006A`. Active Directory showed a bad-password count of five and a locked account, while DC01 recorded Event 4740 identifying CLIENT01 as the caller computer. I correlated those events, then used a delegated Help Desk account rather than Domain Admin to perform password recovery and require a password change. Finally, I verified successful domain authentication and confirmed the account was no longer locked.

---

## 25. Phase Result

Phase 13 successfully demonstrated:

- Domain account-lockout policy enforcement
- Controlled failed interactive authentication
- Active Directory lockout investigation
- Windows Security Event 4625 analysis
- Windows Security Event 4740 analysis
- Endpoint-to-domain-controller event correlation
- Failure-code interpretation
- Delegated Help Desk account recovery
- Forced password change workflow
- Successful post-recovery authentication
- Least-privilege support administration
- Structured authentication troubleshooting

**Phase 13 Status: COMPLETE**
