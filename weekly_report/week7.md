---
title: Post-Hardening Security Audit & Final Executive Assessment
description: Execution of a post-implementation security audit to verify remediation of previously identified vulnerabilities. Black-box validation confirms the effectiveness of the defense-in-depth architecture.
author: Aayusha Linbu
date: Week 7: Final Security Audit
---

## 1. Executive Summary

This project set out to design, implement, and validate a secure Linux management node using an evidence-driven, defense-in-depth methodology. Over seven weeks, the system evolved from a default, vulnerable configuration into a hardened and actively defended infrastructure component.

Week 7 serves as the **final verification phase**. No new security controls are introduced. Instead, previously identified vulnerabilities are re-tested using the same tooling and techniques applied during the Week 2 baseline assessment.

The results of this audit confirm that all critical risks identified earlier in the project have been either fully remediated or reduced to an acceptable residual level. The system now enforces cryptographic identity, minimizes network exposure, actively resists intrusion attempts, and preserves forensic integrity.

## 2. Audit Methodology & Scope

To ensure objectivity, a **black-box audit perspective** was adopted wherever possible. Validation focused on externally observable behavior rather than internal configuration assumptions.

The audit scope includes:

- Authentication surface validation
- Brute-force attack resilience
- Network perimeter enforcement
- Verification of active defensive services

The assessment reuses identical tools and attack patterns from Week 2 to enable direct before-and-after comparison.

## 3. Authentication Surface Validation

Password-based authentication was identified as the primary compromise vector during the Week 2 assessment. Its removal was a core remediation objective in Week 4.

### 3.1 Verification via Network Enumeration

The SSH service was re-enumerated using the same network scanning methodology previously applied.

Observed Results:
- Week 2: SSH advertised support for both publickey and password authentication
- Week 7: SSH advertises publickey authentication only

This confirms that password-based authentication has been fully removed at the protocol level.

Security Impact:
Authentication is now strictly cryptographic. Without possession of the corresponding Ed25519 private key, credential compromise through guessing or replay is not feasible.

![Figure 1: Final network scan confirming removal of password authentication.](/images/week7/nmap.png)

## 4. Brute-Force Attack Resilience

To validate resistance against automated credential attacks, the same brute-force simulation executed in Week 2 was repeated against the hardened system.

### 4.1 Controlled Exploitation Attempt

The identical Hydra attack parameters were reused to ensure methodological consistency.

Command Executed:
hydra -l aayusha -P gen.txt ssh://192.168.56.104 -t 4 -V

Result Comparison:
- Week 2: Successful credential compromise
- Week 7: Immediate failure due to unsupported authentication method

Interpretation:
The attack was rejected during protocol negotiation, before any authentication attempts occurred. This demonstrates that the threat class has been eliminated rather than merely mitigated.

![Figure 2: Hydra attack failing due to hardened authentication policy.](/images/week7/hydra.png)

## 5. Operational Defense Verification

Beyond authentication controls, the audit verified that active defensive mechanisms introduced in Week 5 remain persistent and functional.

Command Executed:
sudo fail2ban-client status sshd
sudo ufw status verbose

Verification Results:
- Fail2Ban: SSH jail active and monitoring authentication logs
- Firewall (UFW): Default deny policy enforced, with SSH restricted to the management subnet only

These findings confirm that the system continues to enforce both preventive and reactive defenses without manual intervention.

![Figure 3: Active status of intrusion prevention and firewall enforcement.](/images/week7/status.png)

## 6. Final Risk Assessment Matrix

| ID | Vulnerability | Week 2 Risk Level | Week 7 Status | Residual Risk |
|----|---------------|------------------|---------------|---------------|
| VULN-01 | Password Authentication | High | Remediated | None |
| VULN-02 | Brute-Force Attacks | High | Remediated | Low (IPS Active) |
| VULN-03 | Privilege Escalation | High | Mitigated | Low (RBAC + MAC) |
| VULN-04 | Performance Overhead | N/A | Accepted | Medium (Documented Trade-off) |

Residual risks are either structurally mitigated or consciously accepted based on system role and threat model.

## 7. Final Project Conclusion

This project successfully delivered a secure, auditable Linux management node engineered according to enterprise security principles.

While Week 6 quantified a significant performance cost associated with active monitoring and extensive logging, the final audit confirms that this overhead directly enables measurable security gains. The system prioritizes integrity, resilience, and observability over raw throughput, which is appropriate for its intended function.

The final architecture demonstrates:
- Structural prevention of credential compromise
- Autonomous resistance to network-based attacks
- Continuous operational awareness
- Verifiable forensic readiness

The project objectives have been met in full, and the system is validated as Secure by Design.

## 8. End of Assessment
