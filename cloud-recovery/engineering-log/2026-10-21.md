# CloudRecover — Development Session Log

**Date:** October 21, 2026  
**Project:** CloudRecover  
**Focus:** Terraform infrastructure provisioning, module integration, bastion host configuration, and private-instance SSH connectivity  
**Status:** Infrastructure configuration is progressing; initial provisioning encountered AWS errors; private SSH connectivity remains unresolved.

## 1. Today's objectives

The primary objective was to continue building CloudRecover's multi-region disaster recovery infrastructure using Terraform, validate the module composition, and begin provisioning the AWS resources.

The practical goals were to:

- Integrate reusable Terraform modules into the primary regional configuration.
- Provision the required networking and compute infrastructure.
- Configure a bastion host for SSH access to private EC2 instances.
- Establish secure connectivity between the bastion, application servers, and monitoring servers.
- Clean up provisioned resources when finished to control AWS costs.

## 2. Terraform architecture and module integration

Continued working on the reusable Terraform architecture for CloudRecover.

The infrastructure is organized around separate regional configurations and reusable modules. The primary region is `eu-west-1` (Ireland), while the recovery region is `eu-west-2` (London).

The modules discussed and integrated include:

- Networking
- Security groups
- Application servers
- Database
- Load balancer
- Monitoring
- Bastion host

The root configuration composes these modules by passing outputs from one module into the inputs of another. For example, networking outputs supply subnet IDs and the VPC ID, while security-group outputs are passed to the EC2, load-balancer, and database modules.

This reinforces an important Terraform principle: **reusable modules define resources and expose interfaces; root modules compose them into a complete infrastructure.**

### Networking configuration

The primary VPC uses the CIDR `172.40.0.0/16`, with dedicated subnet ranges for the different infrastructure tiers.

| Tier | Subnet CIDRs |
|---|---|
| Public | `172.40.10.0/24`, `172.40.11.0/24` |
| Application | `172.40.20.0/24`, `172.40.21.0/24` |
| Monitoring | `172.40.30.0/24`, `172.40.31.0/24` |
| Database | `172.40.40.0/24`, `172.40.41.0/24` |

The intended routing design is:

- Public subnets route through the Internet Gateway.
- Private application and monitoring subnets use NAT Gateways for outbound internet access.
- Database subnets use a dedicated route table without a default internet route.
- The database subnet group spans two Availability Zones.

Networking values were moved into root-level variables and `terraform.tfvars`, while the reusable networking module receives its configuration through variables and exposes resource IDs through outputs.

The provider's region configuration remains a root-level responsibility.

## 3. Security group design

The security module contains security groups for the application servers, load balancer, Prometheus, Grafana, RDS, and bastion host.

The intended traffic flows include:

- HTTP traffic to the load balancer on port `80`.
- Load-balancer traffic to the application on port `3000`.
- Prometheus scraping the application's metrics endpoint and `node_exporter` on port `9100`.
- Grafana querying Prometheus on port `9090`.
- Application-to-RDS PostgreSQL traffic on port `5432`.
- Administrative SSH access restricted to the administrator's public IP.
- SSH from the bastion to private application and monitoring servers.

The bastion security group has a dedicated inbound SSH rule sourced from the administrator's IP and an outbound rule allowing traffic.

A key architectural detail is that **the bastion's outbound permission alone does not authorize SSH into private instances**. Each destination instance's security group must allow inbound TCP port `22` from the bastion security group.

The bastion is intended to use an existing public subnet rather than introducing a separate subnet tier.

## 4. Bastion host module

Created and configured a reusable `modules/bastion-host` module.

The EC2 instance configuration includes:

- An AMI supplied through a variable.
- A configurable instance type.
- A public subnet ID.
- An EC2 key pair.
- The bastion security group.
- A 20 GB `gp3` root volume.
- Instance tags.

The key pair resource registers a public key with AWS. The corresponding private key must remain available to the SSH client and must not be committed to Git.

The instance security group assignment was corrected conceptually to use `vpc_security_group_ids`, appropriate for an instance in a custom VPC. Public IP assignment is also needed for direct SSH access through the internet, alongside a public-subnet route to the Internet Gateway.

The module's intended outputs are:

- `bastion_instance_id`
- `bastion_public_ip`
- `bastion_key_pair_name`

### SSH key decision

A separate key was chosen for the private application and monitoring servers rather than reusing the bastion key.

The selected key name is:

`cloudrecover-private-servers`

This separates access to the private instances from access to the bastion. The private key should be protected and the corresponding public key installed on the target instances.

## 5. Terraform errors encountered

### A. AWS EC2 regional verification restriction

The initial `terraform apply` encountered an AWS error similar to:

`PendingVerification: Your request for accessing resources in this region is being validated...`

AWS indicated that additional instance launches could be blocked while the regional verification was pending, potentially for up to four hours.

**Interpretation:** This was an AWS account or regional access restriction, not a Terraform syntax problem.

The appropriate next step is to check for AWS verification messages and retry only after the restriction is cleared.

### B. Unsupported PostgreSQL engine version

RDS creation failed with:

`InvalidParameterCombination: Cannot find version 15.4 for postgres`

The configured PostgreSQL version was not available for the requested deployment in the selected region.

The intended resolution is to query the available PostgreSQL engine versions in both `eu-west-1` and `eu-west-2`, then choose a supported version. If consistent versions are desired across regions, verify that the selected version is available in both.

This remains unresolved.

### C. Bastion SSH public-key path

Terraform reported that the file did not exist at:

`../../primary/cloudrecover-bastion.pub`

The configuration used:

`file(pathexpand(var.public_key_path))`

The function call itself is valid inside a Terraform `.tf` configuration file. The failure occurred because the resolved file path did not point to an existing public-key file.

A subsequent attempt placed a function call directly in `terraform.tfvars`:

`public_key_path = file("${path.module}/infrastructure/primary/cloudrecover-bastion.pub")`

Terraform rejected this because **function calls are not allowed in `.tfvars` files**.

The correction is to provide a literal path string in `terraform.tfvars`, pointing to the actual public-key file. Path resolution must account for the directory from which Terraform is being run.

This issue must be verified as resolved before applying the configuration again.

### D. Terraform plan proposed resource destruction

A plan reported:

- 4 resources to add
- 0 resources to change
- 4 resources to destroy

This was a warning that the proposed configuration and Terraform state did not agree on the desired resource lifecycle.

The plan must be inspected to identify the exact resources marked for destruction before applying. Particular care is needed if the resources are existing networking or shared infrastructure.

No assumption should be made that the proposed replacements are safe merely because the plan is executable.

### E. Load-balancer target attachment

An earlier issue involved passing a single application instance ID into a module variable expecting `list(string)`. The one-instance form is a list containing the ID.

Another issue involved using `toset(var.instance_ids)` with instance IDs that are unknown until apply. Terraform cannot determine resource addresses from unknown set members during planning.

The proposed fix was to use stable, index-based keys for the target-group attachments so Terraform can establish their addresses during planning.

These are important module-integration considerations to validate in the current configuration.

## 6. Private SSH connectivity test

The bastion host was reachable, and its shell prompt showed:

`ubuntu@ip-172-40-10-97`

The target private application server was:

`172.40.20.174`

An SSH attempt was made using a key file on the bastion:

`ssh -i sshkey ubuntu@172.40.20.174`

The connection stalled. A verbose attempt with a ten-second connection timeout produced:

`connect to address 172.40.20.174 port 22: Connection timed out`

### Diagnosis

This failure occurs before SSH authentication begins. It therefore does not yet establish that the private key is incorrect.

The first checks should be:

1. Confirm that the target application server's attached security group allows inbound TCP port `22` from the bastion security group.
2. Confirm that the bastion is actually attached to the intended security group.
3. Check subnet network ACLs for SSH and return-traffic permissions.
4. Confirm that the target instance is running and its SSH service is available.
5. Once network connectivity works, verify that the correct public key is installed for the `ubuntu` user on the target server.

Both servers are in the same VPC, so a NAT Gateway is not required for the bastion-to-private-instance connection.

The same access design will be required for private monitoring servers if SSH administration through the bastion is intended.

## 7. Cost-control decision

At the end of the session, the decision was to stop and clean up the infrastructure rather than leave potentially billable resources running.

The planned action is to run `terraform destroy` from the primary regional configuration after reviewing the destruction plan and confirming the correct AWS account and Terraform state.

This matters because resources such as NAT Gateways, load balancers, RDS instances, and running EC2 instances can incur costs.

**Important status distinction:** the decision to destroy resources was made, but successful completion of `terraform destroy` has not yet been confirmed in this session. Terraform only manages resources represented in the selected state, so any partially created resources should also be checked in AWS afterward.

## 8. Current status at the end of the session

| Area | Status |
|---|---|
| Terraform module architecture | Developed and being integrated |
| VPC and subnet design | Defined in configuration |
| Security-group architecture | Defined; private SSH path needs validation |
| Bastion module | Created and configured |
| SSH key separation | Decision made: separate private-server key |
| Terraform public-key path | Needs confirmation after correcting `.tfvars` |
| AWS EC2 provisioning | Blocked by AWS verification restriction during the initial apply |
| RDS provisioning | PostgreSQL engine version error unresolved |
| Bastion-to-web SSH | Fails with a connection timeout |
| Full infrastructure provisioning | Not confirmed successful |
| Terraform cleanup | Planned, completion not confirmed |

## 9. Resume plan

When work resumes, proceed from the actual state of the infrastructure rather than assuming the previous apply completed successfully.

1. **Verify cleanup:** confirm whether `terraform destroy` completed and check for remaining billable resources.
2. **Resolve the AWS restriction:** confirm EC2 launches are permitted in the primary region.
3. **Fix the database engine version:** select a PostgreSQL version supported in both regions if required by the architecture.
4. **Verify key paths and module wiring:** ensure `public_key_path` points to an existing public key and the bastion uses its intended security group.
5. **Review Terraform plans carefully:** investigate any resources marked for destruction or replacement before applying.
6. **Provision and validate incrementally:** confirm networking, security groups, EC2 instances, RDS, and the load balancer in the intended order.
7. **Debug private SSH:** verify the bastion-to-instance security-group rules first, then network ACLs and SSH authentication.
8. **Validate monitoring access:** confirm the intended Prometheus and Grafana connectivity after private-server access is established.

## 10. Key learning outcomes

Today's work strengthened practical understanding of:

- Terraform module inputs, outputs, and root-level composition.
- The distinction between Terraform configuration errors and AWS service restrictions.
- Why `.tfvars` files accept literal values rather than arbitrary Terraform expressions.
- Terraform planning limitations when resource identifiers are unknown until apply.
- How security-group references control private SSH access.
- Why a successful bastion login does not automatically imply connectivity to private instances.
- The difference between SSH network timeouts and authentication failures.
- The need to inspect Terraform state and planned destructions before applying changes.
- AWS cost management through deliberate infrastructure cleanup.

**Overall conclusion:** CloudRecover's infrastructure design and Terraform configuration are progressing. Today's main outcome was moving into real deployment and troubleshooting, uncovering several AWS and Terraform integration issues, and establishing a clear next step for private SSH connectivity. The next session should begin by verifying cleanup and resolving the outstanding issues rather than rebuilding the design from scratch.
