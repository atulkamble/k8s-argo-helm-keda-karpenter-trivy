# Helm

A chart packages templates and default values. A release is a named installation of that chart. An upgrade records a new revision and a rollback reapplies a previous revision.

Run from the repository root. The chart uses a ClusterIP Service and local port forwarding.

```bash
helm lint ./helm/webapp
helm template webapp ./helm/webapp --namespace webapp-lab
helm upgrade --install webapp ./helm/webapp --namespace webapp-lab --create-namespace --wait --timeout 3m
kubectl get deployment,pods,svc -n webapp-lab
kubectl port-forward -n webapp-lab svc/webapp 8081:80
```

In a second terminal, run `curl -fsS http://localhost:8081/` and look for the Nginx welcome page. Stop port forwarding with Ctrl+C.

## Upgrade and roll back

This block assumes a fresh lab release with revision 1. Use `helm history` to select the original revision when reusing a release.

```bash
helm upgrade webapp ./helm/webapp -n webapp-lab --set replicaCount=2 --wait --timeout 3m
kubectl get deployment webapp -n webapp-lab
helm history webapp -n webapp-lab
helm rollback webapp 1 -n webapp-lab --wait --timeout 3m
kubectl get deployment webapp -n webapp-lab
```

Expect two ready replicas after the upgrade and one after rollback.

## Cleanup and handoff

```bash
helm uninstall webapp -n webapp-lab --wait --timeout 3m
```

Keep the namespace for Argo CD. For a standalone Helm and KEDA walkthrough, keep the release until KEDA cleanup finishes.
