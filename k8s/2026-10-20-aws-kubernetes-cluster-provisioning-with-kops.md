# AWS Kubernetes Cluster Provisioning with kOps
## Reusable DevOps Lab Runbook

**Last verified:** October 20, 2026  
**Operating system:** Ubuntu Linux  
**Cloud provider:** AWS  
**kOps version used:** 1.36.2  
**Kubernetes cluster name:** `cluster.k8s.local`  
**AWS region:** `us-east-1`  
**State store:** `s3://kopsstateval05`  
**CNI:** Cilium  
**Cluster topology:** 1 control-plane node + 2 worker nodes

---

# 1. Objective

This runbook documents how to provision a Kubernetes cluster on AWS using kOps, from preparing the management machine to validating the finished cluster.

The workflow is:

1. Launch a management EC2 instance.
2. Create and attach an IAM role with the required permissions.
3. Install and configure AWS CLI.
4. Install kubectl.
5. Install kOps.
6. Create an SSH key pair.
7. Prepare an S3 state store.
8. Choose the cluster name and DNS strategy.
9. Create the cluster configuration.
10. Review and apply the infrastructure changes.
11. Validate the Kubernetes cluster.
12. Verify nodes and workloads.
13. Clean up the environment when finished.

## 1.1 Architecture

The management EC2 instance is separate from the EC2 instances that form the Kubernetes cluster.

| Machine or service | Responsibility |
|---|---|
| Management EC2 | Runs AWS CLI, kOps and kubectl commands |
| Control-plane EC2 | Runs Kubernetes control-plane components |
| Worker EC2 1 | Provides compute capacity for Pods |
| Worker EC2 2 | Provides compute capacity for Pods |
| Amazon S3 | Stores kOps cluster configuration and state |
| AWS Auto Scaling | Maintains the configured EC2 capacity |
| Network Load Balancer | Provides access to the Kubernetes API endpoint |
| Cilium | Provides Kubernetes pod networking |

The management machine provisions the cluster. Once Kubernetes is running, its control plane schedules and manages workloads across the worker nodes.

---

# 2. Launch the management EC2 instance

Launch an Ubuntu EC2 instance to act as the management machine.

Ensure that:

- You can connect to the instance.
- It has outbound network access to download the required tools.
- Its IAM role allows the required AWS operations.
- The instance is in the intended AWS account and region.

For this learning exercise, the management instance was separate from the three Kubernetes nodes.

Connect to the instance:

```bash
ssh -i /path/to/your-key.pem ubuntu@YOUR_MANAGEMENT_INSTANCE_IP
```

Replace the example key path, username and IP address with your actual values.

Update the package index and install basic utilities:

```bash
sudo apt update
sudo apt install -y curl unzip jq
```

Check the machine architecture:

```bash
uname -m
```

The installation commands below assume `x86_64`, which is appropriate for an x86-64 EC2 instance. ARM-based machines require the corresponding ARM binaries.

---

# 3. Create and attach an IAM role

Create an IAM role for the management EC2 instance and attach it through an EC2 instance profile.

The role must provide the AWS permissions needed for the operations you intend to perform with kOps, including creating and managing the necessary cluster resources.

Depending on the kOps setup, these can include EC2, Auto Scaling, ELB, IAM, S3, and networking-related permissions.

Follow the official kOps AWS prerequisites to determine the required permissions for your version and deployment.

**Security note:** Do not automatically grant unrestricted administrator access to every machine. Use the minimum permissions practical for the lab, and keep management permissions distinct from the permissions assigned to Kubernetes worker nodes.

After attaching the role, verify that the AWS CLI can retrieve credentials.

---

# 4. Install and configure AWS CLI

Install AWS CLI v2 on Ubuntu x86-64 using the official installer:

```bash
curl -fsSL https://awscli.amazonaws.com/v2/install.sh -o aws-install.sh
sudo bash aws-install.sh --install-dir /usr/local/aws-cli --bin-dir /usr/local/bin
```

Verify the installation:

```bash
aws --version
```

Check the AWS identity:

```bash
aws sts get-caller-identity
```

Set the default region for the current shell:

```bash
export AWS_DEFAULT_REGION=us-east-1
```

Confirm the region:

```bash
aws configure get region
echo "$AWS_DEFAULT_REGION"
```

When using an attached EC2 IAM role, you normally do not need to run `aws configure` with long-lived access keys. The AWS CLI can obtain temporary credentials from the instance role.

Do not hard-code AWS access keys in scripts or save them in source control.

---

# 5. Install kubectl

`kubectl` is the Kubernetes command-line client. It communicates with the Kubernetes API server to inspect and manage cluster resources.

Install the current stable kubectl binary for Linux x86-64:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -fsSL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

Install it:

```bash
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
```

Verify the installation:

```bash
kubectl version --client
```

Note that installing kubectl does not create a cluster or guarantee that it has a kubeconfig configured. Cluster access becomes available after the appropriate credentials and context are created.

---

# 6. Install kOps

kOps, or Kubernetes Operations, automates the provisioning and lifecycle management of Kubernetes clusters.

Install the latest kOps release for Linux x86-64:

```bash
curl -fsSL -o kops \
  "https://github.com/kubernetes/kops/releases/download/$(curl -fsSL https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '\"' -f 4)/kops-linux-amd64"
```

Make it executable and move it into the system path:

```bash
chmod +x kops
sudo mv kops /usr/local/bin/kops
```

Verify the installation:

```bash
kops version
```

For reproducibility, record the version you used. The version in this lab was `1.36.2`.

If you need that exact version in the future, download the corresponding versioned release rather than assuming the latest release will always behave identically.

---

# 7. Generate an SSH key pair

kOps can use an SSH public key when configuring access to cluster machines.

Generate a dedicated key pair:

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/kops-cluster
```

This creates:

- `~/.ssh/kops-cluster` — the private key.
- `~/.ssh/kops-cluster.pub` — the public key.

Protect the private key. Never share it or commit it to Git.

If you use a non-default key name, pass the public key to the cluster creation command using `--ssh-public-key`.

For example:

```bash
--ssh-public-key ~/.ssh/kops-cluster.pub
```

SSH access is useful for troubleshooting, but it is not the only way to investigate a cluster. AWS instance status checks, system logs, Auto Scaling activities, and other diagnostics can help when SSH is unavailable.

---

# 8. Prepare the kOps S3 state store

kOps needs a state store to retain the cluster configuration and related state.

This lab uses:

```text
s3://kopsstateval05
```

Verify access to the existing bucket:

```bash
aws s3 ls s3://kopsstateval05
```

If you are creating a new state bucket, use a globally unique bucket name:

```bash
aws s3api create-bucket \
  --bucket YOUR_UNIQUE_KOPS_BUCKET \
  --region us-east-1
```

Enable versioning on the bucket:

```bash
aws s3api put-bucket-versioning \
  --bucket YOUR_UNIQUE_KOPS_BUCKET \
  --versioning-configuration Status=Enabled
```

Set the state-store environment variable:

```bash
export KOPS_STATE_STORE=s3://kopsstateval05
```

This avoids having to type the state-store path repeatedly.

Confirm it:

```bash
echo "$KOPS_STATE_STORE"
```

**Important:** The S3 state bucket is not the Kubernetes cluster itself. It stores kOps state; the cluster's compute and networking resources are provisioned separately in AWS.

Keep the bucket if you plan to reuse it. Do not delete it just because you remove a cluster.

---

# 9. Choose the cluster name and DNS strategy

The original attempt used:

```text
kubevpro.in
```

A Route 53 public hosted zone had been created, but the domain was not registered and publicly delegated to those nameservers. A public DNS lookup returned `NXDOMAIN`.

Creating a hosted zone alone does not register a domain or automatically configure delegation at the registrar.

Because purchasing a domain was not practical for this lab, the cluster was created using:

```text
cluster.k8s.local
```

Set the cluster name:

```bash
export NAME=cluster.k8s.local
```

**DNS limitation:** `.k8s.local` is a legacy kOps naming approach. The behavior depends on the DNS mode supported by the installed kOps version. This lab succeeded without a public Route 53 hosted zone for that name, but that does not mean every cluster name can be used without DNS configuration.

For production environments, use a properly registered and delegated domain and follow the current kOps DNS requirements.

---

# 10. Create the cluster configuration

The cluster used:

- One control-plane node.
- Two worker nodes.
- Two availability zones.
- Cilium networking.
- `c7i-flex.large` for all three cluster instances.

The final working command was:

```bash
kops create cluster \
  --name=cluster.k8s.local \
  --state=s3://kopsstateval05 \
  --zones=us-east-1a,us-east-1b \
  --networking=cilium \
  --node-count=2 \
  --node-size=c7i-flex.large \
  --control-plane-size=c7i-flex.large \
  --control-plane-count=1 \
  --ssh-public-key=~/.ssh/kops-cluster.pub
```

If you generated a different SSH key path, replace the example path. If you rely on a default public key that kOps detects, you can omit this option.

The command defines the desired cluster configuration. Unless you use `--yes` directly with the creation command, you should not treat this step as confirmation that the AWS infrastructure has been provisioned.

### Why one control-plane node?

An earlier attempt used 12 control-plane nodes and failed validation. A single control-plane node is a simpler choice for a learning environment.

A production high-availability control plane requires deliberate design, including an appropriate number of control-plane and etcd voting members. One control-plane node is not highly available.

### Inspect the instance groups

```bash
kops get ig \
  --name=cluster.k8s.local \
  --state=s3://kopsstateval05
```

The expected groups are:

```text
control-plane-us-east-1a
nodes-us-east-1a
nodes-us-east-1b
```

Confirm the roles, instance types, counts and zones before proceeding.

---

# 11. Review and apply the configuration

This is a critical distinction in the workflow.

`kops create cluster` defines the desired cluster configuration. `kops update cluster` previews the AWS changes needed to match that configuration and, when confirmed, applies them.

First, review the changes:

```bash
kops update cluster \
  --name=cluster.k8s.local \
  --state=s3://kopsstateval05
```

Read the output carefully. Confirm that the instance types, network configuration, control-plane count and worker counts are what you intended.

If the proposed changes are correct, apply them:

```bash
kops update cluster \
  --name=cluster.k8s.local \
  --state=s3://kopsstateval05 \
  --yes
```

The `--yes` flag bypasses the confirmation prompt. It does not mean that the cluster has already finished initializing.

AWS provisioning and Kubernetes bootstrap happen asynchronously. You must verify that they succeed.

---

# 12. Verify EC2 provisioning

Check that the cluster's EC2 instances have launched:

```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters \
    "Name=tag:KubernetesCluster,Values=cluster.k8s.local" \
    "Name=instance-state-name,Values=pending,running" \
  --query 'Reservations[].Instances[].[InstanceId,State.Name,InstanceType,PrivateIpAddress]' \
  --output table
```

The successful lab had three running `c7i-flex.large` instances.

Inspect the Auto Scaling Groups:

```bash
aws autoscaling describe-auto-scaling-groups \
  --region us-east-1 \
  --query "AutoScalingGroups[?contains(AutoScalingGroupName, 'cluster.k8s.local')].[AutoScalingGroupName,DesiredCapacity,Instances[].[InstanceId,LifecycleState,HealthStatus]]" \
  --output json
```

Expected result:

- Control-plane group: desired capacity 1.
- First worker group: desired capacity 1.
- Second worker group: desired capacity 1.
- Instances eventually become `InService` and `Healthy`.

These checks confirm AWS-level infrastructure health, not Kubernetes readiness.

---

# 13. Validate the cluster

Run:

```bash
kops validate cluster \
  --name=cluster.k8s.local \
  --state=s3://kopsstateval05 \
  --wait 10m
```

This allows kOps to retry validation while the cluster initializes.

The final successful output was:

```text
Your cluster cluster.k8s.local is ready
```

Inspect the Kubernetes nodes:

```bash
kubectl get nodes -o wide
```

Inspect system Pods:

```bash
kubectl get pods -n kube-system
```

If the nodes are not ready yet, give the cluster time to initialize and examine the actual errors before making changes.

A running EC2 instance does not automatically mean Kubernetes has finished bootstrapping.

---

# 14. Troubleshooting guide

## A. DNS errors

**Symptom:** kOps cannot resolve the cluster's public DNS name.

**Likely cause:** The domain is not registered or publicly delegated correctly.

**Next steps:**

- Verify the domain registration.
- Verify authoritative nameserver delegation.
- Confirm that the cluster's DNS mode is appropriate for the kOps version.
- Do not assume creating a Route 53 hosted zone alone resolves public DNS.

## B. AWS instance launches fail

Inspect Auto Scaling activities:

```bash
aws autoscaling describe-scaling-activities \
  --region us-east-1 \
  --auto-scaling-group-name control-plane-us-east-1a.masters.cluster.k8s.local \
  --max-items 10 \
  --query 'Activities[].[StartTime,StatusCode,StatusMessage]' \
  --output table
```

During this lab, AWS rejected `t3.medium` with a Free Tier eligibility error. The instance groups were changed to `c7i-flex.large`, after which the instances launched successfully.

Do not assume that a type shown as Free Tier eligible guarantees that the entire Kubernetes setup is free. Pricing depends on the account, current terms, region and associated resources.

## C. EC2 instances are running, but Kubernetes is not ready

Inspect:

1. EC2 instance status checks.
2. Auto Scaling health and launch activities.
3. Control-plane bootstrap logs.
4. API server and etcd health.
5. Kubernetes node readiness.
6. API load-balancer target health.

If SSH access is available, inspect the bootstrap service on the affected control-plane node:

```bash
sudo systemctl status kops-configuration.service --no-pager
```

Read recent logs:

```bash
sudo journalctl -u kops-configuration.service --no-pager -n 100
```

Use the service and log names actually present on the node; do not assume every failure will be recorded in the same log.

## D. `kubectl` cannot connect

Check:

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes
```

If the API endpoint is unreachable, verify the API server and load-balancer health before troubleshooting application workloads.

---

# 15. Cost management and cleanup

The lab used three `c7i-flex.large` instances. These instances and associated resources may incur charges.

Potential costs include:

- EC2 compute.
- EBS volumes.
- Load balancers.
- Public IPv4 addresses.
- Data transfer and other provisioned resources.

Before leaving the lab running, check the current AWS pricing and your account's applicable credits or free-tier terms.

When finished, preview the deletion:

```bash
kops delete cluster \
  --name=cluster.k8s.local \
  --state=s3://kopsstateval05
```

Review the proposed deletions. If they are correct and you no longer need the cluster, delete it:

```bash
kops delete cluster \
  --name=cluster.k8s.local \
  --state=s3://kopsstateval05 \
  --yes
```

Then inspect AWS for any remaining resources and charges. Do not assume every resource has been removed until you verify it.

Keep the S3 state bucket if you intend to use it for future labs.

---

# 16. Final reusable checklist

Before provisioning:

- [ ] Management EC2 is reachable.
- [ ] IAM role and permissions are correct.
- [ ] AWS CLI works and points to the intended account.
- [ ] kubectl and kOps are installed.
- [ ] SSH key pair exists.
- [ ] S3 state store is accessible.
- [ ] DNS strategy is compatible with the kOps version.
- [ ] Instance types and cost are understood.

During provisioning:

- [ ] Cluster configuration created.
- [ ] Instance groups reviewed.
- [ ] `kops update cluster` preview inspected.
- [ ] Changes applied.
- [ ] EC2 instances launched.
- [ ] Auto Scaling groups report healthy instances.

After provisioning:

- [ ] `kops validate cluster` succeeds.
- [ ] `kubectl get nodes` shows all nodes ready.
- [ ] System Pods are healthy.
- [ ] Cost exposure is understood.
- [ ] Cleanup plan is clear.

---

# 17. Key lessons from this lab

The workflow has distinct stages:

1. **Prepare:** Configure the management machine, credentials, tools, SSH access and state store.
2. **Define:** Create the desired cluster configuration.
3. **Apply:** Use kOps to provision AWS infrastructure.
4. **Bootstrap:** Allow Kubernetes components to initialize.
5. **Validate:** Confirm the control plane and worker nodes are healthy.
6. **Operate:** Deploy and manage workloads using Kubernetes.
7. **Clean up:** Remove resources when the learning environment is no longer needed.

The most important lesson is that AWS infrastructure health and Kubernetes cluster health are different. EC2 instances can be running while Kubernetes is still initializing.

**Verified result:** On October 20, 2026, the `cluster.k8s.local` cluster reached a successful kOps validation state with one control-plane node and two worker nodes.

This runbook is a learning reference, not a production deployment blueprint. Review the current kOps documentation and AWS requirements before reusing it with different software versions, account permissions, instance types or DNS configurations.