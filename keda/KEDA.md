## KEDA Basics + Very Basic Hands-on Practice

KEDA (Kubernetes Event-driven Autoscaling) extends Kubernetes autoscaling so workloads can scale based on events and external metrics such as queues, Kafka, Prometheus, cron schedules, CPU, and memory. KEDA monitors the trigger and works with Kubernetes HPA to adjust replicas. ([KEDA][1])

### 1. Architecture

```text
        Event / Metric Source
                |
                v
        +----------------+
        |      KEDA      |
        |    Operator    |
        +-------+--------+
                |
                | Metrics
                v
        +----------------+
        |      HPA       |
        +-------+--------+
                |
                v
        +----------------+
        |   Deployment   |
        +----------------+
          |     |     |
        Pod   Pod   Pod
```

Simple flow:

```text
Metric increases
      ↓
KEDA detects it
      ↓
HPA receives metric
      ↓
Deployment scales out
      ↓
More Pods

Metric decreases
      ↓
Deployment scales in
```

For deployment-style workloads, KEDA monitors the event source and supplies metrics that HPA uses for scaling. ([KEDA][1])

### 2. Important KEDA Objects

The main object you'll use is a `ScaledObject`:

```text
Deployment
    ↑
ScaledObject
    ↑
Trigger
```

A basic `ScaledObject` contains:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: my-scaler
spec:

  scaleTargetRef:
    name: my-app

  minReplicaCount: 1
  maxReplicaCount: 5

  triggers:
    - type: cpu
      metricType: Utilization
      metadata:
        value: "50"
```

`scaleTargetRef` identifies the workload, while `triggers` defines what metric/event drives scaling. KEDA also supports `ScaledJob` for job-style workloads. ([KEDA][2])

---

# Practice 1 — Install KEDA with Helm

This works well on an existing Kubernetes cluster such as EKS.

Check your cluster:

```bash
kubectl get nodes

kubectl get pods
```

Check Helm:

```bash
helm version
```

Add the official KEDA Helm repository:

```bash
helm repo add kedacore https://kedacore.github.io/charts

helm repo update
```

Install:

```bash
helm install keda kedacore/keda \
  --namespace keda \
  --create-namespace
```

Verify:

```bash
kubectl get ns

kubectl get pods -n keda
```

You should see KEDA components including the operator and metrics API server. These Helm installation commands follow KEDA's deployment documentation. ([KEDA][3])

---

# Practice 2 — Deploy a Simple Application

Create:

```bash
nano deployment.yaml
```

Add:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app

spec:
  replicas: 1

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
          image: nginx:latest

          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"

            limits:
              cpu: "500m"
              memory: "128Mi"
```

Deploy:

```bash
kubectl apply -f deployment.yaml
```

Verify:

```bash
kubectl get deployments

kubectl get pods
```

The CPU `requests` are important for a utilization-based CPU scaler. KEDA's CPU scaler also requires Kubernetes Metrics Server. ([KEDA][4])

---

# Practice 3 — Check Metrics Server

Run:

```bash
kubectl top nodes

kubectl top pods
```

If these commands return CPU and memory values, Metrics Server is working.

This is required for the CPU-based lab because KEDA's CPU and memory scalers use Kubernetes Metrics Server. On some environments, including EKS, it may not be installed by default. ([KEDA][4])

---

# Practice 4 — Create KEDA CPU Scaler

Create:

```bash
nano scaledobject.yaml
```

Add:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject

metadata:
  name: nginx-cpu-scaler

spec:

  scaleTargetRef:
    name: nginx-app

  minReplicaCount: 1
  maxReplicaCount: 5

  triggers:

    - type: cpu
      metricType: Utilization

      metadata:
        value: "50"
```

Apply:

```bash
kubectl apply -f scaledobject.yaml
```

Verify:

```bash
kubectl get scaledobject

kubectl describe scaledobject nginx-cpu-scaler
```

Check HPA:

```bash
kubectl get hpa
```

You should notice that KEDA has created an HPA for the `ScaledObject`.

```text
ScaledObject
     |
     v
    KEDA
     |
     v
    HPA
     |
     v
nginx-app
```

Here, the target is approximately **50% CPU utilization**, with replicas constrained between **1 and 5**. ([KEDA][4])

---

# Practice 5 — Generate CPU Load

First check:

```bash
kubectl get pods
```

Open a shell inside the Nginx pod:

```bash
kubectl exec -it <pod-name> -- /bin/sh
```

Generate CPU activity:

```bash
yes > /dev/null &
```

You can run it multiple times:

```bash
yes > /dev/null &
yes > /dev/null &
yes > /dev/null &
```

Exit:

```bash
exit
```

Watch scaling:

```bash
kubectl get pods -w
```

In another terminal:

```bash
kubectl get hpa -w
```

Also check:

```bash
kubectl top pods
```

Expected concept:

```text
CPU < 50%
   ↓
1 Pod

CPU > 50%
   ↓
KEDA/HPA
   ↓
2 Pods
   ↓
3 Pods
   ↓
up to max 5 Pods
```

The exact replica count depends on observed utilization rather than simply adding one pod each time.

---

## Important KEDA Parameters

```yaml
spec:

  pollingInterval: 30

  cooldownPeriod: 300

  minReplicaCount: 1

  maxReplicaCount: 5
```

`pollingInterval` controls how frequently KEDA checks applicable triggers and defaults to 30 seconds. `cooldownPeriod` defaults to 300 seconds and applies to KEDA's scale-to-zero behavior; scaling between 1 and N is primarily handled by HPA. ([KEDA][2])

One important detail for this CPU-only example: **CPU alone cannot provide true scale-to-zero**. KEDA documents that CPU/memory need another non-CPU/non-memory trigger for scale-to-zero. ([KEDA][4])

---

## Practice 6 — Very Simple Cron Scaling

Cron is useful for demonstrating event-driven scaling without generating CPU load.

Example use case:

```text
Office Hours
6 AM → 8 PM
      ↓
Keep application running

Night
8 PM → 6 AM
      ↓
Scale down
```

Create:

```bash
nano cron-scaler.yaml
```

Example:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject

metadata:
  name: nginx-cron-scaler

spec:

  scaleTargetRef:
    name: nginx-app

  minReplicaCount: 0
  maxReplicaCount: 5

  triggers:

    - type: cron

      metadata:
        timezone: Asia/Kolkata
        start: 0 6 * * *
        end: 0 20 * * *
        desiredReplicas: "2"
```

KEDA's cron scaler supports timezone-aware start/end schedules and a desired replica count during that interval. ([KEDA][5])

For practice, change the start/end times to a window around your current time so you can observe the change immediately.

---

## Useful Troubleshooting Commands

```bash
kubectl get scaledobject

kubectl describe scaledobject nginx-cpu-scaler

kubectl get hpa

kubectl describe hpa

kubectl get pods

kubectl top pods

kubectl get pods -n keda

kubectl logs -n keda deployment/keda-operator
```

Delete the practice:

```bash
kubectl delete -f scaledobject.yaml

kubectl delete -f deployment.yaml
```

Remove KEDA:

```bash
helm uninstall keda -n keda

kubectl delete namespace keda
```

[KEDA documentation](https://keda.sh/docs/2.20/?utm_source=chatgpt.com)

### Points to remember for students

**KEDA = event-driven autoscaling.** Kubernetes HPA commonly handles resource/custom metrics; KEDA adds integrations with external event sources and exposes the required metrics to Kubernetes. A `ScaledObject` is used for deployment-style workloads, while `ScaledJob` is for job-style processing. CPU is a good first lab, but a queue-based scaler such as SQS is a better demonstration of KEDA's core event-driven value. ([KEDA][1])

For your **AWS/EKS training**, the natural next practical is:

```text
User
 ↓
Amazon SQS
 ↓
Messages waiting
 ↓
KEDA SQS Scaler
 ↓
HPA
 ↓
EKS Deployment
 ↓
Worker Pods
 ↓
Process messages

0 messages → 0 Pods
100 messages → Multiple Pods
```

That lab demonstrates **EKS + SQS + KEDA + scale-to-zero** much better than CPU scaling.

[1]: https://keda.sh/docs/2.20/concepts/scaling-deployments/?utm_source=chatgpt.com "Scaling Deployments, StatefulSets & Custom Resources | KEDA"
[2]: https://keda.sh/docs/2.21/reference/scaledobject-spec/?utm_source=chatgpt.com "ScaledObject specification | KEDA"
[3]: https://keda.sh/docs/2.21/deploy/?utm_source=chatgpt.com "Deploying KEDA | KEDA"
[4]: https://keda.sh/docs/2.20/scalers/cpu/?utm_source=chatgpt.com "CPU | KEDA"
[5]: https://keda.sh/docs/2.20/scalers/cron/?utm_source=chatgpt.com "Cron | KEDA"
