# 🛠️ DevOps Lab: Troubleshooting AWS EC2 SSH Connectivity

This lab provides a hands-on troubleshooting scenario where an **EC2 instance is running, but you cannot SSH into it**. When a connection times out or drops, the issue almost always lies in the networking path, firewalls, or credential verification. 

---

## 🔍 The Infrastructure Diagnostic Checklist

When your terminal command hangs indefinitely on `ssh -i lab-ssh-key.pem ec2-user@<PUBLIC_IP>`, walk through the infrastructure stacks sequentially from the outside world inward to isolate the bottleneck:

### 🌐 1. Public IP Architecture Check
*   **The Problem:** The target EC2 instance does not have an accessible public routing endpoint. Note that restarting standard instances reallocates public IPs dynamically unless a static Elastic IP (EIP) is tied to it.
*   **Verification:** Navigate to the **EC2 Dashboard** ➡️ **Instances** and select your machine. Ensure the `Public IPv4 address` or `Public IPv4 DNS` field contains an assigned value.

### 🛣️ 2. Internet Gateway (IGW) & Route Tables
*   **The Problem:** The instance resides in a subnet configured as an isolated environment without an outbound gateway target map. It cannot see the public internet.
*   **Verification:** Open the **VPC Console** ➡️ **Subnets** ➡️ Select your subnet ➡️ View the **Route Table** tab.
*   **Remediation:** There must be a destination rule for `0.0.0.0/0` explicitly pointing to your attached Internet Gateway ID (`igw-xxxxxx`).

### 🛡️ 3. Network Access Control Lists (NACL)
*   **The Problem:** NACLs function on a **stateless** paradigm. Allowing traffic inbound via port 22 does not automatically authorize the corresponding exit payload.
*   **Verification:** Open the **VPC Console** ➡️ **Subnets** ➡️ Select your subnet ➡️ View the **Network ACL** tab.
*   **Remediation:** 
    *   *Inbound:* Explicitly allow Port 22 from your origin client IP or `0.0.0.0/0`.
    *   *Outbound:* Explicitly allow traffic targeting **Ephemeral Ports (1024-65535)** so the response handshake can travel back to your machine.

### 🔒 4. Security Group (SG) Rules
*   **The Problem:** The cloud firewall wrapping the instance's Elastic Network Interface (ENI) drops incoming SSH handshakes.
*   **Verification:** Go to the **EC2 Console** ➡️ Select your instance ➡️ Check the **Security** tab.
*   **Remediation:** Verify or append an **Inbound Rule** establishing Type: `SSH`, Port: `22`, Source: `My IP` or `0.0.0.0/0`. *(Security groups are stateful; outbound rules don't require explicit responses matching inbound allowances)*.

### 🔑 5. Operating System Level Validation (SSH Service & Key Permissions)
*   **The Problem:** The physical network connectivity maps correctly, but the host Operating System abruptly closes or denies authorization due to client security policies or bad key contexts.
*   **Remediation & Validation Checks:**
    *   **File Permissions:** If your private key configuration file permits wide-open visibility, OpenSSH clients intentionally terminate operations. Enforce strict permissions via your local terminal: `chmod 400 lab-ssh-key.pem`.
    *   **OS Username Matrix:** Verify your connection command targets the precise default system user context bundled by the target AMI vendor:
        *   `ec2-user` for Amazon Linux 2 / Amazon Linux 2023
        *   `ubuntu` for Ubuntu Images
        *   `admin` for Debian distributions
        *   `centos` for CentOS Images
