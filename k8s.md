# Kubernetes Deployments, ReplicaSets, StatefulSets, Ingress Controllers & CoreDNS

Complete training notes with concepts, architecture diagrams, YAML manifests, commands, practical labs, and interview points.

## 1. Kubernetes Deployment

A Deployment is a Kubernetes workload resource used to manage stateless applications. It automatically manages ReplicaSets, maintains the desired number of Pods, and supports rolling updates and rollbacks.

Key features: scalability, self-healing, declarative updates, rollout history, and rollback support.

Deployment architecture

### Deployment YAML

Save as `deployment.yaml`.

```

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80

```

### Deployment commands

```

kubectl apply -f deployment.yaml

kubectl get deployments
kubectl get replicasets
kubectl get pods -o wide

kubectl describe deployment nginx-deployment

# Scale application
kubectl scale deployment nginx-deployment --replicas=5

# Check status
kubectl rollout status deployment/nginx-deployment

```

## 2. Rolling Updates and Rollbacks

Rolling updates gradually replace old Pods with new Pods, helping maintain application availability during releases.

Rolling update example: 3 replicas

Before

v1

v1

v1

During

v1

v2

v2

v2

After

v2

v2

v2

Old version

New version

With maxSurge: 1, Kubernetes can temporarily create a fourth Pod.

### Practical commands

```

# Update image
kubectl set image deployment/nginx-deployment \
  nginx=nginx:1.28

# Monitor rolling update
kubectl rollout status deployment/nginx-deployment

# View revision history
kubectl rollout history deployment/nginx-deployment

# Roll back to previous revision
kubectl rollout undo deployment/nginx-deployment

# Roll back to specific revision
kubectl rollout undo deployment/nginx-deployment \
  --to-revision=1

# Pause and resume rollout
kubectl rollout pause deployment/nginx-deployment
kubectl rollout resume deployment/nginx-deployment

```

| Strategy      | Description                                                                                 |
| ------------- | ------------------------------------------------------------------------------------------- |
| RollingUpdate | Gradually replaces Pods; default strategy                                                   |
| Recreate      | Terminates old Pods before creating new ones                                                |
| Blue-Green    | Uses separate environments and switches traffic; requires additional configuration          |
| Canary        | Routes limited traffic to a new version; requires additional deployment or routing controls |

## 3. ReplicaSet

A ReplicaSet ensures that a specified number of identical Pod replicas are running.

If a Pod fails or is deleted, the ReplicaSet creates a replacement.

```

apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-rs
  template:
    metadata:
      labels:
        app: nginx-rs
    spec:
      containers:
        - name: nginx
          image: nginx:1.27

```

```

kubectl apply -f replicaset.yaml
kubectl get rs
kubectl get pods

# Delete a Pod and observe self-healing
kubectl delete pod <pod-name>
kubectl get pods -w

```

Important: In production, use Deployments rather than manually managing ReplicaSets for most stateless applications.

## 4. StatefulSet

A StatefulSet manages stateful applications requiring stable Pod identities, ordered deployment, and persistent storage.

Examples include MySQL, PostgreSQL, MongoDB, and clustered databases.

StatefulSet architecture

Each Pod has a predictable identity and its own persistent volume claim.

### StatefulSet YAML

```

apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
    - port: 3306
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.4
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 5Gi

```

Create the referenced Secret before applying this example. A suitable StorageClass must also exist. This manifest demonstrates stable storage and identity; it does not configure MySQL replication or clustering.

```

kubectl create secret generic mysql-secret \
  --from-literal=root-password='ChangeMe123!'

kubectl apply -f statefulset.yaml
kubectl get statefulsets
kubectl get pods
kubectl get pvc

```

## 5. Deployment vs ReplicaSet vs StatefulSet

| Feature                | Deployment     | ReplicaSet              | StatefulSet                        |
| ---------------------- | -------------- | ----------------------- | ---------------------------------- |
| Primary use            | Stateless apps | Maintain Pod count      | Stateful apps                      |
| Stable Pod names       | No             | No                      | Yes                                |
| Automatic recovery     | Yes            | Yes                     | Yes                                |
| Rolling updates        | Yes            | Not directly            | Yes                                |
| Rollback history       | Yes            | No                      | Limited; manual revision selection |
| Stable storage per Pod | Not built in   | Not built in            | Yes, via PVC templates             |
| Typical examples       | Nginx, APIs    | Deployment-managed Pods | Databases                          |

## 6. Kubernetes Ingress

An Ingress is a Kubernetes API resource that defines HTTP and HTTPS routing rules for traffic entering services inside the cluster.

An Ingress resource requires an Ingress Controller to implement those rules.

Ingress routing architecture

### Ingress YAML — Path-based routing

```

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: demo.example.com
      http:
        paths:
          - path: /app
            pathType: Prefix
            backend:
              service:
                name: app-service
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80

```

This requires an installed controller supporting the `nginx` IngressClass and existing `app-service` and `api-service` Services.

The `/app` prefix is forwarded as-is unless the controller is configured to rewrite it.

### Host-based routing

| Hostname            | Backend Service |
| ------------------- | --------------- |
| `app.example.com`   | app-service     |
| `api.example.com`   | api-service     |
| `admin.example.com` | admin-service   |

```

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-service
                port:
                  number: 80
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80

```

## 7. Ingress Controllers

An Ingress Controller watches Ingress resources and configures a proxy or load balancer to route traffic according to their rules.

| Controller                               | Common environment             |
| ---------------------------------------- | ------------------------------ |
| ingress-nginx (community)                | Existing/self-managed clusters |
| F5 NGINX Ingress Controller              | Enterprise NGINX environments  |
| Traefik                                  | Kubernetes and microservices   |
| AWS Load Balancer Controller             | Amazon EKS, AWS ALB            |
| Azure Application Gateway for Containers | Azure AKS                      |

For new deployments, also evaluate the Kubernetes Gateway API, especially because the community ingress-nginx controller is being retired in March 2026 and is no longer a recommended choice for new production installations.

### Common Ingress commands

```

kubectl get ingress
kubectl get ingressclass

kubectl describe ingress web-ingress

kubectl apply -f ingress.yaml

kubectl get svc -A

kubectl delete ingress web-ingress

```

## 8. Kubernetes CoreDNS

CoreDNS is the DNS server commonly deployed in Kubernetes clusters. It provides internal service discovery so Pods can communicate using DNS names rather than hardcoded IP addresses.

For example:

`web-service.default.svc.cluster.local`

| Component       | Meaning                    |
| --------------- | -------------------------- |
| `web-service`   | Service name               |
| `default`       | Namespace                  |
| `svc`           | Kubernetes service domain  |
| `cluster.local` | Default cluster DNS domain |

### DNS resolution flow

CoreDNS resolves the Service name; network traffic then reaches the Service and its backend Pods through the cluster networking implementation.

### CoreDNS troubleshooting commands

```

# Check CoreDNS Pods
kubectl get pods -n kube-system \
  -l k8s-app=kube-dns

# Check DNS service
kubectl get svc kube-dns -n kube-system

# Check CoreDNS configuration
kubectl get configmap coredns -n kube-system -o yaml

# View DNS logs
kubectl logs -n kube-system deployment/coredns

# Test DNS using BusyBox
kubectl run dns-test --image=busybox:1.36 \
  --restart=Never -- sleep 3600

kubectl exec dns-test -- \
  nslookup kubernetes.default.svc.cluster.local

kubectl delete pod dns-test

```

## 9. Practical lab — Deploy, Scale, Update and Expose Nginx

Suitable for Minikube, AKS, EKS, or another configured Kubernetes cluster.

Hands-on checklist

0/8 completed

1\. Create Deployment

`kubectl create deployment webapp --image=nginx:1.27 --replicas=3`

2\. Verify Pods and ReplicaSet

`kubectl get deploy,rs,pods`

3\. Expose with ClusterIP Service

`kubectl expose deployment webapp --port=80 --target-port=80`

4\. Scale to 5 replicas

`kubectl scale deployment webapp --replicas=5`

5\. Perform rolling update

`kubectl set image deployment/webapp nginx=nginx:1.28`

6\. Check rollout

`kubectl rollout status deployment/webapp`

7\. Roll back

`kubectl rollout undo deployment/webapp`

8\. Test internal DNS

`kubectl run dns-check --rm -it --restart=Never --image=busybox:1.36 -- nslookup webapp.default.svc.cluster.local`

To test Ingress externally, install a supported controller, create an Ingress resource targeting the `webapp` Service, and configure DNS or a local Host header to reach the controller.

## 10. Important interview questions

| Question                                | Key answer                                                                           |
| --------------------------------------- | ------------------------------------------------------------------------------------ |
| What is a Deployment?                   | Controller for declarative management of stateless applications                      |
| What happens if a Pod fails?            | ReplicaSet creates a replacement                                                     |
| Deployment vs ReplicaSet?               | Deployment manages ReplicaSets and application updates                               |
| What is a rolling update?               | Gradual replacement of old Pods                                                      |
| What is a rollback?                     | Reverting to a previous Deployment revision                                          |
| Why use StatefulSet?                    | Stable identities and persistent storage                                             |
| What is Ingress?                        | HTTP/HTTPS routing configuration                                                     |
| Does Ingress work without a controller? | Rules alone do not implement traffic routing                                         |
| What is CoreDNS?                        | Cluster DNS service and service discovery                                            |
| Ingress vs LoadBalancer Service?        | Ingress defines HTTP routing; LoadBalancer exposes a Service through a load balancer |

## 11. Quick revision — Commands

```

kubectl get deployments
kubectl get replicasets
kubectl get statefulsets
kubectl get pods -o wide
kubectl get svc
kubectl get ingress
kubectl get ingressclass

kubectl scale deployment webapp --replicas=3
kubectl rollout status deployment/webapp
kubectl rollout history deployment/webapp
kubectl rollout undo deployment/webapp

kubectl describe deployment webapp
kubectl describe ingress web-ingress

kubectl get pods -n kube-system
kubectl get configmap coredns -n kube-system

```

Recommended learning sequence: Deployment → ReplicaSet → Scaling → Rolling Update → Rollback → StatefulSet → Service → Ingress Controller → Ingress Rules → CoreDNS → Troubleshooting.

This order makes it easier to understand how Kubernetes maintains application availability and routes traffic from users to application Pods.
