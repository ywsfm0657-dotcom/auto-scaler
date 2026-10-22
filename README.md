# Kubernetes Autoscaling Under Real Load

## 📌 Project Overview

This project demonstrates how to configure Kubernetes Horizontal Pod Autoscaling (HPA) to automatically scale application replicas based on CPU utilization under load.

The project uses Nginx as the application, Kubernetes Deployments to manage replicas, a Service to distribute traffic, and Metrics Server to provide resource usage metrics.

A load generator sends repeated HTTP requests to the application so that scaling behavior can be observed and tested.

The goal is to understand how Kubernetes responds to changing workloads, how resource requests and limits affect scaling, and why scaling up and scaling down take time.

## 🎯 Project Objectives

- Deploy an application using Kubernetes.
- Configure CPU and memory requests and limits.
- Install and enable Kubernetes Metrics Server.
- Configure Horizontal Pod Autoscaling.
- Generate traffic to test application behavior under load.
- Monitor CPU utilization and replica counts.
- Observe scale-up and scale-down behavior.
- Understand autoscaling delays and cluster resource constraints.
- Practice troubleshooting with `kubectl`.

## 🏗️ Architecture

```text
                  Load Generator
                        |
                        | HTTP Requests
                        v
                ┌─────────────────┐
                │ Nginx Service   │
                └────────┬────────┘
                         |
              ┌──────────┼──────────┐
              v          v          v
          ┌────────┐ ┌────────┐ ┌────────┐
          │ Pod 1  │ │ Pod 2  │ │ Pod 3  │
          │ Nginx  │ │ Nginx  │ │ Nginx  │
          └────────┘ └────────┘ └────────┘
                         ^
                         |
                ┌─────────────────┐
                │ HPA Controller  │
                │ CPU Target: 50% │
                └─────────────────┘
                         ^
                         |
                ┌─────────────────┐
                │ Metrics Server  │
                └─────────────────┘
```

*The number of Pods changes dynamically according to observed CPU utilization, subject to the configured minimum and maximum replica counts.*

## 🛠️ Technologies Used

- Ubuntu Linux
- Kubernetes
- Minikube
- Docker
- Nginx
- Metrics Server
- Horizontal Pod Autoscaler (HPA)
- YAML
- kubectl
- Git and GitHub

## 📋 Prerequisites

Install the following tools before starting:

- Docker
- Minikube
- kubectl

Verify the installations:

```bash
docker --version
minikube version
kubectl version --client
```

Start Minikube:

```bash
minikube start
```

Verify the cluster:

```bash
kubectl get nodes
```

The Minikube node should be in the `Ready` state.

## 📂 Project Structure

```text
k8s-autoscaling/
├── README.md
├── deployment.yaml
├── service.yaml
└── hpa.yaml
```

## 🚀 Implementation

### Step 1: Enable Metrics Server

Metrics Server collects resource usage metrics that Kubernetes can use for autoscaling.

Enable the Minikube addon:

```bash
minikube addons enable metrics-server
```

Check the Metrics Server Pod:

```bash
kubectl get pods -n kube-system
```

Verify node metrics:

```bash
kubectl top nodes
```

Verify Pod metrics:

```bash
kubectl top pods
```

Wait for metrics to become available if they are not immediately reported.

### Step 2: Deploy the Application

Create `deployment.yaml` with the following configuration:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx-app

spec:
  replicas: 2

  selector:
    matchLabels:
      app: nginx-app

  template:
    metadata:
      labels:
        app: nginx-app

    spec:
      containers:
        - name: nginx
          image: nginx:1.28

          ports:
            - containerPort: 80

          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"

            limits:
              cpu: "500m"
              memory: "128Mi"
```

Apply the Deployment:

```bash
kubectl apply -f deployment.yaml
```

Check the Pods:

```bash
kubectl get pods
```

**Resource configuration:**

- CPU request: 100 millicores.
- Memory request: 64 MiB.
- CPU limit: 500 millicores.
- Memory limit: 128 MiB.

CPU requests are particularly important because the HPA calculates CPU utilization relative to the requested CPU.

### Step 3: Create a Kubernetes Service

Create `service.yaml`:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: nginx-service

spec:
  type: ClusterIP

  selector:
    app: nginx-app

  ports:
    - port: 80
      targetPort: 80
```

Apply the Service:

```bash
kubectl apply -f service.yaml
```

Verify:

```bash
kubectl get services
```

The Service provides a stable internal endpoint for accessing the Nginx Pods.

### Step 4: Configure Horizontal Pod Autoscaling

Create `hpa.yaml`:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler

metadata:
  name: nginx-hpa

spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-app

  minReplicas: 2
  maxReplicas: 10

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50

  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
```

Apply the configuration:

```bash
kubectl apply -f hpa.yaml
```

Check the HPA:

```bash
kubectl get hpa
```

Inspect its status:

```bash
kubectl describe hpa nginx-hpa
```

**HPA configuration:**

| Setting | Description |
|---|---|
| `scaleTargetRef` | Identifies the Deployment to scale. |
| `minReplicas: 2` | Maintains at least two replicas. |
| `maxReplicas: 10` | Allows up to ten replicas. |
| `averageUtilization: 50` | Targets average CPU utilization of 50% of the requested CPU. |
| `stabilizationWindowSeconds: 300` | Helps prevent premature scale-down decisions by considering recent recommendations over five minutes. |

### Step 5: Generate Load

Create a temporary load-generator Pod:

```bash
kubectl run load-generator \
  --image=busybox:1.36 \
  --restart=Never \
  -- /bin/sh -c \
  "while true; do wget -q -O- http://nginx-service; done"
```

This Pod repeatedly sends HTTP requests to the Nginx Service.

**Note:** Nginx serves static content efficiently, so this traffic may not generate enough CPU utilization to trigger autoscaling. Successful HTTP requests do not necessarily mean the CPU target has been reached.

For a meaningful autoscaling test, use a CPU-intensive application or a load-testing tool that generates sufficient processing demand.

### Step 6: Monitor Autoscaling

Watch the HPA:

```bash
kubectl get hpa -w
```

Watch the Pods:

```bash
kubectl get pods -w
```

Monitor CPU and memory:

```bash
kubectl top pods
```

Inspect the HPA configuration and events:

```bash
kubectl describe hpa nginx-hpa
```

When the observed CPU utilization exceeds the target, Kubernetes may increase the replica count.

When CPU utilization falls, Kubernetes can reduce the replica count, subject to the minimum replica count and scale-down behavior.

The actual number of replicas depends on observed metrics, workload intensity, scheduling capacity, and application startup time.

### Step 7: Test Scale-Down

Stop the load generator:

```bash
kubectl delete pod load-generator
```

Monitor the HPA:

```bash
kubectl get hpa -w
```

Monitor the Deployment:

```bash
kubectl get deployment nginx-app -w
```

Observe how the replica count changes as CPU utilization decreases.

The Deployment should not scale below two replicas while this HPA configuration is active.

## 🔍 Troubleshooting

| Problem | Diagnostic command |
|---|---|
| Metrics unavailable | `kubectl top pods` |
| HPA shows `<unknown>` | `kubectl describe hpa nginx-hpa` |
| Pods are not starting | `kubectl describe pods` |
| CPU utilization remains low | `kubectl top pods` |
| Load generator fails | `kubectl logs load-generator` |
| Pods remain Pending | `kubectl describe pod <pod-name>` |
| Deployment status | `kubectl get deployment nginx-app` |
| Cluster events | `kubectl get events --sort-by=.metadata.creationTimestamp` |

## 🧠 Key Concepts Learned

### 1. Resource Requests and Limits

Resource requests influence scheduling and CPU utilization calculations. Limits constrain container resource consumption.

### 2. Metrics Server

Metrics Server provides resource usage metrics for Kubernetes components such as HPA.

### 3. Horizontal Pod Autoscaler

HPA automatically adjusts the number of replicas according to observed metrics and configured targets.

### 4. Load Testing

Generating workload helps demonstrate how an application responds to increased demand.

### 5. Scaling Delays

Metrics collection, HPA evaluation, scheduling, image downloads, and application startup can all contribute to scaling delays.

### 6. Scale-Down Stabilization

Stabilization helps prevent rapid replica-count fluctuations when resource utilization changes.

### 7. Cluster Capacity

HPA can request more Pods, but additional Pods may remain Pending if the cluster lacks sufficient resources.

## 🚧 Future Improvements

- Replace Nginx with a CPU-intensive application to produce measurable scaling behavior.
- Test with dedicated load-testing tools such as k6 or hey.
- Configure readiness and liveness probes.
- Add monitoring dashboards using Prometheus and Grafana.
- Configure alerts for high CPU utilization and unavailable replicas.
- Explore Vertical Pod Autoscaler (VPA).
- Explore Cluster Autoscaler on a supported cluster.
- Add the project to a CI/CD pipeline.
- Record scaling results and compare scale-up and scale-down times.

## 👨‍💻 Author

**Mahmoud Youssef**

Aspiring DevOps Engineer

**Areas of Interest:**
Linux | Docker | Kubernetes | CI/CD | Terraform | Ansible | AWS

---

⭐ This project is part of my hands-on DevOps learning journey, focusing on Kubernetes autoscaling, resource management, load testing, and workload reliability.
