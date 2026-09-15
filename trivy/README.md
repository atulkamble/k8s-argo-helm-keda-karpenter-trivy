# Trivy: scan an image

Trivy runs locally, so this lesson needs commands rather than a Kubernetes manifest.
Install the Trivy CLI, then run from the repository root:

```bash
trivy image nginx:latest
trivy image --severity HIGH,CRITICAL nginx:latest
```

Read the package name, severity, installed version, and fixed version in the
results. Scan the same image tag that you deploy in the Helm chart.

Optional: return a failure status when HIGH or CRITICAL vulnerabilities exist:

```bash
trivy image --severity HIGH,CRITICAL --exit-code 1 nginx:latest
```

The scan needs registry access and Trivy's vulnerability database.
