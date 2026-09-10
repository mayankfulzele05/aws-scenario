# 🛠️ DevOps Lab: Application Works on Localhost but Not Through EC2 Public IP

This lab covers the scenario where your application (e.g., Node.js, Python/Flask, Docker container) runs flawlessly inside the EC2 instance via `localhost`, but attempting to access it from the outside world using `http://<EC2_PUBLIC_IP>:<PORT>` results in a **Connection Timed Out** or **Connection Refused** error.

---

## 🔍 The Infrastructure & Application Diagnostic Checklist

When an app works locally but fails externally, the issue is caused by one of two boundaries: **AWS Network Security** (blocking the traffic) or **Application Binding** (the app refusing to listen to external traffic). Follow this workflow to isolate the bug:

### 🔒 1. Security Group Inbound Rules
*   **The Problem:** The stateful AWS firewall wrapping the EC2 instance is blocking the specific port your application is listening on.
*   **Verification:** Go to the **EC2 Console** ➡️ Select the instance ➡️ Click the **Security** tab ➡️ View **Inbound rules**.
*   **Remediation:** 
    *   If your app runs on port `8080`, `3000`, or `5000`, there must be an explicit **Custom TCP** inbound rule for that exact port.
    *   Ensure the source is set to `0.0.0.0/0` (Anywhere) for testing, or your specific local IP address. 
    *   *Note: Standard HTTP (Port 80) and HTTPS (Port 443) rules will not allow traffic through to custom application ports.*

### 🔄 2. Application Binding (The `0.0.0.0` vs `127.0.0.1` Trap)
*   **The Problem:** Your application framework is configured to bind exclusively to the local loopback interface (`127.0.0.1` or `localhost`). When configured this way, the operating system drops any incoming network requests arriving from the external internet via the EC2 Public IP.
*   **Verification:** SSH into your EC2 instance and run the network statistics command to see what interface your port is listening on:
    ```bash
    sudo netstat -tuln | grep <YOUR_PORT>
    # OR
    sudo ss -tuln | grep <YOUR_PORT>
    ```
    *   ❌ **Bad:** If you see `127.0.0.1:8080`, your app is isolated inside the host.
    *   ✅ **Good:** If you see `0.0.0.0:8080` (IPv4) or `:::8080` (IPv6), it is listening to all network interfaces.
*   **Remediation:** Modify your application code or startup script to bind to `0.0.0.0`.
    *   *Flask:* `app.run(host='0.0.0.0', port=8080)`
    *   *Node.js:* `server.listen(8080, '0.0.0.0');`
    *   *FastAPI / Uvicorn:* `uvicorn main:app --host 0.0.0.0 --port 8080`

### 🐳 3. Docker Container Port Mapping
*   **The Problem:** If you are running the application inside a Docker container, the app might be running fine *inside* the container container network, but the port hasn't been exposed or mapped to the EC2 host machine.
*   **Verification:** Run `docker ps` on the EC2 instance. Look at the `PORTS` column.
    *   ❌ **Bad:** `8080/tcp` (The port is open internally in the container, but not exposed to the EC2 host).
    *   ✅ **Good:** `0.0.0.0:8080->8080/tcp` (The host port 8080 points directly to container port 8080).
*   **Remediation:** Stop the container and restart it using the explicit `-p` port mapping flag:
    ```bash
    docker run -d -p 8080:8080 my-app-image
    ```

### 🧱 4. Host OS Internal Firewall (UFW / iptables)
*   **The Problem:** AWS Security Groups are allowing the traffic, but the local Linux operating system running inside your EC2 instance has its own built-in software firewall blocking the port.
*   **Verification:** Check the internal firewall status.
    ```bash
    # For Ubuntu/Debian:
    sudo ufw status
    
    # For RHEL/Amazon Linux:
    sudo firewall-cmd --state
    ```
*   **Remediation:** Open the port locally within the OS:
    ```bash
    # For Ubuntu/Debian:
    sudo ufw allow 8080/tcp
    
    # For RHEL/Amazon Linux:
    sudo firewall-cmd --permanent --add-port=8080/tcp
    sudo firewall-cmd --reload
    ```
