# General DevOps Learning Journal — 20 October 2026

## Today's Focus

Today's hands-on work reinforced Terraform module composition, infrastructure configuration, cross-module debugging, and Kubernetes cluster management using `kOps`.

The common theme was learning to operate infrastructure by understanding how its components fit together, rather than treating individual commands or resources as isolated tasks.

## 1. Terraform and Infrastructure as Code

### Reusable modules and root configurations

Worked on composing reusable Terraform modules from a regional root configuration.

Key concepts reinforced:

- A **module** groups related infrastructure resources and exposes a defined interface.
- **Input variables** let a module receive configuration from its caller.
- **Outputs** expose selected values for use by the caller or other modules.
- A **root module** composes child modules and wires their inputs and outputs together.
- Module source paths are resolved relative to the calling configuration's directory.
- `terraform.tfvars` supplies environment-specific values for declared variables.

A module's implementation should remain encapsulated. The root configuration should use the module's declared inputs and outputs instead of relying on its internal resource names.

### Type checking and debugging

Worked through examples of Terraform integration errors:

- Referencing an output name that did not exist.
- Referencing a security group that was not declared in the current module.
- Passing a string to a variable that expects a list of strings.
- Checking module source paths and resource references.

These errors illustrate why module interfaces should be treated like API contracts. The caller and module must agree on names, types, and ownership boundaries.

### Planning versus applying

Reinforced the distinction between preparing infrastructure changes and making those changes in AWS.

- `terraform validate` checks whether the configuration is structurally valid.
- `terraform plan` proposes changes based on the configuration and Terraform state.
- `terraform apply` executes the approved changes.

A plan must be reviewed before applying it, particularly when it proposes resource replacement or destruction. A successful plan also does not prove that the deployed application, network connectivity, or AWS permissions will work as intended.

## 2. AWS Architecture and Networking

Today's Terraform work involved configuring the relationships between:

- VPCs and subnets.
- Public and private routing.
- Internet and NAT Gateways.
- Application and monitoring instances.
- Load balancer security groups.
- Database subnet groups and RDS security groups.

An important design principle is to give each module clear responsibility and pass resource identifiers between modules explicitly.

For a multi-region architecture, regional resources must be created and configured in their own Region. Resource identifiers such as subnet IDs and security group IDs cannot simply be reused across Regions.

## 3. Kubernetes and kOps Practice

Today's broader DevOps practice also included creating a Kubernetes cluster using `kOps`.

The workflow covered the main cluster-creation prerequisites:

1. Launch an EC2 instance to act as the management machine.
2. Create and configure the required IAM role and permissions.
3. Install and configure `kubectl`.
4. Install and configure `kOps`.
5. Configure AWS credentials.
6. Generate an SSH key with `ssh-keygen`.
7. Prepare the cluster configuration and execute the cluster-creation command.
8. Verify the resulting cluster.

I also revisited a DNS resolution problem involving `kubevpro.in`.

The troubleshooting included checking the domain's NS records using:

```bash
dig NS kubevpro.in
dig +short NS kubevpro.in
```

The important distinction was that creating a Route 53 hosted zone does not, by itself, mean that the domain has been registered or that the public domain's authoritative nameservers point to that hosted zone.

A hosted zone stores DNS records. Domain registration and delegation determine whether the wider DNS system directs queries to those nameservers.

The domain situation also highlighted the need to distinguish a free subdomain supplied by a hosting platform from a separately registered domain that can be delegated to Route 53. The exact DNS setup depends on who controls the parent domain and its nameserver configuration.

I then repeated the cluster-creation process independently and successfully. Redoing the process without relying entirely on a previous walkthrough helped reinforce the workflow and test my understanding.

## 4. General Lessons

### Understand ownership boundaries

Whether working with Terraform modules, AWS security groups, DNS zones, or Kubernetes resources, it is important to know which component owns a resource and which component consumes it.

### Diagnose the layer that is actually failing

A DNS issue is not necessarily a Kubernetes issue. A Terraform type mismatch is not necessarily an AWS service issue. Identifying the failing layer helps narrow the investigation and avoid unnecessary changes.

### Repeating a workflow builds independence

Repeating cluster creation from memory provides a practical check of understanding. The goal is not only to get a successful result but also to understand the prerequisites, configuration, dependencies, and verification steps.

### Successful execution is only one part of validation

A command completing successfully does not automatically prove that every part of the system is correct. Infrastructure should be checked at the appropriate level: Terraform configuration, AWS resources, DNS resolution, cluster health, application readiness, and network connectivity.

## 5. Practical Commands and Checks

| Purpose | Command |
|---|---|
| Inspect authoritative nameservers | `dig NS kubevpro.in` |
| Return nameservers without additional output | `dig +short NS kubevpro.in` |
| Format Terraform configuration | `terraform fmt -recursive` |
| Validate Terraform configuration | `terraform validate` |
| Review proposed infrastructure changes | `terraform plan` |

Run Terraform commands from the appropriate root configuration directory. The exact Kubernetes verification commands depend on the cluster configuration and whether the cluster has finished provisioning.

## 6. Next Steps

- [ ] Finish resolving CloudRecover's remaining Terraform module-interface errors.
- [ ] Validate the primary-region Terraform configuration.
- [ ] Review the Terraform plan carefully before applying changes.
- [ ] Continue validating the Kubernetes cluster and DNS configuration.
- [ ] Keep a record of the exact commands used for cluster creation and verification.
- [ ] Document errors, their root causes, and the verified fixes instead of recording only successful outcomes.

## Daily Reflection

Today's work combined infrastructure design with practical debugging. The most useful lesson was that infrastructure automation depends on correctly defined interfaces and dependencies—not just valid individual resources.

By working through the Terraform module connections and independently repeating the Kubernetes cluster-creation process, I strengthened both my troubleshooting process and my ability to reproduce infrastructure workflows.
