# Chapter 5 - Arculus Portal Set-Up

## 5.1 Overview

In cloud security, guardrails are predefined, automated controls (e.g., security groups, IAM boundaries, routing rules) that prevent unsafe configurations and enforce desired ones. Guardrails do not block work; they shape it safely by default and make deviations explicit and reviewable.

This chapter demonstrates least-privilege network access for a simple service. A new Ubuntu t2.medium instance is launched with Grafana running on TCP/3000 via user_data. An inbound rule is first opened to the world for quick verification, then tightened to a single /32 address. A URL is emitted for testing, and an optional no-ingress approach (SSM port-forwarding) is outlined for later use.

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
