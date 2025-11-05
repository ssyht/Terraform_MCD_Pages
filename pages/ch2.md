# **Chapter 2** - Provisioning EC2 using CloudShell & Terraform

## 2.1 Overview

In this chapter, you’ll use AWS CloudShell (browser-based shell with AWS CLI) to run Terraform. We’ll keep the setup lightweight by installing Terraform to /tmp for this session and storing Terraform state and plugin cache in /tmp to avoid home-directory limits. 

You’ll write a minimal Terraform config that creates a VPC, public subnet, internet route, a unique egress-only security group, and an Ubuntu 22.04 t2.medium EC2 instance—then apply, verify outputs, and destroy when finished.

## 2.2 Launch EC2 Instance

* Sign into your <a href = "https://console.aws.amazon.com/">*AWS Management Console*</a>
* Make sure to select the US East (N. Virginia) region in the top-right part of your screen.

<a name = "fig2.1"></a><img src = "../img/ch.2_AWS_region.png" align = "center"/></center>


In the top search bar, type "CloudShell" and select **CloudShell** from the services list.

<a name = "fig2.1"></a><img src = "../img/ch2_CloudShell_search.png" align = "center"/></center>


