# Argo CD — Very Basic Practice

**Argo CD** is a GitOps Continuous Delivery tool for Kubernetes.

**Basic idea:**

```text
Developer
   │
   │ git push
   ▼
GitHub Repository
   │
   │ Argo CD watches Git
   ▼
Argo CD
   │
   │ Sync
   ▼
Kubernetes Cluster
   │
   ▼
Application / Pods
```

## 1. Prerequisites

```bash
kubectl version --client
kubectl get nodes
```

You need:

```text
Kubernetes Cluster
kubectl
GitHub Repository
Argo CD
```

## 2. Install Argo CD

Create namespace:

```bash
kubectl create namespace argocd
```

Install Argo CD:

```bash
kubectl apply -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Check:

```bash
kubectl get pods -n argocd
```

Wait until pods are:

```text
Running
```

## 3. Access Argo CD UI

For basic local practice:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open:

```text
https://localhost:8080
```

Username:

```text
admin
```

## 4. Get Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d
echo
```

Login using:

```text
Username: admin
Password: <generated-password>
```

## 5. Create Simple Kubernetes Files

Create `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
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
        image: nginx:latest
        ports:
        - containerPort: 80
```

Create `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

## 6. Push Files to GitHub

Example repository structure:

```text
argocd-basic-demo/
│
├── deployment.yaml
└── service.yaml
```

Commands:

```bash
git init
git add .
git commit -m "Add Kubernetes manifests"
git branch -M main
git remote add origin <YOUR-GITHUB-REPO>
git push -u origin main
```

## 7. Create Application in Argo CD UI

Go to:

```text
Applications
   ↓
NEW APP
```

Enter:

```text
Application Name: nginx-app
Project: default

Repository URL: <YOUR-GITHUB-REPO>
Revision: main
Path: .

Cluster:
https://kubernetes.default.svc

Namespace:
default
```

Click:

```text
CREATE
```

## 8. Sync Application

Initially you may see:

```text
OutOfSync
```

Select:

```text
SYNC
   ↓
SYNCHRONIZE
```

After deployment:

```text
Synced
Healthy
```

Verify from terminal:

```bash
kubectl get deployments
kubectl get pods
kubectl get svc
```

## 9. Test GitOps

Change:

```yaml
replicas: 2
```

to:

```yaml
replicas: 3
```

Push:

```bash
git add .
git commit -m "Scale nginx to 3 replicas"
git push
```

Argo CD detects:

```text
Git changed
     ↓
OutOfSync
     ↓
Sync
     ↓
Kubernetes updated
```

Verify:

```bash
kubectl get pods
```

You should now see **3 nginx pods**.

## Points to Remember

* **Git is the source of truth.**
* Argo CD continuously compares **Git desired state** with **Kubernetes actual state**.
* `OutOfSync` = Git and cluster differ.
* `Synced` = Git and cluster match.
* `Healthy` = Kubernetes resources are operating normally.
* **Manual Sync** is easiest for first practice.
* **Auto Sync** can deploy Git changes automatically.
* Avoid manually changing Kubernetes resources managed by Argo CD; make the desired change in Git.
* Argo CD handles **CD/deployment**, not typically the CI image-build process.

**One-line flow:**

```text
Code → Git Push → Argo CD Detects → Sync → Kubernetes Deploys
```

Official project: [Argo CD documentation](https://argo-cd.readthedocs.io/?utm_source=chatgpt.com)


# Argo CD: deploy from Git

Run commands from the repository root, after the Helm lesson.

## Install

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open https://localhost:8080 and use username `admin`. In another terminal,
get the initial password with `argocd admin initial-password -n argocd`.

## Connect the chart

Update `repoURL` in `argocd/application.yaml` if you use a fork. Commit and push
the chart to the `main` branch before applying; Argo CD reads the remote repo.

```bash
kubectl apply -f argocd/application.yaml
kubectl get applications -n argocd
```

Change `replicaCount` in `helm/webapp/values.yaml`, commit, and push. Watch Argo
CD sync the change. Use Git for changes once Argo CD manages the app.
Keep KEDA disabled for this fixed-replica lesson. If you already tried KEDA,
delete its ScaledObject, remove the optional patch settings below, and set
`autoscaling.enabled: false` in Git first.

## Optional: use KEDA together with Argo CD

The patch lets KEDA control replicas without Argo CD resetting them:

```bash
kubectl patch application webapp -n argocd --type merge --patch-file argocd/autoscaling-patch.yaml
```

Wait for sync, then follow the KEDA lesson, skipping its `helm upgrade` command.
To return to fixed replicas, delete the ScaledObject and remove the patch:

```bash
kubectl delete -f keda/scaledobject.yaml --ignore-not-found
kubectl patch application webapp -n argocd --type merge -p '{"spec":{"source":{"helm":{"parameters":null}},"ignoreDifferences":null,"syncPolicy":{"syncOptions":null}}}'
```

## Cleanup

```bash
kubectl delete -f argocd/application.yaml
```

The Application has no deletion finalizer, so the workload stays until you
run the Helm cleanup. If you deployed only through Argo CD, remove it with:

```bash
helm template webapp ./helm/webapp | kubectl delete -n default -f -
```
