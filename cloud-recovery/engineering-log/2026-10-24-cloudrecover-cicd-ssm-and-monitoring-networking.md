# CloudRecover Learning Log — CI/CD, SSM, and Monitoring Networking

**Date:** 2026-10-24  
**Project:** CloudRecover  
**AWS primary region:** `eu-west-1`

## Objectives for the session

1. Stabilize the GitHub Actions CI/CD deployment pipeline and validate deployment to the application EC2 instance.
2. If possible, provision and access the Prometheus and Grafana monitoring servers.

## 1. Production dependency security checks

The GitHub Actions pipeline ran:

```bash
npm audit --omit=dev --audit-level=high
```

It initially flagged high-severity denial-of-service advisories affecting `body-parser` and `multer`, with `@nestjs/platform-express` implicated through its dependency on Multer.

I ran `npm audit fix` locally. The `body-parser` finding disappeared, but the Multer issue remained. Re-running `npm audit fix` did not resolve it.

Dependency inspection showed:

```text
@nestjs/platform-express@11.2.7
└── multer@2.4.0
```

The npm metadata inspection also showed that the next available NestJS 11 patch, `@nestjs/platform-express@11.2.8`, still requested Multer `2.4.0`. I explored the option of overriding that transitive dependency to a patched version instead of upgrading the entire NestJS major version.

**Lesson:** An audit failure can be caused by a transitive dependency, and `npm audit fix` cannot always repair it automatically. Investigate the dependency tree, upstream version constraints, and application compatibility before forcing upgrades.

**Verification status:** The conversation does not include a final post-override audit/build/test result, so the security finding should not be marked resolved solely on the basis of the proposed override.

## 2. GitHub Actions deployment through AWS Systems Manager

The pipeline failed during:

```bash
aws ssm send-command ...
```

with:

```text
InvalidInstanceId: Instances not in a valid state for account
```

The GitHub Actions debug log confirmed the deployment script and SSM parameters were constructed, but the failure occurred when sending the command. The remote deployment script had **not** yet begun execution, so this error did not establish a failure in S3 download, Prisma migrations, systemd, or health checks.

### Diagnostic steps

I queried EC2 and SSM in `eu-west-1`:

```bash
aws ec2 describe-instances \
  --region eu-west-1 \
  --filters "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].{ID:InstanceId,Name:Tags[?Key=='Name']|[0].Value,State:State.Name}" \
  --output table

aws ssm describe-instance-information \
  --region eu-west-1 \
  --query "InstanceInformationList[].{ID:InstanceId,Status:PingStatus,AgentVersion:AgentVersion}" \
  --output table
```

The running instances were the application (`web01`), bastion host, Grafana, and Prometheus. Only `web01` appeared as **Online** in SSM, with SSM Agent version `3.3.4121.0`.

I then added a GitHub Actions diagnostic using `aws sts get-caller-identity` and `aws ssm describe-instance-information`. GitHub Actions successfully assumed `CloudRecoverGitHubActionsDeployRole`, but the SSM query returned `AccessDeniedException`: the role lacked `ssm:DescribeInstanceInformation`.

This highlighted the difference between:

- **AWS authentication:** OIDC and STS role assumption succeeded.
- **AWS authorization:** Each API action, including diagnostic read actions, must be allowed in the role's identity-based policy.

The IAM policy shared during debugging was an old copy with an outdated instance ID; I clarified that the deployed policy was already correct. The actual GitHub Actions target ID and region are separate configuration values and must be checked independently of the IAM policy.

I subsequently reported that the issue was working. The conversation does not contain the final deployment execution log, so record the SSM issue as **reported resolved**, with end-to-end application verification still worth retaining as an explicit check.

### Architecture takeaway

Because I often destroy and recreate this Terraform environment to control costs, hardcoding EC2 instance IDs creates deployment maintenance work. Future improvement: discover the active deployment target from trustworthy Terraform outputs or unique EC2 tags and consider tag-scoped IAM authorization, while keeping the pipeline least-privileged.

## 3. Monitoring infrastructure provisioned

The EC2 inventory confirmed that **Grafana and Prometheus servers were already running**, alongside `web01` and the bastion host. This satisfies the provisioning portion of the day's secondary objective, but browser dashboard access was still unfinished.

## 4. Private subnet routing and NAT Gateway

I clarified that **a NAT Gateway is placed in a public subnet**, while a private subnet that needs outbound internet access can route `0.0.0.0/0` to that NAT Gateway.

The conventional distinctions are:

| Network element | Purpose |
| --- | --- |
| Internet Gateway | Supports internet connectivity for appropriately routed, publicly addressed resources |
| NAT Gateway | Lets private instances initiate outbound connectivity; does not provide unsolicited inbound browser access |
| Private subnet route to NAT | Allows outbound connections from private resources through the NAT Gateway |
| Public subnet route to Internet Gateway | Allows internet routing, subject to public addressing, security rules, and service listeners |

**Main lesson:** Outbound connectivity and inbound access from my browser are different requirements. Merely adding a NAT route or an inbound security group rule does not make a private instance's dashboard directly reachable from the public internet.

## 5. Attempting browser access to Grafana and Prometheus

I wanted to open the monitoring dashboards from my local browser. The proposed approach was SSM port forwarding, but both session attempts failed with:

```text
TargetNotConnected
```

This is consistent with the earlier SSM listing, which showed only the application server online. To use SSM port forwarding on the monitoring instances, I would need to verify their SSM Agent installation and state, IAM instance profiles, and outbound connectivity to SSM endpoints (through NAT or suitable VPC endpoints).

I also examined my Grafana security group rule:

```hcl
resource "aws_vpc_security_group_ingress_rule" "grafana_ui_from_my_ip" {
  security_group_id = aws_security_group.grafana.id
  cidr_ipv4         = var.my_ip
  from_port         = 3000
  ip_protocol       = "tcp"
  to_port           = 3000
  description       = "Grafana UI access from the administrator IP"
}
```

That rule correctly permits traffic to Grafana port `3000` **if a network path already exists**; it does not by itself supply a public IP address or public routing.

For **direct browser access to the instances** (rather than SSM tunneling), the required setup is a public subnet route to an Internet Gateway, public addresses on the instances, appropriately restricted security group rules, and running listeners on their respective ports:

- Grafana: `http://<grafana-public-ip>:3000`
- Prometheus: `http://<prometheus-public-ip>:9090`

A public route and IP do not override security groups, network ACLs, or a service listening only on loopback. This direct-public-IP approach is acceptable for a tightly restricted temporary lab, but authenticated or private access is preferable for a durable deployment. The conversations do not confirm that the monitoring instances were moved to public subnets or that either dashboard became accessible.

## 6. Session results

| Work item | Status at end of session |
| --- | --- |
| Investigate production npm audit findings | Investigated; final clean audit not shown |
| Diagnose CI/CD SSM deployment failure | Investigated and reported working |
| Confirm app EC2 is registered with SSM | Verified Online |
| Provision Grafana and Prometheus EC2 | Verified running |
| Access monitoring dashboards in browser | Not completed |
| Clarify NAT vs Internet Gateway vs SG behavior | Completed conceptually |

## Next session

1. Confirm the full CI/CD run completes and verify application service, ALB health, and a subsequent release deployment.
2. Re-run production dependency audit, build, and tests after the chosen Multer fix.
3. Decide how to expose monitoring UIs for this lab: tightly restricted direct browser access via public networking, or SSM/private access after registering the monitoring hosts.
4. Verify Grafana and Prometheus services are running and listening on `3000` and `9090` respectively.
5. Configure Prometheus scraping of application metrics (`3000`) and Node Exporter (`9100`) and connect Grafana to Prometheus.
6. Tear down infrastructure when testing is finished to limit AWS charges.

## Reflection

Today's work reinforced that deploying an application is not one operation. GitHub OIDC, IAM permissions, SSM managed-node state, deployment targeting, dependency security, EC2 networking, and service health are separate links in the chain. I also sharpened the distinction between a security group **permitting** a connection and the underlying network **providing a path** for that connection. My next step is to validate these layers together instead of assuming one green check proves the whole system is healthy.
