# Chapter 3 Guardrails (Expose, Verify, Lock Down)

## 3.1 Overview

In cloud security, guardrails are predefined, automated controls (e.g., security groups, IAM boundaries, routing rules) that prevent unsafe configurations and enforce desired ones. Guardrails do not block work; they shape it safely by default and make deviations explicit and reviewable.

This chapter demonstrates least-privilege network access for a simple service. A new Ubuntu t2.medium instance is launched with Grafana running on TCP/3000 via user_data. An inbound rule is first opened to the world for quick verification, then tightened to a single /32 address. A URL is emitted for testing, and an optional no-ingress approach (SSM port-forwarding) is outlined for later use.
