# Kubernetes Cloud Native Tools

## 1. What each tool does

| Tool          | Purpose                    | Demo                      |
| ------------- | -------------------------- | ------------------------- |
| **Helm**      | Kubernetes package manager | Deploy Nginx app          |
| **Argo CD**   | GitOps Continuous Delivery | Sync Helm app from GitHub |
| **Trivy**     | Security scanner           | Scan container image      |
| **KEDA**      | Pod autoscaling            | Scale app Pods            |
| **Karpenter** | Node autoscaling           | Add EC2 worker nodes      |

Helm packages and installs Kubernetes applications as charts. ([Helm][2]) KEDA monitors a trigger and works with Kubernetes autoscaling to scale Deployments or StatefulSets. ([KEDA][3])

## 2. Architecture

```text
                       DEVELOPER
                           │
                    git push / change
                           │
                           ▼
                 ┌──────────────────┐
                 │     GitHub       │
                 │   Helm Chart     │
                 └────────┬─────────┘
                          │
                          │ watches Git
                          ▼
                 ┌──────────────────┐
                 │     Argo CD      │
                 │      GitOps      │
                 └────────┬─────────┘
                          │
                          │ Helm Chart
                          ▼
┌─────────────────────────────────────────────────────────┐
│                     Amazon EKS                          │
│                                                        │
│              ┌────────────────────┐                    │
│              │   Nginx Deployment │                    │
│              └─────────┬──────────┘                    │
│                        │                               │
│                  ┌─────┴─────┐                         │
│                  │           │                         │
│                Pod 1       Pod 2                       │
│                                                        │
│         KEDA ───────────────► Scale Pods               │
│                                                        │
│         Karpenter ──────────► Add / Remove Nodes       │
│                                                        │
└─────────────────────────────────────────────────────────┘

                 Trivy
                   │
                   └────► Scan nginx image
                          for vulnerabilities
```

The main teaching point is:

```text
Helm       → Package application
Argo CD    → Deploy/sync application
Trivy      → Scan application image
KEDA       → Scale Pods
Karpenter  → Scale Nodes
```

---

# 3. Prerequisites

```bash
aws --version
kubectl version --client
eksctl version
helm version
docker --version
trivy --version
```

Verify AWS:

```bash
aws sts get-caller-identity
```

Verify Kubernetes later with:

```bash
kubectl get nodes
```

---

# 4. Create EKS Cluster

For a classroom demo, assume an EKS cluster already exists.

Example:

```bash
eksctl create cluster \
  --name k8s-tools-demo \
  --region us-east-1 \
  --nodes 2
```

Configure `kubectl`:

```bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name k8s-tools-demo
```

Check:

```bash
kubectl get nodes
```

---

# 5. Helm Demo

### Definition

> Helm is the package manager for Kubernetes.

Create a chart:

```bash
helm create webapp
```

Structure:

```text
webapp/
├── Chart.yaml
├── values.yaml
└── templates/
```

Edit:

```bash
nano webapp/values.yaml
```

Keep the important values simple:

```yaml
replicaCount: 1

image:
  repository: nginx
  tag: latest

service:
  type: LoadBalancer
  port: 80
```

Install:

```bash
helm install webapp ./webapp
```

Check:

```bash
helm list

kubectl get pods

kubectl get svc
```

Upgrade:

```bash
helm upgrade webapp ./webapp
```

Rollback:

```bash
helm rollback webapp 1
```

Helm gives you reusable Kubernetes packaging plus install, upgrade and rollback workflows. ([Helm][2])

---

# 6. Trivy Demo

### Definition

> Trivy scans containers and Kubernetes resources for security problems.

Scan Nginx:

```bash
trivy image nginx:latest
```

Only HIGH and CRITICAL:

```bash
trivy image \
  --severity HIGH,CRITICAL \
  nginx:latest
```

Students should understand:

```text
Container Image
      │
      ▼
    Trivy
      │
      ├── Vulnerabilities
      ├── OS Packages
      └── Application Dependencies
```

This can happen **before deployment**:

```text
Build Image
    │
    ▼
Trivy Scan
    │
    ▼
Push Image
    │
    ▼
Deploy Kubernetes
```

---

# 7. Install KEDA

Install with Helm:

```bash
helm repo add kedacore https://kedacore.github.io/charts

helm repo update
```

```bash
helm install keda kedacore/keda \
  --namespace keda \
  --create-namespace
```

Check:

```bash
kubectl get pods -n keda
```

KEDA uses a `ScaledObject` to identify the workload and scaling trigger. ([KEDA][4])

---

# 8. Very Basic KEDA Demo

For the simplest classroom demonstration, use the CPU scaler.

First add CPU requests to the Deployment.

```bash
kubectl edit deployment webapp
```

Conceptually:

```yaml
resources:
  requests:
    cpu: 100m
  limits:
    cpu: 500m
```

Create:

```bash
nano keda.yaml
```

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject

metadata:
  name: webapp-scaler

spec:
  scaleTargetRef:
    name: webapp

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
kubectl apply -f keda.yaml
```

Check:

```bash
kubectl get scaledobject
```

```bash
kubectl get hpa
```

Watch Pods:

```bash
kubectl get pods -w
```

Concept:

```text
High CPU
   │
   ▼
 KEDA
   │
   ▼
 Kubernetes HPA
   │
   ▼
1 Pod → 2 Pods → 3 Pods → 5 Pods
```

For real projects, the bigger value of KEDA is event-driven scaling from systems such as queues and messaging platforms rather than only CPU. ([KEDA][3])

---

# 9. Argo CD Installation

Create namespace:

```bash
kubectl create namespace argocd
```

Install:

```bash
kubectl apply \
  -n argocd \
  --server-side \
  --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

This matches the current Argo CD getting-started installation method. ([Argo CD][5])

Check:

```bash
kubectl get pods -n argocd
```

Access UI:

```bash
kubectl port-forward \
  svc/argocd-server \
  -n argocd \
  8080:443
```

Open:

```text
https://localhost:8080
```

Get password:

```bash
argocd admin initial-password -n argocd
```

Username:

```text
admin
```

Argo CD supports both CLI/UI workflows and declarative Kubernetes `Application` resources. ([Argo CD][5])

---

# 10. GitHub Repository

Keep this structure:

```text
k8s-tools-demo/
│
└── webapp/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
```

Push:

```bash
git init

git add .

git commit -m "Add Helm application"
```

```bash
git branch -M main
```

```bash
git remote add origin \
https://github.com/YOUR-USERNAME/k8s-tools-demo.git
```

```bash
git push -u origin main
```

---

# 11. Connect Argo CD

Create:

```bash
nano application.yaml
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: webapp
  namespace: argocd

spec:
  project: default

  source:
    repoURL: https://github.com/YOUR-USERNAME/k8s-tools-demo.git
    targetRevision: main
    path: webapp

  destination:
    server: https://kubernetes.default.svc
    namespace: default

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Apply:

```bash
kubectl apply -f application.yaml
```

Check:

```bash
kubectl get applications -n argocd
```

Now the flow becomes:

```text
Developer
   │
   │ git push
   ▼
GitHub
   │
   ▼
Argo CD
   │
   ▼
Helm Chart
   │
   ▼
Kubernetes
```

---

# 12. Test Argo CD

Change:

```yaml
replicaCount: 2
```

Commit:

```bash
git add .

git commit -m "Scale application"

git push
```

Argo CD detects:

```text
Git desired state = 2 Pods
         │
         ▼
Current cluster = 1 Pod
         │
         ▼
      OutOfSync
         │
         ▼
        Sync
         │
         ▼
       2 Pods
```

Watch:

```bash
kubectl get pods -w
```

---

# 13. Karpenter Demo

### Definition

> Karpenter provides Kubernetes node capacity when Pods cannot be scheduled.

Karpenter specifically watches cluster scheduling requirements and can ask the cloud provider for new compute capacity. ([Karpenter][6])

The simplest explanation:

```text
KEDA
 │
 │ creates more Pods
 ▼
More Pods
 │
 │ insufficient CPU/RAM
 ▼
Pending Pods
 │
 ▼
Karpenter
 │
 ▼
AWS EC2
 │
 ▼
New Worker Node
 │
 ▼
Pending Pods Scheduled
```

### Important

Unlike Helm, Argo CD, KEDA and Trivy, **Karpenter needs AWS infrastructure/IAM configuration**. It cannot be demonstrated correctly by simply applying one YAML file. The official EKS getting-started flow configures the AWS permissions and cluster integration before the controller provisions nodes. ([Karpenter][6])

Assuming Karpenter is already installed, check it with:

```bash
kubectl get pods -n kube-system | grep karpenter
```

Check objects:

```bash
kubectl get nodepool
```

```bash
kubectl get ec2nodeclass
```

---

# 14. Trigger Karpenter

Check existing nodes:

```bash
kubectl get nodes
```

Scale application aggressively:

```bash
kubectl scale deployment webapp \
  --replicas=20
```

Watch:

```bash
kubectl get pods -w
```

Some Pods may become:

```text
Pending
```

In another terminal:

```bash
kubectl get nodes -w
```

Flow:

```text
2 Nodes
   │
   ▼
20 Pods requested
   │
   ▼
No Capacity
   │
   ▼
Pods = Pending
   │
   ▼
Karpenter detects requirement
   │
   ▼
EC2 instance launched
   │
   ▼
New Kubernetes Node
   │
   ▼
Pods Running
```

Check:

```bash
kubectl get pods -o wide
```

---

# 15. Complete Demo Architecture

```text
                         Developer
                            │
                         git push
                            │
                            ▼
                    ┌──────────────┐
                    │    GitHub    │
                    │  Helm Chart  │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │   Argo CD    │
                    │    GitOps    │
                    └──────┬───────┘
                           │
                    Helm deployment
                           │
                           ▼
          ┌──────────────────────────────────┐
          │              EKS                 │
          │                                  │
          │       Nginx Deployment           │
          │             │                    │
          │      ┌──────┼──────┐             │
          │      ▼      ▼      ▼             │
          │     Pod    Pod    Pod            │
          │                                  │
          │  KEDA ─────► Pod Scaling         │
          │                                  │
          │  Karpenter ─► Node Scaling       │
          │                                  │
          └──────────────────────────────────┘
                           ▲
                           │
                         AWS EC2

Developer / CI
      │
      ▼
 Container Image
      │
      ▼
    Trivy
      │
      ▼
Security Scan
```

---

# 16. How all five work together

```text
1. Developer creates application
              │
              ▼
2. Helm packages Kubernetes configuration
              │
              ▼
3. Trivy scans container image
              │
              ▼
4. Code + Helm chart pushed to GitHub
              │
              ▼
5. Argo CD detects Git changes
              │
              ▼
6. Argo CD deploys Helm application
              │
              ▼
7. KEDA monitors workload
              │
              ▼
8. KEDA increases Pods
              │
              ▼
9. Cluster runs out of Node capacity
              │
              ▼
10. Karpenter creates EC2 worker node
              │
              ▼
11. Kubernetes schedules Pods
```

## 17. Commands students should remember

```bash
# Kubernetes
kubectl get nodes
kubectl get pods
kubectl get svc
kubectl get pods -w


# Helm
helm create webapp
helm install webapp ./webapp
helm list
helm upgrade webapp ./webapp
helm rollback webapp 1


# Trivy
trivy image nginx:latest


# KEDA
kubectl get scaledobject
kubectl get hpa


# Argo CD
kubectl get pods -n argocd
kubectl get applications -n argocd


# Karpenter
kubectl get nodepool
kubectl get ec2nodeclass
kubectl get nodes -w
```

## 18. One-line explanations for live delivery

```text
Helm
Package Manager for Kubernetes.

Argo CD
GitOps Continuous Delivery tool for Kubernetes.

Trivy
Security scanner for container images and Kubernetes workloads.

KEDA
Event-driven Pod autoscaler.

Karpenter
Kubernetes Node autoscaler / node provisioning system.
```

The most useful final comparison for students is:

```text
                 WHAT IS BEING MANAGED?

Helm        → Application Packaging
Argo CD     → Application Deployment
Trivy       → Application Security
KEDA        → Pods
Karpenter   → Worker Nodes
```

And the key relationship is:

```text
Traffic / Events Increase
          │
          ▼
        KEDA
          │
          ▼
      More Pods
          │
          ▼
   Need More Capacity
          │
          ▼
      Karpenter
          │
          ▼
      More Nodes
```

### References

[1]: https://karpenter.sh/docs/getting-started/?utm_source=chatgpt.com "Getting Started | Karpenter"
[2]: https://helm.sh/docs/v3/intro/quickstart/?utm_source=chatgpt.com "Quickstart Guide | Helm"
[3]: https://keda.sh/docs/2.20/concepts/scaling-deployments/?utm_source=chatgpt.com "Scaling Deployments, StatefulSets & Custom Resources | KEDA"
[4]: https://keda.sh/docs/2.21/reference/scaledobject-spec/?utm_source=chatgpt.com "ScaledObject specification | KEDA"
[5]: https://argo-cd.readthedocs.io/en/latest/getting_started/?utm_source=chatgpt.com "Getting Started - Argo CD - Declarative GitOps CD for Kubernetes"
[6]: https://karpenter.sh/v1.0/getting-started/getting-started-with-karpenter/?utm_source=chatgpt.com "Getting Started with Karpenter | Karpenter"
