# Kubernetes Learning Journal — October 22, 2026

## 1. Session Overview

- **Focus:** Kubernetes fundamentals and the relationship between Docker, containers, Pods, the control plane, worker nodes, ReplicaSets, and Deployments.
- **Learning approach:** Conceptual understanding through questions, reasoning, and connecting previously learned concepts.

### Session objective

Develop a clear mental model of how Kubernetes schedules workloads and how containers actually run inside a cluster, rather than simply memorizing Kubernetes commands.

---

## 2. Key Concepts Learned

### A. Connecting Docker to Kubernetes

I connected my previous Docker learning to Kubernetes, particularly the concepts of `CMD` and `ENTRYPOINT`.

- `ENTRYPOINT` defines the executable that should run in a container.
- `CMD` provides default arguments to the entrypoint, or a default command when no entrypoint is specified.
- A container image packages the application and its required environment.
- A container is a running instance of an image.

This helped me understand how Kubernetes can use an existing image to run an application without needing to build the image from the Pod specification.

### B. Understanding the Control Plane and Worker Nodes

One of my most important breakthroughs was understanding the relationship between the control plane and worker nodes.

Previously, I knew that a Kubernetes cluster had a control plane and worker nodes, but I did not fully understand how they interacted to execute workloads.

I now understand that:

1. I write a Pod specification in a YAML file.
2. I submit it using `kubectl`, for example, `kubectl create -f pod.yaml`.
3. `kubectl` communicates with the Kubernetes API server.
4. The scheduler, operating as part of the control plane, selects a suitable worker node.
5. The kubelet on that worker node observes the assignment and works with the container runtime to start the Pod's containers.

The YAML file does not have to reside on the control-plane node. I can submit it from my local machine or another machine with appropriate cluster access.

**Key takeaway:** The control plane coordinates and schedules workloads; the worker nodes provide the resources and execute them.

### C. Understanding Pods and Containers

I clarified the distinction between a Pod and a container.

- A **Pod** is Kubernetes' smallest deployable unit.
- A **container** runs the application process within the Pod.
- A Pod can contain one or more containers.
- The worker node runs the Pod, and the container runtime starts its containers.

I also learned that saying a Pod is running generally means Kubernetes has created the Pod and its containers are running, although Pod status and individual container status are distinct and should be inspected when troubleshooting.

### D. Understanding the Container Runtime

I learned that worker nodes have a container runtime responsible for running containers.

Kubernetes does not necessarily require Docker Engine on its worker nodes. A common runtime is `containerd`.

The main responsibilities are:

- **Kubelet:** Ensures assigned Pods are running according to their specifications.
- **Container runtime:** Pulls container images when necessary and creates, starts, and stops containers.
- **Container:** Runs the application process.

I can inspect the runtime configured on a node with:

```bash
kubectl get nodes -o wide
kubectl describe node <worker-node-name>
```

The node description includes its container runtime version.

### E. Understanding ReplicaSets and Multiple Pods

I connected the Pod model to ReplicaSets.

A ReplicaSet maintains a desired number of Pod replicas. For example, if configured for five replicas, it attempts to maintain five Pods.

I worked through an example involving five Pods, each containing five containers:

- 5 Pods × 5 containers per Pod = 25 containers.

This helped me understand how workloads can grow beyond a single container.

However, I also learned that Kubernetes does not guarantee that all five Pods will run on one worker node. The scheduler considers resource availability, scheduling rules, and other constraints.

The cluster must have sufficient capacity to run the requested workload.

### F. Understanding the Deployment Hierarchy

I consolidated the relationship between Deployments, ReplicaSets, Pods, and containers.

**Deployment → ReplicaSet → Pods → Containers**

- A Deployment manages application rollouts and the ReplicaSets associated with its desired state.
- A ReplicaSet maintains the desired number of Pod replicas.
- Pods provide the execution environment for containers.
- Containers run the application processes.

This hierarchy explains how Kubernetes manages multiple instances of an application instead of requiring me to create and manage every Pod independently.

---

## 3. Commands Practiced or Discussed

```bash
# Create a Pod from a YAML specification
kubectl create -f pod.yaml

# Apply a manifest declaratively
kubectl apply -f pod.yaml

# List Pods and identify the node running each Pod
kubectl get pods -o wide

# List cluster nodes
kubectl get nodes -o wide

# Inspect a Pod and its events
kubectl describe pod <pod-name>

# Inspect a node, including its container runtime
kubectl describe node <worker-node-name>
```

I also learned the distinction between `kubectl create -f` and `kubectl apply -f`: both can create resources from a manifest, but `apply` is designed for declarative management of resources over time.

---

## 4. Important Mental Models Established

By the end of this session, I could connect the main components into one execution flow:

**Pod YAML → API server → Scheduler → Worker node → Kubelet and container runtime → Containers → Application**

I also understood that a worker node does not independently choose an arbitrary Pod to run. The scheduler assigns the Pod, and the kubelet on the selected node works to bring it to the desired state.

This is a significant improvement over knowing the components individually without understanding their responsibilities.

---

## 5. What Made This Session Valuable?

I began by connecting Docker concepts, especially `CMD` and `ENTRYPOINT`, to Kubernetes Pods and containers. That led me to investigate how the control plane and worker nodes interact.

The important shift was moving from memorizing definitions to understanding the execution model.

I can now explain why a Pod specification is submitted through the API server, why a worker node is selected, and how the containers inside the Pod actually start.

This foundation will help me reason about scheduling failures, image-pull errors, resource constraints, and workload scaling in future Kubernetes sessions.

---

## 6. Next Learning Objectives

For the next session, I can continue by reviewing Kubernetes Services and consolidating the relationship between networking and workloads.

Suggested topics:

1. Review `ClusterIP`, `NodePort`, and `LoadBalancer`.
2. Understand how Services use labels and selectors to identify Pods.
3. Understand why Services provide stable endpoints when Pod IP addresses change.
4. Connect Deployments, ReplicaSets, Pods, and Services in a single application architecture.
5. Continue practicing with the kOps cluster when it is available, while verifying its state and managing AWS costs.

---

## 7. Personal Reflection

Today's session was valuable because I developed a deeper understanding of how Kubernetes components work together.

I am learning to ask not only what a component does, but also why it exists, what it interacts with, and where it fits into the larger system.

My approach is to understand the fundamentals, connect them to practical experience, and build on them incrementally rather than rushing through topics.

**Today's key lesson:** Understanding the relationship between components is more valuable than memorizing each component in isolation.

I do not need to master all of Kubernetes in one session. I can let today's concepts settle and continue building my understanding in the next session.
