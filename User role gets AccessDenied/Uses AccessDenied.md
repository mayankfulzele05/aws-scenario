# 🛠️ DevOps Lab: Troubleshooting IAM AccessDenied Errors

This lab covers the scenario where an **IAM User or IAM Role receives an `AccessDenied` or `UnauthorizedOperation` error** when attempting to interact with an AWS resource (e.g., creating an S3 bucket, terminating an EC2 instance, or reading a DynamoDB table). 

---

## 🔍 The IAM Policy Evaluation Diagnostic Checklist

AWS evaluates permissions using a strict hierarchy. When troubleshooting an `AccessDenied` error, look at the five major policy layers that control the final authorization decision:

### 🛑 1. The Explicit Deny Rule (The Absolute Override)
*   **The Problem:** An `AccessDenied` error occurs because a policy explicitly contains `"Effect": "Deny"`. In AWS, an **Explicit Deny always wins** over any `Allow` rule, no matter where it is defined.
*   **Verification:** Search through all attached policies for the word `"Deny"`.
*   **Remediation:** Check if your IP address, time of day, or tag restrictions are triggering a conditional Deny block (e.g., a policy that denies all actions if Multi-Factor Authentication is not active).

### 👤 2. Identity-Based Policies
*   **The Problem:** The IAM User or Role executing the command simply lacks an attached policy granting permission for that specific API action on that resource.
*   **Verification:** Open the **IAM Console** ➡️ Go to **Users** or **Roles** ➡️ Select your identity ➡️ Check the **Permissions** tab.
*   **Remediation:** Attach a managed policy (like `AmazonEC2FullAccess`) or construct an inline customer-managed policy explicitly allowing the action:
    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": "s3:ListBucket",
                "Resource": "arn:aws:s3:::my-target-bucket"
            }
        ]
    }
    ```

### 📦 3. Resource-Based Policies
*   **The Problem:** You are trying to access a resource (like an S3 bucket, SQS queue, KMS key, or Secrets Manager secret) that has its own security policy attached, and that policy does not trust or allow your IAM identity.
*   **Verification:** Open the specific resource's console (e.g., **S3 Console** ➡️ Select your bucket ➡️ **Permissions** tab ➡️ **Bucket Policy**).
*   **Remediation:** Ensure the resource-based policy explicitly lists your IAM User/Role ARN as a **Principal** with allowed actions, or ensure it does not contain a blanket `Deny` statement blocking external accounts.
    *   *Note on S3:* Also verify that the **Block Public Access** settings are not conflicting with your intended operations.

### 🌐 4. Service Control Policies (SCPs)
*   **The Problem:** Your application is running inside an AWS account that belongs to an **AWS Organization**. The parent Organization administrator has applied a Service Control Policy (SCP) that restricts that specific service or action across the entire account.
*   **Verification:** (Requires administrative root organization access) Open the **AWS Organizations Console** ➡️ Select the **AWS Account** ➡️ View the **Policies** tab to see attached SCPs.
*   **Remediation:** Even if an IAM Role has `AdministratorAccess` inside its local account, an SCP restricting `ec2:*` will completely block everyone in that account from launching instances. The parent organization admin must modify the SCP.

### 🚧 5. Permissions Boundaries
*   **The Problem:** A Permissions Boundary is an advanced feature used to delegate permissions. It sets the **maximum allowable permissions** an identity can ever perform. If an action is allowed in your Identity Policy but *not* allowed in your Permissions Boundary, the action is blocked.
*   **Verification:** Open the **IAM Console** ➡️ Select the User or Role ➡️ Scroll down to the **Permissions Boundary** section to see if one is applied.
*   **Remediation:** Update the Permissions Boundary policy to include the missing actions. The final permission is the intersection (the overlap) of the Identity Policy and the Permissions Boundary.

---

## 🛠️ Pro-Tip: Automated Troubleshooting Tools

Instead of guessing which layer is blocking you, utilize these native AWS diagnostic tools:

1.  **AWS CloudTrail:** Look up the failed event in CloudTrail log history. The error message often specifies if it was an explicit deny or missing permission.
2.  **IAM Policy Simulator:** Go to the AWS IAM Policy Simulator console, select your user/role, type in the failing action (e.g., `s3:GetObject`), and run a simulation to see exactly which policy is causing the block.
3.  **AWS CLI dry-run flag:** For many services like EC2, append `--dry-run` to your command. This checks if you have permissions without actually creating or destroying resources:
    ```bash
    aws ec2 terminate-instances --instance-ids i-0123456789abcdef0 --dry-run
    ```
