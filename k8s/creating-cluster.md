# Runbook: Provisioning a Kubernetes Cluster on AWS with kOps

**Last tested:** October 20, 2026  
**kOps version:** 1.36.2  
**AWS region:** `us-east-1`  
**Cluster name:** `cluster.k8s.local`  
**State store:** `s3://kopsstateval05`  
**Networking:** Cilium  
**Status:** Successfully validated

---

## 1. Objective

This runbook documents how to provision, configure, validate, troubleshoot, and clean up a Kubernetes cluster on AWS using kOps.

It is intended to be reused when creating future Kubernetes learning environments.

### Final architecture

| Component | Configuration |
|---|---|
| Cluster management | kOps |
| Cloud provider | AWS |
| Region | `us-east-1` |
| Availability Zones | `us-east-1a`, `us-east-1b` |
| Control plane | 1 EC2 instance |
| Worker nodes | 2 EC2 instances |
| Instance type | `c7i-flex.large` |
| Networking | Cilium |
| Cluster DNS name | `cluster.k8s.local` |
| kOps state | S3 |
| API endpoint | AWS Network Load Balancer |

The cluster consists of three EC2 instances: one control-plane node and two worker nodes.

**Important:** `c7i-flex.large` was the instance type that successfully launched during this session. Its eligibility and cost depend on the AWS account and applicable pricing rules; do not assume the complete cluster is free.

---

## 2. Prerequisites

Before creating a cluster, verify that the management machine has:

- AWS CLI installed and configured.
- kOps installed.
- `kubectl` installed.
- AWS credentials with the necessary EC2, Auto Scaling, ELB, IAM, Route 53 where applicable, and S3 permissions.
- An S3 bucket for kOps state.
- An SSH key pair if SSH access is needed.
- An AWS region and availability zones selected.

Check the tools:

```bash
aws --version
kops version
kubectl version --client
aws sts get-caller-identity
```

The last command confirms which AWS account and identity your CLI is using.

Verify the region:

```bash
aws configure get region
```

If necessary, set it:

```bash
export AWS_DEFAULT_REGION=us-east-1
```

Confirm the state bucket exists and is accessible:

```bash
aws s3 ls s3://kopsstateval05
```

If the bucket does not exist, create it before proceeding. For this runbook, the bucket already exists.

---

## 3. Understand the domain requirement

My initial attempt used `kubevpro.in`.

I had created a public hosted zone in Route 53, but I had not registered the domain or configured delegation at the domain registrar.

I ran:

```bash
dig NS kubevpro.in @8.8.8.8
```

The result was `NXDOMAIN`.

### Why this failed

A Route 53 hosted zone creates a place to manage DNS records. It does not, by itself, register a domain or make the domain's nameservers discoverable through public DNS.

For a publicly delegated domain, the registrar or parent DNS zone must point to the appropriate authoritative nameservers.

### The solution used

Because I did not want to purchase a domain for this learning exercise, I switched to:

```text
cluster.k8s.local
```

This avoided the need to register `kubevpro.in` for this cluster.

**Important limitation:** `.k8s.local` is a legacy kOps DNS approach. DNS behavior and support depend on the kOps version and configuration; this setup should not be assumed to be suitable for production. For production, use a properly registered and delegated domain.

---

## 4. Create the cluster configuration

I initially tried using 12 control-plane nodes. That configuration failed validation because an etcd control-plane membership should use an odd number of voting members for quorum.

For a learning cluster, one control-plane node is sufficient. Two worker nodes provide a useful environment for scheduling workloads across nodes.

The working configuration uses:

- 1 control-plane node
- 2 worker nodes
- 2 availability zones
- Cilium networking
- `c7i-flex.large` instances

Create the cluster:

```bash
kops create cluster \
  --name=cluster.k8s.local \
  --state=s3://kopsstateval05 \
  --zones=us-east-1a,us-east-1b \
  --networking=cilium \
  --node-count=2 \
  --node-size=c7i-flex.large \
  --control-plane-size=c7i-flex.large \
  --control-plane-count=1
```

This command creates the desired cluster configuration in the kOps state store. Depending on the kOps version and options, it may also initiate configuration-related checks.

It does **not** mean that Kubernetes is ready.

### Inspect the instance groups

```bash
kops get ig \
  --name=cluster.k8s.local \
  --state=s3://kopsstateval05
```

Expected instance groups:

```text
control-plane-us-east-1a
nodes-us-east-1a
nodes-us-east-1b
```

Verify that there is one control-plane instance group and two worker instance groups.

---

## 5. Apply the configuration to AWS

After reviewing the configuration, apply it:

```bash
kops update cluster \
  --name=cluster.k8s.local \
  --state=s3://kopsstateval05 \
  --yes
```

The `--state` argument is important. It tells kOps which S3 state store contains the cluster configuration.

If you omit it, kOps may not find the cluster configuration you created.

The update operation applies the desired infrastructure configuration. AWS Auto Scaling then launches the instances.

### Verify that the instance groups have the intended instance type

```bash
kops get ig \
  --name=cluster.k8s.local \
  --state=s3://kopsstateval05
```

Check that all three groups use the intended machine type.

### Admin kubeconfig credentials

If you need administrator credentials in your kubeconfig, use the supported kOps admin credential workflow. For example, the `--admin` option can be used when updating a cluster in kOps versions that support it:

```bash
kops update cluster \
  --name=cluster.k8s.local \
  --state=s3://kopsstateval05 \
  --yes \
  --admin
```

Follow any credential-expiration settings or additional output provided by your installed kOps version.

---

## 6. Verify EC2 instance provisioning

Do not immediately assume that a successful `kops update cluster` means the instances are running.

Inspect the cluster instances:

```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters \
    "Name=tag:KubernetesCluster,Values=cluster.k8s.local" \
    "Name=instance-state-name,Values=pending,running" \
  --query 'Reservations[].Instances[].[InstanceId,State.Name,InstanceType,PrivateIpAddress]' \
  --output table
```

Inspect Auto Scaling Groups:

```bash
aws autoscaling describe-auto-scaling-groups \
  --region us-east-1 \
  --query "AutoScalingGroups[?contains(AutoScalingGroupName, 'cluster.k8s.local')].[AutoScalingGroupName,DesiredCapacity,Instances[].[InstanceId,LifecycleState,HealthStatus]]" \
  --output json
```

Expected outcome:

- One control-plane instance.
- One worker in `us-east-1a`.
- One worker in `us-east-1b`.
- Instances in the `running` state.
- Auto Scaling instances in `InService` with health status `Healthy`.

These checks confirm AWS infrastructure health, not Kubernetes health.

---

## 7. Validate the Kubernetes cluster

Run:

```bash
kops validate cluster \
  --name=cluster.k8s.local \
  --state=s3://kopsstateval05 \
  --wait 10m
```

The `--wait 10m` option allows kOps to retry validation while the cluster initializes.

A successful validation should end with:

```text
Your cluster cluster.k8s.local is ready
```

The node status should show all three nodes as ready:

| Role | Expected status |
|---|---|
| Control plane | `True` |
| Worker node 1 | `True` |
| Worker node 2 | `True` |

You can also inspect the nodes with:

```bash
kubectl get nodes -o wide
```

Inspect system pods:

```bash
kubectl get pods -n kube-system
```

Inspect the cluster API and other resources only after the API server is reachable.

---

## 8. Troubleshooting encountered during this session

### Problem A: Public DNS lookup failed

**Symptom**

```text
error doing DNS lookup for NS records
no such host
```

**Cause**

A Route 53 hosted zone had been created, but the domain had not been registered or delegated through public DNS.

**Resolution**

Use a domain-free learning configuration such as `cluster.k8s.local` where supported, or use a domain that is properly registered and delegated.

Do not repeatedly run `kops update cluster` expecting it to fix missing public DNS delegation.

### Problem B: Too many control-plane nodes

**Symptom**

Cluster configuration validation failed after specifying 12 control-plane nodes.

**Cause**

The control-plane membership was configured with an unsuitable etcd voting-member count for quorum.

**Resolution**

For this learning cluster, use one control-plane node and two workers. A production high-availability control plane requires a deliberately designed multi-node configuration.

### Problem C: The `--state` argument was missing

**Symptom**

kOps could not find the intended cluster configuration during an update.

**Resolution**

Include:

```bash
--state=s3://kopsstateval05
```

in subsequent kOps commands so the correct state store is used.

### Problem D: AWS rejected `t3.medium` instance launches

**Symptom**

Auto Scaling activities reported:

```text
The specified instance type is not eligible for Free Tier.
```

**Cause**

AWS rejected the requested instance type under the account's applicable Free Tier eligibility restriction.

**Resolution**

The instance groups were changed to `c7i-flex.large`, and the updated configuration was applied. AWS subsequently launched all three instances.

To inspect Auto Scaling failures in the future:

```bash
aws autoscaling describe-scaling-activities \
  --region us-east-1 \
  --auto-scaling-group-name control-plane-us-east-1a.masters.cluster.k8s.local \
  --max-items 10 \
  --query 'Activities[].[StartTime,StatusCode,StatusMessage]' \
  --output table
```

Do not assume that a machine type being listed as eligible guarantees that every resource in the cluster is free of charge.

### Problem E: EC2 instances were running, but the cluster was not ready

**Symptoms**

- EC2 instances were running.
- Auto Scaling groups reported `InService` and `Healthy`.
- `kops validate cluster` showed that the control plane was not ready.
- The API server, controller manager, scheduler, or kOps system pods were not ready.
- Worker machines had not joined the cluster.

**Cause**

Infrastructure had been provisioned, but Kubernetes bootstrap and node registration were still in progress.

**Resolution**

Allow time for initialization and rerun:

```bash
kops validate cluster \
  --name=cluster.k8s.local \
  --state=s3://kopsstateval05 \
  --wait 10m
```

The validation eventually succeeded, with the control-plane node and both workers reporting `Ready`.

If validation continues to fail, inspect the relevant Auto Scaling activities, EC2 instance status checks, bootstrap service logs, API server health, and load-balancer target health before changing the cluster configuration.

---

## 9. What this exercise taught me

The troubleshooting sequence matters:

1. **DNS:** Can the chosen cluster name be used with the DNS mode supported by this kOps setup?
2. **Configuration:** Has kOps stored the intended cluster and instance-group configuration?
3. **Infrastructure provisioning:** Did AWS successfully launch the EC2 instances?
4. **Instance health:** Are the instances running and healthy according to AWS?
5. **Kubernetes bootstrap:** Has the control plane initialized?
6. **Node registration:** Have the workers joined the cluster?
7. **Cluster validation:** Does kOps confirm that the cluster is healthy?

A successful step does not guarantee that the next step has completed.

In this session, the first problem was public DNS delegation. After switching to a domain-free cluster name, the next issue was AWS instance eligibility. After correcting the instance type, the infrastructure came up, but Kubernetes needed additional time to initialize.

The final validation confirmed that all three nodes were ready.

---

## 10. Safely clean up the learning environment

Kubernetes learning clusters can incur AWS charges even when no applications are deployed.

When you have finished experimenting, decide whether to keep the cluster running or delete it.

**Warning:** Deleting the cluster destroys its associated infrastructure. Confirm that you no longer need the cluster or its workloads before proceeding.

To delete the cluster and its managed infrastructure:

```bash
kops delete cluster \
  --name=cluster.k8s.local \
  --state=s3://kopsstateval05 \
  --yes
```

Review the output and verify the result in AWS. Do not assume that every associated resource or the S3 state bucket has been removed. Check for any remaining load balancers, storage volumes, public IPv4 addresses, and other resources that may continue to incur charges.

Keep the S3 state bucket if you intend to reuse it for future kOps clusters. Do not delete the bucket merely because you have deleted this cluster.

---

## 11. Quick reference: next time I create a cluster

1. Confirm AWS credentials and region.
2. Confirm the S3 state bucket is accessible.
3. Select a DNS strategy compatible with the kOps version.
4. Create a cluster configuration with a suitable control-plane count.
5. Inspect the instance groups.
6. Apply the configuration with `kops update cluster`.
7. Verify EC2 instances and Auto Scaling health.
8. Run `kops validate cluster --wait 10m`.
9. Confirm all nodes are ready using `kubectl get nodes`.
10. Monitor costs and delete the environment when finished.

**Final verified result from October 20, 2026:** `cluster.k8s.local` was successfully provisioned in AWS `us-east-1` with one control-plane node, two worker nodes, Cilium networking, and kOps state stored in `s3://kopsstateval05`.
