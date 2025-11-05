# Chapter 3 - Guardrails (Expose, Verify, Lock Down)

## 3.1 Overview

In cloud security, guardrails are predefined, automated controls (e.g., security groups, IAM boundaries, routing rules) that prevent unsafe configurations and enforce desired ones. Guardrails do not block work; they shape it safely by default and make deviations explicit and reviewable.

This chapter demonstrates least-privilege network access for a simple service. A new Ubuntu t2.medium instance is launched with Grafana running on TCP/3000 via user_data. An inbound rule is first opened to the world for quick verification, then tightened to a single /32 address. A URL is emitted for testing, and an optional no-ingress approach (SSM port-forwarding) is outlined for later use.

## 3.1 CloudShell Setup (same pattern as Chapter 2)

**What this does**: Installs Terraform to /tmp, uses CloudShell role creds, stores TF state/plugins in /tmp, and prepares the ch3 working directory.

```bash
export AWS_REGION=${AWS_REGION:-us-east-1}

unset AWS_PROFILE AWS_SDK_LOAD_CONFIG AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN

TF_VERSION="1.9.5"
ARCH=$(uname -m); case "$ARCH" in x86_64) TF_ARCH="amd64" ;; aarch64) TF_ARCH="arm64" ;; *) echo "Unsupported arch: $ARCH"; exit 1 ;; esac
mkdir -p /tmp/bin /tmp/arculus/ch3
curl -fsSLo /tmp/terraform.zip "https://releases.hashicorp.com/terraform/${TF_VERSION}/terraform_${TF_VERSION}_linux_${TF_ARCH}.zip"
unzip -o /tmp/terraform.zip -d /tmp/bin >/dev/null
export PATH="/tmp/bin:$PATH"
terraform -version

export TF_DATA_DIR=/tmp/.tfdata
export TF_PLUGIN_CACHE_DIR=/tmp/.tfplugins
mkdir -p "$TF_PLUGIN_CACHE_DIR"

# Work directory
cd /tmp/arculus/ch3
```

## 3.2 Write Main.tf (Grafana on 3000 + guardrail SG)

This creates a tiny VPC, public subnet, IGW + route, a unique SG that allows only app_port from allow_cidr, then boots Ubuntu and installs Grafana via user_data. An HTTP URL is emitted.

```bash
# From your Chapter 3 work dir
cd /tmp/arculus/ch3

# Overwrite main.tf with valid multi-line blocks
cat > main.tf <<'HCL'
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# -------- Variables (multi-line blocks) --------
variable "region" {
  type    = string
  default = "us-east-1"
}

variable "project" {
  type    = string
  default = "arculus-ch3"
}

variable "az" {
  type    = string
  default = "us-east-1a"  # change at apply time if needed
}

variable "instance_type" {
  type    = string
  default = "t2.medium"
}

variable "app_port" {
  type    = number
  default = 3000
}

# Start open for verification; tighten to /32 later
variable "allow_cidr" {
  type    = string
  default = "0.0.0.0/0"
}

# -------- Provider --------
provider "aws" {
  region = var.region
}

# -------- Minimal Networking --------
resource "aws_vpc" "vpc" {
  cidr_block           = "10.1.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = { Name = "${var.project}-vpc" }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.vpc.id
  tags   = { Name = "${var.project}-igw" }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.vpc.id
  cidr_block              = "10.1.1.0/24"
  map_public_ip_on_launch = true
  availability_zone       = var.az
  tags = { Name = "${var.project}-subnet" }
}

resource "aws_route_table" "rt" {
  vpc_id = aws_vpc.vpc.id
  tags   = { Name = "${var.project}-rt" }
}

resource "aws_route" "default" {
  route_table_id         = aws_route_table.rt.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.igw.id
}

resource "aws_route_table_association" "assoc" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.rt.id
}

# -------- Security Group: app port ingress (variable), all egress --------
resource "aws_security_group" "app" {
  name_prefix = "sample_terra-guardrails-"
  description = "Allow app port; all egress"
  vpc_id      = aws_vpc.vpc.id

  ingress {
    from_port   = var.app_port
    to_port     = var.app_port
    protocol    = "tcp"
    cidr_blocks = [var.allow_cidr]
    description = "App port"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "sample_terra_guardrails" }
}

# -------- Ubuntu 22.04 LTS AMI (Canonical 099720109477) --------
data "aws_ami" "ubuntu_jammy" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
  filter {
    name   = "architecture"
    values = ["x86_64"]
  }
}

# -------- EC2: Grafana via user_data on port 3000 --------
locals {
  user_data = <<-BASH
    #!/bin/bash
    set -euxo pipefail
    apt-get update -y
    apt-get install -y apt-transport-https software-properties-common wget
    wget -q -O - https://packages.grafana.com/gpg.key | apt-key add -
    add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
    apt-get update -y
    apt-get install -y grafana
    systemctl enable grafana-server
    systemctl start grafana-server
  BASH
}

resource "aws_instance" "vm" {
  ami                         = data.aws_ami.ubuntu_jammy.id
  instance_type               = var.instance_type
  subnet_id                   = aws_subnet.public.id
  vpc_security_group_ids      = [aws_security_group.app.id]
  associate_public_ip_address = true
  user_data                   = local.user_data

  tags = {
    Name    = "${var.project}-vm"
    Project = var.project
    Role    = "grafana"
  }
}

# -------- Outputs --------
output "public_ip" {
  value = aws_instance.vm.public_ip
}

output "url" {
  value = "http://${aws_instance.vm.public_ip}:${var.app_port}"
}

output "sg_id" {
  value = aws_security_group.app.id
}

output "az_used" {
  value = var.az
}
HCL

# Re-init and apply
terraform init -reconfigure
terraform fmt
terraform validate
terraform apply -auto-approve -var="region=${AWS_REGION}" -var="az=us-east-1a"
```

## 3.3 Init & Apply
```bash
terraform init -reconfigure
terraform fmt
terraform validate
terraform apply -auto-approve -var="region=${AWS_REGION}" -var="az=us-east-1a"
```
 * If the AZ rejects t2.medium, re-run with a supported one:

```bash
terraform apply -auto-approve -var="region=${AWS_REGION}" -var="az=us-east-1b"
```

## 3.4 Verifying the Service
```bash
terraform output # Open the printed `url` in the browser, e.g., http://PUBLIC_IP:3000
```

* Once you've entered the link based on the output, you will arrive in the Grafana UI.
* Here you can enter the default "Username: Admin" and "Password: Admin". Then, you can reset your password and enter the portal.

* Lock it down to a single /32
Get the current public IP and re-apply with a /32:
```bash
MYIP="$(curl -s https://checkip.amazonaws.com)/32"
echo "$MYIP"   # sanity check

terraform apply -auto-approve \
  -var="region=${AWS_REGION}" \
  -var="az=$(terraform output -raw az_used)" \
  -var="allow_cidr=${MYIP}"
```

## 3.5 Cleanup
```bash
terraform destroy -auto-approve \
  -var="region=${AWS_REGION}" \
  -var="az=$(terraform output -raw az_used 2>/dev/null || echo us-east-1a)"
```

## As a Result:

This chapter demonstrated network guardrails as policy-as-code: a minimal Grafana service was deployed, verified via an open ingress rule, and then constrained to a single /32, reducing exposure while preserving functionality. Controls were codified in Terraform (security group, routing, outputs), showing a repeatable pattern to expose, verify, and tighten access. The chapter also introduced a no-ingress direction (SSM port-forwarding) for stricter Zero-Trust edge deployments.
