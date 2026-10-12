# DevOps Learning Journal — Day 6
 
**Date:** October 12, 2026  
**Focus:** Terraform infrastructure management, AWS SSM troubleshooting, EC2 storage expansion, and project scope decisions.

## 1. Today's Learning Objectives

- Inspect the existing Terraform configuration and identify the resources that define an environment.
- Define the inputs required to create a temporary environment.
- Understand how temporary AWS environments could be provisioned and cleaned up.
- Troubleshoot an AWS Systems Manager deployment failure caused by insufficient disk space.
- Understand the relationship between EBS volumes, Linux partitions, and filesystems.
- Make a cost-conscious decision about the remaining project scope.

## 2. Inspecting the Existing Terraform Configuration

I reviewed the existing Terraform configuration in my `infrastructure/` directory.

The configuration is organized into separate files:

- `instance.tf` — defines the EC2 instance.
- `InstID.tf` — contains the Ubuntu AMI data source.
- `keypair.tf` — manages the EC2 key pair.
- `provider.tf` — configures the AWS provider.
- `SecGrp.tf` — defines security groups.
- `variables.tf` — declares input variables.

I confirmed that the existing Terraform configuration relies on AWS default networking rather than explicitly selecting a subnet in the EC2 resource configuration.

### EC2 storage configuration

My EC2 instance configuration includes:

```hcl
root_block_device {
  volume_size = 20
  volume_type = "gp3"
}
```

This configures the root EBS volume to use 20 GiB of General Purpose SSD storage (gp3).

I applied the Terraform change to increase the configured root volume size.

## 3. Troubleshooting the AWS SSM Disk-Space Issue

### The problem

During a remote deployment through AWS Systems Manager (SSM), I encountered an error similar to:

```text
No space left on device
```

The error occurred while SSM was attempting to write a temporary worker file under its local working directory.

This prevented the remote deployment operation from proceeding successfully.

### Investigating the cause

I investigated the EC2 instance's storage usage and discovered that the root filesystem was full.

Initially, the root filesystem had approximately 7.6 GiB of total space and was at 100% utilization.

I then inspected disk usage to identify the directories consuming the available storage. The investigation revealed significant usage from:

- `/opt/taskflow-api` — the application deployment directory and its releases.
- `/root/.npm` — the npm cache.
- `/var` — system and application-related files.

The npm cache alone occupied approximately 879 MiB.

### First corrective action: clean the npm cache

I ran:

```bash
npm cache clean --force
```

This removed unnecessary cached npm data and freed approximately 1 GiB of disk space.

This was a useful immediate mitigation, but it did not address the underlying problem of having insufficient root filesystem capacity for the workload.

### Second corrective action: increase EBS storage

I updated the Terraform EC2 configuration to increase the root EBS volume to 20 GiB and applied the change.

However, increasing the EBS volume did not initially make the entire additional capacity available to Linux. The root partition and filesystem still needed to be expanded.

I subsequently expanded the root partition and filesystem.

### Verification

After the storage expansion, I checked the root filesystem using:

```bash
df -h /
```

The resulting storage figures were approximately:

| Metric | Result |
|---|---:|
| Total root filesystem size | 20 GiB |
| Used space | 6.8 GiB |
| Available space | 13 GiB |
| Utilization | 35% |

I also inspected the block-device layout with:

```bash
lsblk
```

The root disk was approximately 20 GiB, and the root partition occupied almost the entire disk.

### What I learned from this incident

This troubleshooting exercise taught me several important operational lessons:

1. **EBS storage and the Linux filesystem are different layers.** Increasing an EBS volume does not always automatically expand the partition and filesystem to use the new capacity.
2. **Disk-space problems can affect deployment tooling.** SSM needs local disk space to execute remote commands and create temporary files.
3. **Inspect before making changes.** Commands such as `df -h`, `lsblk`, and `du` help identify whether the issue is filesystem capacity, partition sizing, or directory-level consumption.
4. **Cleaning caches is a mitigation, not always the permanent solution.** Removing unnecessary files can restore capacity, but the underlying storage allocation must also be appropriate.
5. **Deployment reliability depends on host maintenance.** Automated deployment pipelines can fail even when the application code and CI/CD workflow are correct.

## 4. Understanding the Self-Service Environment Provisioning Design

I explored how temporary AWS environments could be provisioned, deployed, and cleaned up with minimal manual intervention.

The intended lifecycle is:

1. A user requests an environment.
2. GitHub Actions validates the request.
3. GitHub Actions assumes the appropriate AWS role through OIDC.
4. Terraform provisions the temporary EC2 environment.
5. The existing deployment approach deploys TaskFlow through SSM and systemd.
6. The environment is destroyed when requested or when its expiry policy is reached.

I also began separating temporary environment resources from the existing infrastructure.

The proposed Terraform layout includes:

```text
infrastructure/
├── modules/
│   └── temporary_environment/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── environments/
│   └── temporary/
│       ├── main.tf
│       └── variables.tf
└── existing Terraform files...
```

The intention is to keep the temporary environment configuration and state separate from the existing infrastructure.

I identified several inputs that a temporary environment would require:

- Environment name.
- Ubuntu AMI ID.
- EC2 instance type.
- EC2 key pair.
- Subnet ID.
- Security group IDs.
- Public IP configuration.

I also recognized that SSM-based deployment requires appropriate IAM permissions and network connectivity. A temporary instance should not simply reuse the existing application's security group without reviewing the access rules and dependencies.

**Status:** I explored the design and began preparing the Terraform structure. I did not complete or validate the full temporary environment provisioning workflow.

## 5. Cost Management and Project Scope

I decided to stop the project at its current level rather than continue implementing temporary environment provisioning, controlled destruction, and automatic expiry cleanup.

The decision was based on the additional AWS investment and the practical experience I had already gained.

I concluded that the existing project provides useful experience in:

- Infrastructure provisioning with Terraform.
- EC2 and EBS management.
- CI/CD automation with GitHub Actions.
- OIDC-based AWS authentication.
- Remote deployment through SSM.
- Application process management with systemd.
- Monitoring and observability using Prometheus and Grafana.
- Operational troubleshooting.

I can explore development, staging, and production environment provisioning in a future project, designed with those requirements from the beginning.

Before shutting down or deleting any AWS resources, I need to identify dependencies and preserve anything I want to retain. I should not destroy the existing Terraform-managed infrastructure indiscriminately.

## 6. Key Takeaways

Today's most important lesson was that reliable DevOps work extends beyond writing a successful CI/CD pipeline.

Infrastructure capacity, filesystem management, deployment tooling, access permissions, monitoring, and cost control all influence whether a deployment succeeds.

The SSM incident was particularly valuable because I investigated the actual failure, traced it to disk exhaustion, freed space by cleaning the npm cache, increased the EBS volume through Terraform, expanded the Linux partition and filesystem, and verified the available capacity.

I also practiced an important engineering decision: keeping a project's scope manageable rather than adding more infrastructure simply for the sake of complexity.

## 7. Next Steps

- Document the final architecture of the current project.
- Verify that the CI/CD workflow, application health checks, and monitoring remain functional.
- Review AWS resources and recurring costs.
- Preserve the Terraform state and any configuration needed to maintain the existing infrastructure.
- Plan the next project with environment separation and cleanup requirements included in its initial architecture.

**Day 6 conclusion:** I strengthened my understanding of Terraform-managed infrastructure, AWS storage troubleshooting, SSM deployment failures, and the operational decisions needed to maintain reliable and cost-conscious cloud systems.