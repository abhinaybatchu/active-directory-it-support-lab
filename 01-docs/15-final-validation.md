# Phase 15 — Final Environment Validation

## 1. Objective

The objective of this phase was to perform a final end-to-end validation of the completed Abhinay Labs Active Directory IT Support Home Lab.

Rather than introducing new configuration, this phase verified that the major components built throughout the project continued to operate together correctly.

Final validation covered:

- Domain controller availability
- Active Directory replication
- DNS and domain service discovery
- Organizational Unit structure
- User and group configuration
- AGDLP group nesting
- Help Desk security group membership
- Domain authentication
- Account state
- Group Policy processing
- SMB departmental resources
- Departmental access control
- CLIENT01 domain membership
- Domain-controller discovery
- Repository integrity
- Git synchronization

A replication issue discovered during final validation was also investigated and resolved before the environment was considered healthy.

---

# Final Environment

## 2. Validated Architecture

The completed environment consisted of:

| System   | Role                                                     | IPv4 Address     |
| -------- | -------------------------------------------------------- | ---------------- |
| DC01     | Domain Controller, AD DS, DNS, lab file-share host       | `192.168.170.10` |
| DC02     | Additional Domain Controller, AD DS, DNS, Global Catalog | `192.168.170.11` |
| CLIENT01 | Windows 11 domain workstation                            | `192.168.170.20` |

Domain:

```text
abhinaylabs.internal
```

NetBIOS domain:

```text
ABHINAYLABS
```

CLIENT01 DNS configuration:

```text
Preferred DNS: 192.168.170.10
Alternate DNS: 192.168.170.11
```

The final environment therefore provided two Active Directory-integrated DNS servers and two writable domain controllers.

---

# Domain Controller Validation

## 3. Active Directory Replication

Replication health was checked using:

```cmd
repadmin /replsummary
```

During final validation, the command unexpectedly reported replication error:

```text
8524
```

Although replication had previously been healthy, DC02 had been powered off for an extended period as part of normal home-lab resource management.

The replication error therefore required investigation rather than being ignored simply because earlier tests had passed.

---

## 4. Replication Troubleshooting

The investigation was performed systematically.

The following areas were checked before making corrective changes:

- IP connectivity
- DNS configuration
- DNS name resolution
- Active Directory SRV records
- Domain-controller discovery
- Domain controller GUID DNS records
- Replication status
- Individual naming-context replication state

The investigation showed that general DNS and domain connectivity were functioning.

DC02's Directory System Agent GUID was identified as:

```text
059ff4ff-12f3-4595-a1bf-87c8b523efd4
```

The corresponding GUID-based DNS information was available, which helped confirm that the problem was not simply a missing domain-controller DNS identity.

---

## 5. Isolating the Failing Naming Context

Detailed replication information was inspected using:

```cmd
repadmin /showrepl DC01 /verbose
```

The results showed that the problem was not a complete replication failure across every Active Directory partition.

Other naming contexts had recent successful replication information, while the Schema naming context was stale and failing.

The affected naming context was:

```text
CN=Schema,CN=Configuration,DC=abhinaylabs,DC=internal
```

This narrowed the investigation from:

```text
"Active Directory replication is broken"
```

to the more precise finding:

```text
"The Schema naming context requires replication recovery."
```

This distinction prevented unnecessary changes to DNS or the overall domain configuration.

---

## 6. Targeted Replication Recovery

Instead of rebuilding the domain controller or changing otherwise healthy infrastructure, targeted replication was initiated for the affected Schema naming context.

The command used was:

```cmd
repadmin /replicate DC01 DC02 "CN=Schema,CN=Configuration,DC=abhinaylabs,DC=internal"
```

The targeted replication completed successfully.

This demonstrated the importance of isolating the specific failing component before applying remediation.

---

## 7. Post-Recovery Replication Validation

After the targeted replication completed, replication health was checked again using:

```cmd
repadmin /replsummary
```

The final replication summary reported healthy replication with no remaining failures.

Replication diagnostics were also validated using:

```cmd
dcdiag /test:Replications
```

The replication test passed.

Final result:

```text
DC01 <----> DC02
Replication healthy
0 replication failures
```

The replication issue was therefore considered resolved.

---

# DNS and Domain Services Validation

## 8. DNS Validation

The final validation confirmed that Active Directory DNS remained operational.

Validation included:

- Domain name resolution
- Domain controller name resolution
- Active Directory service records
- Domain-controller GUID DNS information
- CLIENT01 use of the domain DNS servers

The final CLIENT01 DNS configuration remained:

```text
192.168.170.10
192.168.170.11
```

This confirmed that CLIENT01 continued to use the Active Directory DNS infrastructure rather than the VMware NAT gateway as its domain DNS resolver.

---

## 9. Domain Service Connectivity

CLIENT01 connectivity to important domain services was validated.

The tested services included:

```text
DNS       TCP 53
Kerberos  TCP 88
LDAP      TCP 389
```

PowerShell `Test-NetConnection` validation confirmed successful TCP connectivity to the required services.

This provided network-level evidence that CLIENT01 could reach the services required for domain operations.

---

# Active Directory Object Validation

## 10. Organizational Unit Structure

The final Active Directory OU structure was verified.

The environment retained the intended organization:

```text
abhinaylabs.internal
├── Corp
│   ├── Groups
│   ├── Servers
│   ├── Users
│   │   ├── Disabled-Users
│   │   ├── Finance
│   │   ├── HR
│   │   ├── IT
│   │   └── Sales
│   └── Workstations
└── Domain Controllers
```

This confirmed that users, workstations, groups, disabled identities, and server objects remained logically organized.

---

## 11. Core Synthetic Users

The original synthetic department users remained available:

| Department | User           | Username  |
| ---------- | -------------- | --------- |
| HR         | David Miller   | `dmiller` |
| Finance    | Elena Rivera   | `erivera` |
| Sales      | Joseph Daniel  | `jdaniel` |
| IT         | Pauline Hudson | `phudson` |

The onboarding/offboarding workflow also used the synthetic Finance employee:

```text
Sophia Carter
scarter
```

whose account was disabled and moved to the Disabled-Users OU during offboarding.

---

# Group and AGDLP Validation

## 12. Security Groups

The departmental Global groups remained present:

```text
GG_HR_Users
GG_Finance_Users
GG_Sales_Users
GG_IT_Users
GG_IT_Helpdesk
```

Resource-oriented Domain Local groups included:

```text
DL_HR_Share_RW
DL_Finance_Share_RW
```

---

## 13. AGDLP Validation

The final validation confirmed the intended AGDLP relationships.

HR:

```text
David Miller
    ↓
GG_HR_Users
    ↓
DL_HR_Share_RW
    ↓
HR resource permissions
```

Finance:

```text
Elena Rivera
    ↓
GG_Finance_Users
    ↓
DL_Finance_Share_RW
    ↓
Finance resource permissions
```

This preserved the design:

```text
Accounts
   ↓
Global Groups
   ↓
Domain Local Groups
   ↓
Permissions
```

rather than assigning departmental permissions directly to individual users.

---

# Help Desk Validation

## 14. Help Desk Security Model

The Help Desk account remained associated with:

```text
GG_IT_Helpdesk
```

The account was not a member of Domain Admins.

The delegated Help Desk design therefore continued to follow least privilege.

Earlier project validation had already demonstrated that the Help Desk role could perform its delegated password-reset function while being unable to perform unrestricted administrative actions such as creating arbitrary domain users.

Final validation confirmed that the supporting group structure remained intact.

---

# Group Policy Validation

## 15. Domain Account Policy

The domain account policy remained configured with the project's intended controls.

Validated settings included:

```text
Minimum password length: 12
Password history: 24
Password complexity: Enabled
Account lockout threshold: 5
Account lockout duration: 15 minutes
Reset lockout counter after: 15 minutes
```

These settings supported the authentication and account-lockout troubleshooting scenarios used later in the project.

---

## 16. Workstation and User GPO Processing

Group Policy processing was validated from CLIENT01 using commands including:

```cmd
gpupdate /force
```

and:

```cmd
gpresult /r
```

Computer and user policy processing remained functional.

This confirmed that the domain workstation could continue to locate the domain infrastructure and process applicable Group Policy.

---

# File Share and Authorization Validation

## 17. Departmental SMB Resources

The project retained the departmental shares:

```text
\\DC01\HR
```

and:

```text
\\DC01\Finance
```

The corresponding AGDLP-based authorization model remained in place.

---

## 18. Authorized and Unauthorized Access

David Miller's HR account was used to validate authorization behavior.

Expected access:

```text
\\DC01\HR
→ Allowed
```

Expected unauthorized access:

```text
\\DC01\Finance
→ Denied
```

Both behaviors were confirmed.

This demonstrated that successful domain authentication did not automatically grant access to every domain resource.

Authentication answered:

```text
Who is the user?
```

Authorization answered:

```text
What is the authenticated user allowed to access?
```

The final result confirmed that the HR and Finance resource boundaries remained effective.

---

## 19. Mapped Drive Validation

The HR Group Policy drive mapping remained functional.

The mapping was:

```text
H:
```

to:

```text
\\DC01\HR
```

The mapping could be validated using:

```cmd
net use
```

This provided additional evidence that DNS, authentication, Group Policy, SMB connectivity, and authorization were functioning together.

---

# CLIENT01 Validation

## 20. Domain Authentication

David Miller successfully authenticated to CLIENT01 as a domain user.

The authenticated identity was validated using:

```cmd
whoami
```

Expected domain identity:

```text
abhinaylabs\dmiller
```

---

## 21. Logon Server

The domain logon server was validated using:

```cmd
echo %LOGONSERVER%
```

This confirmed that CLIENT01 was using domain infrastructure rather than only a local Windows account.

---

## 22. Domain Controller Discovery

CLIENT01 domain-controller discovery was validated using:

```cmd
nltest /dsgetdc:abhinaylabs.internal
```

Successful discovery demonstrated that CLIENT01 could locate Active Directory domain-controller services.

---

## 23. Domain Membership

CLIENT01 remained joined to:

```text
abhinaylabs.internal
```

This confirmed that the workstation's domain relationship remained intact after the complete sequence of project changes and troubleshooting exercises.

---

# Account State Validation

## 24. David Miller Account

The David Miller account had previously been intentionally locked during the authentication troubleshooting phase.

After recovery, the final account state was verified as:

```text
Enabled: True
LockedOut: False
BadPwdCount: 0
```

A search for locked accounts did not identify David as currently locked.

CLIENT01 domain authentication also succeeded after recovery.

This confirmed that the troubleshooting exercise did not leave the primary HR test account in a broken state.

---

# Repository Validation

## 25. Repository Inventory

The repository was reviewed to verify that the project documentation and screenshots were present in the expected structure.

The repository contained:

```text
README.md
.gitignore
01-docs/
02-screenshots/
03-scripts/
04-sample-logs/
05-reports/
06-assets/
```

The technical documentation contained the completed phase files through the onboarding/offboarding workflow.

The screenshot collection contained the project evidence sequence from:

```text
01-vmware-ad-lab-network.png
```

through:

```text
32-disabled-account-logon-denied.png
```

Empty repository folders were retained as part of the standard portfolio structure but were not filled with artificial content solely to make them non-empty.

---

## 26. Documentation Audit

The technical documentation was reviewed for consistency with the completed environment.

The final network-design documentation was updated to reflect the completed three-system architecture:

```text
DC01
DC02
CLIENT01
```

and the final redundant DNS design.

The authentication troubleshooting documentation was also adjusted so that failed authentication evidence was not presented as automatically proving benign password mistyping.

The final wording recognizes that authentication failures require contextual analysis before determining cause.

After this audit, the Phase 1–14 technical documentation was considered complete and locked against unnecessary cosmetic rewriting.

---

# Git Integrity Validation

## 27. Git Repository State

Git status was checked after project documentation work.

The expected final repository state is:

```text
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
```

This confirms that:

- Local project changes were committed
- The local branch matched the remote branch
- No unintended files remained untracked or modified

Temporary audit files were removed rather than committed to the public repository.

---

# Final Troubleshooting Case

## 28. Replication Error 8524

The most significant issue discovered during final validation was Active Directory replication error:

```text
8524
```

The important troubleshooting sequence was:

```text
repadmin /replsummary
        ↓
Replication failure detected
        ↓
Validate DNS and connectivity
        ↓
Validate AD DNS/SRV/GUID information
        ↓
repadmin /showrepl DC01 /verbose
        ↓
Isolate stale Schema naming context
        ↓
Targeted Schema replication
        ↓
repadmin /replsummary
        ↓
dcdiag /test:Replications
        ↓
Replication healthy
```

The recovery command was:

```cmd
repadmin /replicate DC01 DC02 "CN=Schema,CN=Configuration,DC=abhinaylabs,DC=internal"
```

This was preferable to immediately rebuilding DC02, changing healthy DNS configuration, or making broad infrastructure changes.

---

## 29. Troubleshooting Lesson

The primary lesson from the replication incident was:

> A broad error should be narrowed to the specific failing component before remediation is attempted.

The initial symptom was a replication error.

Further evidence showed:

- Network connectivity was functioning
- DNS resolution was functioning
- Domain service records were available
- Domain-controller GUID DNS information was available
- Not every naming context was failing
- The Schema naming context required targeted recovery

The remediation was therefore specific to the observed failure.

This evidence-based approach is applicable to IT Support, systems administration, NOC, and security operations work.

---

# Enterprise Relevance

## 30. IT Support Relevance

The completed validation exercised skills relevant to IT Support and Service Desk roles:

- Windows domain authentication
- DNS troubleshooting
- Domain connectivity
- Group Policy validation
- Account lockout investigation
- Password-reset workflows
- Least-privilege administration
- File-share troubleshooting
- Access-control validation
- User onboarding and offboarding
- Windows administrative tools
- PowerShell and command-line validation

---

## 31. Security Analyst / SOC Relevance

The project also developed security-analysis fundamentals through:

- Authentication-event investigation
- Account-lockout analysis
- Security Event Log review
- Least privilege
- Access control
- Separation of authentication and authorization
- Correlation of user, workstation, timestamp, and account state
- Evidence-based troubleshooting

---

## 32. IAM Relevance

Identity and Access Management concepts demonstrated include:

- User provisioning
- User deprovisioning
- Account disabling
- Organizational Units
- Group-based access
- AGDLP
- Least privilege
- Delegated administration
- Password reset
- Account lockout
- Authentication
- Authorization
- Access removal during offboarding

---

## 33. NOC Relevance

Infrastructure troubleshooting included:

- Static IPv4 configuration
- DNS
- Service connectivity
- Domain-controller availability
- Replication health
- Server-to-server communication
- Client-to-server communication

These skills overlap with entry-level network and infrastructure monitoring responsibilities.

---

# Final Project Validation Result

## 34. Validation Summary

| Area                              | Final Result |
| --------------------------------- | ------------ |
| DC01 availability                 | PASS         |
| DC02 availability                 | PASS         |
| Active Directory replication      | PASS         |
| DNS resolution                    | PASS         |
| AD service discovery              | PASS         |
| CLIENT01 domain membership        | PASS         |
| Domain authentication             | PASS         |
| Group Policy processing           | PASS         |
| OU structure                      | PASS         |
| User/group structure              | PASS         |
| AGDLP relationships               | PASS         |
| Help Desk least-privilege model   | PASS         |
| HR share authorization            | PASS         |
| Finance access denial for HR user | PASS         |
| HR mapped drive                   | PASS         |
| Account recovery state            | PASS         |
| Repository inventory              | PASS         |
| Documentation audit               | PASS         |
| Git synchronization               | PASS         |

---

# Interview Explanation

A concise interview explanation of final validation is:

> After completing the Active Directory lab, I performed end-to-end validation rather than assuming each earlier phase was still healthy. I checked replication, DNS, domain discovery, authentication, Group Policy, AGDLP permissions, SMB access, account state and Git repository integrity. During that process, repadmin reported replication error 8524. I verified DNS, connectivity and domain-controller records, then used repadmin /showrepl to narrow the issue to a stale Schema naming context. Instead of rebuilding the server or changing healthy DNS settings, I initiated targeted Schema replication and then confirmed zero replication failures with repadmin and a passing dcdiag replication test. I also revalidated CLIENT01 authentication, GPO processing and departmental access controls before considering the environment complete.

---

# Key Lessons Learned

1. Final validation is necessary because a component that worked earlier may no longer be healthy.
2. Active Directory health depends on DNS, network connectivity, authentication services, and replication working together.
3. Replication errors should be narrowed to the affected naming context before remediation.
4. A successful DNS test does not automatically prove healthy Active Directory replication.
5. A successful login does not automatically prove correct resource authorization.
6. AGDLP separates user membership from resource permissions and makes access easier to manage.
7. Least-privilege Help Desk delegation is safer than granting broad administrative privileges.
8. Authentication events should be interpreted using surrounding evidence rather than isolated event counts.
9. Onboarding and offboarding are security workflows as well as administrative workflows.
10. Technical troubleshooting should end with validation that the original problem is actually resolved.
11. Documentation and Git state should also be validated before a portfolio project is considered complete.

---

# Phase Result

The completed Abhinay Labs environment successfully demonstrated an integrated Windows enterprise identity and IT Support workflow:

```text
VMware Network
      ↓
DC01 + DC02
      ↓
AD DS + DNS
      ↓
Replication
      ↓
Users + Groups + OUs
      ↓
CLIENT01 Domain Membership
      ↓
Authentication
      ↓
Group Policy
      ↓
AGDLP Authorization
      ↓
Help Desk Delegation
      ↓
Account Troubleshooting
      ↓
Employee Lifecycle Management
      ↓
Final Environment Validation
```

The environment passed final validation after the discovered replication issue was investigated, isolated, remediated, and retested.

**Phase 15 Status: COMPLETE**
