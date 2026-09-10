# 🛠️ DevOps Lab: Securely Connecting an EC2 Instance to S3

This lab covers the scenario where an application running on an **EC2 instance needs programmatic access to an Amazon S3 bucket** (e.g., uploading user avatars, downloading application configuration files, or reading log data). 

---

## 🏗️ The Golden Rule: Use IAM Instance Profiles (No Hardcoded Keys!)

> ❌ **The Anti-Pattern:** Generating an IAM User access key pair and embedding it inside your server code or environment variables. This creates a severe security risk if the instance is compromised.
> 
>  **The Best Practice:** Attach an **IAM Role** directly to the EC2 instance using an **IAM Instance Profile**. The AWS SDK running on the machine will automatically fetch short-lived, self-rotating credentials behind the scenes.

---

## 🔍 The Connection & Access Diagnostic Checklist

If your application throws an `AccessDenied` or a `Timeout` error when attempting to reach your S3 bucket from the EC2 instance, walk through this checklist:

### 🎭 1. IAM Instance Profile & Role Attachment
*   **The Problem:** The EC2 instance does not have an IAM role attached, or the attached role lacks permissions to access S3.
*   **Verification:** Go to the **EC2 Console** ➡️ Select your instance ➡️ View the **IAM Role** field in the **Details** tab. 
*   **Remediation:** 
    1. Create an IAM Role with a Trust Policy that allows the `://amazonaws.com` service to assume it.
    2. Attach an identity policy to that role granting permission to your target bucket:
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
                        "arn:aws:s3:::my-application-bucket",
                        "arn:aws:s3:::my-application-bucket/*"
                    ]
                }
            ]
        }
        ```
    3. Attach this role to your EC2 instance via the console or CLI (**Actions** ➡️ **Security** ➡️ **Modify IAM Role**).

### 🤖 2. IMDSv2 (Instance Metadata Service) Check
*   **The Problem:** The AWS SDK on your instance needs to call the Instance Metadata Service (IMDS) at the link-local IP `http://169.254.169.254` to fetch its IAM credentials. If IMDS is turned off or if its "hop limit" is too restrictive, the SDK cannot get tokens.
*   **Verification:** SSH into your instance and try to manually fetch a token:
    ```bash
    TOKEN=\$(curl -X PUT "http://169.254.169" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
    curl -H "X-aws-ec2-metadata-token: \$TOKEN" http://169.254.169
    ```
*   **Remediation:** If the command fails, verify that IMDSv2 is **Enabled** in your EC2 instance details. *Note: If your application runs inside a Docker container on the EC2 instance, you must increase the IMDS Hop Limit to **2** in the EC2 configuration, otherwise the token cannot pass through the Docker bridge network to reach the container.*

### 🧱 3. S3 Bucket Policy & Block Public Access
*   **The Problem:** The EC2 instance's IAM role has permissions, but the S3 bucket's own **Bucket Policy** explicitly blocks access or fails to trust the account.
*   **Verification:** Open the **S3 Console** ➡️ Select your bucket ➡️ View the **Permissions** tab.
*   **Remediation:** Ensure there is no explicit `Deny` statement blocking your instance's IAM Role. If the EC2 instance is located in a different AWS account, the target S3 Bucket Policy must explicitly list your EC2 Role ARN as an allowed **Principal**.

### 🛣️ 4. VPC Routing to S3 (Public vs Private Subnets)
*   **The Problem:** The application hangs and times out whenever it calls S3 API endpoints (`*.s3.amazonaws.com`).
*   **Verification:** Run `curl -I https://amazonaws.com` from inside the EC2 instance.
*   **Remediation:** 
    *   *If the EC2 is in a Public Subnet:* Ensure it has an assigned Public IP and its Route Table points `0.0.0.0/0` to an Internet Gateway.
    *   *If the EC2 is in a Private Subnet:* It requires either a **NAT Gateway** to reach the public S3 endpoints or, more ideally, an **S3 VPC Gateway Endpoint**. 
    *   *Pro-Tip:* Setting up an S3 Gateway Endpoint is completely free and routes S3 traffic securely through AWS's internal network without touching the public internet. Simply go to **VPC Console** ➡️ **Endpoints** ➡️ **Create Endpoint** ➡️ Select `com.amazonaws.<region>.s3` (Gateway type) and associate it with your subnet's Route Table.
