# 🛠️ DevOps Lab: Troubleshooting a Completely Unreachable EC2 Instance

This lab covers the scenario where an **EC2 instance is completely unreachable** via any network protocol (SSH, HTTP, ICMP/Ping). The instance appears "dead" to the outside world, even though the AWS Console states its status is `Running`.

---

## 🔍 The Infrastructure Diagnostic Checklist

When an instance goes completely dark, follow this workflow moving from core AWS infrastructure down to the hardware hypervisor and operating system level:

### 🚦 1. Status Check Failures
*   **The Problem:** The instance is failing its foundational health checks. AWS performs two checks every minute: **System Status** (AWS hardware issues) and **Instance Status** (OS/software issues).
*   **Verification:** Navigate to the **EC2 Dashboard** ➡️ **Instances** ➡️ Select your machine ➡️ View the **Status checks** tab.
*   **Remediation:** 
    *   *System Status Failure (Hardware):* Stop and start the instance. This forces AWS to migrate the virtual machine to a different, healthy physical host.
    *   *Instance Status Failure (Software):* The OS is likely frozen, kernel-panicked, or out of memory. Reboot the instance via the console.

### 🌐 2. VPC & Subnet Network Isolation
*   **The Problem:** The instance was launched into a subnet that has no structural path to the outside world, or its Network Interface (ENI) has detached/misconfigured IP targeting.
*   **Verification:** Check the instance details in the **EC2 Console**.
*   **Remediation:** 
    *   Ensure the instance has a **Public IPv4 address** assigned (if you are trying to reach it directly over the public internet).
    *   Verify the subnet ID and check the **VPC Route Table** to ensure the entire subnet isn't isolated from your Internet Gateway (`0.0.0.0/0` must point to an `igw-xxxxxx`).

### 🛡️ 3. Total Firewall Block (Security Groups & NACLs)
*   **The Problem:** The firewalls are dropping **all** inbound and outbound traffic, mimicking a dead server.
*   **Verification:** Open the **Security** tab of the instance and check both the Security Group rules and the Subnet's Network ACL rules.
*   **Remediation:** 
    *   *Security Group:* Ensure there isn't a total absence of inbound rules. SGs drop all traffic by default if no explicit allow rules exist.
    *   *NACL:* Ensure there isn't an explicit `DENY` rule with a lower rule number than your `ALLOW` rules. NACLs process rules in strict numerical order.

### 📼 4. Boot Diagnostics & Instance Console Output
*   **The Problem:** The operating system crashed during boot, encountered a corrupted disk sector, or failed to configure its internal network interfaces during startup.
*   **Verification:** Go to **EC2 Console** ➡️ Select the instance ➡️ Click **Actions** ➡️ **Monitor and troubleshoot** ➡️ **Get system log**.
*   **Remediation:** Analyze the kernel dump or boot log text. Look for phrases like `Kernel panic`, `Waiting for root device`, or `Failed to start Network Service`. If the disk is corrupted, you will need to detach the EBS volume and attach it to a recovery instance to fix the OS configuration.

### 🖼️ 5. Instance Screenshot Verification
*   **The Problem:** The system log is blank, but the operating system might be stuck on a specific boot screen or Windows blue screen (BSOD).
*   **Verification:** Go to **EC2 Console** ➡️ Select the instance ➡️ Click **Actions** ➡️ **Monitor and troubleshoot** ➡️ **Get instance screenshot**.
*   **Remediation:** This provides a literal picture of what a physical monitor plugged into the VM would show. If it shows a crash screen, a forced reboot or volume recovery is required.
