# 🛠️ DevOps Lab: Two EC2 Instances Cannot Communicate

This lab covers the scenario where **Instance A (Client/Frontend) cannot establish a connection with Instance B (Server/Database/Backend)**. This applies whether they are trying to communicate via Ping (ICMP), SSH, HTTP, or a custom database port.

---

## 🔍 The Cross-Instance Network Diagnostic Checklist

To resolve communication failures between two cloud instances, work your way from the local instance settings out to the global AWS network boundaries:

### 📁 1. The Single vs. Multi-VPC Boundary
*   **The Problem:** The two instances are running in completely different Virtual Private Clouds (VPCs). By default, separate VPCs are isolated networks and cannot route traffic directly to each other.
*   **Verification:** Open the **EC2 Console** ➡️ Select **Instance A** ➡️ Check its `VPC ID`. Repeat for **Instance B**.
*   **Remediation:** 
    *   *If in the same VPC:* Move to Step 2.
    *   *If in different VPCs:* They cannot communicate over private IPs without networking bridges. You must set up a **VPC Peering Connection** or a **Transit Gateway**, and update the **Route Tables** in both VPCs to route target CIDR blocks through that peering connection.

### 🛣️ 2. Subnet Routing & Target IP Addressing (Public vs. Private IPs)
*   **The Problem:** Instance A is trying to reach Instance B using its **Public IP** instead of its **Private IP**, forcing the traffic out to the public internet and back, or they are in different subnets with broken routing.
*   **Verification:** Check your application configuration or connection string. Ensure you are targeting the **Private IPv4 Address** of Instance B.
*   **Remediation:** Always use Private IPs for internal AWS traffic. If they are in the same VPC but different subnets, check the **VPC Route Tables** for both subnets. Ensure there is a default local route (e.g., Destination: `10.0.0.0/16`, Target: `local`) allowing subnets within the same VPC to talk to each other.

### 🔒 3. Security Group Self-Reference & Port Allowances
*   **The Problem:** The Security Group protecting Instance B is blocking the specific port or incoming IP address of Instance A.
*   **Verification:** Go to **EC2 Console** ➡️ Select **Instance B** ➡️ Click the **Security** tab ➡️ View **Inbound rules**.
*   **Remediation:** 
    *   *The Tight Security Approach:* Instead of opening the port to the entire VPC CIDR block, modify the Inbound Rules of Instance B's Security Group. Add a rule allowing the specific protocol/port (e.g., PostgreSQL Port `5432`), and set the **Source** to the **Security Group ID of Instance A** (e.g., `sg-0123456789abcdef0`).
    *   AWS natively allows Security Groups to reference each other. This ensures that even if Instance A changes its private IP due to a reboot, it will still retain access.

### 🛡️ 4. Subnet Network ACL (NACL) Boundaries
*   **The Problem:** The two instances live in different subnets within the same VPC, and a custom **Network ACL (NACL)** attached to one of the subnets is blocking the cross-subnet traffic.
*   **Verification:** Go to **VPC Console** ➡️ **Subnets** ➡️ Check the **Network ACL** tab for both subnets.
*   **Remediation:** Because NACLs are **stateless**, you must ensure that:
    *   Subnet A's NACL allows *Outbound* traffic to Subnet B's CIDR, and *Inbound* traffic from Subnet B on **Ephemeral Ports (1024-65535)**.
    *   Subnet B's NACL allows *Inbound* traffic from Subnet A's CIDR, and *Outbound* traffic back to Subnet A on **Ephemeral Ports (1024-65535)**.

### 🧬 5. Internal Host Firewalls (UFW / firewalld)
*   **The Problem:** The AWS network infrastructure is completely clear, but the local Linux Operating System operating inside Instance B is actively dropping the packets.
*   **Verification:** SSH into both instances. From Instance A, run a netcat or telnet test: `nc -zv <Instance_B_Private_IP> <Port>`. If it says "Connection refused" immediately (rather than timing out), the port is likely closed or blocked by the OS firewall.
*   **Remediation:** SSH into Instance B and check the native OS firewall status:
    ```bash
    # For Ubuntu/Debian:
    sudo ufw status
    
    # For RHEL/Amazon Linux:
    sudo firewall-cmd --state
    ```
    If active, add an explicit exemption rule within the OS to allow internal traffic from Instance A's private IP block.
