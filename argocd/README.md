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
