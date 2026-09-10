# 🥼 Hands-On DevOps Lab: Troubleshooting Cross-EC2 Communication Failures

## 🎯 Lab Objectives
1. Deploy two EC2 instances inside a custom VPC.
2. Manually break the infrastructure to simulate a real-world network outage.
3. Use standard Linux and AWS diagnostic tools to identify and fix the network bottlenecks.

---

## 🏗️ Phase 1: Lab Architecture Setup (The Baseline)

Log into your **AWS Management Console** and build the following architecture components manually:

### 1. Networking Infrastructure
*   **VPC:** Create a custom VPC named `Lab-VPC` with a CIDR block of `10.0.0.0/16`.
*   **Subnets:** Create two distinct subnets inside your new VPC:
    *   `Subnet-Alpha` (CIDR: `10.0.1.0/24`)
    *   `Subnet-Beta` (CIDR: `10.0.2.0/24`)
*   **Internet Gateway:** Create an IGW, attach it to `Lab-VPC`, and add a default route (`0.0.0.0/0` ➡️ `igw-xxxxxx`) in the subnets' Route Tables so you can SSH into your lab targets.

### 2. Compute Instances
*   Launch two instances using the standard **Amazon Linux 2023** or **Ubuntu** AMI:
    *   **Instance-Alpha** (Frontend Node): Deploy into `Subnet-Alpha`. Assign it a unique Security Group named `SG-Alpha`.
    *   **Instance-Beta** (Backend Target): Deploy into `Subnet-Beta`. Assign it a unique Security Group named `SG-Beta`.

---

## 🛑 Phase 2: Intentionally Breaking the Environment

To make this a genuine troubleshooting exercise, implement these **three deliberate misconfigurations**:

1.  **The Security Group Wall:** Go to `SG-Beta` (the backend instance's firewall) and **delete all inbound rules**. It should be completely blank.
2.  **The Stateless NACL Trap:** Go to the Network ACL attached to `Subnet-Beta`. Add a custom **Inbound Deny Rule** at rule number `50` blocking traffic coming from the `Subnet-Alpha` CIDR block (`10.0.1.0/24`).
3.  **The Local OS Block:** (We will execute this on the OS layer in Phase 3).

---

## 🔍 Phase 3: The Hands-On Troubleshooting Execution

### 📋 Scenario Brief
You are the on-call DevOps Engineer. The application developers report that the application on **Instance-Alpha** cannot send database queries to **Instance-Beta** on Port `5432` (PostgreSQL), and they cannot even ping the server. 

Your job is to log into `Instance-Alpha` and trace the failure through the network stacks.

### Step 1: Initial Discovery & Connectivity Verification
SSH into **Instance-Alpha** from your workstation and attempt to test the network socket of **Instance-Beta**:

```bash
# 1. Attempt to ping the backend private IP
ping <INSTANCE_B_PRIVATE_IP>
# (Observe: The command hangs indefinitely)

# 2. Use Netcat or Telnet to test the specific database application port
nc -zv <INSTANCE_B_PRIVATE_IP> 5432
# (Observe: The terminal hangs and times out)
```

> 🤔 **DevOps Diagnosis:** A connection that **hangs/times out** means packets are being silently dropped by a firewall (AWS Security Group or NACL). If it returned "Connection Refused", it would mean the network path is open but the app isn't running.

---

### Step 2: Breaking Through Layer 1 — The AWS Security Group
1.  Navigate to the AWS Console ➡️ **EC2 Instances** ➡️ Select `Instance-Beta`.
2.  Click the **Security** tab and open `SG-Beta`.
3.  **The Fix:** Click **Edit inbound rules**. Add a new rule:
    *   **Type:** Custom TCP
    *   **Port Range:** `5432`
    *   **Source:** Instead of typing an IP block, type `SG-Alpha` and select the Security Group ID of your frontend instance. 
4.  Go back to your terminal on `Instance-Alpha` and re-run `nc -zv <INSTANCE_B_PRIVATE_IP> 5432`.
    *   *Result:* The command **still times out!** We have cleared the Security Group layer, but another firewall layer is dropping the packets.

---

### Step 3: Breaking Through Layer 2 — The Subnet Network ACL
1.  Navigate to the AWS Console ➡️ **VPC Dashboard** ➡️ **Subnets** ➡️ Select `Subnet-Beta`.
2.  Click the **Network ACL** tab. Notice the explicit Deny rule blocking `10.0.1.0/24`.
3.  **The Fix:** Edit the inbound rules. **Delete** the rule that explicitly denies traffic from `Subnet-Alpha`'s subnet block, or change its action to `ALLOW`.
4.  Return to your terminal on `Instance-Alpha` and test the connection:
    ```bash
    nc -zv <INSTANCE_B_PRIVATE_IP> 5432
    ```
    *   *Result:* The connection will now return `Connection refused`. 
    *   *Why?* The network path across the AWS cloud infrastructure is now **100% open**, but there is no application currently listening on port 5432 inside the target OS!

---

### Step 4: Final Validation — Mocking the Backend Application
To prove that your network engineering fixes worked, you must simulate an active application running on port `5432` inside **Instance-Beta**:

1. Open a separate terminal window and **SSH directly into Instance-Beta**.
2. Run a temporary netcat listener on the database port to fake a running database service:
   ```bash
   sudo nc -l 5432
   ```
3. Return to your original terminal window on **Instance-Alpha** and run the test a final time:
   ```bash
   nc -zv <INSTANCE_B_PRIVATE_IP> 5432
   ```

### 🎉 Expected Success Output:
```text
Connection to <INSTANCE_B_PRIVATE_IP> 5432 port [tcp/postgres] succeeded!
```
