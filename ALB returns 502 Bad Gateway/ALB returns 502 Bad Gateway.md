# 🛠️ DevOps Lab: Troubleshooting AWS ALB 502 Bad Gateway Errors

This lab covers the scenario where your Application Load Balancer (ALB) successfully listens for public client traffic, but requests fail at the backend proxy boundary, returning an **HTTP 502 Bad Gateway** response to the user's browser.

---

## 🔍 The Load Balancer Triage Checklist

Follow this systematic checklist to identify why the ALB is failing to parse or maintain connection streams with your backend targets:

### 🔄 1. Target Group Protocol Verification
*   **The Problem:** The ALB is attempting an encrypted handshake, but the backend is serving unencrypted text, or vice versa.
*   **Verification:** 
    1. Go to the **EC2 Console** ➡️ **Target Groups** ➡️ Select your target group.
    2. Look at the **Protocol** and **Port** configuration in the details pane.
*   **Remediation:** 
    *   If your Docker container or EC2 application is listening on HTTP port `8080`, your Target Group must be configured explicitly for **HTTP:8080**.
    *   Do not configure the Target Group protocol to HTTPS unless you have installed matching SSL/TLS certificates locally inside your target machine's web server engine (e.g., Nginx/Apache configs).

### ⏳ 2. Web Server Keep-Alive Alignment (The Race Condition)
*   **The Problem:** The backend application server drops the TCP connection socket prematurely while the ALB is still trying to push data down the pipe.
*   **Verification:** Check your internal application web server configuration files (e.g., `/etc/nginx/nginx.conf` or your Node/Python startup configs).
*   **Remediation:** Adjust your internal server metrics so they outlast the ALB idle window. The rule of thumb is **Backend Timeout > ALB Timeout**:
    *   **Nginx Configuration:**
        ```nginx
        keepalive_timeout 65; # Ensure this is greater than the ALB default of 60s
        ```
    *   **Apache Configuration:**
        ```apache
        KeepAliveTimeout 65
        ```
    *   **Node.js Server:**
        ```javascript
        const server = app.listen(8080);
        server.keepAliveTimeout = 65000; // 65 seconds
        server.headersTimeout = 66000;   // Must be greater than keepAliveTimeout
        ```

### 🐳 3. Docker Container Networking & Port Remapping
*   **The Problem:** The application runs inside Docker, and although the target group routes traffic to the EC2 host, the port mapping configuration inside the container engine drops incoming proxy packets.
*   **Verification:** Run `docker ps` on the target EC2 host. Inspect the `PORTS` tracking map.
*   **Remediation:** Ensure your container is actively bound to the host network interface. If your Target Group sends traffic to host port `80`, your container launch statement must translate that interface accurately down to the container's internal listening block:
    ```bash
    docker run -d -p 80:8080 my-web-app
    ```

### 📜 4. Response Header Buffer Constraints
*   **The Problem:** The application returns massive cookie elements, heavily encrypted JWT payloads, or extensive tracing headers that exceed standard ALB processing limits.
*   **Verification:** Enable **ALB Access Logs** and save them to S3. Review the `elb_status_code` alongside the `target_status_code`. If the `elb_status_code` reads `502` and the `target_status_code` reads `-`, the ALB dropped the packet after evaluating the backend header data.
*   **Remediation:** Optimize your application to streamline cookies and headers, or use cloud management rules to clip down extraneous payload tags before sending data up the proxy pipeline.
