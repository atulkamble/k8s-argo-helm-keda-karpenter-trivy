# Argo CD

Argo CD reconciles Kubernetes resources against manifests rendered from Git. This Application renders the existing Helm chart. `Synced` describes alignment with the desired revision, while `Healthy` describes the state of the live resources.

## Install and connect

Finish Helm cleanup first. Reuse an existing Argo CD installation when available. For a fresh lab installation, run from the repository root.

```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl rollout status -n argocd deployment/argocd-repo-server --timeout=5m
kubectl rollout status -n argocd statefulset/argocd-application-controller --timeout=5m
```

`application.yaml` targets this repository's `main` branch. If you teach from a fork or review branch, set `repoURL` and `targetRevision` to the published source containing your chart before applying.

```bash
kubectl create namespace webapp-lab --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f argocd/application.yaml
kubectl wait -n argocd application/webapp --for=jsonpath='{.status.sync.status}'=Synced --timeout=5m
kubectl wait -n argocd application/webapp --for=jsonpath='{.status.health.status}'=Healthy --timeout=5m
kubectl get deployment,pods,svc -n webapp-lab
```

## Demonstrate reconciliation

Set `replicaCount: 2` in `helm/webapp/values.yaml` and push the change to the Application's configured source branch. In a shared classroom, the instructor can push once for everyone.

```bash
git add helm/webapp/values.yaml
git commit -m "Demo: scale webapp to two replicas"
git push
```

```bash
kubectl annotate -n argocd application/webapp argocd.argoproj.io/refresh=hard --overwrite
kubectl wait -n webapp-lab deployment/webapp --for=jsonpath='{.spec.replicas}'=2 --timeout=5m
kubectl rollout status -n webapp-lab deployment/webapp --timeout=3m
kubectl get -n argocd application/webapp
```

Automated sync applies Git changes. Self-heal corrects tracked live drift. Prune removes tracked resources deleted from Git.

## Optional UI

Run `kubectl port-forward -n argocd svc/argocd-server 8080:443` and open `https://localhost:8080`. On a fresh install, use username `admin` and retrieve its initial password in a private terminal.

```bash
kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
echo
```

## Cleanup

If continuing to KEDA, leave the Application running. After KEDA cleanup, run the following. The Application uses a cascading deletion finalizer to remove its managed Deployment and Service.

```bash
kubectl delete -f argocd/application.yaml --ignore-not-found --timeout=5m
kubectl get deployment,svc -n webapp-lab
```

[Argo CD sync options](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/)
