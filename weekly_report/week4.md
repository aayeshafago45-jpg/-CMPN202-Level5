---
title: System Hardening & Defense-in-Depth Implementation
description: Transitioning from a vulnerable baseline to a hardened bastion. Implementation of Enterprise RBAC (Group-Based Access Control), Ed25519 cryptographic identity, strict network whitelisting, and immutable audit trails.
author: Aayusha Linbu
date: Week 4: Security Hardening
---

## 1. Objective & Strategic Intent

The vulnerability assessment conducted in Week 2 confirmed that the default system posture was unsuitable for any production or security-sensitive environment. The most critical findings included:

- Password-based SSH authentication enabling rapid credential compromise
- Unrestricted network exposure of the management plane
- A flat privilege model allowing immediate escalation to root following authentication

The objective of Week 4 is to eliminate these attack vectors by transitioning the system from a permissive baseline to a hardened bastion host using a layered **Defense-in-Depth** strategy. Rather than applying isolated configuration changes, this phase restructures the system’s trust and access model to align with enterprise security expectations and high-compliance audit standards.

The guiding principle of this phase is simple:
Security must be enforced structurally, not procedurally.

## 2. Hardening Strategy & Design Rationale

The hardening strategy focuses on four complementary control domains:

1. **Role-Based Access Control (RBAC):** Enforcing separation of duties through group-based policy rather than individual user discretion.
2. **Cryptographic Identity Assurance:** Eliminating reusable secrets in favor of high-entropy asymmetric authentication.
3. **Protocol-Level Enforcement:** Hardening SSH to enforce access policy before authentication occurs.
4. **Forensic Integrity & Perimeter Defense:** Ensuring that intrusion attempts are both restricted and reliably recorded.

Each control layer is designed to remain effective even if another layer is partially bypassed.

## 3. Phase 1: Enterprise Role-Based Access Control (RBAC)
### 3.1 Separation of Duties Model

To mitigate privilege escalation risk, a group-based RBAC model was implemented. This approach ensures that remote access capability and administrative authority are mutually exclusive by default.

Two distinct trust domains were defined:

- **Remote Access Domain:** Users permitted to authenticate via SSH but granted no administrative privileges.
- **Administrative Control Domain:** Users with sudo authority who are explicitly denied remote login capability.

This design prevents direct exposure of privileged accounts to network-based attack vectors.

### 3.2 Implementation

A dedicated administrative controller account was created, and the primary login user was demoted to a non-privileged role.

**Command Executed:**
```bash
    # Create administrative controller
    sudo adduser master_admin

    # Grant administrative privileges
    sudo usermod -aG sudo master_admin

    # Define SSH access group
    sudo groupadd ssh-users
    sudo usermod -aG ssh-users aayusha

    # Remove sudo privileges from entry user
    sudo deluser aayusha sudo
```

![Figure 1: Verification of Group Membership creation and User Demotion.](/images/week4/sudo_group.png)

### 3.3 Verification

Post-implementation validation confirmed that:

- The user `aayusha` retained remote access but lacked administrative authority
- The user `master_admin` possessed full administrative control but was not eligible for SSH authentication

This establishes a strict separation between access and authority, eliminating direct privilege escalation following credential compromise.

## 4. Phase 2: Cryptographic Identity Enforcement

Password-based authentication was identified as the highest-risk control failure during the Week 2 assessment. To remediate this, the authentication model was migrated from knowledge-based secrets to asymmetric cryptographic identity.

### 4.1 Key Generation & Deployment

An Ed25519 key pair was generated on the management workstation and deployed to the target system. The Ed25519 algorithm was selected for its strong security properties, reduced key size, and performance efficiency.

**Command Executed (Host Side):**
```bash
ssh-keygen -t ed25519 -C "week4_hardening"
cat .ssh\id_ed25519.pub | ssh aayusha@192.168.56.104 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

![Figure 2: Successful generation and transfer of Ed25519 keys.](/images/week4/sshkey_added.png)

This establishes a non-reusable, workstation-bound identity that cannot be brute-forced or replayed.

## 5. Phase 3: SSH Protocol Hardening & Policy Enforcement

With cryptographic identity in place, the SSH daemon was hardened to enforce access policy at the protocol level.

### 5.1 SSH Daemon Configuration

The SSH configuration was modified to enforce group-based access control and disable all legacy authentication mechanisms.

**Configuration Applied:**
```ini
# 1. RBAC Enforcement (The "Different" Approach)
# Only allow members of the 'ssh-users' group to connect.
# 'master_admin' is NOT in this group, so they are blocked from remote login.
AllowGroups ssh-users

# 2. Disable Passwords (The Kill Switch)
PasswordAuthentication no
ChallengeResponseAuthentication no
PermitRootLogin no

# 3. Enable Public Key Auth
PubkeyAuthentication yes

# 4. Enhanced Logging (Forensic Readiness)
# Increases log detail to capture key fingerprints for every auth event.
LogLevel VERBOSE
```

![Figure 3: Hardened sshd_config file showing Group Whitelisting and Verbose Logging.](/images/week4/ssh_config_update.png)

### 5.2 Security Impact

This configuration enforces access restrictions before authentication is attempted. Administrative accounts are never exposed to network-based attacks, and password-based brute-force vectors are fully eliminated.

Verbose logging ensures that all authentication attempts are recorded with sufficient detail to support forensic analysis and intrusion detection tooling.

## 6. Phase 4: Forensic Integrity & Network Perimeter Defense

### 6.1 Audit Log Immutability

To preserve forensic integrity, authentication logs were protected using append-only filesystem attributes.

**Command Executed:**
```bash
# 1. Verify Logging Infrastructure
sudo systemctl status rsyslog

# 2. Apply Immutability (Anti-Tamper)
sudo chattr +a /var/log/auth.log

# 3. Verify the Lock
lsattr /var/log/auth.log
```

![Figure 4: Application of the Immutable Attribute (+a) to the authentication log.](/images/week4/chattr.png)
Applying the append-only attribute prevents deletion or modification of authentication logs, even by privileged users. This introduces friction and detectability into any post-compromise cleanup attempt.

### 6.2 Host-Based Firewall Enforcement

To reduce network exposure, a strict default-deny firewall policy was implemented using UFW.

**Command Executed:**
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
# Whitelist ONLY the management workstation IP
sudo ufw allow from 192.168.56.1 to any port 22 proto tcp
sudo ufw enable
```

![Figure 5: UFW Status showing strict IP-based whitelisting.](/images/week4/ufw_setup.png)

Only the trusted management workstation is permitted to initiate SSH connections. All other inbound traffic is silently dropped.

## 7. Verification & Outcome Assessment

Post-hardening validation confirmed the effectiveness of all control layers:

- Privileged accounts cannot authenticate remotely
- Password-based authentication is fully disabled
- Network access is restricted to a single trusted IP
- Authentication logs cannot be altered or deleted

The system has transitioned from a permissive default configuration to a hardened bastion host with enforced separation of duties, cryptographic identity assurance, and reliable forensic visibility.

## 8. Conclusion & Transition

Week 4 successfully establishes a hardened security baseline suitable for hosting sensitive services and supporting advanced security instrumentation. All critical weaknesses identified during the vulnerability assessment have been structurally remediated rather than procedurally mitigated.

With the security perimeter and trust model enforced, the system is now prepared for Week 5, where advanced containment, intrusion prevention, and observability mechanisms will be deployed on top of this hardened foundation.

## 9. References

1.  **NIST.** (2013). *Security Control CA-7: Non-Repudiation*. NIST Special Publication 800-53 Revision 4.
2.  **OpenSSH.** (2024). *sshd_config — OpenSSH daemon configuration file*.
3.  **Kerrisk, M.** (2024). *chattr(1) — Linux manual page*.