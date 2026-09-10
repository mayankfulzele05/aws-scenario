# 🛠️ DevOps Lab: Troubleshooting AWS ALB 503 Service Unavailable Errors

This lab covers the scenario where your Application Load Balancer (ALB) returns an **HTTP 503 Service Unavailable** response to users. This indicates that the load balancer cannot find a viable, healthy backend container or server to handle the incoming request.

---

## 🔍 The Load Balancer Triage Checklist

Follow this systematic workflow to determine why your backend targets are dropping out of the load balancer's routing pool:

### 🏥 1. Target Group Health Check Auditing
*   **The Problem:** The application is running inside the EC2 instance or Docker container, but the ALB marks it as `Unhealthy` because it gets an unexpected response code during its periodic pings.
*   **Verification:** 
    1. Go to the **EC2 Console** ➡️ **Target Groups** ➡️ Select your target group.
    2. Click the **Targets** tab and inspect the **Health Status** column.
*   **Remediation:** 
    *   Check the exact path your ALB is pinging under the **Health checks** tab (e.g., is it hitting `/` or `/health`?).
    *   Ensure your application code explicitly serves an **HTTP 200 OK** response on that exact path. If your app redirects `/` to a login page (`HTTP 302`), the ALB will mark it as unhealthy by default unless you append `302` to the **Success codes** match string.

### 🔌 2. Security Group Network Path Block
*   **The Problem:** Your application is fine, but a firewall is blocking the ALB from hitting the application's port, making the health check fail.
*   **Verification:** Go to **EC2 Console** ➡️ **Instances** ➡️ Select your host instance ➡️ View the **Security** tab.
*   **Remediation:** 
    *   The Security Group attached to your **EC2 instances / Docker host** must explicitly allow inbound traffic on your application port (e.g., Port `8080`).
    *   **Best Practice:** Set the **Source** of that inbound rule to the **Security Group ID of the ALB**. This locks down your backend so *only* the load balancer can talk to it, keeping it hidden from the public internet.

### 📁 3. Empty Target Registration
*   **The Problem:** Your infrastructure is running, but it has not been wired up to the load balancer's listener.
*   **Verification:** View the **Targets** tab inside your Target Group.
*   **Remediation:** If the registered targets count is `0`, your instances were never associated. Manually click **Register targets** to add them for testing, or update your Auto Scaling Group (ASG) / ECS Service configuration to automatically register new nodes into this Target Group upon boot.

### 🐳 4. Docker Container Resource Exhaustion
*   **The Problem:** The app starts fine, but as soon as traffic hits, the container exhausts its allocated memory or CPU, freezes, and drops the ALB's health check packets.
*   **Verification:** SSH into your EC2 host and run container resource diagnostics:
    ```bash
    docker stats
    ```
*   **Remediation:** Look for instances where memory utilization is pinned at nearly 100%. If your container hits its hard memory limit, the Linux kernel will trigger an OOM (Out Of Memory) kill event, or the app will become unresponsive. Increase the container resource allocations or scale out your architecture with more EC2 target nodes.
