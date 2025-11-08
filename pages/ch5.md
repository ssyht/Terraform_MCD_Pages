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
