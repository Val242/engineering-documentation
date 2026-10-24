# Kubernetes Learning Journal — October 24, 2026

**Main Focus:** Kubernetes Services, networking, horizontal scaling, Ingress, and Helm.

## 1. Session Overview

Today, I continued my Kubernetes learning by exploring how Services expose applications, how traffic flows from external clients to Pods, and how multiple application replicas are managed.

I also practised working with Kubernetes resources on AWS using kOps and connected Kubernetes networking concepts to my existing knowledge of AWS Load Balancers, Target Groups, and EC2 instances.

The session helped me strengthen my understanding of Kubernetes architecture rather than simply memorizing commands.

## 2. Kubernetes Cluster Setup

I revisited the process of creating a Kubernetes cluster on AWS using kOps.

My cluster configuration uses:

- **Cluster name:** `cluster.k8s.local`
- **AWS region:** `us-east-1`
- **Control plane:** 1 node
- **Worker nodes:** 2
- **Instance type:** `c7i-flex.large`
- **Networking:** Cilium
- **State store:** `s3://tinoscript-kops-cluster-state`

I reviewed the commands for creating the cluster, exporting kubeconfig credentials, validating the cluster, and inspecting nodes.

This reinforced the relationship between AWS infrastructure and Kubernetes cluster components.

## 3. Kubernetes Services

### Why Services exist

Pods are temporary resources. They can be replaced, recreated, or rescheduled, and their IP addresses may change.

A Kubernetes Service provides a stable network endpoint through which clients can communicate with matching Pods.

I learned that:

- A Service uses label selectors to discover eligible Pods.
- Multiple Pods can share the same label.
- A single Service can route traffic to multiple ready Pods.
- The Service remains stable even when the underlying Pods are replaced.
- A Service is a networking abstraction, not a separate application running inside a Pod.

### Labels and selectors

For example, a Service configured with:

```yaml
selector:
  app: my-app
```

selects Pods with the matching label:

```yaml
labels:
  app: my-app
```

If a Deployment creates five matching Pods, those Pods can become eligible backends for the same Service.

**Key lesson:** A Deployment manages application replicas, while a Service provides network access to them.

## 4. Understanding Kubernetes Ports

One of the most important concepts I clarified today was the distinction between `port`, `targetPort`, and `nodePort`.

Consider this example:

```yaml
ports:
  - port: 80
    targetPort: 3000
    nodePort: 30001
```

### Service port

`port: 80` specifies the port exposed by the Kubernetes Service.

Clients connecting directly to the Service use this port.

### Target port

`targetPort: 3000` identifies the port on which the application listens inside the Pod.

The Service forwards traffic to the selected Pod on this port.

### NodePort

`nodePort: 30001` exposes the application through a port on the cluster's nodes.

An external client can use a reachable node IP address and this port to access the application.

### Important understanding

The ports do not need to have the same number.

For example:

**Client → NodePort 30001 → Service routing → Pod port 3000**

The Service port is 80, but Kubernetes networking can map the NodePort traffic to the application's target port without requiring a separate network hop through a Service server.

I also understood that an application listening on port 3000 cannot receive traffic sent to port 8080 unless something else is listening or forwarding on that port.

**Key lesson:** The Service's `targetPort` must correspond to the actual application listening port.

## 5. NodePort vs. LoadBalancer Services

### NodePort

A NodePort Service exposes an application through a port on the cluster's nodes.

Example:

```text
External Client
      |
Node IP:30001
      |
Kubernetes Service Routing
      |
Application Pod:3000
```

The client needs a reachable node address and the allocated NodePort.

NodePort does not automatically provision an external cloud load balancer.

### LoadBalancer

A LoadBalancer Service requests an external load balancer from the underlying infrastructure or a compatible controller.

On AWS, this can provide an externally accessible DNS endpoint for the application.

Example:

```text
External Client
      |
AWS Load Balancer
      |
Kubernetes Backend Networking
      |
Application Pod
```

Depending on the load balancer integration, traffic may be forwarded through worker-node NodePorts or directly to Pod IP addresses.

### Comparison

| Feature | NodePort | LoadBalancer |
|---|---|---|
| Entry point | Node IP and NodePort | External load balancer |
| External load balancer | Not automatically created | Provisioned when supported |
| Typical use | Labs and direct node exposure | Cloud application exposure |
| Additional LB costs | None inherently | Usually applicable |
| Backend routing | Kubernetes networking | Cloud LB plus Kubernetes networking |

### Connection to AWS

My previous understanding of AWS Load Balancers helped me understand Kubernetes Services.

I connected the following ideas:

- AWS Load Balancer listeners accept incoming requests.
- Target Groups identify backend destinations.
- Kubernetes Services identify eligible application Pods through selectors.
- `targetPort` specifies the application's listening port.
- The exact AWS backend port depends on the load balancer's target configuration.

This connection made Kubernetes networking easier to understand.

## 6. Services and Horizontal Scaling

I learned how Kubernetes Services support horizontally scaled applications.

Suppose a Deployment has one replica and is scaled to five replicas.

The Deployment and ReplicaSet ensure the required Pods exist, while the Service can continue routing traffic to the ready Pods without needing its configuration changed.

Example command:

```bash
kubectl scale deployment my-app --replicas=5
```

The Service discovers eligible Pods through their labels.

### Pod scaling vs. node scaling

I also reinforced the distinction between:

**Horizontal Pod Scaling:** Increasing the number of application Pods.

**Horizontal Node Scaling:** Increasing the number of worker nodes available to run Pods.

Multiple Pods can run on the same worker node.

For example, ten Pods may run across two worker nodes if those nodes have sufficient capacity.

The Kubernetes scheduler considers resource requests, node capacity, scheduling constraints, and existing workloads.

If sufficient capacity is unavailable, new Pods may remain Pending.

### Autoscaling

I reviewed two related components:

- **Horizontal Pod Autoscaler (HPA):** Adjusts the number of application replicas based on supported metrics.
- **Cluster Autoscaler:** Can adjust worker-node capacity when scheduling requirements cannot be met, if installed and configured.

**Key lesson:** Increasing the number of Pods does not automatically increase the number of worker nodes.

## 7. ReplicaSets and Fault Recovery

I reviewed the relationship between Deployments, ReplicaSets, Pods, and worker nodes.

```text
Deployment
    |
ReplicaSet
    |
Multiple Pods
    |
Worker Nodes
```

A Deployment manages ReplicaSets, while a ReplicaSet maintains the desired number of Pod replicas.

If a worker node fails, the Kubernetes node controller eventually detects the failure.

Once the affected Pods are removed from the active replica count, the ReplicaSet can create replacements.

The scheduler then attempts to place those replacement Pods on available nodes.

If insufficient resources are available, replacement Pods may remain Pending.

I also learned that the recovery process is not necessarily instantaneous because node failure detection and Pod eviction take time.

**Key lesson:** ReplicaSets help maintain the desired replica count, but recovery also depends on healthy worker nodes and available capacity.

## 8. Practical Ingress and Service Configuration

I worked with a Kubernetes application named `my-app`.

The application configuration included:

- Deployment: `my-app`
- Service: `my-app`
- Service type: `ClusterIP`
- Service port: `80`
- Original target port: `8080`
- Ingress: `vpro-ingress`
- Ingress path: `/login`

### Ingress routing

The intended request flow was:

```text
Browser
   |
AWS Load Balancer
   |
NGINX Ingress Controller
   |
my-app Service
   |
Application Pod
```

The Ingress defines the HTTP routing rule, while the Service provides access to eligible Pods.

I modified the original Ingress example to remove its hostname restriction, allowing it to match the `/login` path without requiring that specific hostname.

I also corrected an important configuration mistake: the Ingress backend must reference the Service port, which was 80, rather than the application's target port, which was 8080.

### Ingress address

Kubernetes reported an AWS load balancer DNS address for `vpro-ingress`, showing that an external address had been assigned.

However, receiving an Ingress address alone does not prove that the application is serving successful HTTP requests.

## 9. Troubleshooting ImagePullBackOff

During the practical deployment, my Pod entered the `ImagePullBackOff` state.

I investigated using:

```bash
kubectl get pods -o wide
kubectl get deployment my-app
kubectl describe pod <pod-name>
```

The Pod events reported an image pull authorization failure for:

```text
imranvisualpath/vproapp:V1
```

The error indicated that the image repository was unavailable, inaccessible, or required authorization.

This explained why the Deployment had zero available replicas and why the Service had no endpoints.

### What I learned

- Kubernetes must successfully pull the container image before starting the container.
- A Pod that cannot start cannot become ready.
- A Service without ready backend endpoints cannot successfully route requests to the intended application.
- `kubectl describe pod` and its Events section are important troubleshooting tools.
- Successful creation of a Deployment or Ingress does not guarantee application availability.

I discussed switching to a public NGINX image as an alternative for learning, but did not confirm successful application access during this session.

## 10. Introduction to Helm

I also reviewed Helm.

**Helm is a package manager for Kubernetes that helps package, install, upgrade, and manage related Kubernetes resources as an application release.**

The three main concepts are:

### Chart

A package containing Kubernetes resource templates, metadata, and default configuration.

### Release

A specific installation of a Helm chart in a Kubernetes cluster.

### Values

Configuration settings used to customize chart templates.

For example, a Helm chart can contain templates for:

- Deployment
- Service
- ConfigMap
- Secret
- Ingress

Instead of managing every resource separately, Helm can manage them together as an application release.

**Key lesson:** Helm simplifies application deployment and lifecycle management, but Kubernetes remains responsible for running and coordinating the resources.

This was a conceptual introduction; I did not complete a Helm installation exercise today.

## 11. Key Architectural Understanding

Today's session strengthened the following mental model:

```text
External Client
       |
Load Balancer / Ingress
       |
Kubernetes Service
       |
Ready Application Pods
       |
Worker Nodes
```

At the same time:

```text
Deployment
     |
ReplicaSet
     |
Desired Pod Replicas
     |
Kubernetes Scheduler
     |
Worker Nodes
```

The two flows represent different responsibilities:

- **Deployment and ReplicaSet:** Desired application state and replica management.
- **Scheduler:** Placement of Pods on worker nodes.
- **Service:** Stable network access to eligible Pods.
- **Ingress:** HTTP/HTTPS routing to Services.
- **Cloud Load Balancer:** External traffic entry point.
- **Helm:** Packaging and managing related Kubernetes resources.

Understanding these responsibilities helps explain how Kubernetes provides scalability, availability, and application networking.

## 12. Reflection

Kubernetes has become easier for me to understand because I am connecting new concepts to technologies I already know.

My previous experience with Docker, AWS EC2, Load Balancers, infrastructure management, and system design has provided a useful foundation.

At the same time, practical Kubernetes exercises are helping the architecture become more intuitive.

Today, I moved beyond simply knowing that Services expose Pods. I explored why Services are needed, how port mapping works, how multiple Pods share a Service, and how networking interacts with horizontal scaling.

I also learned that configuring Kubernetes resources is different from verifying whether an application is operational.

My current understanding of Kubernetes fundamentals is improving, while more advanced topics such as production networking, storage, security, and cluster operations remain areas for future study.

## 13. Next Learning Objectives

For upcoming sessions:

1. Practise exposing applications with different Service types.
2. Verify end-to-end Ingress routing to a healthy backend application.
3. Explore Kubernetes readiness and liveness probes.
4. Practise scaling Deployments and observing Service endpoints.
5. Deploy an application using a Helm chart.
6. Continue exploring Kubernetes networking and troubleshooting.

---

**Session conclusion:** Today's main achievement was developing a clearer mental model of how Kubernetes manages application replicas and routes traffic between external clients, Services, and Pods.