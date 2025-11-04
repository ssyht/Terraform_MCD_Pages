<a name = "Pg1"></a>

# **Chapter 1** - Overview & Getting Started with Terraform 

## 1.1 Purpose of the Lab
This module introduces Terraform as the Infrastructure-as-Code (IaC) backbone for the Arculus edge-security testbed. You’ll learn how to describe cloud resources declaratively (VPCs, subnets, security groups, EC2 instances, IAM, SSM, KMS, and Secrets Manager), apply guardrails (least-privilege IAM, permission boundaries, encrypted storage, SSM-only access), and package repeatable environments as modules for students and instructors.
By the end of Chapter 1, you’ll have a working local Terraform setup, a reproducible AWS scaffold, and a clear path to layer the Arculus services on top (testbed nodes, micro-segmentation, mTLS, and zero-trust administration).

## 1.2 Prerequisites
To follow this lab, you should have:

**Accounts & Access**

* An AWS account with programmatic access (IAM user/role) and an administrative sandbox or teaching account.

* MFA enabled for the console; an access key/secret (or AWS SSO) for CLI use.

* Local Tooling

* Terraform (v1.6+ recommended).

* AWS CLI (v2+), configured via aws configure or SSO.

* A code editor (VS Code recommended) with HCL extensions.

**Knowledge & Concepts (lightweight)**

* Basic AWS networking (VPC, subnets, routing).

* IAM fundamentals (roles, policies, permission boundaries).

* Why IaC matters: versioning, reviewability, repeatability, and drift detection.

**What we provide in this chapter:**

* Starter Terraform workspace & folder layout.

* A minimal yet secure AWS baseline (VPC, subnets, SSM-managed EC2, KMS CMK, logging).

## 1.3 References to guide lab work
Please use the links below to learn the related information for this lab. 

* <a href = "https://developer.hashicorp.com/terraform/tutorials">*Terraform by HashiCorp*</a> - Introductions & Tutorials
* <a href = "https://registry.terraform.io/providers/hashicorp/aws/latest">*Terraform AWS Provider*</a> - Registry
* <a href="https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html">*AWS CLI*</a> - v2
* <a href ="https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html">*AWS Systems Manager SSM*</a> - Session Manager
* <a href = "https://docs.aws.amazon.com/kms/latest/developerguide/overview.html">*AWS KMS*</a> - Key Management Service
* <a href = "https://aws.amazon.com/">*AWS (Amazon Web Services)*</a> - A cloud platform that offers a variety of services, including storage, compute, and databases. AWS allows users to build and run applications on-demand.
## 1.4 Goals/Outcomes:
By the end of this lab module, you will be able to:

(i)	Understand Confidential Computing Concepts

* Explain the fundamentals of Confidential Computing and its role in securing multi-cloud and distributed environments.
* Understand the importance of trust establishment and policy enforcement in secure applications.

(ii) Set Up and Configure the Certifier Framework

* Successfully install and configure the Certifier Framework on their system.
* Generate cryptographic keys for secure communication.
* Define, sign, and provision security policies for trusted interactions.

(iii) Develop Secure Applications Using the Certifier API

* Utilize the Certifier API to enable attestation, secure storage, and trust management in applications.
* Abstract hardware complexities to make applications portable across different Confidential Computing environments.

(iv) Deploy and Manage Trusted Services

* Set up the Certifier Service to evaluate attestations and enforce security policies.
* Manage trust relationships and ensure compliance with defined security policies.

(v) Test and Validate Secure Communication

* Run example applications to demonstrate secure client-server interactions.
* Verify policy compliance through real-time attestations and trust evaluations.
