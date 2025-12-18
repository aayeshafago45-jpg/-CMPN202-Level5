---
title: Enterprise Infrastructure Deployment & Baseline Configuration
description: Technical documentation regarding the provisioning of a secure, headless Ubuntu Server 24.04 environment. Detailed analysis of multi-core resource allocation, split-horizon network topology, and the resolution of declarative network configuration (Netplan) anomalies.
author: Aayusha Linbu
date: Week 1: Infrastructure Foundation
---

## 1. Architectural Intent & Security Scope
The objective of Phase 1 was not merely to deploy a virtual machine, but to **engineer a controlled infrastructure baseline** that reflects the design constraints of an enterprise Tier-3 environment within a single-host laboratory scope.

Rather than deploying a general-purpose Linux workstation, the system was intentionally provisioned as a **headless Ubuntu Server**. This decision enforces two foundational security principles from the outset:

1. **Attack Surface Minimization** : The exclusion of a GUI stack (X11, Wayland, display managers, desktop services) removes a substantial dependency chain that historically introduces privilege-escalation and remote-execution vectors. This ensures that all exposed services are explicit, intentional, and auditable.

2. **Deterministic Resource Allocation** : By reserving system resources exclusively for infrastructure services and security instrumentation, performance measurements in later phases remain attributable to security controls rather than UI overhead or background desktop processes.

This phase establishes a “clean-room baseline” against which all subsequent vulnerability assessment (Week 2), hardening (Week 4), and performance regression (Week 6) activities are measured.

## 2. Virtualization Layer & Hardware Provisioning

The infrastructure was deployed using Oracle VirtualBox 7, selected for its fine-grained control over CPU topology, memory allocation, and multi-adapter networking features required to simulate enterprise segmentation patterns.

### 2.1 Hardware Resource Strategy
Unlike default lab templates, the virtual hardware profile was deliberately non-minimal, ensuring that future security testing would not be constrained by artificial bottlenecks.

- **vCPU Allocation: 4 Cores:** Security subsystems such as auditd, fail2ban, AppArmor enforcement hooks, and stress-testing utilities are inherently concurrent. Allocating four vCPUs prevents scheduler starvation and ensures that later performance degradation can be attributed to security overhead, not CPU scarcity.

- **Memory Allocation: 2048 MB:** Memory was intentionally capped rather than maximized. This enforces realistic constraints for detecting memory-pressure scenarios and denial-of-service behavior while still providing sufficient headroom for IDS/IPS tooling.

- **Storage: 25 GB (Dynamically Allocated):** A dynamically expanding disk ensures sufficient capacity for verbose security logs (`/var/log`, audit trails) without prematurely consuming host storage.

![Figure 1: Hardware resource map highlighting the optimized 4-Core CPU allocation.](/images/week1/resources_allocation.png)

![Figure 2: Virtual Machine summary verifying the configuration.](/images/week1/vm_summery.png)

## 3. Identity & Privilege Model

During unattended installation, direct root login was intentionally avoided. Instead, a named administrative user (`aayusha`) was created with `sudo-mediated` privilege escalation.

![Figure 3: Identity & Privilege Model.](/images/week1/useraccount_setup.png)

### Security Rationale
- **Non-repudiation:** All privileged actions require explicit escalation and are recorded in `/var/log/auth.log`.
- **Audit Integrity:**  This prevents anonymous root activity, aligning the system with enterprise accountability requirements.
- **Forward Compatibility:** The model supports later role separation and SSH key enforcement without architectural rework.

## 4. Network Architecture: Split-Horizon Design

To reflect enterprise management separation, a dual-homed (split-horizon) network topology was implemented.

| Interface | Role                         | Exposure                 |
| --------- | ---------------------------- | ------------------------ |
| `enp0s3`  | Management Plane (Host-Only) | Trusted, non-routable    |
| `enp0s8`  | Update Plane (NAT)           | Controlled outbound only |

This design ensures that:

- Administrative access is isolated from upstream traffic
- Internet connectivity is available only when explicitly required
- Future firewall rules can enforce asymmetric trust boundaries

## 5. Incident Analysis: Netplan Interface Enumeration Failure

### 5.1 Fault Detection

Post-installation verification revealed an inconsistency:

- `enp0s3` correctly obtained a management IP (`192.168.56.x`)
- `enp0s8` existed at the kernel level but remained inactive, blocking outbound connectivity

![Figure 3: Netplan Interface Enumeration Failure.](/images/week1/verification_caught_issue.png)

### 5.2 Root Cause Analysis
The Ubuntu installer generates Netplan configuration only for interfaces present at first boot. Because the NAT adapter was added after initial VM creation, it was not declared in `/etc/netplan/00-installer-config.yaml`.

This highlights a critical operational insight:
> Declarative networking systems do not infer intent from hardware presence.

## 6. Remediation: Declarative Network Correction (Netplan)

Rather than applying transient fixes (`ip link set up`), the resolution was implemented correctly and persistently using Netplan’s declarative model.

### Corrected Configuration
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: true
      dhcp6: true

```
```bash
sudo chmod 600 /etc/netplan/00-installer-config.yaml
sudo netplan generate
sudo netplan apply
```

![Figure 4:  Declarative Network Correction (Netplan).](/images/week1/fix_adapter.png)

This remediation ensures:

- Interface persistence across reboots
- Predictable routing behavior
- Compatibility with future firewall enforcement

## 7. Baseline Verification & System Integrity

A full verification sweep was conducted to confirm system readiness.

![Figure 5: A full verification sweep.](/images/week1/verification_success_full.png)

### 7.1 Kernel & Platform Validation

- Kernel: Linux 6.8.0-90-generic
- Significance: Enables modern observability features (eBPF hooks, enhanced scheduling telemetry)

### 7.2 Resource Efficiency

- Idle Memory Usage: ~320–340 MiB
- Confirms headless efficiency and sufficient capacity for security tooling

### 7.3 Network State Confirmation

- enp0s3: Active, management-only
- enp0s8: Active, outbound capable

## 8. Security Posture Assessment (Baseline)
At the conclusion of Phase 1:

- ✔ Architecture hardened by design (not configuration)
- ✔ No unnecessary services exposed
- ✔ Deterministic networking in place
- ⚠ Default authentication and firewall policies intentionally untouched

This is a measured insecure baseline, suitable for controlled exploitation and subsequent remediation.

## 9. Conclusion & Transition
Phase 1 successfully delivered a stable, auditable, enterprise-style infrastructure baseline.
The discovery and resolution of the Netplan anomaly reinforces the importance of post-deployment verification, even in automated installs.

The system is now prepared for Week 2, where a white-box vulnerability assessment will be performed to empirically identify weaknesses in the default configuration before any hardening is applied.

## 10. References

- Canonical Ltd. (2024). Netplan Reference Documentation. https://netplan.io/reference
- NIST. (2020). SP 800-53 Rev.5 – Security and Privacy Controls.
- Oracle. (2024). VirtualBox Networking Architecture. https://www.virtualbox.org/manual/ch06.html