# 🛠️ DevOps Lab: EC2 Instance Cannot Access an S3 Bucket

This lab covers the scenario where an application, tool, or Docker container running inside an **EC2 instance fails to access an Amazon S3 bucket**. This manifests either as an immediate `AccessDenied` authorization error or a network connection `Timeout`.

---

## 🔍 Diagnostic & Resolution Workflow

Follow this systematic checklist from the internal instance environment outward to pinpoint and repair the infrastructure block:

### 🎭 1. IAM Instance Profile & Identity Verification
*   **The Symptoms:** The application immediately throws an `AccessDenied` or `ExpiredToken` error.
*   **The Culprit:** The EC2 instance lacks a valid IAM Identity wrapper, or the attached policy does not grant explicit permissions to your specific bucket resource name.
*   **How to Resolve:**
    1. Open the **IAM Console** and create an IAM Role targeting the Trusted Entity: `://amazonaws.com`.
    2. Attach a customer-managed permission policy allowing access to your exact bucket:
        ```json
        {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Action": [
                        "s3:GetObject",
                        "s3:PutObject",
                        "s3:ListBucket"
                    ],
                    "Resource": [
                        "arn:aws:s3:::my-target-bucket",
                        "arn:aws:s3:::my-target-bucket/*"
                    ]
                }
            ]
        }
        ```
    3. Navigate to the **EC2 Console** ➡️ Select your target instance ➡️ Click **Actions** ➡️ **Security** ➡️ **Modify IAM Role** and bind your new role.

### 🤖 2. The IMDSv2 Docker Hop Limit (Container Isolation)
*   **The Symptoms:** The AWS CLI works perfectly on the base EC2 host, but your application running inside a **Docker Container** or **Kubernetes Pod** fails to fetch credentials.
*   **The Culprit:** The Instance Metadata Service (IMDSv2) token has a default network hop limit of `1`. The token gets dropped the second it passes through the Docker bridge network to reach the container.
*   **How to Resolve:**
    1. Navigate to the **EC2 Console** ➡️ Select the instance.
    2. Click **Actions** ➡️ **Instance Settings** ➡️ **Modify instance metadata options**.
    3. Change the **Metadata response hop limit** from `1` to `2`. This lets the token pass safely from the host interface down into your containerized network stack.

### 🧱 3. S3 Bucket Policy Hardening
*   **The Symptoms:** The IAM Instance Profile is configured correctly, but you still receive an `AccessDenied` error message.
*   **The Culprit:** The S3 bucket has a resource-level **Bucket Policy** containing an explicit `Deny` statement (e.g., enforcing specific IP ranges or blocking public access) that overrides your IAM role permissions.
*   **How to Resolve:**
    1. Navigate to the **S3 Console** ➡️ Select your bucket ➡️ Open the **Permissions** tab.
    2. Review the **Bucket Policy**. Ensure there is no explicit `Deny` block contradicting your actions. 
    3. If your EC2 instance resides in a *different* AWS account, you must add an explicit `Allow` statement to the S3 bucket policy listing the EC2 Role ARN as the trusted **Principal**.

### 🛣️ 4. VPC Network Routing (Connection Timeouts)
*   **The Symptoms:** The command completely hangs in the terminal and eventually drops with a `Connection Timeout` error.
*   **The Culprit:** The EC2 instance cannot find a structural route out of the VPC network to reach the public S3 web endpoints.
*   **How to Resolve:**
    *   *If the EC2 is in a Public Subnet:* Ensure the instance possesses a valid Public IP and its Route Table directs `0.0.0.0/0` to an Internet Gateway (`igw-xxxxxx`).
    *   *If the EC2 is in a Private Subnet (Best Practice):* Deploy an **S3 VPC Gateway Endpoint** to bypass the internet entirely.
    *   *Deployment Steps:* Open the **VPC Console** ➡️ **Endpoints** ➡️ **Create Endpoint**. Search for the service name `com.amazonaws.<region>.s3` (Verify Type is **Gateway**). Select your VPC, check the box next to your private subnet's Route Table, and save. AWS will automatically update your routing tables to securely send S3 traffic across its private internal network framework for free.
