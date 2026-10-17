# CloudRecover — Daily Progress Report

**Date:** October 17, 2026  
**Project:** CloudRecover — Multi-Region AWS Disaster Recovery Platform  
**Focus:** Terraform Security Module and Network Access Control

## 1. Objective

Continue building the reusable Terraform security module for CloudRecover, defining security groups and access rules for the application, Application Load Balancer (ALB), Prometheus, Grafana, and Amazon RDS.

The goal was to establish controlled communication between infrastructure components while keeping the configuration reusable across the primary and recovery AWS regions.

## 2. Work Completed

### A. Designed the Security Group Structure

Defined the security groups required for the main infrastructure components:

- **Web application security group:** Controls access to the application EC2 instances.
- **ALB security group:** Controls public HTTP access and communication from the load balancer to the application.
- **Prometheus security group:** Controls access to the monitoring server and Prometheus interface.
- **Grafana security group:** Controls access to the Grafana dashboard.
- **RDS security group:** Controls access to the PostgreSQL database.

The module uses Terraform resources for security groups and individual ingress and egress rules.

### B. Configured Application Access Rules

Worked on the following application server access requirements:

- SSH access on port `22` restricted to the configured administrator IP.
- Application traffic on port `3000` from the ALB security group.
- Prometheus scraping of application metrics on port `3000`.
- Node Exporter metrics collection on port `9100`, restricted to the Prometheus security group.
- Outbound IPv4 and IPv6 traffic rules.

The intended design is to avoid exposing application and host-metrics ports directly to the public internet.

### C. Configured ALB-to-Application Communication

Defined the ALB security group with public HTTP access on port `80`.

Reviewed the traffic flow between the ALB and the application server and identified a configuration that needs correction: the NestJS application listens on port `3000`, so the ALB's outbound rule and the application's inbound rule must permit traffic on port `3000`.

The web-server inbound rule on port `80` is unnecessary unless another service is listening on that port.

### D. Designed Monitoring Access Rules

Defined the access requirements for Prometheus and Grafana:

- Prometheus interface: port `9090`.
- Grafana interface: port `3000`.
- Node Exporter: port `9100`.
- Application metrics: port `3000`.
- SSH access: port `22`, restricted to the configured administrator IP.

The intended monitoring flow is:

1. Prometheus collects application metrics from the NestJS API.
2. Prometheus collects host metrics from Node Exporter.
3. Grafana queries Prometheus on port `9090`.
4. Authorized users access Grafana's dashboard.

The Grafana public-access rule requires further restriction to an administrator IP or another trusted access path.

### E. Configured Database Access

Created the RDS security group and defined an inbound PostgreSQL rule on port `5432`, allowing traffic from the web application security group.

This follows the principle of granting database access to the application tier rather than opening PostgreSQL to the public internet.

## 3. Architecture and Terraform Concepts Reinforced

Today's work reinforced the following concepts:

- Security groups act as stateful virtual firewalls for AWS resources.
- Ingress rules define permitted inbound traffic, while egress rules define permitted outbound traffic.
- Security group references allow access to be granted based on another security group rather than a broad IP range.
- The ALB receives client traffic on port `80` and forwards requests to the application target group on port `3000`.
- Prometheus needs network access to both the application's metrics endpoint and Node Exporter.
- Grafana communicates with Prometheus to retrieve monitoring data.
- RDS PostgreSQL should accept connections only from authorized application resources.
- Reusable Terraform modules should receive required infrastructure identifiers through input variables.
- Terraform module outputs expose values to the calling configuration, while input variables allow the parent configuration to supply required values.

## 4. Outstanding Corrections

The security module is still being refined. The following items remain:

- [ ] Align the ALB egress rule with the application target port `3000`.
- [ ] Remove or justify the application server's inbound port `80` rule.
- [ ] Restrict Grafana's port `3000` access instead of allowing unrestricted public access.
- [ ] Ensure all security groups use the intended VPC ID consistently.
- [ ] Review the networking module dependency and avoid unnecessary nested module calls if the root configuration already composes networking and security.
- [ ] Confirm that all Prometheus, Grafana, application, and database rules match the intended traffic flows.
- [ ] Run `terraform fmt` and `terraform validate` from the relevant Terraform root.
- [ ] Review `terraform plan` before applying any infrastructure changes.

## 5. Next Session

Continue refining and validating the security module before proceeding with the remaining CloudRecover infrastructure modules.

Recommended sequence:

1. Finalize the security group rules.
2. Validate the networking and security module integration.
3. Review the Terraform plan.
4. Continue with the application infrastructure module.
5. Proceed to the database, load balancer, and monitoring modules as appropriate.

## 6. End-of-Day Summary

Today's focus was translating CloudRecover's architecture into Terraform security-group rules. The main components and their intended communication paths have been defined, and the review identified several configuration details that need correction before the module is ready to apply.

**Status:** Security module in progress; validation and final rule corrections remain.

The next milestone is a validated security module that enforces the intended access boundaries between the application, load balancer, monitoring services, and database.