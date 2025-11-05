# **Chapter 2** - Provisioning EC2 using CloudShell & Terraform

## 2.1 Overview

In this chapter, you’ll use AWS CloudShell (a browser-based shell with AWS CLI pre-configured) to run Terraform. Keep everything temporary and lightweight by installing Terraform into /tmp for the current session only. Store Terraform state and plugin cache in /tmp to avoid CloudShell home-directory quotas. Write a minimal Terraform config that creates a VPC, public subnet, route to the internet, a unique egress-only security group (sample_terra-*), and a t2.medium Ubuntu 22.04 EC2 instance. Launch infrastructure with terraform apply, verify outputs, and destroy when done.
