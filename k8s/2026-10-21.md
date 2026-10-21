# Kubernetes Learning Journal — October 21, 2026

## 1. Today's Objectives

- Recreate a Kubernetes cluster using kOps without relying on yesterday's step-by-step guidance.
- Understand how kOps state storage and cluster configuration work.
- Troubleshoot a `kubectl` configuration problem.
- Learn the purpose of Pods and Namespaces.
- Continue learning Kubernetes without rushing through the course.
- Manage AWS resources responsibly to avoid unnecessary costs.

---

## 2. Practical Work Completed

### 2.1 Independently recreated a Kubernetes cluster

I successfully recreated a Kubernetes cluster on AWS using kOps, relying on my understanding of the previous day's work.

The cluster was configured with:

- **1 control-plane node**
- **2 worker nodes**
- **Instance type:** `c7i-flex.large`
- **AWS region:** `us-east-1`
- **Availability Zones:** `us-east-1a` and `us-east-1b`
- **Networking:** Cilium
- **Cluster name:** `cluster.k8s.local`

This was an important milestone because I was able to repeat the process independently rather than simply follow instructions.

### 2.2 Created a dedicated kOps state bucket

I created a new S3 bucket for the cluster's state:

`tinoscript-kops-cluster-state`

I also enabled S3 bucket versioning.

Commands used:

```bash
aws s3api create-bucket \
  --bucket tinoscript-kops-cluster-state \
  --region us-east-1

aws s3api put-bucket-versioning \
  --bucket tinoscript-kops-cluster-state \
  --versioning-configuration Status=Enabled
```

I verified the versioning configuration:

```bash
aws s3api get-bucket-versioning \
  --bucket tinoscript-kops-cluster-state
```

The result confirmed that versioning was enabled.

**What I learned:** The kOps state store is where kOps records the configuration and state it uses to manage a cluster. A dedicated state bucket helps separate one cluster's state from another cluster's state.

S3 versioning preserves previous versions of objects when they are changed or deleted. However, versioning alone does not guarantee that a cluster can be fully restored. Recovery also depends on having the required certificates, keys, configuration, and AWS resources.

### 2.3 Created and applied the cluster configuration

I used the following command to define the cluster:

```bash
kops create cluster \
  --name=cluster.k8s.local \
  --state=s3://tinoscript-kops-cluster-state \
  --zones=us-east-1a,us-east-1b \
  --networking=cilium \
  --node-count=2 \
  --node-size=c7i-flex.large \
  --control-plane-size=c7i-flex.large \
  --control-plane-count=1
```

I then applied the configuration:

```bash
kops update cluster \
  --name=cluster.k8s.local \
  --state=s3://tinoscript-kops-cluster-state \
  --yes \
  --admin
```

Initially, I received a `State Store: Required value` error because I had omitted the `--state` argument.

I corrected the command by explicitly specifying the S3 bucket.

**What I learned:** kOps needs to know which state store contains the cluster configuration. I can provide this using `--state` on individual commands or set the `KOPS_STATE_STORE` environment variable.

For example:

```bash
export KOPS_STATE_STORE=s3://tinoscript-kops-cluster-state
```

If I use the environment variable, I should still verify that it points to the intended bucket before running cluster-management commands.

### 2.4 Validated the cluster successfully

I ran:

```bash
kops validate cluster \
  --name=cluster.k8s.local \
  --state=s3://tinoscript-kops-cluster-state
```

The validation output showed three healthy nodes:

- Two worker nodes
- One control-plane node

The final result was:

`Your cluster cluster.k8s.local is ready`

This confirmed that kOps had successfully provisioned the infrastructure and that the cluster passed validation.

I also verified the cluster listing:

```bash
kops get clusters \
  --state=s3://tinoscript-kops-cluster-state
```

The cluster appeared under the expected name.

**Important distinction:** A successful `kops validate cluster` confirms that the cluster meets kOps validation checks. It does not, by itself, prove that an application has been deployed or that every application-level function works.

---

## 3. Troubleshooting: Understanding kubeconfig

### 3.1 The original error

When I attempted to apply a Kubernetes manifest, I encountered an error similar to:

```text
failed to download openapi:
Get "http://localhost:8080/openapi/v2?timeout=32s":
connect: connection refused
```

I also discovered that:

```bash
kubectl config current-context
```

reported that no current context was set, and:

```bash
kubectl config get-contexts
```

showed no configured contexts.

### 3.2 What the error taught me

`kubectl` is the command-line client used to communicate with a Kubernetes API server.

It needs connection information and credentials, usually stored in a kubeconfig file. That configuration identifies the cluster, the user or credentials used to access it, and the current context.

A **context** connects a cluster, a user identity, and optionally a namespace. The current context tells `kubectl` which configured target to use by default.

When `kubectl` has no usable context, it may fall back to trying `localhost:8080`. Because there was no Kubernetes API server listening there, the request failed.

This was a client-configuration problem, not necessarily a failure of the cluster itself.

### 3.3 Why kOps state matters

I initially attempted to export kubeconfig for a cluster using the wrong state bucket. The cluster was not found there.

When I checked the older state bucket, I found a cluster configuration, but kubeconfig export failed because the required CA certificate could not be found.

This taught me that:

- A cluster name alone is not enough to identify the correct kOps state.
- The cluster name and state bucket must match the cluster I intend to manage.
- Kubernetes API access depends on valid configuration and credentials, including the required certificates.
- Finding old cluster metadata in S3 does not guarantee that the cluster can still be accessed or restored.

For future deployments, I should verify the state store and export kubeconfig from the correct cluster configuration before running `kubectl` commands.

---

## 4. Kubernetes Concepts Learned

### 4.1 Pods

A Pod is the smallest deployable unit in Kubernetes. It runs one or more containers that share certain resources, including networking and storage configuration.

Most application Pods contain one main application container. Multiple containers can share a Pod when they are tightly coupled and need to work together.

Key points:

- Pods run on cluster nodes.
- Kubernetes schedules Pods onto suitable worker nodes.
- Each Pod has its own network identity and IP address.
- Pod IP addresses can change when Pods are replaced.
- Deleting a standalone Pod does not automatically cause Kubernetes to recreate it.
- A Deployment can manage Pods through a ReplicaSet and replace missing replicas to maintain the desired count.

Useful commands:

```bash
kubectl get pods -A
kubectl get pods -o wide
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

The `-A` option lists Pods across all namespaces. The `-o wide` option provides additional details, including the node on which each Pod is running and its IP address.

### 4.2 Namespaces

A Namespace is a logical scope for organizing Kubernetes resources within a cluster.

Namespaces can help separate resources belonging to different applications, teams, or environments.

For example, a cluster might contain:

- `development`
- `staging`
- `production`

Namespaces help with organization and resource naming, and they can be used alongside access controls and policies.

However, **a Namespace does not automatically provide complete security isolation**. Proper isolation may also require:

- Role-Based Access Control (RBAC)
- NetworkPolicies
- Resource quotas and limits
- Appropriate handling of Secrets and credentials

I now understand that Namespaces are more than folders for resources, but they are not a complete security boundary by themselves.

### 4.3 Services — next topic

I planned to study Services today but decided to leave the topic for the next session.

The question I want to answer next is:

**If Pod IP addresses can change, how do other Pods and applications reliably communicate with them?**

Kubernetes Services address this problem by providing a stable network endpoint for a logical set of Pods.

I will study how Services select Pods, how selectors and labels work together, and how traffic reaches the correct application Pods.

---

## 5. Important Lessons About Cluster Naming and State

Today's troubleshooting reinforced several distinctions:

| Component | Purpose |
|---|---|
| Cluster name | Identifies the cluster in kOps configuration and commands |
| S3 state bucket | Stores kOps cluster state and configuration |
| AWS infrastructure | Provides the actual compute, networking, and supporting resources |
| kubeconfig | Tells `kubectl` how to access a Kubernetes cluster |
| Kubernetes context | Selects a configured cluster, identity, and optional namespace |

The name `cluster.k8s.local` was a configuration choice, not a pre-existing Kubernetes cluster. The `.k8s.local` suffix is commonly seen in older kOps learning setups; DNS behavior depends on the kOps configuration and version.

I also learned that creating a public Route 53 hosted zone is not the same as registering a domain or delegating that domain to the hosted zone. A hosted zone alone does not guarantee that a domain is publicly resolvable.

---

## 6. Cost Management and Cleanup

Because the cluster uses AWS resources that may incur charges, I decided to clean it up after validation rather than leave it running unnecessarily.

I previewed the deletion using:

```bash
kops delete cluster \
  --name=cluster.k8s.local \
  --state=s3://tinoscript-kops-cluster-state
```

The preview listed resources associated with the cluster and indicated that `--yes` was required to proceed.

The command to execute deletion is:

```bash
kops delete cluster \
  --name=cluster.k8s.local \
  --state=s3://tinoscript-kops-cluster-state \
  --yes
```

**Cleanup status:** The deletion command was prepared, but I have not yet confirmed from its completion output whether the cluster was actually deleted.

After executing it, I should verify that the EC2 instances and associated infrastructure have been removed. I should also check for any remaining billable resources rather than assuming that one deletion command removed everything.

---

## 7. My Learning Reflection

Yesterday, creating the cluster was difficult because I was still trying to understand the requirements, especially the domain and DNS issue.

Today, recreating the cluster independently felt smoother. I was able to use what I learned yesterday, identify the state-store mistake, correct it, and validate a fresh cluster.

The most important progress was not memorizing the commands. It was developing a clearer understanding of the relationship between:

- kOps and AWS infrastructure
- Cluster state and kubeconfig
- The Kubernetes API server and `kubectl`
- Worker nodes and Pods
- Namespaces and resource organization

I also recognized that I do not need to force myself to finish every planned topic in one session. I stopped before rushing through Services, leaving myself room to understand the topic properly next time.

My approach is to practise, document, reflect, and repeat. The documentation is a reference I can return to when needed; I do not need to memorize every command before moving forward.

---

## 8. Plan for the Next Session

1. Confirm whether today's cluster has been deleted.
2. If it has been deleted, recreate it independently and validate it.
3. Verify the nodes and kubeconfig before interacting with Kubernetes resources.
4. Study Services, labels, and selectors.
5. Understand how Services provide stable access to Pods whose IP addresses can change.
6. Test the concepts practically when the cluster is available.
7. Clean up AWS resources after the lab and verify the cleanup.

### Final takeaway

**Today I proved to myself that I can recreate and validate a kOps-managed Kubernetes cluster independently. I also learned to distinguish infrastructure state from Kubernetes client configuration, and established a foundation in Pods and Namespaces. My next step is to understand how Services connect applications reliably.**
