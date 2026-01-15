---
title: Quantitative Security Regression Analysis & Performance Impact Assessment
description: Comparative benchmarking to quantify the operational cost of defense-in-depth controls. Analysis of CPU, memory, and storage regression against the Week 3 performance baseline.
author: Aayusha Linbu
date: Week 6: Performance Cost of Security
---

## 1. Objective & Engineering Scope

Across Weeks 1–5, the system was intentionally transformed from an insecure default installation into a hardened, actively monitored defensive platform. This evolution introduced multiple persistent security controls, including cryptographic authentication, mandatory access control, intrusion prevention, and continuous telemetry.

The objective of Week 6 is to answer a single engineering question:

What is the measurable operational cost of this security architecture?

To ensure scientific validity, the exact benchmarking methodology established in Week 3 was repeated without modification. By comparing the unsecured baseline metrics against the current secured state, performance deltas can be directly attributed to security enforcement rather than workload variance.

This phase does not introduce new controls. It exists solely to measure, interpret, and justify the cost of security.

## 2. Operational State Verification

Prior to executing any benchmarks, the system state was verified to ensure that all defensive services introduced in Weeks 4 and 5 were active and consuming resources.

Command Executed:
```bash
sudo systemctl status fail2ban postfix week5-monitor.service
```

Status Assessment:
- Fail2Ban: Active and monitoring authentication logs
- Postfix: Active and ready for secure SMTP relay
- Monitoring Service: Active and polling system metrics continuously

This confirms that all subsequent benchmarks reflect a realistic “active defense” operational state rather than an artificially idle environment.

![Figure 1: Verification of active security services prior to benchmarking.](/images/week6/servivces-status.png)

## 3. Memory Overhead Analysis (Idle Cost)

The first regression test evaluated the baseline memory overhead introduced by persistent security services. Measurements were taken during idle conditions with no artificial load applied.

| State | Metric | Value |
| :--- | :--- | :--- |
| Week 3 (Unsecured) | Used RAM | 372 MiB |
| Week 6 (Secured) | Used RAM | 377 MiB |
| Delta | Increase | +1.3% |

Engineering Interpretation:
Despite the addition of multiple background services (Fail2Ban, Postfix, and the custom monitoring daemon), the total memory increase is approximately 5 MiB. This negligible overhead validates the efficiency of the headless architecture and confirms that memory is not a limiting factor for the deployed security stack.

![Figure 2: Idle memory comparison showing minimal regression.](/images/week6/initial-btop.png)

## 4. 4. Computational Overhead Analysis (CPU)

To measure computational regression under load, the same CPU-intensive matrix multiplication workload used in Week 3 was executed.

Command Executed:
```bash
stress-ng --cpu 4 --cpu-method matrixprod --timeout 40s --metrics-brief
```

Regression Results:

| State | Throughput (Bogo Ops/s) |
| :--- | :--- |
| Week 3 (Unsecured) | 8,853.77 |
| Week 6 (Secured) | 1,558.71 |
| Performance Impact | -82.3% |

Root Cause Analysis:
The significant reduction in raw computational throughput is primarily attributed to persistent background activity introduced in Week 5. The monitoring service periodically samples system metrics, while Fail2Ban continuously parses authentication logs. Combined with AppArmor enforcement, this introduces frequent context switching and scheduler contention.

This phenomenon demonstrates the classic observer effect: continuous observability imposes a measurable computational cost, particularly within a resource-constrained virtual environment.

![Figure 3: CPU regression results showing the impact of active monitoring.](/images/week6/stress-report.png)

## 5. Storage I/O Overhead Analysis

The most pronounced regression was observed during storage I/O testing. Unlike Week 3, the disk subsystem is now actively handling continuous log writes and telemetry data.

Command Executed:
```bash
fio --name=baseline_test --ioengine=libaio --rw=randwrite --bs=4k --size=512M --numjobs=1 --time_based --runtime=15 --group_reporting
```

Regression Results:

| State | Bandwidth | IOPS |
| :--- | :--- | :--- |
| Week 3 (Unsecured) | 132 MiB/s | 33,700 |
| Week 6 (Secured) | 15.7 MiB/s | 4,021 |
| Performance Impact | -88.1% | Severe |

Root Cause Analysis:
The disk subsystem is now subject to sustained contention from multiple security-related write streams:

1. Double-write behavior to system and authentication logs
2. Forced write synchronization to preserve forensic integrity
3. Concurrent CSV logging from the monitoring service

This confirms a critical operational reality: logging-heavy security architectures impose significant I/O overhead. In high-security systems, disk throughput becomes a constrained and valuable resource.

![Figure 4: Disk I/O regression demonstrating contention under active logging.](/images/week6/fio-r4eport.png)

## 6. Security Validation Check (Lynis Audit)

To confirm that the observed performance cost resulted in tangible security improvements, a final host-based audit was performed.

Command Executed:
```bash
sudo lynis audit system --quick
```

Audit Summary:
- Hardening Index: 61
- Firewall: Active and verified
- SSH Hardening: Passed

Score Context:
While the numerical score appears moderate, deductions primarily relate to controls intentionally excluded from project scope, such as malware scanners and dedicated partitioning. Core security objectives—firewall enforcement, authentication hardening, and logging integrity—passed validation.

![Figure 5: Final Lynis security audit confirms hardening status.](/images/week6/lynis-report.png)

## 7. Conclusion: Engineering Trade-Off Assessment

The regression analysis yields a definitive conclusion:

Security is not free.

The system gained:
- Kernel-level process confinement
- Autonomous intrusion prevention
- Immutable, high-fidelity audit logging
- Real-time telemetry and alerting

The system paid:
- Significant reductions in peak CPU throughput
- Severe storage I/O contention under load

Final Engineering Verdict:
For a hardened management node, this trade-off is acceptable and expected. The system is not designed for high-performance computation but for survivability, integrity, and observability under attack.

The architecture is therefore validated as Secure by Design, prioritizing control, detection, and resilience over raw performance.

## 8. Transition to Final Audit

With performance impact quantified and justified, the project proceeds to its final phase.

Week 7 will perform a post-hardening security audit to formally validate exposure reduction, service minimization, and residual risk using industry-standard assessment tooling.