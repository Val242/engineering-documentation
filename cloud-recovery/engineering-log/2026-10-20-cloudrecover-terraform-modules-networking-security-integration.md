# CloudRecover — Daily Project Progress

**Date:** 20 October 2026  
**Focus:** Terraform module composition, networking, security, and infrastructure integration

## Project Context

CloudRecover is a multi-region AWS disaster recovery platform for a NestJS inventory API. The intended architecture uses Terraform to provision AWS infrastructure, PostgreSQL on Amazon RDS, GitHub Actions with AWS OIDC federation, Systems Manager for remote deployment, and monitoring through Prometheus and Grafana.

The primary environment is planned for `eu-west-1` (Ireland), with a recovery environment in `eu-west-2` (London).

## Work Completed Today

### 1. Terraform root configuration and module composition

Continued integrating reusable Terraform modules through the regional root configuration.

The root configuration is responsible for composing the infrastructure modules and passing the appropriate outputs from one module into another.

The modules being integrated include:

- Networking — VPC, subnets, gateways, routes, and route tables.
- Security — security groups for the application, load balancer, monitoring services, and database.
- Application server — EC2 instance for the NestJS API.
- Load balancer — Application Load Balancer and target registration.
- Database — Amazon RDS PostgreSQL.
- Monitoring — Prometheus and Grafana infrastructure.

The objective is to keep resource implementation inside reusable modules while the root configuration controls how those modules work together.

### 2. Networking module configuration

Worked on the networking module's inputs, outputs, and root-level configuration.

The network design separates subnets by purpose:

- Public subnets for internet-facing infrastructure.
- Application subnets for the API server.
- Monitoring subnets for monitoring infrastructure.
- Database subnets for RDS.

The design distributes subnets across two Availability Zones and separates public, private application/monitoring, and database routing.

The primary VPC uses `172.40.0.0/16`, with separate `/24` subnet CIDRs for the public, application, monitoring, and database tiers.

The recovery environment is intended to use a separate, non-overlapping CIDR range.

### 3. Variables and environment configuration

Continued organizing Terraform configuration so that environment-specific values are supplied through root variables and `terraform.tfvars`, rather than being embedded throughout the reusable modules.

Configuration includes:

- AWS Region and Availability Zones.
- VPC and subnet CIDRs.
- Resource names and tags.
- EC2 instance settings.
- AMI selection.
- Database configuration.
- Administrative IP restrictions.

The primary and recovery configurations must supply values appropriate to their respective Regions and infrastructure.

### 4. Module outputs and references

Worked on exposing networking resource identifiers through module outputs so that other modules can consume them.

Examples include:

- VPC ID.
- Public and private subnet IDs.
- NAT Gateway IDs.
- Route table IDs.
- Security group IDs.
- Application EC2 instance ID.
- RDS endpoint information.

The root configuration uses these outputs to connect resources without directly referencing resources declared inside another module.

### 5. Debugging cross-module integration

Worked through Terraform errors caused by incorrect references and mismatched module interfaces.

**Issue: Incorrect subnet output name**

The root configuration referenced `module.vpc.application_subnet_az1_id`, but the networking module exposed `app_subnet_az1_id`.

The reference was corrected to match the actual output name.

**Lesson:** Terraform output names are an interface contract. A root configuration must reference the exact output exposed by the child module.

**Issue: Database module referenced an undeclared security group**

The database module attempted to reference `aws_security_group.rds.id`, even though the database module did not declare that security group.

The intended design is for the security module to own the RDS security group and expose its ID. The database module then receives that ID through a variable such as `rds_security_group_id`.

**Lesson:** Keep resource ownership clear. A module should receive an external resource's identifier through its inputs rather than reach into another module's internal resources.

**Issue: Load balancer expected a list of instance IDs**

The load balancer module's `instance_ids` input expects `list(string)`, while the application module exposes a single instance ID as a string.

For a single application instance, the root configuration must wrap that value in a one-element list:

`[module.app_server.app_server_instance_id]`

This aligns the caller's value with the module's declared type.

**Lesson:** Terraform distinguishes strings from lists of strings, even when the list contains only one item.

**Issue: Module source path**

The module source path must be relative to the directory containing the calling Terraform configuration. The correct path depends on the repository's actual directory layout.

### 6. Infrastructure design decisions reinforced

- Keep the networking and security modules separate.
- Let the root configuration compose the modules.
- Pass security group IDs into the modules that need them.
- Use module outputs instead of accessing another module's internal resources.
- Use separate Terraform root configurations and state for the primary and recovery Regions.
- Keep regional resource identifiers and configuration specific to each Region.

## Current State

Terraform configuration work has progressed to cross-module integration and debugging. The errors encountered have helped identify mismatches between declared variables, exposed outputs, and values passed between modules.

**Important:** Successful configuration evaluation or progress through `terraform plan` does not mean that infrastructure has been deployed successfully. An apply and subsequent AWS validation are separate steps.

## Remaining Tasks

- [ ] Resolve the remaining load balancer `instance_ids` type mismatch.
- [ ] Review every root-to-module input against the corresponding variable declaration.
- [ ] Verify all module output names against their definitions.
- [ ] Check resource references and Terraform resource labels.
- [ ] Verify module source paths from each regional root directory.
- [ ] Run `terraform fmt -recursive`.
- [ ] Run `terraform validate` in each regional root configuration.
- [ ] Run and review `terraform plan` for the primary environment.
- [ ] Resolve all remaining plan errors and investigate unexpected changes.
- [ ] Review resource replacement and destruction actions before applying.
- [ ] Validate the primary infrastructure after deployment.
- [ ] Repeat the configuration and validation process for the recovery environment.

## End-of-Day Summary

Today's focus was not simply writing individual Terraform resources. It was connecting independently defined modules through a consistent root configuration.

The main progress was improving the module interfaces and diagnosing integration errors involving output names, resource ownership, source paths, and collection types.

The next step is to finish validating the module interfaces, obtain a clean and understood plan, and only then proceed with infrastructure deployment.
