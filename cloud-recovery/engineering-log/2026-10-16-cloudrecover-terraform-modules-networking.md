# CloudRecover — Terraform Learning Log

**Dates:** October 15–16, 2026  
**Project:** CloudRecover — Multi-Region AWS Disaster Recovery Platform  
**Focus:** Terraform modules, AWS networking, infrastructure architecture, and multi-region deployment design

## 1. Learning Objectives

Over these two days, I focused on understanding how to organize AWS infrastructure using Terraform modules and how to structure CloudRecover for multi-region disaster recovery.

My objectives were to:

- Understand how reusable Terraform modules work.
- Build a custom networking module for CloudRecover.
- Organize AWS networking resources into logical components.
- Understand Terraform variables, outputs, and module interfaces.
- Learn how root configurations call reusable modules.
- Distinguish deployment environments from AWS regions.
- Establish a clear infrastructure structure for the primary and recovery regions.

---

## 2. Day One — October 15, 2026

### 2.1 Understanding Terraform Modules

I studied how Terraform modules help organize infrastructure into reusable and maintainable components.

A Terraform module is a collection of Terraform configuration files in a directory. The root module is the configuration from which Terraform commands are executed, while child modules encapsulate reusable infrastructure definitions.

I learned that a module does not have to be defined in a single `main.tf` file. Its configuration can be divided into several files according to their responsibilities.

For CloudRecover, I decided to build a custom networking module rather than depend on a community VPC module. This gives me direct control over the infrastructure design and helps me understand the resources being provisioned.

### 2.2 Organizing the Networking Module

I learned how to organize networking resources into separate files:

- `vpc.tf` — VPC configuration.
- `subnets.tf` — subnet definitions.
- `internet-gateway.tf` — Internet Gateway configuration.
- `nat-gateway.tf` — NAT Gateways and related configuration.
- `routes.tf` — route tables and routes.
- `variables.tf` — module input variables.
- `outputs.tf` — values exposed to the calling module.

The key lesson was that Terraform loads all `.tf` files in the same directory as one module. The filenames help organize the configuration, but they do not create separate modules.

### 2.3 Designing the VPC and Subnets

I worked on the networking design for CloudRecover's primary region.

The VPC uses the CIDR block `172.40.0.0/16`, with subnets organized by infrastructure tier.

The subnet layout is:

| Tier | Availability Zone 1 | Availability Zone 2 |
|---|---|---|
| Public | `172.40.10.0/24` | `172.40.11.0/24` |
| Application | `172.40.20.0/24` | `172.40.21.0/24` |
| Monitoring | `172.40.30.0/24` | `172.40.31.0/24` |
| Database | `172.40.40.0/24` | `172.40.41.0/24` |

This design separates resources according to their responsibilities while distributing the infrastructure across two Availability Zones.

I also configured the VPC with DNS support and DNS hostnames enabled.

### 2.4 Understanding AWS Gateways

I reviewed the roles of the Internet Gateway and NAT Gateways.

**Internet Gateway (IGW)**

The Internet Gateway enables internet connectivity for resources in public subnets when the associated routing and security configuration permits it.

**NAT Gateways**

NAT Gateways provide outbound internet connectivity for resources in private subnets. For CloudRecover, I planned two NAT Gateways, each associated with a public subnet in a different Availability Zone.

Each NAT Gateway uses an Elastic IP address.

This design allows application and monitoring resources in private subnets to initiate outbound connections without requiring inbound internet connections to those resources.

### 2.5 Route Tables and Routing

I learned how route tables determine where network traffic is directed.

The networking design contains:

- One public route table using the Internet Gateway for the default route.
- Two private route tables, each using its corresponding NAT Gateway.
- One dedicated database route table without a default internet route.

The application and monitoring subnets in Availability Zone 1 use the first private route table, while their counterparts in Availability Zone 2 use the second.

Both database subnets use the dedicated database route table.

I also learned that AWS automatically creates a local route for communication within a VPC. I do not need to add that route manually.

The database subnets can communicate with application resources across subnets through the VPC's local routing, provided that the relevant security-group rules and other network controls permit the traffic.

A database subnet's route table and the RDS DB subnet group serve different purposes: the route table controls routing, while the DB subnet group identifies the subnets available for database deployment.

### 2.6 Route Table Associations

I implemented route table associations to connect each subnet to the appropriate route table.

The associations follow the intended network design:

- Both public subnets use the public route table.
- The application subnet in Availability Zone 1 uses private route table 1.
- The application subnet in Availability Zone 2 uses private route table 2.
- Monitoring subnets follow the same per-AZ private routing arrangement.
- Both database subnets use the dedicated database route table.

I learned that a route table can be associated with multiple subnets, including subnets in different Availability Zones. Route tables are VPC-level resources, while each subnet belongs to a specific Availability Zone.

---

## 3. Day Two — October 16, 2026

### 3.1 Understanding Terraform Outputs

I worked on defining outputs for the networking module.

Outputs expose selected information about resources so that the root module or another calling configuration can consume it.

For example, the networking module can expose:

- VPC ID.
- Public, application, monitoring, and database subnet IDs.
- Internet Gateway ID.
- NAT Gateway IDs.
- Elastic IP allocation IDs.
- Route table IDs.
- Route table association IDs.

I learned that each output must have a unique output name within its module. Reusing the same output name for multiple subnets or gateways causes a configuration error.

I also improved output names and descriptions to make them understandable. For example, names such as `public_subnet_az1_id` and `database_subnet_az2_id` communicate the purpose and location of each resource more clearly.

### 3.2 Understanding Terraform Variables

I reviewed the input variables required by the networking module.

Variables allow the calling configuration to provide values to a module rather than hardcoding every setting inside it.

I learned to include:

- A clear description of each variable.
- An appropriate type.
- A default value when a sensible default is appropriate.

I also learned to remove variables that are not referenced by the module.

For example, an AMI variable normally belongs in the application module rather than the networking module if no networking resource uses it.

Regional settings and Availability Zone selections must also be considered carefully. Defaults that assume `eu-west-1` should not be reused blindly when deploying to another region.

### 3.3 Understanding the Root Module and Child Modules

I clarified how a root `main.tf` calls a child module.

A module block such as `module "vpc"` identifies the child module using its `source` path. A relative source such as `./modules/network` points to the module directory relative to the root configuration.

Terraform loads all the `.tf` files in that directory, not only the file named `main.tf`.

The responsibilities are:

- **Child `main.tf` and other resource files:** define the infrastructure.
- **Child `variables.tf`:** declare the module's inputs.
- **Child `outputs.tf`:** expose selected resource attributes.
- **Root `main.tf`:** call the child module and pass its inputs.
- **Root `outputs.tf`:** expose values from the root configuration when needed.

The root can reference a child module's exported values using expressions such as `module.vpc.vpc_id`.

This helped me understand that a module's resources are defined in its configuration, while its outputs form the interface through which the calling configuration can access selected values.

### 3.4 Understanding Terraform Plan and Outputs

I investigated why `terraform plan` continued to display infrastructure-related information after I commented out several outputs.

I learned that outputs and resources have different purposes.

Commenting out an output does not remove the underlying AWS resource. The output controls whether a value is exposed through the module's interface; it does not determine whether Terraform manages the resource.

Terraform plan evaluates the desired infrastructure configuration against the current state and reports proposed infrastructure changes. It is not simply a display of the outputs currently declared.

I also learned that removing an output may break references to that output in the root module or another calling module.

This distinction helped me understand why output declarations should not be confused with resource definitions.

### 3.5 Distinguishing Deployment Environments from Regions

One of the most important concepts I clarified was the difference between deployment environments and AWS regions.

**Deployment environments** describe the purpose or lifecycle stage of an application:

- `dev` — development.
- `test` — testing and validation.
- `prod` — production.

**Regional roles** describe where infrastructure runs and how it supports disaster recovery:

- `primary` — the region that normally serves production traffic.
- `recovery` — the separate region used to restore service following a primary-region failure.

These concepts are related but not interchangeable.

For CloudRecover, the primary and recovery infrastructure will run in different AWS regions. Development, testing, and production are separate environment concerns that can be designed independently.

I do not necessarily need six fully provisioned environments. The number of environments and the regional deployment strategy should reflect the project's requirements and cost constraints.

### 3.6 Planning the Terraform Root Structure

I established the intended separation between the primary and recovery roots.

The planned structure is:

```text
CloudRecover/
├── modules/
│   ├── networking/
│   ├── security/
│   ├── application/
│   ├── database/
│   ├── load-balancer/
│   └── monitoring/
│
└── infrastructure/
    ├── primary/
    │   ├── main.tf
    │   ├── variables.tf
    │   ├── outputs.tf
    │   └── terraform.tfvars
    │
    └── recovery/
        ├── main.tf
        ├── variables.tf
        ├── outputs.tf
        └── terraform.tfvars
```

The reusable modules define infrastructure components, while the primary and recovery roots compose those components for their respective regional deployments.

The primary root will be my starting point for integrating and deploying the main infrastructure. The recovery root will remain independently manageable so that disaster-recovery resources can be provisioned or restored separately.

I also learned that Terraform initialization is performed from each root configuration. Terraform loads the referenced child modules as part of that root's configuration.

Each root can maintain a separate state, providing independent tracking and management of regional resources.

---

## 4. Current Architecture Decisions

The following decisions guide the current CloudRecover implementation:

1. Build a custom Terraform networking module.
2. Organize networking resources into logical files and reusable components.
3. Separate public, application, monitoring, and database subnets.
4. Use two Availability Zones for the planned primary-region subnet layout.
5. Use an Internet Gateway for public routing and two NAT Gateways for private-subnet outbound connectivity.
6. Keep database subnets on a dedicated route table without a default internet route.
7. Use descriptive, uniquely named outputs to expose resource IDs.
8. Remove unused module variables and define appropriate types and descriptions.
9. Reuse modules from separate primary and recovery root configurations.
10. Manage the primary and recovery regions independently, with separate state boundaries.
11. Distinguish deployment environments such as dev, test, and prod from the primary and recovery regional roles.

---

## 5. Next Steps

My next implementation session will focus on completing the remaining Terraform modules and integrating them into the primary root.

### Immediate tasks

- [ ] Finish and validate the networking module.
- [ ] Build the security module.
- [ ] Build the application module.
- [ ] Build the database module.
- [ ] Build the load-balancer module.
- [ ] Build the monitoring module as required by the architecture.
- [ ] Define consistent variables and outputs for the modules.
- [ ] Call the modules from the primary root `main.tf`.
- [ ] Pass the required resource IDs between modules through inputs and outputs.
- [ ] Run Terraform formatting and validation.
- [ ] Review the execution plan before applying infrastructure changes.
- [ ] Integrate the recovery root after the primary configuration is established.

### Validation priorities

Before considering the modules complete, I need to verify that:

- Every module has the inputs it actually requires.
- Output names are unique and referenced correctly.
- Resource dependencies are expressed correctly.
- Security groups permit only the intended traffic.
- The database is reachable from the application through the intended private network paths.
- Regional configurations use appropriate CIDRs, Availability Zones, and AWS providers.
- The recovery configuration can be managed independently of the primary configuration.

---

## 6. Key Takeaway

Over October 15 and 16, I progressed from configuring individual AWS networking resources toward understanding how Terraform modules compose into a larger infrastructure architecture.

I learned how to organize networking resources, connect subnets to route tables, expose resource attributes through outputs, define module inputs, and call child modules from a root configuration.

I also clarified the architectural distinction between deployment environments and regional roles.

My next goal is to apply these concepts consistently across the remaining CloudRecover modules, integrate them through the primary root, and then establish the recovery deployment.

The focus is not simply to make Terraform execute successfully. It is to build an infrastructure configuration that is understandable, reusable, independently manageable, and suitable for a multi-region disaster-recovery system.
