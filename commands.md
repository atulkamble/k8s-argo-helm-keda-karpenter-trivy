# helm 
```
eksctl create cluster --name mycluster --region us-east-1 --nodegroup-name mynodes --node-type t3.medium --nodes 2 --nodes-min 2 --nodes-max 2 --managed

aws eks update-kubeconfig --name mycluster --region us-east-1

kubectl get nodes 
kubectl get pods 
kubectl get services 
kubectl get ns

Helm - the package manager
Charts - community curated charts

https://helm.sh/docs/intro/install

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh

// install on widndows 

choco install kubernetes-helm
OR 
winget install Helm.Helm

>> importing 

helm version 
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo list
helm repo update
helm search repo nginx
helm install my-nginx bitnami/nginx
helm list
kubectl get pods
kubectl get svc
helm uninstall my-nginx
http://ad8f0271dd7484f6dadf9c809891fa0f-1482572670.us-east-1.elb.amazonaws.com/

// wordpress

helm install my-wordpress bitnami/wordpress
OR
helm install my-wordpress bitnami/wordpress --set persistence.enabled=false --set mariadb.primary.persistence.enabled=false

a3f17651cc63f428181928e8d721561d-696092515.us-east-1.elb.amazonaws.com

// get password 

kubectl get secret my-wordpress \
  -o jsonpath="{.data.wordpress-password}" | base64 -d
echo

// wp-admin
a3f17651cc63f428181928e8d721561d-696092515.us-east-1.elb.amazonaws.com/wp-admin

// username - user 
password - dqoo0zBqmh

helm list
kubectl get pods
kubectl get svc
helm uninstall my-wordpress

kubectl get secret my-wordpress \
  -o jsonpath="{.data.wordpress-password}" | base64 -d
echo


>> manual

helm create myapp
cd myapp
helm lint .
helm install myapp .
helm list
kubectl get pods
kubectl get svc
kubectl get deployments
helm upgrade myapp . --set replicaCount=2

```
# ArgoCD
```
1. // create eks cluster 
eksctl create cluster --name mycluster --region us-east-1 --nodegroup-name mynodes --node-type t3.medium --nodes 2 --nodes-min 2 --nodes-max 2 --managed

2. // update kubeconfig 
aws eks update-kubeconfig --name mycluster --region us-east-1

3. // check kubectl 
kubectl version --client

4. // check details
kubectl get nodes
kubectl get ns
kubectl get svc

5. // create Github Repo OR fork https://github.com/atulkamble/argocd-app
git clone https://github.com/atulkamble/argocd-app
cd argocd-app
code .

6. Install ArgoCD

kubectl create ns argocd 

kubectl get ns 

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

OR 

kubectl apply --server-side --force-conflicts -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl get pods -n argocd

7. Port Forwarding for ArgoCD

kubectl port-forward svc/argocd-server -n argocd 8080:443

// Run in Background 

kubectl port-forward svc/argocd-server -n argocd 8080:443 > /tmp/argocd-port-forward.log 2>&1 &

// on mac

nohup kubectl port-forward svc/argocd-server -n argocd 8080:443 > /tmp/argocd-port-forward.log 2>&1 &

// TIP : keep it running in one of tab 

8. // retrive password for logging in to ArgoCD UI 

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
echo

9. // Login to ArgoCD Server - https://localhost:8080

username - admin 
password - AhZmoaLQTfpUC9wg

10. // App Deployment 

Application Name: nginx-app
Project: default
Repository URL: https://github.com/atulkamble/argocd-app
Revision: main
Path: .
Cluster: https://kubernetes.default.svc
Namespace: default

11. click on CREATE

12. Kill Process

pkill -f "kubectl port-forward svc/argocd-server"

eksctl delete cluster --name mycluster --region us-east-1
```
