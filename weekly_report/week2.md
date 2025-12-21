---
title: Service Operationalization & Gray-Box Vulnerability Audit
description: Structured activation and assessment of the management plane using a Gray-Box security methodology. Internal socket validation, external service enumeration, and controlled credential-based exploitation were performed to empirically measure system exposure.
author: Aayusha Linbu
date: Week 2: Attack Surface Analysis
---

## 1. Assessment Objective & Methodological Rationale

Following the completion of the infrastructure foundation in Week 1, the objective of Week 2 is to operationalize the remote management plane and conduct a structured vulnerability assessment against the default system configuration.

A Gray-Box security methodology was intentionally selected. Unlike a purely external (Black Box) assessment, this approach leverages partial internal knowledge to correlate host-level service behavior with external observability. This allows for higher confidence findings by validating vulnerabilities from multiple perspectives.

The assessment workflow was executed in four deliberate stages:

1. Management service activation (SSH)
2. Internal verification of socket exposure and process state
3. External enumeration using network reconnaissance tooling
4. Controlled exploitation to quantify real-world impact

This methodology ensures that identified weaknesses are supported by empirical evidence rather than assumption.

## 2. Management Plane Operationalization

As the system was deployed without a graphical interface, remote access via Secure Shell (SSH) is a mandatory operational requirement. To establish this management plane, the OpenSSH daemon was explicitly enabled using the systemd service manager.

### 2.1 SSH Service Activation

The SSH service was configured to start immediately and persist across reboots.

**Command Executed:**
```bash
sudo systemctl enable --now ssh
sudo systemctl status ssh
```

**Evidence of Availability:**
As demonstrated in the process logs, the service successfully initialized and entered the `active (running)` state. The log timestamps confirm immediate availability.

![Figure 1: Service activation via systemctl.](/images/week2/ssh_enable.png)

The service successfully entered the active (running) state, confirming that the management plane is operational. Service logs indicate a clean initialization without warnings or errors, validating readiness for remote administration.

At this stage, the system is intentionally exposed to network access in order to create a measurable and testable attack surface.

## 3. Internal Exposure Validation (Host-Level Audit)

Before initiating external reconnaissance, an internal audit was performed to validate how the SSH service is bound at the network layer.

### 3.1 Socket Binding Analysis
The `ss` (socket statistics) utility was used to enumerate listening TCP sockets and their owning processes.


**Command Executed:**
```bash
sudo ss -ltnp | grep :22
```
![Figure 2: Socket verification confirming port 22 is open internally.](/images/week2/ltnp.png)

**Technical Analysis:**

The output confirms that the SSH daemon is bound to 0.0.0.0:22, meaning it is listening on all available network interfaces. The owning process is identified as `sshd`, confirming legitimate service ownership.

Security Implication:  
Binding to 0.0.0.0 maximizes availability but also maximizes exposure. In the absence of firewall controls, the service is reachable from any connected subnet, including the Host-Only management network.

This internal confirmation establishes an expectation for the results observed during external scanning.


## 4. External Perimeter Enumeration

With internal exposure verified, the assessment transitioned to an external perspective from the management workstation.

### 4.1 Reachability & Service Discovery

A comprehensive Nmap scan was conducted to identify open ports, running services, and exposed metadata.

**Command Executed:**
bash
nmap -T4 -A -v 192.168.56.104

![Figure 3: Nmap confirmation of Open Port 22.](/images/week2/nmap-intense-scan.png)


The scan confirms that TCP port 22 is open and running OpenSSH version 9.6p1. No evidence of packet filtering, connection throttling, or intrusion prevention was observed.

Security Impact:  
The absence of host-based firewall controls means the management service is reachable by any device on the local subnet. Additionally, version disclosure enables attackers to align exploits with known vulnerabilities.

## 5. Authentication Surface Enumeration (NSE Analysis)#

To assess authentication resilience, the Nmap Scripting Engine (NSE) was used to enumerate supported login mechanisms and cryptographic parameters.

### 5.1 Accepted Authentication Methods

**Command Executed:**
```bash
nmap -p 22 --script ssh-auth-methods,ssh2-enum-algos 192.168.56.104
```

![Figure 4: NSE Script confirming 'password' authentication is enabled.](/images/week2/namp-script-scan.png)

The script output explicitly advertises password-based authentication as an accepted login method.

Security Impact:  
Password authentication enables multiple attack classes, including dictionary attacks, credential stuffing, and authentication flooding. This confirms the presence of VULN-01: Password Authentication Enabled.

## 6. Controlled Exploitation: Brute-Force Simulation

To move beyond theoretical risk assessment, a controlled brute-force attack was executed using THC-Hydra in order to empirically measure time-to-compromise.

### 6.1 Attack Execution

**Command Executed:**
```bash
hydra -l aayusha -P gen.txt ssh://192.168.56.104 -t 4 -q -I
```

![Figure 5: Successful compromise of credentials using Hydra.](/images/week2/hydra-ssh-comprimised.png)

The attack successfully recovered valid credentials without triggering lockouts, delays, or defensive responses.

Interpretation:  
The absence of rate limiting, intrusion prevention, and cryptographic identity enforcement results in near-zero resistance to automated credential attacks once a weak password exists.

This confirms that authentication is the dominant risk vector in the current configuration.

## 7. Vulnerability Risk Matrix

Based on the collected evidence, the following vulnerabilities were identified and validated:

| ID | Component | Description | Severity | Evidence |
|----|----------|-------------|----------|----------|
| VULN-01 | Authentication | Password-based SSH login enabled | High (7.5) | NSE ssh-auth-methods output |
| VULN-02 | Network Perimeter | No firewall or access restriction | Medium (5.0) | Nmap port discovery |
| VULN-03 | Privilege Model | Broad sudo privileges | High (8.8) | Verified administrative scope |

Severity levels are aligned with CVSS impact assessment rather than exploit complexity alone.

## 8. Conclusion

This Gray-Box assessment demonstrates that while the management plane is functionally sound, it is insecure by default. The system is reachable, unfiltered, and accepts password-based authentication without resistance.

These weaknesses were intentionally preserved to establish a measurable baseline. The successful exploitation confirms the necessity of defensive controls prior to production deployment.

## 9. Forward Strategy

The findings from this assessment directly inform the remediation roadmap:

- Week 4: Eliminate password authentication through Ed25519 SSH keys and strict daemon policy
- Week 5: Deploy Fail2Ban to introduce active intrusion prevention and rate limiting

Subsequent audits will reuse the same methodology to verify remediation effectiveness.

## 10. References

1.  **Lyon, G.** (2009). *Nmap Network Scanning: The Official Nmap Project Guide*. Insecure.com LLC.
2.  **OpenSSH Project.** (2024). *Manual Pages: sshd_config*. Available at: https://man.openbsd.org/sshd_config
3.  **MITRE Corporation.** (2024). *Common Vulnerabilities and Exposures (CVE)*. Available at: https://cve.mitre.org/