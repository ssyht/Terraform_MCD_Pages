# Chapter 5 - Arculus Portal Set-Up

## 5.1 Overview

In zero-trust edge security, the Arculus portal acts as the policy and visibility plane: it issues identities, enforces service-to-service trust (mTLS), and provides an operator UI/API to observe and control workloads.

This chapter walks through launching an Ubuntu instance (or AMI pre-baked by the instructor) and bootstrapping the Arculus stack via user_data/Docker Compose. Required inbound ports (e.g., 80/443 for the UI/API, 3000 for the local admin UI, 8440–8443 control channels, 3003, 179, 10250; and UDP 14550–14558/8285 if using telemetry later) are added explicitly, tested, and then tightened. You’ll claim the portal, create an organization/workspace, generate API tokens, and enroll the mission/adversary/research nodes for Chapter 4.

## 5.2 CloudShell Setup (same pattern as Chapter 4)

**What this does**: Installs Terraform to /tmp, uses CloudShell role creds, stores TF state/plugins in /tmp, and prepares the chapter 5 working directory.

```bash
export AWS_REGION=${AWS_REGION:-us-east-1}

unset AWS_PROFILE AWS_SDK_LOAD_CONFIG AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN

TF_VERSION="1.9.5"
ARCH=$(uname -m;); case "$ARCH" in x86_64) TF_ARCH="amd64" ;; aarch64) TF_ARCH="arm64" ;; *) echo "Unsupported arch: $ARCH"; exit 1;; esac
mkdir -p /tmp/bin /tmp/arculus/ch5
curl -fsSLo /tmp/terraform.zip "https://releases.hashicorp.com/terraform/${TF_VERSION}/terraform_${TF_VERSION}_linux_${TF_ARCH}.zip"
unzip -o /tmp/terraform.zip -d /tmp/bin >/dev/null
export PATH="/tmp/bin:$PATH"
terraform -version

export TF_DATA_DIR=/tmp/.tfdata
export TF_PLUGIN_CACHE_DIR=/tmp/.tfplugins
mkdir -p "$TF_PLUGIN_CACHE_DIR"

cd /tmp/arculus/ch5
```

## 5.3 Write Main.tf 

```bash
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    aws  = { source = "hashicorp/aws",  version = "~> 5.0" }
    http = { source = "hashicorp/http", version = "~> 3.4" }
  }
}

provider "aws" {
  region = var.region
}

# Caller IP for tight /32 allow-list
data "http" "me" {
  url = "https://checkip.amazonaws.com/"
}

locals {
  caller_ip_cidr = var.allow_cidr != "" ? var.allow_cidr : "${trimspace(data.http.me.response_body)}/32"
}

# Default VPC + one subnet
data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

# Ubuntu 22.04 LTS (Jammy)
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

# -------- Security Group (NO SSH, NO IAM) --------
resource "aws_security_group" "portal_sg" {
  name        = "${var.name}-sg"
  description = "Arculus portal SG"
  vpc_id      = data.aws_vpc.default.id

  # TCP ports
  dynamic "ingress" {
    for_each = toset([80, 443, 3000, 3003, 8440, 8441, 8442, 8443, 179, 10250])
    content {
      description = "Arculus TCP ${ingress.value}"
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = [local.caller_ip_cidr]
    }
  }

  # UDP ranges/ports
  ingress {
    description = "Arculus UDP 14550-14558"
    from_port   = 14550
    to_port     = 14558
    protocol    = "udp"
    cidr_blocks = [local.caller_ip_cidr]
  }
  ingress {
    description = "Arculus UDP 8285"
    from_port   = 8285
    to_port     = 8285
    protocol    = "udp"
    cidr_blocks = [local.caller_ip_cidr]
  }

  # Egress (apt/git/docker pulls)
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = var.common_tags
}

# -------- user_data script (installer is fully non-interactive) --------
locals {
  user_data = <<-EOT
    #!/bin/bash
    set -eux

    DB_PASSWORD="${var.db_password}"

    export DEBIAN_FRONTEND=noninteractive
    apt-get update -y
    apt-get install -y git curl npm docker.io expect
    systemctl enable --now docker || true

    cd /opt
    [ -d arculus-sw ] || git clone https://github.com/arculus-zt/arculus-sw.git
    cd arculus-sw
    chmod +x setup_controller.sh || true
    chmod +x arculus-setup.sh || true

    cat >/opt/arculus-sw/noninteractive_setup.expect <<'EOF'
    #!/usr/bin/expect -f
    set timeout -1
    set dbpwd [lindex $argv 0]
    set genkey ""

    spawn bash -lc "./arculus-setup.sh"

    expect {
      -re "Enter database password:*" {
        send -- "$dbpwd\r"
        exp_continue
      }
      -re "(?i)(generated|encryption).*[ :\\-]([A-Fa-f0-9]{64})" {
        set genkey $expect_out(2,string)
        exp_continue
      }
      -re "Enter a 256-bit encryption key:*" {
        send -- "$genkey\r"
        exp_continue
      }
      -re "Enter CHN URL.*:"        { send -- "\r"; exp_continue }
      -re "Enter CHN API key.*:"    { send -- "\r"; exp_continue }
      -re "Enter CHN deploy key.*:" { send -- "\r"; exp_continue }
      eof
    }
    EOF

    chmod +x /opt/arculus-sw/noninteractive_setup.expect
    /opt/arculus-sw/noninteractive_setup.expect "$DB_PASSWORD"

    # Persist a copy of the key if installer didn't
    FOUND_KEY="$(grep -RhoE '[A-Fa-f0-9]{64}' /opt/arculus-sw 2>/dev/null | head -n1 || true)"
    if [ -n "$FOUND_KEY" ]; then
      mkdir -p /opt/arculus
      echo -n "$FOUND_KEY" > /opt/arculus/ENCRYPTION_SECRET.txt
      chmod 600 /opt/arculus/ENCRYPTION_SECRET.txt
    fi

    # PM2 autostart if present
    if command -v pm2 >/dev/null 2>&1; then
      su - ubuntu -c "pm2 save" || true
      pm2 startup systemd -u ubuntu --hp /home/ubuntu || true
      systemctl enable pm2-ubuntu || true
      systemctl start pm2-ubuntu || true
    fi
  EOT
}

# -------- EC2 Instance (no key pair, no instance profile) --------
resource "aws_instance" "portal" {
  ami                         = data.aws_ami.ubuntu.id
  instance_type               = var.instance_type
  subnet_id                   = element(data.aws_subnets.default.ids, 0)
  vpc_security_group_ids      = [aws_security_group.portal_sg.id]
  associate_public_ip_address = true

  user_data = local.user_data

  tags = merge(var.common_tags, { Name = var.name })
}
```

5.4 Variable.tf Script
```bash
variable "name" {
  type    = string
  default = "arculus-portal"
}

variable "region" {
  type    = string
  default = "us-east-1"
}

variable "instance_type" {
  type    = string
  default = "t2.large"
}

variable "portal_port" {
  type    = number
  default = 3000
}

# If empty, we auto-scope to caller /32
variable "allow_cidr" {
  type    = string
  default = ""
}

# DB password fed to the installer
variable "db_password" {
  type    = string
  default = "vimanceri"
}

variable "common_tags" {
  type = map(string)
  default = {
    Project = "MizzouCloudDevOps"
    Module  = "Terraform Chapter 4"
    Owner   = "Student"
  }
}
```
