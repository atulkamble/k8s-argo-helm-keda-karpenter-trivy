# KEDA: scale Pods

Run commands from the repository root after deploying `webapp` in `default`.

## Install

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace --wait
kubectl top pods -n default
```

CPU scaling needs Metrics Server. If `kubectl top` fails, install or repair
Metrics Server first. CPU requests are already in the Helm chart.

## Try scaling

For a Helm-managed app:

```bash
helm upgrade webapp ./helm/webapp --set autoscaling.enabled=true
```

For an Argo CD-managed app, use the patch in the Argo CD lesson instead.

```bash
kubectl apply -f keda/scaledobject.yaml
kubectl get scaledobject,hpa -n default
kubectl apply -f keda/load-generator.yaml
kubectl get hpa -n default -w
```

The scaler targets 50% CPU utilization and allows 1–5 replicas. Watch Pods
with `kubectl get pods -w`. Traffic runs for five minutes; `DeadlineExceeded`
is expected when the job stops. If CPU stays below 50%, increase the job's
`spec.parallelism`, delete the job, and apply it again. Scaling is not guaranteed
by simply sending a few requests. Scale-down can take several minutes.

## Cleanup

```bash
kubectl delete -f keda/load-generator.yaml --ignore-not-found
kubectl delete -f keda/scaledobject.yaml --ignore-not-found
```

Return `autoscaling.enabled` to `false` using Helm or remove the Argo CD patch,
depending on which tool manages your app.
