# Kubernetes Research and Transition Documentation

## 1. Introduction to Kubernetes (K8s)

Kubernetes (K8s) is a container orchestration platform designed for managing containerized applications at scale. While tools such as Docker and Podman are commonly used for container creation and runtime management, Kubernetes focuses on clustering, orchestration, scalability, and automated deployment.

Originally developed by Google, Kubernetes is widely used in cloud-native development to efficiently deploy, manage, and scale applications across distributed environments.

---

## 2. Cluster Architecture

The Metropolia Garage environment utilizes a distributed Kubernetes cluster consisting of:

* A centralized main server
* Multiple edge computing devices

Kubernetes provides a standardized method for running workloads across all devices connected to the cluster.

### Control Plane

The Control Plane acts as the central management interface of the cluster. It is responsible for:

* Scheduling workloads
* Managing cluster state
* Monitoring resources
* Handling deployments and scaling

Users interact with the cluster primarily through the Control Plane using tools such as:

```bash
kubectl
```

### Nodes

Nodes are the worker machines within the cluster. These can include:

* Centralized servers
* Edge devices
* Embedded computers

Kubernetes distributes workloads automatically across available nodes.

---

## 3. Core Concepts for Migration

### Pods

Pods are the smallest deployable units in Kubernetes.

A pod can contain:

* One or more containers
* Shared networking
* Shared storage volumes
* Runtime configurations

Pods encapsulate application components and their required resources.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  containers:
    - name: nginx
      image: nginx
```

---

### Namespaces

Namespaces provide logical separation between services and environments inside a cluster.

Benefits include:

* Resource isolation
* Service separation
* Access control
* Organizational structure

By default, applications in different namespaces cannot directly access each other unless networking permissions are explicitly configured.

Example:

```bash
kubectl create namespace research
```

---

### Deployments

Deployments manage application lifecycle operations such as:

* Rolling updates
* Scaling
* Replica management
* Restart policies

Deployments in Kubernetes are conceptually similar to Docker Compose services.

Applications are commonly deployed using:

```bash
kubectl apply -f deployment.yaml
```

Example deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
```

---

## 4. Storage and Networking

### Storage Classes

The current environment utilizes the `local-path` storage class for persistent storage.

Characteristics:

* Simple local node storage
* Suitable for development and lightweight workloads
* Storage tied to individual nodes

Future migration plans may include the use of **Longhorn** for distributed storage across cluster nodes.

Advantages of Longhorn:

* Cross-node persistence
* Replication
* Improved fault tolerance
* Easier storage scalability

---

### Internal Networking

Kubernetes provides built-in networking between pods and services.

Features include:

* Automatic DNS resolution
* Internal service discovery
* Flat networking model
* Pod-to-pod communication

Containers inside the cluster can communicate without additional manual configuration.

---

### Ingress

To expose services outside the Kubernetes cluster, an Ingress layer is required.

Common Ingress solutions include:

* Traefik
* NGINX Ingress Controller
* HAProxy

Ingress responsibilities:

* HTTP/HTTPS routing
* TLS termination
* Domain-based routing
* External access management

Example architecture:

```text
Internet
    ↓
Ingress Controller (Traefik)
    ↓
Kubernetes Services
    ↓
Pods
```

---

## 5. Podman to Kubernetes Translation

Podman includes functionality for generating Kubernetes-compatible YAML files.

Example:

```bash
podman generate kube <container-name>
```

However, the translation process is not fully one-to-one.

### Manual Modification Requirements

Generated YAML files often require manual adjustment for:

* Namespace compatibility
* Internal networking
* Persistent storage definitions
* Security policies
* Resource limits
* Ingress integration

---

### Template Standardization

Following official internal templates is essential for:

* Consistency
* Maintainability
* Security compliance
* Easier troubleshooting
* Standardized deployment practices

Custom configurations should align with established lab deployment conventions.

---

## 6. Development Environment

Local Kubernetes research and testing are performed before deployment to the production cluster.

Common tools include:

### Minikube

Minikube provides a lightweight local Kubernetes environment for testing and development.

Features:

* Single-node Kubernetes cluster
* Local experimentation
* Dashboard support
* Easy setup

Example startup command:

```bash
minikube start
```

---

### kubectl

`kubectl` is the primary command-line interface used to interact with Kubernetes clusters.

Example commands:

```bash
kubectl get pods
kubectl get services
kubectl apply -f deployment.yaml
```

---

## Figure 1

![Minikube Dashboard](images/dashboard.png)

**Figure 1:** Successful initialization of the local Kubernetes dashboard for resource monitoring.

---

## 7. Summary

Kubernetes provides a scalable and standardized orchestration platform suitable for distributed environments such as the Metropolia Garage infrastructure.

Key migration considerations include:

* Understanding Kubernetes core architecture
* Proper namespace separation
* Standardized deployment templates
* Storage and networking configuration
* Ingress management
* Manual refinement of Podman-generated manifests

Local testing using Minikube and kubectl is essential before deploying applications into the production cluster.
