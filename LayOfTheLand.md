# TryHackMe Write-up: The Lay of the Land

## Engagement Details

- Platform: TryHackMe
- Room: The Lay of the Land
- Author: Ahmad Zulhilmi Mohamad Masri
- Student ID: 52215124142
- Assessment Date: 7 May 2025
- Objective: Perform post-compromise enumeration in a Windows Active Directory environment and recover the challenge flag.

## Executive Summary

This write-up documents a guided post-exploitation workflow in a Windows AD lab. The assessment verified domain membership, enumerated users from a specific Organizational Unit (OU), identified a local listening service, and retrieved the final flag from an internal web endpoint.

The room reinforces a core red-team principle: local enumeration after initial access often reveals internal-only services and high-value identity information that are not externally visible.

## Scope and Access

A provided Windows target was deployed in TryHackMe and accessed using RDP from AttackBox.

- Username: kkidd
- Password: Pass123321@

RDP connection command:

```bash
xfreerdp /v:<MACHINE_IP> /u:kkidd /p:Pass123321@
```

## Methodology

The workflow followed a structured post-exploitation sequence:

1. Confirm AD/domain membership.
2. Enumerate users in the target OU.
3. Identify local services bound to listening ports.
4. Access discovered service and capture proof (flag).

## Technical Walkthrough

### 1) Domain Membership Verification

Command used:

```powershell
systeminfo | findstr Domain
```

Observed result:

- Domain: thmdomain.com

Interpretation:

- The host is domain-joined.
- If the result were WORKGROUP, it would indicate a non-domain standalone machine.

### 2) AD User Enumeration (THM OU)

Command used:

```powershell
Get-ADUser -Filter * -SearchBase "OU=THM,DC=THMREDTEAM,DC=COM"
```

Observed findings:

- Total users discovered in OU: 6
- Administrative UPN identified: thmadmin@thmredteam.com

Security relevance:

- User and identity enumeration provides targeting context for later actions such as credential attacks, Kerberos abuse paths, or privilege mapping.

### 3) Local Service Discovery

Command used:

```powershell
netstat -an | findstr "LISTENING" | findstr "13337"
```

Observed finding:

- A local service was listening on TCP 13337 (localhost).

Security relevance:

- Internal services bound to localhost are often omitted from perimeter scans but may expose sensitive functionality to a compromised session.

### 4) Flag Retrieval

URL accessed from the compromised Windows host:

- http://localhost:13337

Observed flag:

- THM{S3rv1cs_1s_3numerat37ed}

Outcome:

- Task objective completed successfully.

## Key Findings

- The target system was confirmed as AD-integrated.
- OU-level user enumeration was successful and returned actionable identity data.
- A local-only web service was discovered via host-based network inspection.
- The discovered service directly exposed the challenge flag.

## Defensive Takeaways

1. Restrict and monitor local administrative and PowerShell enumeration capabilities where possible.
2. Apply least privilege and segmentation for AD user discovery paths.
3. Audit localhost-bound services for sensitive data exposure.
4. Enable endpoint logging for command-line telemetry and unusual local service access patterns.

## Conclusion

The room provided a practical introduction to Windows AD post-exploitation enumeration. The exercise demonstrates how simple, low-noise host commands can quickly produce meaningful domain context and uncover hidden internal services. These are foundational skills for offensive operations and equally valuable for defensive detection engineering.
