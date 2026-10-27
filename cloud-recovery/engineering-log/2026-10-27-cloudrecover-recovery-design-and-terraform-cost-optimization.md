# CloudRecover Learning Log — Recovery Architecture and Terraform Cost Optimization

**Date:** 2026-10-27  
**Project:** CloudRecover — Multi-Region AWS Disaster Recovery  
**Focus:** Monitoring access, recovery strategy, Terraform resource lifecycle, and cost-aware infrastructure management

## 1. Session overview

Today I made progress on CloudRecover's infrastructure and began preparing for cross-region disaster recovery. I moved from building the primary environment toward understanding what must be available to restore the application and its data in a second AWS Region.

A particularly valuable result was finding a way to preserve my configured EC2 instances while removing two costly NAT Gateways when I am not working on the project.

## 2. Faster, repeatable infrastructure provisioning

I reprovisioned my CloudRecover servers and successfully ran the CI/CD pipeline in **less than 40 minutes**. Earlier in the project, similar work used to take **more than three hours**.

This improvement reinforced the benefit of reusable Terraform modules, repeatable provisioning, and familiarity with my deployment pipeline. It also showed that tasks which originally required extensive troubleshooting are becoming routine.

## 3. Using the existing ALB for monitoring dashboards

I wanted to access Grafana and Prometheus from my browser while keeping their instances in private subnets.

I learned that an existing **internet-facing Application Load Balancer (ALB)** can act as the public entry point and forward requests to private instances over the VPC's local routes. A NAT Gateway is not required for **inbound traffic forwarded by an ALB**.

My monitoring target groups use:

| Target group | Backend service | Backend port |
|---|---|---|
| Application | NestJS API | `3000` |
| Grafana | Grafana | `3000` |
| Prometheus | Prometheus | `9090` |

I learned to distinguish between the ALB's **listener port** and a target group's **backend port**. Although ports `80` and `443` are conventional entry points, distinct listeners on other ports can be useful for a temporary lab without custom hostnames. Listener rules or listeners must actually forward to target groups; creating a target group alone does not make it reachable.

A security group must allow traffic along each relevant path, including browser-to-ALB access and ALB-to-private-instance access. Administrative dashboards should not be exposed without appropriate access restrictions and authentication; plain HTTP does not encrypt credentials.

## 4. Prometheus scraping and monitoring scope

I configured or prepared the Prometheus scrape configuration around three endpoints:

| Job | Endpoint | Purpose |
|---|---|---|
| `prometheus` | `localhost:9090/metrics` | Prometheus's own metrics |
| `cloudrecover-api` | `172.40.20.18:3000/metrics` | NestJS application metrics |
| `node-exporter` | `172.40.20.18:9100/metrics` | EC2 host metrics |

I explored PromQL queries such as `up` to check scrape health and CPU utilization queries based on `node_cpu_seconds_total`.

**Scope decision:** I decided not to spend additional time building Grafana dashboards now. The monitoring foundation is sufficient for this stage; I will return to dashboard design later. The next priority is cross-region backup and recovery.

## 5. Beginning Phase 7: Cross-region backup and recovery

I reviewed the next project phase, which includes:

- Selecting primary and recovery AWS Regions.
- Defining separate VPCs with non-overlapping CIDR ranges.
- Maintaining distinct Terraform root configurations and states for primary and recovery infrastructure.
- Reusing the networking, security, application, database, and load-balancing modules.
- Planning RDS snapshots and cross-region copies, including encryption and KMS permissions.
- Defining recovery-region secrets, configuration, prerequisites, and deployment dependencies.

### Recovery architecture decisions

I recognized that I **do not need to keep an identical copy of my entire primary infrastructure running** in the recovery region. I chose to investigate a **backup-and-restore** strategy because it reduces the cost of idle recovery infrastructure.

My initial list of recovery components includes the VPC, subnets and routing, security groups, web server, RDS, and potentially a bastion host. An ALB is also relevant because the primary application's public entry point depends on one. Monitoring servers can be deferred during initial recovery.

A key distinction is between infrastructure that must **exist during recovery** and infrastructure that must **already be running before failure**. Terraform can provision many of these resources on demand, but this increases recovery time.

### Understanding RDS snapshot restoration

I initially thought I could attach a recent snapshot to an existing RDS instance. I learned that **restoring an RDS snapshot creates a new database instance**. The recovery application must then connect to the restored database endpoint with compatible credentials, network permissions, and configuration.

The snapshot is a **point-in-time copy**, not a live copy of the latest transaction. Backup frequency and successful cross-region copying affect **RPO** (the potential data-loss window). The time needed to provision networking, restore RDS, deploy the API, validate it, and redirect traffic affects **RTO**.

My chosen approach is a cost-conscious starting point, but it must eventually be demonstrated in a recovery test—not just described in Terraform.

## 6. Avoiding unintended EC2 replacements in Terraform

While extending the ALB-related infrastructure, a Terraform plan unexpectedly showed `-/+` replacements of existing EC2 instances and their `aws_ec2_instance_state` resources.

The plan pointed to an important difference between these EC2 configuration arguments:

- `security_groups`: associated with group names and can trigger replacement when used incorrectly for instances in a VPC.
- `vpc_security_group_ids`: the appropriate configuration for security group IDs in the custom VPC; security group associations can generally be updated without instance replacement.

After correcting the configuration to use `vpc_security_group_ids`, the unintended replacements disappeared.

**Lesson:** Always inspect `terraform plan` before applying, especially when modifying running infrastructure. The `-/+` marker means Terraform plans to destroy and recreate a resource, which is substantially different from an in-place `~` update.

## 7. Preserving EC2 instances while reducing NAT Gateway costs

I did not want to destroy my servers because rebuilding their configurations would waste time. At the same time, my primary VPC contained **two NAT Gateways**, which continue incurring hourly charges even while application EC2 instances are stopped.

I therefore modified my Terraform networking module to support an `enable_nat_gateway` boolean. The conditional configuration covers:

- The two NAT Gateways.
- Their two Elastic IPs.
- Their associated private-subnet default routes.

I learned that `count` can make resources conditional, but introducing it changes a resource address from an unindexed address to an indexed one (`[0]`). Terraform's plan showed the existing resources mapped to their new indexed addresses rather than introducing replacements solely for the refactor.

### Why the routes matter

A private route table may contain:

```text
172.40.0.0/16  -> local
0.0.0.0/0      -> NAT Gateway
```

If I delete the NAT Gateway but leave the `0.0.0.0/0` route behind, AWS may show the route as a **blackhole** because its target no longer exists. Internal VPC communication still follows the `local` route, but outbound traffic that relies on the deleted NAT cannot work.

The correct paused-state design is to **remove the NAT-dependent default routes, not the private route tables or subnet associations**.

### Final Terraform plan reviewed

My latest plan showed:

| Planned action | Resource | Purpose |
|---|---|---|
| Create: 1 | `aws_ec2_instance_state.bastion_state` | Manage the existing bastion instance in `stopped` state; **not** create a new EC2 server |
| Update in place: 2 | Two private route tables | Remove the NAT-dependent `0.0.0.0/0` routes |
| Destroy: 4 | Two NAT Gateways and two associated Elastic IPs | Eliminate their ongoing NAT and IP allocation costs |

**Plan summary: `1 to add, 2 to change, 4 to destroy`.** Importantly, the plan did **not** propose destroying or replacing the existing EC2 application, Prometheus, or Grafana servers, nor the VPC, subnets, RDS, or ALB.

This is a **reviewed plan**, not proof of a completed apply. The session does not establish that the NAT Gateways were actually deleted or that the final AWS route tables were verified after applying.

## 8. Cost and connectivity trade-offs

Disabling NAT reduces recurring cost, but private instances will lose **NAT-based outbound connectivity** unless another path exists (for example, suitable VPC endpoints). That can affect SSM, package installation, and artifact downloads.

Stopping EBS-backed EC2 instances preserves their installed software and data on attached EBS volumes, but EBS storage is still billable. RDS and the ALB can also continue generating charges while the application servers are stopped. The bastion should be considered separately because its instance-state management was newly introduced.

This led to an important design distinction:

- **Active mode:** NAT Gateways and outbound private-subnet routes are present; instances can be started and operated normally.
- **Paused mode:** NAT Gateways and NAT routes are removed, while stateful infrastructure and EC2 disks remain preserved for a later session.

On resuming work, I will need to restore valid outbound routes and NAT Gateways (or equivalent connectivity) before assuming SSM and application deployment can work.

## 9. Key takeaways

1. **Terraform modules enable cross-region reuse**, but recovery infrastructure does not need to be identical to primary infrastructure.
2. **RDS snapshots are restored into new database instances**; they are not attached to an already-running instance.
3. **RPO and RTO should drive recovery decisions**, not only whether Terraform can create the resources.
4. **A public ALB can route to services in private subnets** without those services having public IP addresses.
5. **Listener ports and backend ports serve different purposes** and need not be the same.
6. **`terraform plan` is a safety review tool**, particularly when it shows resource replacements.
7. **NAT Gateway lifecycle and route lifecycle must be coordinated** to avoid stale blackhole routes.
8. **Preserving EC2 is different from keeping EC2 running**; stopping compute while retaining EBS is useful for a learning environment.
9. **Cost optimization involves trade-offs**: removing NAT saves money but may temporarily disable private-instance internet and SSM connectivity.

## 10. Follow-up items

- [ ] If not already done, apply the reviewed paused-state Terraform plan using the intended variable values.
- [ ] Confirm both NAT Gateways and dedicated Elastic IP allocations have been removed.
- [ ] Confirm the private route tables no longer contain NAT-dependent default routes.
- [ ] Check which other billable resources remain active, particularly RDS, ALB, EBS, and public IPv4 allocations.
- [ ] Research AWS disaster recovery strategies, RPO/RTO, cross-region RDS snapshots, snapshot encryption, and restore sequencing.
- [ ] Select the recovery region and define a non-overlapping CIDR range.
- [ ] Design the recovery Terraform root configuration and its independent state.
- [ ] Return to Grafana dashboards after the core recovery flow has been demonstrated.

## Reflection

Today's work reinforced that infrastructure engineering is not only about creating resources. It also involves controlling their lifecycle safely, understanding network dependencies, preserving work already performed, and balancing reliability against cost.

I am moving into the most demanding part of CloudRecover—cross-region disaster recovery—with a clearer understanding of both the primary environment and the decisions that should guide recovery design.
