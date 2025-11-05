# **Chapter 2** - Provisioning EC2 using CloudShell & Terraform

## 2.1 Overview

In this chapter, you’ll use AWS CloudShell (browser-based shell with AWS CLI) to run Terraform. We’ll keep the setup lightweight by installing Terraform to /tmp for this session and storing Terraform state and plugin cache in /tmp to avoid home-directory limits. 

You’ll write a minimal Terraform config that creates a VPC, public subnet, internet route, a unique egress-only security group, and an Ubuntu 22.04 t2.medium EC2 instance—then apply, verify outputs, and destroy when finished.

## 2.2 Navigating to CloudShell

* Sign into your <a href = "https://console.aws.amazon.com/">*AWS Management Console*</a>
* Make sure to select the US East (N. Virginia) region in the top-right part of your screen.

<a name = "fig2.1"></a><img src = "../img/ch.2_AWS_region.png" align = "center"/></center>


In the top search bar, type "CloudShell" and select **CloudShell** from the services list.

<a name = "fig2.1"></a><img src = "../img/ch2_CloudShell_search.png" align = "center"/></center>

You will arrive in this CloudShell terminal:

<a name = "fig2.1"></a><img src = "../img/Ch2_CloudShell_StartPage.png" align = "center"/></center>

## 2.2 Running Main.tf Script to Launch EC2 Instance


Install Terraform into /tmp for this session, reset env vars so it uses CloudShell’s role credentials (no profiles), and point Terraform’s state and plugin cache to /tmp to avoid home-directory quotas—then move into the working directory.

```bash
export AWS_REGION=${AWS_REGION:-us-east-1}

unset AWS_PROFILE AWS_SDK_LOAD_CONFIG AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN

TF_VERSION="1.9.5"
ARCH=$(uname -m); case "$ARCH" in x86_64) TF_ARCH="amd64" ;; aarch64) TF_ARCH="arm64" ;; *) echo "Unsupported arch: $ARCH"; exit 1 ;; esac
mkdir -p /tmp/bin /tmp/arculus/ch2
curl -fsSLo /tmp/terraform.zip "https://releases.hashicorp.com/terraform/${TF_VERSION}/terraform_${TF_VERSION}_linux_${TF_ARCH}.zip"
unzip -o /tmp/terraform.zip -d /tmp/bin >/dev/null
export PATH="/tmp/bin:$PATH"
terraform -version

export TF_DATA_DIR=/tmp/.tfdata
export TF_PLUGIN_CACHE_DIR=/tmp/.tfplugins
mkdir -p "$TF_PLUGIN_CACHE_DIR"

# Work directory
cd /tmp/arculus/ch2
<img width="468" height="391" alt="image" src="https://github.com/user-attachments/assets/5e8d20fb-a758-40f8-808c-f697f20cc0d5" />

```
