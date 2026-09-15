# Helm: package and deploy Nginx

Run commands from the repository root. Use an existing Kubernetes cluster.

The chart has four files: chart information, values, a Deployment, and a Service.
Edit `helm/webapp/values.yaml` to change replicas, image, or service type.

```bash
helm lint ./helm/webapp
helm template webapp ./helm/webapp
helm install webapp ./helm/webapp --namespace default
kubectl get pods,svc -n default
kubectl port-forward svc/webapp -n default 8081:80
```

Open http://localhost:8081. The default `LoadBalancer` Service can create a
billable cloud load balancer. Use `service.type: ClusterIP` for local practice.

Upgrade and roll back:

```bash
helm upgrade webapp ./helm/webapp --set replicaCount=2
helm rollback webapp 1
```

Cleanup (remove the Argo CD Application first if you used that lesson):

```bash
helm uninstall webapp --namespace default
```
