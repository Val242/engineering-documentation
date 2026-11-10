# CloudRecover – Daily Engineering Journal

Date: 10 November 2026\
Project: CloudRecover\
Primary Region: `eu-west-1` (Ireland)\
Recovery Region: `eu-west-2` (London)\
Private Hosted Zone: `cloudrecover.internal`\
Application DNS: `api.cloudrecover.internal`

## 1. Overview of today's work

Today's focus was implementing private DNS using Amazon Route 53 for the CloudRecover disaster recovery architecture.

The objective was to configure a single internal DNS name that could resolve to the primary Application Load Balancer during normal operation and switch to the recovery Application Load Balancer when the primary environment becomes unhealthy.

The work completed today included:

- Created a Route 53 private hosted zone named `cloudrecover.internal`.
- Associated the primary VPC in `eu-west-1` with the hosted zone.
- Associated the recovery VPC in `eu-west-2` with the same hosted zone.
- Created primary and secondary DNS failover records.
- Configured both DNS records as ALB alias records.
- Enabled `evaluate_target_health` for both ALB aliases.
- Tested private DNS resolution from an EC2 instance.
- Compared the DNS responses of the application record with the DNS responses of both ALBs.
- Investigated ALB target-group health using the AWS CLI.

The failover test is not yet complete. Both primary and recovery targets were found to be unhealthy during the investigation.

## 2. Understanding DNS and hosted zones

### 2.1 Do I need to purchase a domain name?

Initially, I thought I needed to purchase or register a domain name before configuring DNS in Route 53.

Today, I learned that this is not necessary when working with private DNS.

I can create a private hosted zone using an internal DNS namespace:

```
cloudrecover.internal
```

This is not a publicly registered internet domain. It is a private DNS namespace used within the AWS environment.

Resources in VPCs associated with the private hosted zone can resolve records within that namespace, provided the VPC DNS settings support resolution.

This means I can experiment with private DNS without purchasing a domain name.

### 2.2 What is a hosted zone?

A hosted zone is a container for DNS records belonging to a particular domain or DNS namespace.

My hosted zone is:

```
cloudrecover.internal
```

The application DNS record is:

```
api.cloudrecover.internal
```

Route 53 automatically creates default `NS` and `SOA` records when a hosted zone is created.

Therefore, my hosted zone contains four records:

1. NS record.
2. SOA record.
3. Primary application A alias record.
4. Secondary application A alias record.

The primary and secondary application records have the same DNS name and record type but different failover routing configurations.

### 2.3 What is port 53?

DNS uses port 53.

- UDP port 53 is commonly used for DNS queries and responses.
- TCP port 53 is also used for DNS operations, including larger responses and zone transfers.

When I used commands such as `nslookup` and `dig`, I was querying DNS to resolve my private application name into IP addresses.

DNS resolution and HTTP communication are separate processes:

1. DNS resolves `api.cloudrecover.internal` into IP addresses.
2. The client uses the resolved address to communicate with the application through the ALB.

## 3. Private hosted zone and VPC association

My private hosted zone is associated with two VPCs.

| Environment | AWS Region  | Purpose                        |
| ----------- | ----------- | ------------------------------ |
| Primary     | `eu-west-1` | Normal application environment |
| Recovery    | `eu-west-2` | Disaster recovery environment  |

The primary Terraform state creates the hosted zone and associates the primary VPC.

The recovery Terraform state associates its VPC with the existing hosted zone using `aws_route53_zone_association`.

The recovery state consumes the hosted zone ID created by the primary state rather than creating another hosted zone.

### Important lesson

Associating multiple VPCs with the same private hosted zone allows resources in those VPCs to resolve the DNS records.

However, this does not establish network connectivity between the VPCs.

VPC peering or a Transit Gateway is not required simply to share private hosted-zone DNS resolution between associated VPCs.

## 4. Configuring Route 53 failover records

The application DNS name is:

```
api.cloudrecover.internal
```

I created two DNS records:

| Record   | Routing policy | Destination  |
| -------- | -------------- | ------------ |
| Primary  | PRIMARY        | Primary ALB  |
| Recovery | SECONDARY      | Recovery ALB |

Both records are configured as `A` alias records.

The relevant Terraform configuration is:

```
resource "aws_route53_record" "application" {
  zone_id         = local.hosted_zone_id
  name            = var.record_name
  type            = "A"
  set_identifier  = "${var.project}-${lower(var.failover_type)}"

  alias {
    name                   = var.alb_dns_name
    zone_id                = var.alb_zone_id
    evaluate_target_health = true
  }

  failover_routing_policy {
    type = var.failover_type
  }
}
```

### Understanding `set_identifier`

The primary and recovery records have the same DNS name and record type.

The `set_identifier` distinguishes the individual records.

For example:

```
cloudrecover-primary
cloudrecover-secondary
```

It is not the DNS record name itself.

### Understanding `evaluate_target_health`

This configuration:

```
evaluate_target_health = true
```

instructs Route 53 to evaluate the health of the ALB alias target.

This is important because I want Route 53 to consider the health of the primary and recovery load balancers when deciding which failover record to return.

A separate Route 53 health-check resource is not automatically necessary for this alias-based design.

## 5. DNS commands and practical tests

I performed these tests from an EC2 environment in the primary VPC.

### 5.1 Resolve the private application DNS name

Command:

```
nslookup api.cloudrecover.internal
```

Output:

```
Server: 127.0.0.53
Address: 127.0.0.53#53

Non-authoritative answer:
Name: api.cloudrecover.internal
Address: 79.125.20.65
Name: api.cloudrecover.internal
Address: 52.212.26.238
```

This confirmed that the private DNS name was resolving successfully from my EC2 environment.

### 5.2 Resolve the primary ALB directly

Command:

```
dig +short cloudrecover-primary-alb-510712796.eu-west-1.elb.amazonaws.com
```

Output:

```
79.125.20.65
52.212.26.238
```

### 5.3 Resolve the recovery ALB directly

Command:

```
dig +short cloudrecover-recovery-alb-984913421.eu-west-2.elb.amazonaws.com
```

Output:

```
18.134.201.2
13.43.173.119
```

### 5.4 Resolve the application DNS name again

Command:

```
dig +short api.cloudrecover.internal
```

Output:

```
52.212.26.238
79.125.20.65
```

The order of the IP addresses changed, but they matched the primary ALB's DNS responses.

They did not match the recovery ALB's DNS responses.

Conclusion: At the time of the test, Route 53 was returning the primary ALB as the destination for `api.cloudrecover.internal`.

The two IP addresses did not mean that Route 53 was distributing traffic between the two regions. They were simply the addresses returned for the primary ALB.

ALB IP addresses can change, so this comparison is a point-in-time diagnostic.

### 5.5 Test the application through private DNS

Command:

```
curl http://api.cloudrecover.internal/health/ready
```

Output:

```
{"status":"ok","database":"up"}
```

### 5.6 Test the primary ALB directly

Command:

```
curl http://cloudrecover-primary-alb-510712796.eu-west-1.elb.amazonaws.com/health/ready
```

Output:

```
{"status":"ok","database":"up"}
```

These tests demonstrated that the application was responding through the private DNS name and the primary ALB at the time of testing.

For future tests, I can use `curl -i` to inspect the HTTP response status code as well as the response body:

```
curl -i http://localhost:3000/health/ready
```

## 6. Understanding ALB target-group health checks

An Application Load Balancer uses target-group health checks to determine whether registered targets are healthy.

The health-check configuration includes:

- Protocol.
- Port.
- Path.
- Expected HTTP response code.
- Health-check interval.
- Healthy threshold.
- Unhealthy threshold.

My NestJS application exposes the readiness endpoint:

```
/health/ready
```

The expected successful HTTP response is:

```
200 OK
```

### 6.1 Inspecting the primary target group

Command:

```
aws elbv2 describe-target-groups \
  --region eu-west-1 \
  --names cloudrecover-primary-tg \
  --query 'TargetGroups[*].{Path:HealthCheckPath,Port:HealthCheckPort,Protocol:HealthCheckProtocol,Matcher:Matcher.HttpCode,Interval:HealthCheckIntervalSeconds}' \
  --output table
```

Output:

| Setting  | Value        |
| -------- | ------------ |
| Protocol | HTTP         |
| Port     | traffic-port |
| Path     | `/`          |
| Matcher  | 200          |
| Interval | 30 seconds   |

The primary target group was checking `/`, rather than `/health/ready`.

This was intentional for the outage simulation. I deliberately made the primary target unhealthy and did not change its configuration during this test.

### 6.2 Inspecting the recovery target group

Command:

```
aws elbv2 describe-target-groups \
  --region eu-west-2 \
  --names cloudrecover-recovery-tg \
  --query 'TargetGroups[*].{ARN:TargetGroupArn,Path:HealthCheckPath,Matcher:Matcher.HttpCode}' \
  --output table
```

Output:

| Setting           | Value           |
| ----------------- | --------------- |
| Health-check path | `/health/ready` |
| Expected response | 200             |

The recovery target group's health-check configuration is aligned with my NestJS readiness endpoint.

However, having the correct health-check path does not automatically mean the registered EC2 target is healthy.

## 7. Inspecting target health using AWS CLI

### 7.1 Retrieve the primary target-group ARN

Command:

```
aws elbv2 describe-target-groups \
  --region eu-west-1 \
  --names cloudrecover-primary-tg \
  --query 'TargetGroups[*].{Name:TargetGroupName,ARN:TargetGroupArn,LoadBalancer:LoadBalancerArns}' \
  --output table
```

The primary target-group ARN returned was:

```
arn:aws:elasticloadbalancing:eu-west-1:568740659610:targetgroup/cloudrecover-primary-tg/a1b410cb8ba5c7e1
```

### 7.2 Inspect primary target health

Command:

```
aws elbv2 describe-target-health \
  --region eu-west-1 \
  --target-group-arn arn:aws:elasticloadbalancing:eu-west-1:568740659610:targetgroup/cloudrecover-primary-tg/a1b410cb8ba5c7e1 \
  --query 'TargetHealthDescriptions[*].{Target:Target.Id,State:TargetHealth.State,Reason:TargetHealth.Reason}' \
  --output table
```

Output:

```
Target: i-0b640d76c2b3623d3
State: unhealthy
Reason: Target.ResponseCodeMismatch
```

The primary target was unhealthy.

`Target.ResponseCodeMismatch` means the response received by the health checker did not match the configured success-code matcher.

In this case, the target group was checking `/` and expecting HTTP `200`.

### 7.3 Retrieve the recovery target-group ARN

Command:

```
aws elbv2 describe-target-groups \
  --region eu-west-2 \
  --names cloudrecover-recovery-tg \
  --query 'TargetGroups[*].{ARN:TargetGroupArn,Path:HealthCheckPath,Matcher:Matcher.HttpCode}' \
  --output table
```

The recovery target-group ARN returned was:

```
arn:aws:elasticloadbalancing:eu-west-2:568740659610:targetgroup/cloudrecover-recovery-tg/353ae047201642cc
```

### 7.4 Inspect recovery target health

Command:

```
aws elbv2 describe-target-health \
  --region eu-west-2 \
  --target-group-arn arn:aws:elasticloadbalancing:eu-west-2:568740659610:targetgroup/cloudrecover-recovery-tg/353ae047201642cc \
  --query 'TargetHealthDescriptions[*].{Target:Target.Id,State:TargetHealth.State,Reason:TargetHealth.Reason}' \
  --output table
```

Output:

```
Target: i-0dbdbed3738409e1c
State: unhealthy
Reason: Target.FailedHealthChecks
```

The recovery target was also unhealthy.

`Target.FailedHealthChecks` means the target failed the configured health checks. The exact cause has not yet been established.

The recovery EC2 instance, application process, listening port, security-group rules, and health-check response still need to be investigated.

## 8. Current infrastructure status

At the end of today's session:

| Component                   | Status               |
| --------------------------- | -------------------- |
| Private hosted zone         | Created              |
| Primary VPC association     | Configured           |
| Recovery VPC association    | Configured           |
| Primary failover record     | Created              |
| Recovery failover record    | Created              |
| ALB alias health evaluation | Enabled              |
| Private DNS resolution      | Working              |
| Primary DNS selection       | Confirmed            |
| Primary target health       | Unhealthy            |
| Recovery target health      | Unhealthy            |
| Successful DNS failover     | Not yet demonstrated |
| Automated DR pipeline       | Not yet implemented  |

### Important observation

Both target groups are currently unhealthy.

Therefore, I have not yet established the conditions needed to demonstrate a successful primary-to-recovery failover.

The recovery target must first become healthy.

Once recovery is healthy, I can deliberately make the primary unhealthy and observe whether Route 53 selects the recovery ALB.

If both records are unhealthy, the DNS response should not be treated as proof that the recovery environment is healthy or serving application traffic.

## 9. Plan for the next work session

### Step 1: Investigate recovery target health

- Inspect the recovery EC2 instance.
- Confirm that NestJS is running.
- Confirm that the application listens on the expected port.
- Test `/health/ready` locally.
- Inspect the HTTP response status code.
- Verify that the target group's health-check port and path match the application.
- Verify that the recovery EC2 security group permits traffic from the ALB security group on the application port.
- Recheck target health until the recovery target reports `healthy`.

### Step 2: Perform the DNS failover test

Once recovery is healthy:

- Keep recovery healthy.
- Deliberately make the primary target unhealthy.
- Allow the ALB health checks and Route 53 health evaluation to detect the failure.
- Resolve `api.cloudrecover.internal` again.
- Compare its DNS answers with the recovery ALB's DNS answers.
- Test the application through the private DNS name.
- Confirm that the recovery environment is serving requests.

### Step 3: Test failback

After confirming failover:

- Restore the primary target's health.
- Observe Route 53 behaviour when the primary becomes healthy again.
- Understand DNS caching and TTL effects.
- Confirm that failback behaviour is consistent with the intended DR design.

### Step 4: Automate disaster recovery

After verifying the DNS failover mechanism, proceed with the recovery pipeline.

The pipeline will eventually cover:

1. Failure detection.
2. Recovery orchestration.
3. Infrastructure provisioning.
4. Database snapshot selection and restoration.
5. Application deployment.
6. Application readiness validation.
7. DNS traffic activation or switching.
8. Recovery reporting and validation.

## 10. Key lessons learned

Today's work helped me understand several important infrastructure concepts:

1. I do not need a publicly registered domain to experiment with private DNS in AWS.
2. A Route 53 private hosted zone can be associated with VPCs in different AWS regions.
3. Private hosted-zone association does not establish network connectivity between VPCs.
4. DNS uses port 53.
5. A hosted zone contains DNS records, including automatically created NS and SOA records.
6. Route 53 failover routing uses primary and secondary records for the same DNS name and record type.
7. `set_identifier` distinguishes records that share the same name and type.
8. `evaluate_target_health = true` enables Route 53 to evaluate the health of an ALB alias target.
9. ALB target-group health checks and Route 53 health evaluation are related but distinct mechanisms.
10. `nslookup` and `dig` help verify DNS resolution.
11. `curl -i` helps inspect an application's HTTP response status.
12. `describe-target-groups` reveals target-group health-check configuration.
13. `describe-target-health` reveals the actual health state and reason for registered targets.
14. A healthy recovery destination is necessary to demonstrate the intended failover scenario.

## Final reflection

Today was a productive step in the development of CloudRecover.

I created the private hosted zone, associated both regional VPCs, configured the primary and secondary DNS failover records, and verified that the application DNS name resolves successfully.

I also learned how to investigate ALB target health using the AWS CLI and how target-group health relates to Route 53 failover.

The next objective is to make the recovery target healthy, complete the failover experiment, and then move towards automating the disaster-recovery workflow.

I am building CloudRecover incrementally: understand the infrastructure, test each component, verify the behaviour, and then automate the process.