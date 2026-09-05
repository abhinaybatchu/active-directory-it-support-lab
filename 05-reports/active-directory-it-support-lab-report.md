# Active Directory IT Support Home Lab — Final Project Report

## Project Information

**Project:** Active Directory IT Support Home Lab
**Organization:** Abhinay Labs — Fictional Lab Environment
**Domain:** `abhinaylabs.internal`
**Platform:** VMware Workstation
**Primary Technologies:** Windows Server 2025, Windows 11, Active Directory Domain Services, DNS, Group Policy, PowerShell, SMB, NTFS, RSAT, Windows Event Viewer
**Project Status:** Complete

> Abhinay Labs is a fictional organization created solely for this simulated enterprise IT environment. All employee identities and organizational data used in this project are synthetic.

---

# 1. Executive Summary

This project created a multi-VM Windows Active Directory environment designed to simulate common enterprise IT Support, Service Desk, identity administration, access-control, and troubleshooting workflows.

The environment progressed from a basic private VMware network into an integrated Windows domain containing two writable domain controllers and a Windows 11 domain workstation.

The completed environment included:

- Active Directory Domain Services
- Active Directory-integrated DNS
- Two writable domain controllers
- Active Directory replication
- Organizational Units
- Departmental users and security groups
- AGDLP-based resource authorization
- Windows workstation domain membership
- Domain-user authentication
- Group Policy
- SMB departmental resources
- NTFS and share permissions
- Delegated Help Desk administration
- Least-privilege access
- Password resets
- Account-lockout investigation
- Windows authentication-event analysis
- Employee onboarding and offboarding
- Replication troubleshooting
- Final end-to-end validation

The project emphasized not only configuration, but also validation and troubleshooting. Each major workflow was tested from the perspective of the administrator or end user to confirm that the intended behavior actually occurred.

---

# 2. Project Objectives

The primary objective was to develop practical Windows enterprise support skills relevant to entry-level roles such as:

- IT Support Technician
- Service Desk Analyst
- Help Desk Analyst
- Junior Systems Support
- IT Security Support
- IAM Analyst
- NOC Analyst

Specific technical objectives included:

1. Build a private Windows enterprise lab using VMware Workstation.
2. Deploy a Windows Server Active Directory forest and domain.
3. Configure Active Directory-integrated DNS.
4. Design a structured OU hierarchy.
5. Administer users and security groups.
6. Implement group-based access using AGDLP.
7. Join a Windows 11 workstation to the domain.
8. Validate domain authentication.
9. Deploy and troubleshoot Group Policy.
10. Configure departmental SMB resources.
11. Apply share and NTFS permissions.
12. Delegate Help Desk administration using least privilege.
13. Investigate password and account-lockout problems.
14. Analyze relevant Windows Security events.
15. Simulate employee onboarding and offboarding.
16. Add a second domain controller and validate replication.
17. Troubleshoot an Active Directory replication failure.
18. Perform final end-to-end validation of the environment.

---

# 3. Lab Architecture

The completed environment used the VMware Workstation VMnet8 NAT network.

```text
                           Internet
                              |
                              v
                    VMware NAT Gateway
                      192.168.170.2
                              |
                              v
                  VMnet8 192.168.170.0/24
                              |
              +---------------+---------------+
              |                               |
              v                               v
            DC01                            DC02
       192.168.170.10                  192.168.170.11
       Windows Server                  Windows Server
       AD DS + DNS                     AD DS + DNS
       File Shares                     Global Catalog
              |                               |
              +-------- Replication ----------+
                              |
                              v
                           CLIENT01
                       192.168.170.20
                         Windows 11
                              |
                    Domain Workstation
```

## Systems

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

CLIENT01 DNS:

```text
Preferred DNS: 192.168.170.10
Alternate DNS: 192.168.170.11
```

---

# 4. Network and DNS Design

The environment used:

```text
VMnet8
192.168.170.0/24
```

with VMware NAT gateway:

```text
192.168.170.2
```

The VMware DHCP range began at:

```text
192.168.170.128
```

The infrastructure addresses selected for DC01, DC02, and CLIENT01 were therefore outside the automatic VMware DHCP allocation range.

Active Directory DNS was provided by:

```text
DC01 — 192.168.170.10
DC02 — 192.168.170.11
```

CLIENT01 used these domain DNS servers rather than the VMware NAT gateway as its final DNS configuration.

This allowed the workstation to resolve Active Directory records required for domain-controller discovery, Kerberos, LDAP, and other domain services.

---

# 5. Active Directory Deployment

DC01 was deployed as the first domain controller for:

```text
abhinaylabs.internal
```

Active Directory Domain Services and DNS were installed and the server was promoted to create the new forest.

DC02 was subsequently deployed as an additional writable domain controller.

The completed domain therefore contained:

```text
DC01
  ⇅
Active Directory Replication
  ⇅
DC02
```

DC02 also provided DNS and Global Catalog functionality.

The second domain controller allowed replication, redundant domain DNS, and controlled domain-controller failover behavior to be practiced.

---

# 6. Organizational Unit Design

The domain was organized using the following OU hierarchy:

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

The structure separated:

- Users by department
- Disabled accounts
- Security groups
- Workstations
- Servers
- Domain controllers

This supported logical administration, Group Policy targeting, employee lifecycle management, and delegated Help Desk operations.

---

# 7. User and Group Administration

Synthetic employee accounts were created for multiple departments.

| Department | Employee       | Username  |
| ---------- | -------------- | --------- |
| HR         | David Miller   | `dmiller` |
| Finance    | Elena Rivera   | `erivera` |
| Sales      | Joseph Daniel  | `jdaniel` |
| IT         | Pauline Hudson | `phudson` |

Department Global groups included:

```text
GG_HR_Users
GG_Finance_Users
GG_Sales_Users
GG_IT_Users
GG_IT_Helpdesk
```

Resource Domain Local groups included:

```text
DL_HR_Share_RW
DL_Finance_Share_RW
```

Users were assigned to their corresponding departmental Global groups.

---

# 8. AGDLP Authorization

Resource authorization was designed using:

```text
A → G → DL → P
```

where:

```text
A  = Accounts
G  = Global Groups
DL = Domain Local Groups
P  = Permissions
```

Example HR access path:

```text
David Miller
      ↓
GG_HR_Users
      ↓
DL_HR_Share_RW
      ↓
HR resource permissions
```

The same model was used for Finance.

This separated employee membership from resource permissions and avoided assigning departmental permissions directly to individual user accounts.

---

# 9. Windows Workstation Domain Integration

CLIENT01 was deployed using Windows 11 Enterprise Evaluation and joined to:

```text
abhinaylabs.internal
```

The workstation computer object was placed under:

```text
Corp\Workstations
```

Domain authentication was tested using David Miller.

Validation included:

```cmd
whoami
```

```cmd
echo %LOGONSERVER%
```

and:

```cmd
nltest /dsgetdc:abhinaylabs.internal
```

These tests confirmed:

- Domain-user authentication
- Domain membership
- Domain-controller discovery
- Domain logon-server use

---

# 10. Group Policy

Group Policy was used to implement both security and user configuration.

The project included:

- Domain password policy
- Account lockout policy
- Workstation security configuration
- Interactive logon configuration
- HR-specific user restriction
- HR mapped-drive configuration

The domain account policy included:

```text
Minimum password length:       12
Password history:              24
Password complexity:           Enabled
Account lockout threshold:     5
Account lockout duration:      15 minutes
Lockout counter reset:         15 minutes
```

Group Policy application was validated using:

```cmd
gpupdate /force
```

and:

```cmd
gpresult /r
```

The HR-specific policy was also tested from CLIENT01 to confirm that the intended restriction applied to the HR user.

---

# 11. Departmental File Resources

Two departmental SMB resources were configured on DC01 for the lab:

```text
\\DC01\HR
```

and:

```text
\\DC01\Finance
```

Both share permissions and NTFS permissions were configured.

Resource permissions were assigned through Domain Local groups rather than directly to individual employees.

The HR user was validated with:

```text
\\DC01\HR
→ Allowed
```

and:

```text
\\DC01\Finance
→ Denied
```

This demonstrated that authentication and authorization are separate controls.

David could successfully authenticate to the domain while still being denied access to a resource for which he had not been authorized.

The HR resource was additionally mapped through Group Policy:

```text
H: → \\DC01\HR
```

---

# 12. Help Desk Administration

A dedicated Help Desk account was created:

```text
helpdesk1
```

and assigned to:

```text
GG_IT_Helpdesk
```

The Help Desk account was deliberately not made a Domain Administrator.

Administrative permissions were delegated over the user OU for password-reset operations.

The resulting model was:

```text
Help Desk
    ↓
Delegated password-reset capability
    ↓
No unrestricted domain administration
```

Testing demonstrated:

```text
Password reset     → Allowed
Create domain user → Denied
Domain Admin       → No
```

This implemented the principle of least privilege.

---

# 13. Authentication and Account Lockout Troubleshooting

A controlled password-failure scenario was performed using David Miller.

The account eventually entered a locked state according to the configured domain lockout policy.

The investigation used both the domain controller and CLIENT01.

Relevant Windows Security events included:

```text
4625 — Failed logon
4740 — User account locked out
4724 — Password reset attempt
```

The lab demonstrated that the location of an event matters.

For the tested interactive authentication scenario:

```text
Failed interactive logon
        ↓
CLIENT01
        ↓
Event 4625
```

while:

```text
Account lockout
        ↓
Domain Controller
        ↓
Event 4740
```

The investigation correlated:

- User identity
- Workstation
- Timestamp
- Event ID
- Failure information
- Account state

The account was subsequently recovered and authentication was retested.

Final account state included:

```text
Enabled: True
LockedOut: False
BadPwdCount: 0
```

---

# 14. Employee Onboarding and Offboarding

A synthetic Finance employee was used for the identity lifecycle workflow:

```text
Sophia Carter
scarter
```

## Onboarding

The onboarding process included:

```text
Create account
      ↓
Place in Finance OU
      ↓
Assign GG_Finance_Users
      ↓
Receive AGDLP-based Finance access
      ↓
Validate authentication
      ↓
Validate Finance access
      ↓
Validate HR denial
```

## Offboarding

The offboarding process included:

```text
Disable account
      ↓
Remove departmental membership
      ↓
Move to Disabled-Users OU
      ↓
Attempt authentication
      ↓
Logon denied
```

This demonstrated that identity lifecycle management directly affects enterprise security.

---

# 15. Active Directory Replication Troubleshooting

One of the most important troubleshooting scenarios occurred during final environment validation.

Running:

```cmd
repadmin /replsummary
```

reported replication error:

```text
8524
```

Replication had previously been healthy, but DC02 had been powered off for an extended period during normal home-lab resource management.

The error was investigated systematically rather than immediately changing the environment.

Validation included:

- IP connectivity
- DNS configuration
- DNS resolution
- Active Directory SRV records
- Domain-controller GUID DNS information
- Individual replication relationships
- Naming-context replication state

Detailed replication information was inspected using:

```cmd
repadmin /showrepl DC01 /verbose
```

The evidence showed that other naming contexts had recent successful replication information while the Schema naming context was stale and failing.

The affected naming context was:

```text
CN=Schema,CN=Configuration,DC=abhinaylabs,DC=internal
```

A targeted replication operation was performed:

```cmd
repadmin /replicate DC01 DC02 "CN=Schema,CN=Configuration,DC=abhinaylabs,DC=internal"
```

The operation completed successfully.

Replication was then retested using:

```cmd
repadmin /replsummary
```

and:

```cmd
dcdiag /test:Replications
```

Final result:

```text
Replication healthy
0 replication failures
```

No screenshot of the original 8524 condition was recreated after the issue had been resolved. The troubleshooting sequence is documented based on the actual lab investigation.

---

# 16. Troubleshooting Methodology

A major focus throughout the project was troubleshooting from evidence rather than immediately changing configuration.

A simplified methodology used throughout the lab was:

```text
Identify the symptom
        ↓
Verify current state
        ↓
Test lower-level dependencies
        ↓
Collect evidence
        ↓
Narrow the failure
        ↓
Apply the smallest appropriate correction
        ↓
Retest
        ↓
Confirm the original issue is resolved
```

For domain problems, this often meant checking:

```text
IP configuration
      ↓
DNS
      ↓
Service connectivity
      ↓
Domain-controller discovery
      ↓
Authentication
      ↓
Group Policy / permissions / replication
      ↓
Relevant logs
```

This reduced unnecessary configuration changes and made troubleshooting more repeatable.

---

# 17. Final Validation

After all project phases were complete, the environment was validated as an integrated system.

| Component                    | Result |
| ---------------------------- | ------ |
| DC01 availability            | PASS   |
| DC02 availability            | PASS   |
| AD replication               | PASS   |
| DNS                          | PASS   |
| AD service discovery         | PASS   |
| CLIENT01 domain membership   | PASS   |
| Domain authentication        | PASS   |
| Group Policy                 | PASS   |
| OU structure                 | PASS   |
| Users and groups             | PASS   |
| AGDLP                        | PASS   |
| Help Desk least privilege    | PASS   |
| HR resource access           | PASS   |
| Finance denial for HR user   | PASS   |
| HR mapped drive              | PASS   |
| Recovered user account state | PASS   |
| Documentation audit          | PASS   |
| Git repository integrity     | PASS   |

The final environment was considered healthy after the discovered replication issue was remediated and retested.

---

# 18. Security Principles Demonstrated

The project incorporated several important security principles.

## Least Privilege

Help Desk permissions were delegated according to required job functions instead of granting Domain Admin membership.

## Group-Based Authorization

Resource permissions were assigned through security groups using AGDLP rather than directly to individual users.

## Authentication vs. Authorization

Successful domain authentication did not automatically provide access to departmental resources.

## Account Lifecycle Management

Employee access was provisioned during onboarding and removed during offboarding.

## Account Lockout

Repeated authentication failures triggered the configured domain account-lockout policy.

## Evidence-Based Investigation

Authentication events and replication failures were investigated using surrounding technical evidence before determining cause.

## Network Isolation

The lab remained on a private VMware NAT network and did not require direct public exposure of Active Directory services.

---

# 19. Enterprise Relevance

## IT Support / Service Desk

Relevant skills included:

- Domain-user administration
- Password resets
- Account lockouts
- Windows workstation domain joining
- Group Policy troubleshooting
- File-share troubleshooting
- DNS troubleshooting
- Access validation
- Employee onboarding and offboarding
- RSAT
- Windows administrative tools

## IAM

Relevant skills included:

- Authentication
- Authorization
- Provisioning
- Deprovisioning
- Group-based access
- AGDLP
- Least privilege
- Delegated administration
- Access removal

## NOC / Infrastructure Support

Relevant skills included:

- IP configuration
- DNS
- Service connectivity
- Server availability
- Domain-controller communication
- Replication monitoring
- Infrastructure troubleshooting

## Security Analyst / SOC

Relevant skills included:

- Windows Security Event investigation
- Failed-authentication analysis
- Account-lockout investigation
- Access control
- Least privilege
- Authentication-event correlation
- Evidence-based troubleshooting

---

# 20. Skills and Technologies

The project provided hands-on practice with:

- VMware Workstation
- Windows Server 2025
- Windows 11
- Active Directory Domain Services
- DNS
- Organizational Units
- Active Directory users and groups
- AGDLP
- Group Policy
- SMB
- NTFS permissions
- Windows Event Viewer
- RSAT
- PowerShell
- Command Prompt
- `repadmin`
- `dcdiag`
- `nltest`
- `gpupdate`
- `gpresult`
- `Test-NetConnection`
- `Resolve-DnsName`
- `nslookup`
- `net use`
- Git
- GitHub
- Visual Studio Code
- Markdown

---

# 21. Key Lessons Learned

1. Active Directory depends heavily on correct DNS configuration.
2. A successful ping does not prove that an application or domain service is healthy.
3. DNS resolution does not automatically prove healthy Active Directory replication.
4. Authentication and authorization solve different problems.
5. Group-based permissions scale better than assigning access directly to individual users.
6. Least-privilege delegation is preferable to giving Help Desk personnel broad administrative access.
7. Windows authentication events must be interpreted in the context of where the authentication attempt occurred.
8. Failed authentication attempts alone do not prove whether activity is benign or malicious.
9. Employee onboarding and offboarding are security processes as well as administrative processes.
10. A second domain controller introduces replication-health considerations in addition to redundancy.
11. Broad infrastructure errors should be narrowed to the specific failing component before remediation.
12. Troubleshooting is incomplete until the original problem is retested.
13. Final end-to-end validation can reveal problems that were not present during earlier project phases.
14. Good technical documentation should record what was actually observed rather than manufacture missing evidence.

---

# 22. Project Outcome

The completed project evolved from an empty virtual environment into a functioning Windows domain capable of supporting common enterprise IT workflows.

The final environment demonstrated:

```text
Infrastructure
      ↓
Active Directory
      ↓
DNS
      ↓
Replication
      ↓
Identity Administration
      ↓
Domain Authentication
      ↓
Group Policy
      ↓
Resource Authorization
      ↓
Help Desk Delegation
      ↓
Authentication Troubleshooting
      ↓
Employee Lifecycle Management
      ↓
Final Validation
```

The project strengthened practical skills relevant to entry-level IT Support, Service Desk, IAM, NOC, and security-support roles while providing documented portfolio evidence of the completed work.

---

# 23. Supporting Evidence

Detailed implementation documentation is available under:

```text
01-docs/
```

Sanitized screenshots are available under:

```text
02-screenshots/
```

The repository README provides the recruiter-facing project overview:

```text
README.md
```

Detailed final validation is documented in:

```text
01-docs/15-final-validation.md
```

---

# 24. Project Status

```text
Implementation       COMPLETE
Testing              COMPLETE
Troubleshooting      COMPLETE
Final Validation     COMPLETE
Technical Docs       COMPLETE
README               COMPLETE
Final Report         COMPLETE
```

**Overall Project Status: COMPLETE**
