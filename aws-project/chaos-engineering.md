
---

# Lab Report: Implementing Chaos Engineering with AWS FIS and Terraform

**Date:** January 2, 2026
**Project Type:** Resilience Engineering / SRE
**Tools:** Terraform, AWS FIS, EC2 Auto Scaling, CloudWatch, PowerShell/Bash

---

## 1. Executive Summary

This project shifted focus from standard infrastructure provisioning (DevOps) to **Site Reliability Engineering (SRE)**. The objective was to not only build a highly available web architecture using **Terraform** but to actively verify its resilience by injecting failure using **AWS Fault Injection Service (FIS)**.

We successfully deployed a "Game Day" scenario where we intentionally terminated production-like instances to validate the automatic failover and self-healing capabilities of the Auto Scaling Group (ASG) and Application Load Balancer (ALB).

---

## 2. Architecture Overview

The infrastructure was defined entirely as code (IaC) using Terraform, ensuring reproducibility.

### Core Components

* **Network:** A custom VPC (`10.0.0.0/16`) with public subnets across multiple Availability Zones (AZs) for high availability.
* **Compute:** An Auto Scaling Group (ASG) launching Amazon Linux 2023 instances.
* *Bootstrapping:* User Data script installs Apache (`httpd`) and writes the specific Instance ID to `index.html` for observability.


* **Traffic Management:** An Application Load Balancer (ALB) distributing traffic and performing health checks.
* **The "Gun" (Chaos Engine):** **AWS FIS** configured to target instances tagged with `ChaosTarget=true`.
* **The "Safety Brake" (Observability):** **Amazon CloudWatch** alarms monitoring CPU utilization to abort the experiment if system stress exceeded safe limits.

---

## 3. Implementation Steps

### Phase 1: Infrastructure Deployment

We utilized Terraform to provision the network, security groups, and compute resources. A critical aspect was tagging the ASG instances with a specific key-value pair (`ChaosTarget=true`) so the Chaos engine could identify valid targets without affecting other infrastructure.

### Phase 2: The Chaos Experiment Template

We defined an `aws_fis_experiment_template` resource.

* **Action:** `aws:ec2:terminate-instances`. We chose **Terminate** over Stop to force the ASG to provision a completely new replacement, testing the full recovery lifecycle.
* **Target:** Filtered dynamically by resource tags.
* **Stop Condition:** Linked to a CloudWatch Alarm to prevent total system meltdown.

### Phase 3: Observability

We deployed a custom CloudWatch Dashboard via Terraform to visualize:

1. **Healthy Host Count** (The primary metric for resilience).
2. **Target Response Time** (User experience).
3. **CPU Utilization** (System stress).

---

## 4. Challenges & Troubleshooting

During the implementation, we encountered specific permission and logic errors that required SRE-level debugging.

### Issue 1: The "Blind" Assassin (IAM Permissions)

* **Error:** The FIS experiment failed to start or failed to find targets.
* **Root Cause:** The IAM role assigned to FIS (`fis-experiment-role`) had permission to `Stop` and `Terminate` instances but lacked the permission to *find* them.
* **Resolution:** We updated the `aws_iam_role_policy` to include `ec2:DescribeInstances`. Crucially, we had to set the `Resource` to `*` because AWS does not allow "listing" permissions to be scoped to specific resources.

```hcl
# The Fix in Terraform
Action = [
  "ec2:DescribeInstances", # Added this to allow finding targets
  "ec2:TerminateInstances",
  ...
]
Resource = "*" # Changed from scoped ARN to wildcard

```

### Issue 2: The Service Role Mismatch

* **Error:** `Failed to add AmazonFISServiceRolePolicy to user. Cannot attach AWS reserved policy to an IAM user.`
* **Context:** When trying to grant the *human user* permission to start the experiment via the console, we attempted to attach the AWS-managed policy `AmazonFISServiceRolePolicy`.
* **Root Cause:** This specific policy is a **Service-Linked Role policy**. It is reserved exclusively for the AWS internal service agent (robots) and cannot be attached to human users or standard IAM roles.
* **Resolution:** We created a custom **Inline Policy** for the user (`dartmouthwrld`) that granted:
1. Control plane access (`fis:*`).
2. **`iam:PassRole`**: This was critical. It authorized the human user to "hand over" the destructive Terraform role (`fis-experiment-role`) to the FIS service to execute the attack.



### Issue 3: SSH & Observability

* **Observation:** We removed SSH access (port 22) for security, adhering to immutable infrastructure practices.
* **Challenge:** We needed a way to verify which specific instance was responding to requests without logging in.
* **Resolution:** We injected the `ec2-metadata --instance-id` command directly into the `index.html` via User Data. We then wrote a client-side script (`check.ps1`) to query the Load Balancer repeatedly and parse this HTML output.

---

## 5. "Game Day" Results

### Steady State

Before the attack, the monitoring script confirmed traffic was balanced across 3 healthy nodes.

![Insert Screenshot of Terminal Output here showing green '200 OK' messages toggling between 3 different instance IDs]

### The Injection

We initiated the `Test ASG Recovery Capabilities` experiment in the FIS Console.

* **Event:** FIS invoked the `terminate-instances` action on one node.
* **Immediate Impact:** A momentary drop in capacity. The terminal script showed a brief timeout/failover event before stabilizing on the 2 remaining nodes.

![Insert Screenshot of AWS FIS Console here showing the Experiment State changing from 'Pending' to 'Running' to 'Completed']

### The Recovery

Within minutes, the Auto Scaling Group health checks marked the terminated instance as dead. The ASG launched a new instance to satisfy the `Desired Capacity = 3`.

* **Metric Confirmation:** The CloudWatch Dashboard visualized the "Healthy Host Count" dipping from 3 to 2, flatlining for roughly 90 seconds, and then returning to 3.

![Insert Screenshot of CloudWatch Dashboard here: The 'Healthy Host Count' graph showing a V-shaped dip and recovery]

---

## 6. Conclusion

This project successfully demonstrated the transition from theoretical availability to proven resilience. By implementing Chaos Engineering principles:

1. We verified that the **IAM permissions** were correctly scoped (specifically the need for `PassRole`).
2. We proved the **Auto Scaling Group** correctly detects and replaces terminated instances.
3. We validated that the **Application Load Balancer** successfully routes traffic away from unhealthy targets, maintaining service availability during a partial outage.

**Next Steps:**

* Implement "Connection Draining" on the ASG to further minimize the latency spike during termination.
* Automate the experiment execution using AWS EventBridge to run periodically (e.g., once a week).