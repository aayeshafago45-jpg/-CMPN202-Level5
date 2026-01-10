---
title: Implementation of Active Defense & Autonomous Telemetry
description: Deployment of Mandatory Access Control, Active Intrusion Prevention, and secure telemetry pipelines to transition the system from static hardening to an autonomous defensive posture.
author: Aayusha Linbu
date: Week 5: Active Defense & Observability
---

## 1. Strategic Objective & Security Rationale

Week 4 established a hardened, access-controlled baseline by enforcing cryptographic identity, strict privilege separation, and network whitelisting. While this posture significantly reduces the likelihood of compromise, it remains fundamentally **static**.

Modern threat models assume persistence, automation, and zero-day exploitation. As such, a hardened system must not only *resist* intrusion but also *contain*, *detect*, and *communicate* under adverse conditions.

The objective of Week 5 is to evolve the system from a hardened bastion into an **active defensive platform** capable of:

1. **Process Containment:** Limiting the blast radius of compromised services through Mandatory Access Control.
2. **Autonomous Intrusion Prevention:** Detecting and responding to hostile authentication behavior in real time.
3. **Secure Telemetry & Alerting:** Reliably notifying administrators of security or stability events without exposing inbound services.
4. **Operational Awareness:** Establishing continuous resource monitoring to support later performance regression analysis.

This phase introduces *reactive security controls* that operate independently of administrator intervention.

## 2. Layer 1: Mandatory Access Control (AppArmor)

Traditional Linux Discretionary Access Control (DAC) fails catastrophically once a privileged process is compromised. To address this structural weakness, **AppArmor** was deployed to enforce Mandatory Access Control at the kernel level.

AppArmor constrains processes based on predefined behavioral profiles, ensuring that even exploited services cannot exceed their intended scope.

### 2.1 Kernel Integration & Tooling

User-space utilities were installed to verify and manage AppArmor enforcement state.

**Command Executed:**
```bash
sudo apt install apparmor-utils -y
```

![Figure 1: Installation of AppArmor user-space utilities.](/images/week5/app-armor-installation.png)

### 2.2 Enforcement Verification
To confirm active enforcement, the loaded AppArmor profiles were inspected.

**Command Executed:**
```bash
sudo aa-status
```

![Figure 2: Verification of AppArmor profiles running in Enforce Mode.](/images/week5/a-status.png)

**Assessment:**
The output confirms that multiple profiles are operating in Enforce Mode, including network-facing and system utilities. This ensures that successful exploitation of a service results in process confinement rather than full system compromise.


## 3. Layer 2: Active Intrusion Prevention (Fail2Ban)

Week 2 demonstrated that password-based authentication enables rapid compromise through automated brute-force attacks. While password authentication has since been eliminated, authentication flooding and credential abuse remain relevant denial-of-service vectors.

To mitigate these threats, Fail2Ban was deployed as an autonomous intrusion prevention system.

### 3.1 Deployment & Policy Configuration

Fail2Ban was installed and configured to monitor SSH authentication logs.

**Command Executed:**
```bash
sudo apt install fail2ban -y
```

![Figure 3: Deployment of the Fail2Ban intrusion prevention framework.](/images/week5/fail2ban-installation.png)

A strict SSH jail policy was applied, enforcing:

- Maximum failed attempts: 3
- Ban duration: 1 hour

### Service Activation & Verification

**Command Executed:**
```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

![Figure 4: Fail2Ban status confirming the SSHD jail is active.](/images/week5/fail2ban-start.png)

**Security Impact:**
Any hostile authentication pattern now triggers an automatic network-layer ban. This converts brute-force and authentication-flood attacks from persistent threats into self-defeating events.


## 4. Layer 3: Secure Telemetry & Alerting Pipeline (SMTP)

Detection without notification is operationally ineffective. To enable outbound alerting without introducing inbound exposure, a secure SMTP relay architecture was implemented using Postfix.

This design ensures encrypted alert delivery while avoiding the complexity and risk of operating a public mail server.

### 4.1 SMTP Relay Installation
Required mail transfer and authentication components were installed.

**Command Executed:**
```bash
sudo apt install postfix mailutils libsasl2-modules -y
```

![Figure 5: Installation of Postfix and SASL authentication modules.](/images/week5/mail-util-install.png)

### 4.2 Encrypted Relay Configuration

Due to outbound SMTP restrictions on Port 25, the relay was explicitly configured to use Port 587 with enforced TLS encryption.

**Configuration Applied (main.cf):**
```ini
relayhost = [smtp.gmail.com]:587
smtp_sasl_auth_enable = yes
smtp_tls_security_level = encrypt
```

![Figure 6: Configuring main.cf to force TLS encryption on Port 587.](/images/week5/maincf.png)

### 4.3 Pipeline Validation

A test alert was injected to validate end-to-end delivery.

**Command Executed:**
```bash
echo "This is a test from your Secure Server" | mail -s "Week 5 Alert" [REDACTED]@gmail.com
```

![Figure 7: Successful injection of email payload into the Postfix queue.](/images/week5/postfix.png)

![Figure 8: Confirmation of alert receipt in the administrator's inbox.](/images/week5/testmail-received.png)

**Outcome:**
The alert was successfully delivered, confirming reliable, encrypted telemetry capability.

## 5. Layer 4: Autonomous Resource Monitoring

To support later performance regression analysis (Week 6), a lightweight autonomous monitoring mechanism was implemented. This system continuously observes CPU and memory utilization and triggers alerts under abnormal conditions.

### 5.1 Monitoring Logic
A custom Bash script (week5-monitor.sh) was developed to:
1. Sample CPU and RAM usage
2. Log metrics to CSV for later analysis
3. Trigger SMTP alerts when CPU utilization exceeds defined thresholds

**Conceptual Logic:**
```bash
THRESHOLD=80
if [ "$CPU_HIGH" -eq 1 ]; then
    echo "High CPU Load Detected" | mail -s "CRITICAL ALERT" ...
fi
```

![Figure 9: The custom monitoring script logic.](/images/week5/monitorsh.png)

### 5.2 Service Persistence

To ensure resilience, the monitoring script was deployed as a managed systemd service.

**Command Executed:**
```bash
sudo nano /etc/systemd/system/week5-monitor.service
sudo systemctl enable --now week5-monitor.service
```

![Figure 10: Enabling the custom monitoring daemon via systemctl.](/images/week5/setting-custom-scripts.png)

This guarantees continuous observability across reboots and service restarts.

## 6. Outcome Assessment

At the conclusion of Week 5, the system demonstrates the following defensive capabilities:

- **Containment:** AppArmor enforces kernel-level process restrictions.
- **Prevention:** Fail2Ban autonomously mitigates hostile authentication behavior.
- **Telemetry:** Secure SMTP alerting enables real-time administrator awareness.
- **Observability:** Continuous resource monitoring establishes operational visibility.

The infrastructure has transitioned from a hardened but passive system into an active, self-monitoring defensive platform.

## 7. Conclusion & Transition

Week 5 completes the shift from static hardening to active defense and observability. The system can now detect, contain, and report abnormal behavior without manual intervention.

With security instrumentation in place, the project proceeds to Week 6, where the performance impact of these controls will be quantified through controlled regression testing against the baseline established in Week 3.

## 8. References

1.  **Bauer, M.** (2020). *Linux Server Security: Hack and Defend*. O'Reilly Media.
2.  **Fail2Ban Project.** (2024). *Fail2Ban Manual*. Available at: https://github.com/fail2ban/fail2ban
3.  **Canonical.** (2024). *AppArmor Administration Guide*. Ubuntu Documentation.