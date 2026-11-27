# KEDA

A ScaledObject connects a Deployment to a scaling trigger. KEDA creates an HPA, and the HPA adjusts replicas using CPU metrics from Metrics Server. This lab targets 50% of requested CPU on average and keeps between one and five replicas.

Run from the repository root with a healthy `webapp` Deployment in `webapp-lab`.

```bash
kubectl top pods -n webapp-lab
```

If `kubectl top` fails, verify Metrics Server and wait for the Nginx Pods to report CPU metrics. If KEDA is already installed, skip the installation block.

```bash
helm repo add kedacore https://kedacore.github.io/charts --force-update
helm repo update kedacore
helm upgrade --install keda kedacore/keda -n keda --create-namespace --wait --timeout 5m
kubectl wait --for=condition=Established crd/scaledobjects.keda.sh --timeout=2m
```

## Give the HPA control of replicas

For the **Helm** path, run this command in a shell.

```bash
helm upgrade webapp ./helm/webapp -n webapp-lab --reuse-values --set autoscaling.enabled=true --wait --timeout 3m
```

For the **Argo CD** path, run these commands instead. The patch omits replicas from the rendered chart and tells Argo CD to respect the live HPA-managed replica value during sync.

```bash
kubectl patch application webapp -n argocd --type merge --patch-file argocd/autoscaling-patch.yaml
kubectl wait -n argocd application/webapp --for=jsonpath='{.status.sync.comparedTo.source.helm.parameters[0].value}'=true --timeout=5m
kubectl wait -n argocd application/webapp --for=jsonpath='{.status.sync.status}'=Synced --timeout=5m
```

## Generate load and observe

```bash
kubectl apply -f keda/scaledobject.yaml
kubectl wait -n webapp-lab scaledobject/webapp-scaler --for=condition=Ready --timeout=2m
kubectl delete -f keda/load-generator.yaml --ignore-not-found
kubectl apply -f keda/load-generator.yaml
kubectl get hpa -n webapp-lab -w
```

In another terminal, run `kubectl top pods -n webapp-lab` and `kubectl get pods -n webapp-lab -l app=webapp -w`. Look for CPU above target followed by a replica increase. If CPU remains below target, raise the Job's `parallelism` and `completions` together, then recreate it. After five minutes, `DeadlineExceeded` is expected. Scale-down can take several more minutes.

## Cleanup

```bash
kubectl delete -f keda/load-generator.yaml --ignore-not-found
kubectl delete -f keda/scaledobject.yaml --ignore-not-found
```

For Helm, return to fixed replicas with the following command, then follow Helm cleanup.

```bash
helm upgrade webapp ./helm/webapp -n webapp-lab --reuse-values --set autoscaling.enabled=false --wait --timeout 3m
```

For Argo CD, remove the merge-patch fields, then follow Argo CD cleanup.

```bash
kubectl patch application webapp -n argocd --type merge -p '{"spec":{"source":{"helm":{"parameters":null}},"ignoreDifferences":null,"syncPolicy":{"syncOptions":null}}}'
```

[KEDA CPU scaler](https://keda.sh/docs/2.18/scalers/cpu/)
