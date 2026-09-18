## Trivy — Basic Practice Lab

[Trivy](https://trivy.dev/?utm_source=chatgpt.com) is an open-source security scanner commonly used with Docker, Kubernetes, CI/CD, and Infrastructure as Code. For a basic lab, focus on **container image scanning, filesystem scanning, and Kubernetes scanning**.

### 1. Points to Remember

* **Trivy = Security Scanner**
* Developed as part of the Aqua Security ecosystem.
* Finds **CVEs/vulnerabilities**, **misconfigurations**, **secrets**, and some **license issues**.
* Can scan:

  * Container images
  * Local files/directories
  * Git repositories
  * Kubernetes
  * IaC such as Terraform/Kubernetes manifests
* Commonly integrated into CI/CD pipelines to catch issues before deployment.
* Trivy uses vulnerability databases that are downloaded/updated automatically.

### 2. Architecture

```text
Developer
    |
    v
Application Code
    |
    +----------------------+
    |        TRIVY         |
    +----------------------+
       |       |       |
       v       v       v
     Image    Files    K8s
       |       |       |
       +-------+-------+
               |
               v
     Vulnerability /
     Security Report
```

### 3. Install Trivy on macOS

```bash
brew install trivy
```

Verify:

```bash
trivy --version
```

Update vulnerability database manually if required:

```bash
trivy image --download-db-only
```

---

## 4. Practice 1 — Scan a Docker Image

Pull an image:

```bash
docker pull nginx:latest
```

Scan:

```bash
trivy image nginx:latest
```

Trivy reports vulnerabilities with severity levels such as:

```text
UNKNOWN
LOW
MEDIUM
HIGH
CRITICAL
```

Show only HIGH and CRITICAL:

```bash
trivy image --severity HIGH,CRITICAL nginx:latest
```

Ignore vulnerabilities without an available fix:

```bash
trivy image --ignore-unfixed nginx:latest
```

---

## 5. Practice 2 — Scan Your Own Docker Image

Suppose your application contains:

```text
myapp/
├── app.py
├── requirements.txt
└── Dockerfile
```

Build:

```bash
cd myapp

docker build -t myapp:v1 .
```

Scan:

```bash
trivy image myapp:v1
```

Only important vulnerabilities:

```bash
trivy image \
  --severity HIGH,CRITICAL \
  myapp:v1
```

---

## 6. Practice 3 — Filesystem Scan

Scan the current project:

```bash
trivy fs .
```

Scan vulnerabilities:

```bash
trivy fs --scanners vuln .
```

Scan for exposed secrets:

```bash
trivy fs --scanners secret .
```

Scan both:

```bash
trivy fs --scanners vuln,secret .
```

This is useful **before creating the Docker image**.

```text
Source Code
    |
    | trivy fs .
    v
Security Check
    |
    v
docker build
```

---

## 7. Practice 4 — Scan Kubernetes YAML

Create:

```bash
nano deployment.yaml
```

Example:

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

Scan the manifest:

```bash
trivy config deployment.yaml
```

Or scan manifests in the current directory:

```bash
trivy config .
```

This checks for Kubernetes/IaC **misconfigurations** rather than just CVEs.

---

## 8. Practice 5 — Scan Kubernetes Cluster

Make sure the cluster is accessible:

```bash
kubectl get nodes
```

Then:

```bash
trivy k8s --report summary cluster
```

For a more comprehensive cluster scan:

```bash
trivy k8s --report all cluster
```

Scan a namespace:

```bash
trivy k8s --namespace default all
```

---

## 9. Save Report to File

Table format:

```bash
trivy image nginx:latest > trivy-report.txt
```

JSON:

```bash
trivy image \
  --format json \
  --output trivy-report.json \
  nginx:latest
```

SARIF:

```bash
trivy image \
  --format sarif \
  --output trivy-report.sarif \
  nginx:latest
```

---

## 10. Basic DevSecOps Flow

```text
Developer
   |
   v
Source Code
   |
   |----> trivy fs .
   |
   v
Docker Build
   |
   v
Container Image
   |
   |----> trivy image myapp:v1
   |
   v
Container Registry
   |
   v
Kubernetes
   |
   |----> trivy config .
   |----> trivy k8s ...
   |
   v
Application
```

### Commands to Remember

```bash
# Version
trivy --version

# Image scanning
trivy image nginx:latest

# Important vulnerabilities only
trivy image --severity HIGH,CRITICAL nginx:latest

# Ignore vulnerabilities without fixes
trivy image --ignore-unfixed nginx:latest

# Filesystem
trivy fs .

# Secrets
trivy fs --scanners secret .

# Kubernetes/IaC manifest
trivy config deployment.yaml

# Kubernetes cluster
trivy k8s --report summary cluster

# Save JSON report
trivy image --format json --output report.json nginx:latest
```

For a **very basic classroom demo**, the four commands worth demonstrating are:

```bash
trivy image nginx:latest
trivy image --severity HIGH,CRITICAL nginx:latest
trivy fs .
trivy config deployment.yaml
```

These clearly demonstrate the progression **Image → Vulnerabilities, Source → Security Issues, Kubernetes YAML → Misconfigurations**.
