# 🛠️ DevOps Lab: EC2 Has a Public IP But Cannot Access the Internet

This lab covers the scenario where an **EC2 instance has an assigned Public IP address, but cannot establish any outbound connections to the internet** (e.g., `curl` or `ping` requests hang indefinitely, and package managers like `yum`, `apt`, or `apk` fail to download updates).

---

## 🔍 The Networking Diagnostic Checklist

Having a Public IP is only *half* of the requirement for internet connectivity in AWS. The infrastructure surrounding the instance must also be configured to map that public address to the outside world. Follow this workflow to isolate the configuration failure:

### 🛣️ 1. Internet Gateway (IGW) Attachment & Route Table Mapping
*   **The Problem:** The instance is sitting in a subnet whose Route Table does not know how to send traffic to the public internet. A public IP is useless if the routing layer is broken.
*   **Verification:** 
    1. Go to the **VPC Console** ➡️ **Subnets** ➡️ Select the subnet hosting your EC2 instance.
    2. Click on the **Route Table** tab.
*   **Remediation:** Look at the routes. To access the internet, there must be a default route targeting **all traffic (`0.0.0.0/0`)** pointing directly to your Internet Gateway ID (`igw-xxxxxx`). If this rule is missing, click **Edit routes** and add it.

### ⚓ 2. Internet Gateway (IGW) VPC Attachment State
*   **The Problem:** The Internet Gateway exists and is referenced in the Route Table, but the IGW itself is detached from your specific VPC.
*   **Verification:** Go to the **VPC Console** ➡️ **Internet Gateways** ➡️ Select your IGW.
*   **Remediation:** Check the **State** column. If it says `detached`, select **Actions** ➡️ **Attach to VPC**, and choose the VPC where your EC2 instance resides.

### 🛡️ 3. Stateless Network ACL (NACL) Outbound Restrictions
*   **The Problem:** The Security Group allows your outbound request, but the Subnet's **Network ACL (NACL)** is blocking the return traffic. Because NACLs are **stateless**, you must explicitly allow both the request going out and the response coming back in.
*   **Verification:** Go to the **VPC Console** ➡️ **Subnets** ➡️ Select your subnet ➡️ View the **Network ACL** tab.
*   **Remediation:** 
    *   *Outbound Rules:* Ensure there is an `ALLOW` rule for destination `0.0.0.0/0` on the port you are targeting (or all traffic).
    *   *Inbound Rules:* When your instance downloads a package or calls an external API, the internet sends the response back on **Ephemeral Ports (1024-65535)**. You must ensure your Inbound NACL rules allow traffic on ports `1024-65535` from source `0.0.0.0/0`.

### 🔒 4. Security Group Outbound (Egress) Rules
*   **The Problem:** The Security Group applied to your instance has had its default outbound rules deleted or restricted, blocking the server from initiating connections.
*   **Verification:** Go to the **EC2 Console** ➡️ Select your instance ➡️ Click the **Security** tab ➡️ View **Outbound rules**.
*   **Remediation:** By default, AWS Security Groups allow all outbound traffic (`0.0.0.0/0` All Traffic). If someone hardened the security group and removed this, you must explicitly add an outbound rule allowing your required traffic (e.g., HTTP Port 80 and HTTPS Port 443) to `0.0.0.0/0`.

### 🧬 5. Subnet Auto-Assign Public IP Settings (The "Manual IP" Trap)
*   **The Problem:** If you manually attached a public IP to an instance during launch, but the underlying Subnet configuration has `Auto-assign public IPv4 address` set to **False**, certain automated configuration scripts or secondary network interfaces (ENIs) might lose track of the public routing state.
*   **Verification:** Go to the **VPC Console** ➡️ **Subnets** ➡️ Select your subnet. Look for the property `Auto-assign public IPv4 address`.
*   **Remediation:** For true public subnets, modify the subnet settings and check the box to **Enable auto-assign public IPv4 address**. This ensures any resource spun up in this subnet automatically gains public routing privileges without manual intervention.
