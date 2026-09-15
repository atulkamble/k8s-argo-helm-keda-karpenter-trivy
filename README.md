# Kubernetes Cloud Native Tools

Basic examples for learning one tool at a time.

| Tool      | What you learn               | Instructions                            |
| --------- | ---------------------------- | --------------------------------------- |
| Helm      | Package and deploy Nginx     | [Helm lesson](helm/README.md)           |
| Trivy     | Scan the Nginx image         | [Trivy lesson](trivy/README.md)         |
| Argo CD   | Sync the Helm chart from Git | [Argo CD lesson](argocd/README.md)      |
| KEDA      | Scale application Pods       | [KEDA lesson](keda/README.md)           |
| Karpenter | Add EC2 worker nodes         | [Karpenter lesson](karpenter/README.md) |

## Folder structure

```text
.
├── helm/
│   ├── README.md
│   └── webapp/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           └── service.yaml
├── argocd/
│   ├── README.md
│   ├── application.yaml
│   └── autoscaling-patch.yaml
├── keda/
│   ├── README.md
│   ├── scaledobject.yaml
│   └── load-generator.yaml
├── karpenter/
│   ├── README.md
│   ├── ec2nodeclass.yaml
│   ├── nodepool.yaml
│   └── capacity-demo.yaml
└── trivy/
    └── README.md
```

## Prerequisites

- An existing Kubernetes cluster and configured `kubectl`.
- Helm and Trivy installed locally.
- A GitHub repository for the Argo CD lesson.
- Metrics Server for KEDA CPU scaling.
- EKS, an installed Karpenter controller, and its AWS permissions for node scaling.

Check your cluster:

```bash
kubectl config current-context
kubectl get nodes
helm version
trivy --version
```

## Start here

Run all lesson commands from the repository root. The examples use release
`webapp` in namespace `default`.

```bash
helm lint ./helm/webapp
helm template webapp ./helm/webapp
helm install webapp ./helm/webapp --namespace default
kubectl get pods,svc -n default
```

Follow the lessons in the table's order. Each folder includes commands to apply,
observe, and clean up its example. The `latest` Nginx tag keeps the classroom
example simple; use a tested version when you need repeatable results.

## How the tools connect

```text
Git → Argo CD → Helm chart → Nginx Pods
                               ↑
                         KEDA scales Pods
                               ↓
                         Pending Pods
                               ↓
                    Karpenter adds nodes

Trivy scans the container image before deployment.
```

For the Argo CD replica-change lesson, keep KEDA disabled. When running both
controllers together, use the optional patch in the Argo CD folder so KEDA
controls replicas. The Karpenter lesson uses a separate capacity Deployment.

Cloud load balancers and EC2 nodes incur charges. Follow each lesson's cleanup
steps afterward; the existing cluster and installed controllers remain.
