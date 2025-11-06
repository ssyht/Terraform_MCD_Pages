# Terraform‑for‑Arculus: Zero‑Trust Edge Security Testbed

> **Status**: For the Mizzou Cloud DevOps Modules.
> **Audience**: Instructors & Students.
> **Goal**: A copy‑paste‑friendly lab that teaches Terraform fundamentals while provisioning an Arculus‑style zero‑trust testbed on AWS.

---

## Table of Contents

1. [What You’ll Build](#what-youll-build)
2. [Learning Objectives](#learning-objectives)
3. [Prerequisites](#prerequisites)
4. [Quick Start (5‑Minute Smoke Test)](#quick-start-5-minute-smoke-test)
5. [Repository Layout](#repository-layout)
6. [Chapter Overviews](#chapter-overviews)
7. [Chapter 1 – Overview & Getting Started](#chapter-1--overview--getting-started)
8. [Chapter 2 – Provision 1st EC2 via Terraform (SSM‑only, no SSH)](#chapter-2--provision-1st-ec2-via-terraform-ssm-only-no-ssh)
9. [Chapter 3 – “Hello, Guardrails” (Grafana on TCP/3000 + SG hardening)](#chapter-3--hello-guardrails-grafana-on-tcp3000--sg-hardening)
10. [Chapter 4 – Bootstrap the Arculus Testbed from an **Ubuntu** AMI](#chapter-4--bootstrap-the-arculus-testbed-from-an-ubuntu-ami)
11. [Chapter 5 – To Be Determined (capstone add‑on)](#chapter-5--to-be-determined-capstone-add-on)
12. [Security Model & Zero‑Trust Thread](#security-model--zero-trust-thread)
13. [Cost & Cleanup](#cost--cleanup)
14. [Troubleshooting](#troubleshooting)
15. [FAQ](#faq)
16. [Contributing](#contributing)
17. [License](#license)
18. [Acknowledgments](#acknowledgments)

---

## What You’ll Build

You’ll use **Terraform** to stand up an **Arculus‑inspired zero‑trust testbed** on AWS. The lab progresses from a single, tightly locked down EC2 instance to a small **three‑node topology** (mission / adversary / research) with **SSM‑only administration**, **least‑privilege IAM**, **explicit Security Groups**, and **optional private access (VPC endpoints)**. You’ll also run a simple app (**Grafana**) and practice **opening a port safely, then re‑hardening** the surface.

**High‑level diagram**

```
+-------------------- AWS Account --------------------+
|  VPC (10.0.0.0/16)                                  |
|   +---------+    +----------+    +----------+       |
|   | SubnetA |    | SubnetB  |    | Endpoints|       |
|   +----+----+    +----+-----+    +----+-----+       |
|        |              |              |               |
|   [EC2: student-1]    |        [SSM, S3, Logs]      |
|        |              |              |               |
|   (SSM Session Mgr, no SSH; Grafana in Ch.3)        |
|                                                      |
|   Mission  |  Adversary  |  Research  (Ch.4)         |
|   (EC2s launched from pre-baked Ubuntu AMI)          |
+------------------------------------------------------+
```

---

## Learning Objectives

* **Terraform Fundamentals**: providers, state, variables, outputs, modules.
* **AWS Guardrails by Default**: SSM‑only admin (no inbound 22), explicit SGs, IAM permission boundaries, budget alarms.
* **Secure Exposure**: open a single port for a student service (Grafana on 3000), verify, then **tighten** exposure.
* **Composable Testbed**: launch a **three‑node** Ubuntu‑based topology from an instructor‑baked AMI with lightweight `user_data` bootstrap.
* **Operational Hygiene**: tags, cost estimation, teardown, and reproducibility.

---

## Prerequisites

* **Account**: AWS account with credits or classroom sandbox.
* **Access**: **AWS CloudShell** enabled in the same region you plan to deploy.
* **Permissions**: Ability to create VPC, Subnets, Security Groups, EC2, IAM roles/policies, Budgets/CloudWatch Alarms, SSM, and VPC endpoints (optional).
* **Local Tools** (optional): Terraform ≥ 1.6 if not using CloudShell’s preinstalled version.
* **Instructor‑baked AMI** (Chapter 4): Ubuntu 22.04 LTS with pre‑placed folders/scripts for mission/adversary/research services. (You will be given an **AMI ID**.)

> **Note**: We operate **exclusively through AWS Systems Manager (SSM)** for administration. **No SSH** inbound rules will be used.

---

## Quick Start (5‑Minute Smoke Test)

From **AWS CloudShell**:

```bash
# 2) Initialize & plan
terraform init
terraform plan -out tf.plan \
  -var region="us-east-2" \
  -var student_tag="sanjit-01"

# 3) Apply (type 'yes' when prompted)
terraform apply tf.plan

# 4) Verify SSM connectivity
aws ssm start-session --target $(terraform output -raw instance_id)

# 5) Destroy (if only smoke-testing)
terraform destroy -auto-approve
```

If those commands work, you’re ready for Chapter 2.

---

## Repository Layout

```
terraform-arculus-lab/
├─ README.md                         # This file
├─ modules/                          # Reusable Terraform modules
│  ├─ vpc-minimal/
│  ├─ ec2-ssm-instance/
│  ├─ security-groups/
│  ├─ budget-guardrails/
│  └─ vpc-endpoints-optional/
├─ chapters/
│  ├─ 01-overview-getting-started/
│  │  └─ README.md
│  ├─ 02-single-ec2-ssm-only/
│  │  ├─ main.tf
│  │  ├─ variables.tf
│  │  ├─ outputs.tf
│  │  └─ user_data.sh
│  ├─ 03-hello-guardrails-grafana/
│  │  ├─ main.tf
│  │  ├─ variables.tf
│  │  ├─ outputs.tf
│  │  ├─ user_data_grafana.sh
│  │  └─ examples/
│  └─ 04-arculus-testbed-ubuntu-ami/
│     ├─ main.tf
│     ├─ variables.tf
│     ├─ outputs.tf
│     ├─ user_data_mission.sh
│     ├─ user_data_adversary.sh
│     └─ user_data_research.sh
└─ .github/workflows/ci.yml          # (Optional) lints/validate terraform
```

---

## Chapter Overviews

* **Chapter 1 – Overview & Getting Started**: Context, goals, guardrails, and how to use CloudShell effectively.
* **Chapter 2 – Single EC2 via Terraform (SSM‑only)**: Minimal VPC/SG/IAM, one instance, **no inbound SSH**; verify with SSM.
* **Chapter 3 – Hello, Guardrails**: Reuse the same EC2; open **TCP/3000** for **Grafana**, test via browser, then **restrict** exposure to your IP (`/32`) or remove rule and use SSM port‑forwarding.
* **Chapter 4 – Arculus Testbed from **Ubuntu** AMI**: Provision **three nodes** (mission/adversary/research) from a pre‑baked **Ubuntu** AMI. Enforce SSM‑only, explicit SGs, VPC endpoints (optional), and minimal `user_data` to start each role’s service.
* **Chapter 5 – TBD**: Optional advanced topic (e.g., S3/KMS integration, mTLS between nodes, private‑only access, or Batch workload).

---

## Chapter 1 – Overview & Getting Started

**Theme**: Zero‑trust by default, Infrastructure‑as‑Code for repeatability, and student‑friendly guardrails.

### 1.1 Why Terraform for Arculus?

* Declaratively encode **security controls** (least‑privilege IAM, SSM‑only, explicit SGs).
* **Reproducible** environments across cohorts.
* **Destroy‑ability**: easy teardown for cost control.

### 1.2 CloudShell Basics (we’ll use this everywhere)

```bash
# Create a working folder
mkdir -p ~/labs/terraform-arculus && cd ~/labs/terraform-arculus

# Check Terraform
terraform -version

# Configure default region (choose the one your instructor uses)
aws configure set region us-east-2
```

### 1.3 Guardrails We Enforce Throughout

* **No inbound SSH** (use **SSM Session Manager**).
* **Tag everything** (e.g., `project = arculus-lab`, `owner = <your-name>`).
* **Security Groups**: additive, explicit, minimal.
* **Optional private access** via **VPC endpoints** for SSM/S3/Logs.
* **Budgets/Alarms** when possible.

---

## Chapter 2 – Provision 1st EC2 via Terraform (SSM‑only, no SSH)

**Outcome**: You will launch a minimal VPC + one EC2 with **SSM** enabled; you’ll prove access with an SSM session.

### 2.1 Files (sketch)

`main.tf` (excerpt):

```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}

provider "aws" {
  region = var.region
}

module "vpc" {
  source = "../../modules/vpc-minimal"
  name   = "arculus-lab"
}

module "ec2" {
  source               = "../../modules/ec2-ssm-instance"
  name                 = "student-1"
  subnet_id            = module.vpc.public_subnet_id
  instance_type        = var.instance_type
  ami_id               = var.ami_id # Amazon Linux 2 or Ubuntu; SSM agent must be present
  iam_instance_profile = module.ec2_profile.name
  tags = {
    project = "arculus-lab"
    owner   = var.student_tag
  }
}
```

`variables.tf` (excerpt):

```hcl
variable "region" { type = string }
variable "student_tag" { type = string }
variable "instance_type" { type = string  default = "t3.micro" }
variable "ami_id" { type = string  description = "Base AMI with SSM agent" }
```

`outputs.tf` (excerpt):

```hcl
output "instance_id" { value = module.ec2.instance_id }
output "public_ip"   { value = module.ec2.public_ip }
```

### 2.2 Run

```bash
terraform init
terraform apply \
  -var region="us-east-2" \
  -var student_tag="<your-name>" \
  -var ami_id="ami-xxxxxxxxxxxx"

# Verify SSM
aws ssm start-session --target $(terraform output -raw instance_id)
```

> **Check‑off**: You can open an SSM shell without any inbound SSH rules.

---

## Chapter 3 – “Hello, Guardrails” (Grafana on TCP/3000 + SG hardening)

**Outcome**: Safely expose a simple service (**Grafana**) on **port 3000**, visit it in a browser, then **tighten** exposure.

### 3.1 Minimal Idea

1. Reuse the **EC2** from Chapter 2.
2. Install **Grafana** via `user_data_grafana.sh` or an SSM document.
3. Add a **Security Group rule**: allow **TCP/3000** from `0.0.0.0/0` temporarily.
4. Visit: `http://<public_ip>:3000` → confirm it works.
5. **Harden**: change SG rule to **your /32** only, or remove the open rule and use **SSM port‑forwarding**.

### 3.2 Example Snippets

`user_data_grafana.sh` (excerpt):

```bash
#!/usr/bin/env bash
set -euxo pipefail

# Example for Amazon Linux 2023; adjust for Ubuntu if needed
sudo dnf install -y grafana || sudo yum install -y grafana || true
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

`main.tf` (SG excerpt):

```hcl
resource "aws_security_group" "app_sg" {
  name        = "app-3000-sg"
  description = "Allow Grafana"
  vpc_id      = module.vpc.vpc_id

  # Step 1: temporarily open to all (demo only)
  ingress {
    from_port   = 3000
    to_port     = 3000
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_network_interface_sg_attachment" "attach" {
  security_group_id    = aws_security_group.app_sg.id
  network_interface_id = module.ec2.primary_eni_id
}
```

### 3.3 Harden (Very Important)

Change the SG ingress to your **current public IP** only (replace `X.Y.Z.W/32`):

```hcl
  ingress {
    from_port   = 3000
    to_port     = 3000
    protocol    = "tcp"
    cidr_blocks = ["X.Y.Z.W/32"]
  }
```

Apply:

```bash
terraform apply -auto-approve
```

**Alternative**: Remove the open rule entirely and use **SSM port forwarding**:

```bash
aws ssm start-session \
  --target $(terraform output -raw instance_id) \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["3000"],"localPortNumber":["3000"]}'
# Open http://localhost:3000 locally while the session runs.
```

> **Check‑off**: You demonstrated opening a port, validating the service, and **re‑hardening** to least exposure.

---

## Chapter 4 – Bootstrap the Arculus Testbed from an **Ubuntu** AMI

**Outcome**: Launch **three EC2 nodes** — **mission**, **adversary**, **research** — from an **instructor‑provided Ubuntu AMI**, each with role‑specific `user_data` and **SSM‑only** access.

### 4.1 Why a Pre‑Baked Ubuntu AMI?

* Ensures **consistent folder structure and scripts** across all students.
* Faster launches; fewer moving parts during lab time.
* Instructor can ship updates once (new AMI), students simply pass the AMI ID.

## Cost & Cleanup

**Keep costs tiny** by using `t3.micro`/`t3.small` and destroying labs when done.

```bash
# From any chapter folder you deployed:
terraform destroy -auto-approve
```

> **Does `terraform destroy` terminate the VMs?** Yes — it deletes Terraform‑managed resources, including the EC2 instances those plans created.

**Pro tip**: Add budgets/alarms and the `owner` tag so untagged resources stand out.

---

## Troubleshooting

* **SSM session fails**: Ensure the instance profile has `AmazonSSMManagedInstanceCore`, SSM agent is installed/running, and outbound egress is allowed to SSM endpoints.
* **Grafana not loading**: Confirm SG ingress on 3000 (temporarily), confirm service is running (`systemctl status grafana-server`).
* **Apply fails on endpoints**: Your region may require specific endpoint services; check names and route tables.
* **AMI not found**: You’re in the wrong region or lack permission; verify AMI ID visibility.

---

## FAQ

**Q: Can I switch the instance type later?**
A: Update `instance_type` and run `terraform apply` — note it may recreate instances (downtime).

**Q: Can I avoid any public ingress?**
A: Yes. Use SSM exclusively and add VPC endpoints; remove public IPs.

**Q: Why Ubuntu in Chapter 4?**
A: The instructor‑baked **Ubuntu** AMI guarantees identical paths/scripts for the three roles.

**Q: Where are the logs?**
A: Prefer shipping to CloudWatch Logs (agent) or S3. Add as a Chapter 5 exercise.

---

## Contributing

* Open PRs against feature branches (`feat/ch4-ubuntu-testbed`, etc.).
* Run `terraform fmt` and `terraform validate` before submitting.
* Keep modules small, focused, and documented with examples.

---

## License

Mizzou Cloud DevOps (educational use encouraged). See `LICENSE`.

---

## Acknowledgments

* By Sanjit Subhash - Cybersecurity Research Assistant from CERI (Cyber Education, Research and Infrastructure) from University of Missouri Columbia
* Arculus research collaborators - Roshan Neupane, Harshavardhan Chintapatla

