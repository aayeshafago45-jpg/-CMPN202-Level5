---
title: Workload Characterization & Performance Instrumentation
description: Establishment of a scientific performance control group. Deployment of industry-standard benchmarking suites (stress-ng, fio) to quantify the computational and I/O capacity of the unsecured baseline prior to hardening.
author: Aayusha Linbu
date: Week 3: Performance Baselining
---

## 1. Objectives & Strategic Intent

With the attack surface empirically validated in Week 2, the project transitions into a preparatory engineering phase. Before applying security controls, it is essential to understand the system’s inherent performance characteristics.

Security mechanisms such as cryptographic authentication (SSH keys), packet inspection (UFW), mandatory access control (AppArmor), and intrusion prevention (Fail2Ban) all introduce measurable overhead. Without a pre-hardening control group, any observed degradation in later stages cannot be accurately attributed.

The objective of Week 3 is therefore to establish a **quantitative performance baseline** for the unsecured system. This baseline serves as the reference point against which the cost of security will be measured in Week 6.

The guiding question for this phase is:
Does the introduction of layered security controls degrade system performance beyond acceptable operational thresholds?

## 2. Experimental Methodology & Instrumentation Strategy

To ensure reproducibility and engineering rigor, lightweight utilities were intentionally avoided in favor of benchmarking suites capable of generating deterministic, subsystem-specific load.



The selection criteria prioritized:
- Repeatability
- Subsystem isolation
- Low measurement distortion

| Subsystem | Tool | Rationale |
|---------|------|-----------|
| CPU Scheduler | stress-ng | Provides deterministic CPU workloads with selectable algorithms. The matrix multiplication method was chosen to approximate the floating-point intensity of cryptographic operations. |
| Storage I/O | fio | Enables fine-grained random I/O testing using asynchronous engines, accurately modeling security logging and audit workloads. |
| Observability | btop | Offers real-time and historical visibility, enabling correlation between injected load and system response without influencing workload execution. |

All tests were conducted on an otherwise idle system to eliminate environmental noise.


## 3. Toolchain Provisioning & Environment Control

All benchmarking tools were installed exclusively from the official Ubuntu repositories to ensure version consistency and kernel compatibility.

**Command Executed:**
```bash
    sudo apt update
    sudo apt install stress-ng fio btop -y
```

![Figure 1: Successful installation of the benchmarking toolchain.](/images/week3/install-professional-tools.png)

This approach ensures that future test repetition (Week 6) uses identical binaries, preserving experimental integrity.

## 4. Establishing the Idle Control State

Before injecting artificial load, the system's resting state was documented. This control variable is essential for detecting "Resource Leaks" or background process interference.

**Command Executed:**
```bash
    btop
```

![Figure 2: System idle state confirming high resource efficiency.](/images/week3/btop.png)


### Observed Idle Metrics
- CPU Utilization: ~2%
- Memory Usage: ~370 MiB of 1.92 GiB
- Swap Usage: 0%

Engineering Significance:  
These values confirm the effectiveness of the headless architecture implemented in Week 1. The absence of a graphical environment preserves approximately 1.5 GiB of memory headroom, which will later be consumed by security services without requiring resource scaling.

This idle state acts as a baseline for detecting background interference or resource leakage during subsequent tests.

## 5. CPU Workload Characterization

To approximate the computational cost of cryptographic operations, a CPU-intensive floating-point workload was executed across all allocated cores.

**Command Executed:**
```bash
    stress-ng --cpu 4 --cpu-method matrixprod --timeout 40s --metrics-brief
```

### Baseline CPU Performance
- Throughput: 8,853.77 Bogo Operations per Second

![Figure 3: CPU stress test results showing 8,853 ops/s throughput.](/images/week3/stress.png)


**Interpretation:**
This value represents the server’s raw compute capacity in the absence of firewall inspection, access control mediation, or kernel-level confinement. Any measurable reduction observed after hardening can be causally linked to security enforcement rather than workload variance.

During execution, all four CPU cores were fully saturated, confirming that the scheduler and test harness behaved as expected.

![Figure 4: Real-time visualization of CPU saturation during the stress test.](/images/week3/stress-btop.png)

## 6. Storage Workload Characterization

Security tooling generates sustained small-block write activity, particularly within `/var/log`. To simulate this behavior, a random write workload was executed using asynchronous I/O.

**Command Executed:**
```
    fio --name=baseline_test --ioengine=libaio --rw=randwrite --bs=4k --size=512M --numjobs=1 --time_based --runtime=15 --group_reporting
```

![Figure 5: Disk I/O benchmark showing 132 MiB/s random write speed.](/images/week3/fio.png)

### Baseline Disk Performance
- Bandwidth: 132 MiB/s
- IOPS: ~33,700

Interpretation:  
The results indicate that the storage subsystem can comfortably absorb high-frequency audit and log writes without becoming a bottleneck. This baseline is critical for later comparison once auditd, journald persistence, and intrusion detection logging are enabled.

Any significant deviation from this performance envelope in Week 6 will signal an I/O contention risk.

## 7. Control State Summary

The unsecured performance baseline is summarized below:

| Metric | Week 3 Baseline (Unsecured) |
| :--- | :--- |
| **Security Controls** | None |
| **CPU Throughput** | **8,853.77 Ops/s** |
| **Disk Bandwidth** | **132 MiB/s** |
| **Disk IOPS** | **33,700** |
| **Idle Memory Usage**| **~370 MiB** |

This dataset constitutes the control group for all subsequent regression analysis.

## 8. Conclusion & Transition

Week 3 successfully establishes a scientifically valid performance baseline for the system prior to hardening. By isolating CPU and I/O characteristics under controlled conditions, the project is now equipped to quantify the real cost of security enforcement rather than relying on qualitative assumptions.

The system is therefore ready to enter the Hardening Phase.

In Week 4, cryptographic identity enforcement, firewalling, and privilege restriction will be implemented. These controls are intentionally designed to be measured against the benchmarks defined in this phase.


## 8. References

1.  **Canonical Ltd.** (2024). *Ubuntu Manpage: stress-ng - a tool to load and stress a computer system*. Available at: https://manpages.ubuntu.com/manpages/noble/man1/stress-ng.1.html
2.  **Axboe, J.** (2024). *Fio - Flexible I/O Tester User Guide*. Available at: https://fio.readthedocs.io/en/latest/
3.  **Gregg, B.** (2020). *Systems Performance: Enterprise and the Cloud, 2nd Edition*. Addison-Wesley Professional.
