# Trusted Trouble

## Event: The Gauntlet: Arctic Howl - OffSec

**Category:** Network Forensics / Insider Threat Investigation
**Difficulty:** Hard
**Points:** 80 / 80
**Status:** Completed (8/8)

---

## Challenge Description

The trail ends behind an icy waterfall. The portal seals. Ashka rises, altered and ready to strike. But before the confrontation, evidence reveals the Yeti had already used her augmented instincts to move quietly through nearby systems, collecting and leaking data beyond the Expanse.

One victim was Megacorp One. Days after hiring several new employees, the company discovered sensitive information had been leaked. Nothing appeared broken, but suspicious activity was traced through their MAIL server and several CLIENT machines.

Megacorp One has provided PCAP captures from these systems to help identify who leaked the data and how the breach occurred.

---

## Challenge Files

| File | Description |
|------|-------------|
| MAIL server PCAP | Network capture from the company mail server |
| CLIENT machine PCAPs | Network captures from employee client machines |

---

## Tools Used

- Wireshark / tshark (PCAP analysis)
- Network protocol dissection (SMTP, HTTP, DNS)
- SQLite inspection

---

## Analysis

### Phase 1: Recruitment Process

The investigation began with the MAIL server PCAP, which captured the entire hiring process for Megacorp One. Email traffic revealed a job application cycle where **9 candidates** submitted applications. The hiring manager, **Tatiana Petrov**, reviewed and approved applications, ultimately accepting **3 candidates**: Samuel Adu, Fernanda Ribeiro, and Min-Jun Park.

### Phase 2: VPN Onboarding Issues

After joining, one of the new employees experienced connectivity issues during VPN onboarding. Analysis of the CLIENT PCAPs identified **Fernanda Ribeiro** as the employee who had issues connecting to the company VPN — her traffic showed repeated failed authentication attempts and connection timeouts before eventually establishing a session.

### Phase 3: Policy Violations

Review of client machine traffic identified two employees engaged in activity that violated company policy: **Samuel Adu** and **Min-Jun Park**. Their traffic patterns showed anomalous outbound connections and behaviour inconsistent with standard employee usage.

### Phase 4: Data Exfiltration

Deeper analysis of Samuel Adu's client traffic revealed a covert data exfiltration channel. His machine was making outbound connections to the public IP **203.98.112.47**, transmitting a file named `sensitive.db` — a SQLite database containing a `users` table with plaintext credentials:

| Username | Password |
|----------|----------|
| Robin Schwartz | `5up3r5Tr0NgP@$$w0rd!` |

The exfiltration method used HTTP/S to transfer the database to the external attacker-controlled server, bypassing any internal monitoring that focused only on email and file-share channels.

### Phase 5: Insider Threat Identification

Correlating all evidence — policy violations, suspicious outbound connections, and the exfiltrated database — the insider threat was conclusively identified as **Samuel Adu**, a newly hired employee who leveraged their internal access to steal and exfiltrate sensitive credential data.

---

## Attack Path Summary

```
[Megacorp One Hiring Process]
          |
          | 9 applicants -> 3 accepted (samuel.adu, fernanda.ribeiro, min-jun.park)
          | Hiring manager: tatiana.petrov
          v
[Employees Onboarded]
          |
          | fernanda.ribeiro -> VPN issues (legitimate)
          | samuel.adu + min-jun.park -> policy violations
          v
[samuel.adu - Insider Threat]
          |
          | Accessed sensitive.db (SQLite - credentials database)
          | Exfiltrates via HTTP to external C2
          v
[203.98.112.47 - Attacker-Controlled Server]
          |
          | Receives: sensitive.db
          |   -> Table: users
          |   -> Robin Schwartz / 5up3r5Tr0NgP@$$w0rd!
```

---

## Answers

### Q1: How many people applied to work at MegacorpOne?
**9**

Nine candidates submitted applications to Megacorp One, as captured in the MAIL server PCAP showing the full recruitment email thread.

### Q2: Out of the total applicants, whose application was accepted?
**samuel.adu, fernanda.ribeiro, min-jun.park**

Three applicants were accepted: Samuel Adu, Fernanda Ribeiro, and Min-Jun Park. Their acceptance notices were visible in the SMTP traffic captured on the mail server.

### Q3: What is the name of the hiring manager responsible for approving the applications?
**tatiana.petrov**

Tatiana Petrov was identified as the hiring manager from the email headers and approval correspondence captured in the mail server PCAP.

### Q4: Which of the employees had issues with the company VPN?
**fernanda.ribeiro**

Fernanda Ribeiro's CLIENT PCAP showed repeated failed VPN authentication attempts and connection resets, indicating she had trouble connecting to the company VPN during onboarding.

### Q5: Identify the employee(s) that were violating company policy.
**samuel.adu, min-jun.park**

Both Samuel Adu and Min-Jun Park were identified as violating company policy based on anomalous traffic patterns observed in their CLIENT machine PCAPs — behaviour outside the bounds of acceptable use.

### Q6: What is the public IP address that the insider threat is connecting to in order to exfiltrate data?
**203.98.112.47**

Samuel Adu's CLIENT PCAP showed repeated outbound connections to `203.98.112.47`, an external public IP not associated with any Megacorp One infrastructure — identified as attacker-controlled exfiltration infrastructure.

### Q7: Identify what the insider threat was exfiltrating. Make sure to include the sensitive data in your answer.
**sensitive.db — SQLite database containing a `users` table with credentials: Robin Schwartz / 5up3r5Tr0NgP@$$w0rd!**

The file `sensitive.db` was transferred to the external server. Extracting the SQLite database from the PCAP stream and inspecting its contents revealed a `users` table with at least one set of plaintext credentials belonging to Robin Schwartz.

### Q8: Which of the employees was the insider threat?
**samuel.adu**

Samuel Adu was the insider threat. He violated company policy, maintained covert outbound connections to an external IP, and exfiltrated `sensitive.db` containing internal credentials — all consistent with a malicious insider leveraging new employee access.

---

## Key Takeaways

1. **Insider threats are hardest to detect** — Samuel Adu's activity blended in with normal onboarding traffic, making it easy to overlook without deep PCAP analysis
2. **New employee onboarding is a high-risk window** — fresh access combined with low scrutiny gives insiders an opportunity to act quickly before monitoring baselines are established
3. **Credentials in plaintext databases** are a critical risk — `sensitive.db` contained passwords in cleartext, meaning a single exfiltration event fully compromised the stored accounts
4. **Network forensics across multiple capture points** (mail server + client machines) is essential for building a complete timeline and correlating insider activity
5. **Policy violations as a precursor** — both Samuel Adu and Min-Jun Park showed anomalous behaviour, but only one was the actual insider; not every policy violation indicates a threat, but they warrant investigation
6. **Egress filtering and DLP** would have caught the outbound transfer of `sensitive.db` to an unknown public IP before the data left the network

---

## References

- [Wireshark PCAP Analysis](https://www.wireshark.org/)
- [SQLite Documentation](https://www.sqlite.org/)
- [MITRE ATT&CK - Exfiltration Over C2 Channel (T1041)](https://attack.mitre.org/techniques/T1041/)
- [MITRE ATT&CK - Insider Threat](https://attack.mitre.org/techniques/T1078/)

---

*Challenge from OffSec The Gauntlet: Arctic Howl - Trusted Trouble*
*Completed: 8/8 questions | 80/80 points*
