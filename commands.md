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

# KEDA 
```
eksctl create cluster --name mycluster --region us-east-1 --nodegroup-name mynodes --node-type t3.medium --nodes 2 --nodes-min 2 --nodes-max 2 --managed
aws eks update-kubeconfig --name mycluster --region us-east-1
kubectl get nodes                        
kubectl get pods 
kubectl get svc 
kubectl get ns 
choco install kubernetes-helm
OR 
winget install Helm.Helm
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace
kubectl get ns 
kubectl get pods -n keda

git clone https://github.com/atulkamble/argocd-app
cd argocd-app
kubectl apply -f keda-deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f scaledobject.yaml
kubectl get pods
kubectl get svc 
kubectl get svc nginx-service

http://a3e1bffec0d6f47e6becd9c4b6124a36-1184056075.us-east-1.elb.amazonaws.com/

kubectl get scaledobject
kubectl get hpa
kubectl describe scaledobject nginx-scaledobject

kubectl create ns argocd

kubectl apply --server-side --force-conflicts -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl port-forward svc/argocd-server -n argocd 8080:443 > /tmp/argocd-port-forward.log 2>&1 &

// App Deployment

Application Name: nginx-app
Project: default
Repository URL: https://github.com/atulkamble/argocd-app
Revision: main
Path: .
Cluster: https://kubernetes.default.svc
Namespace: default

kubectl top pods

kubectl run load-generator --image=busybox:1.36 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://nginx-service; done"

kubectl get pods
kubectl top pods
kubectl get hpa
kubectl get scaledobject

kubectl get hpa -w

for i in {1..5}; do
  kubectl run load-generator-$i --image=busybox:1.36 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://nginx-service; done"
done
```
# Trivy 
```
// trivy - security scan

- vulnerabilities - report

choco install trivy

WSL >>

sudo apt-get install wget gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy

trivy --version

// keep docker desktop in running state

docker pull nginx:latest
trivy image nginx:latest
trivy image --severity HIGH,CRITICAL nginx:latest
trivy image --ignore-unfixed nginx:latest

// helloworld.py

print("hello world")
print("username: admin")

# Basic password comment
# password: admin

# Dummy secrets for Trivy practice ONLY
AWS_ACCESS_KEY_ID = "AKIAIOSFODNN7EXAMPLE"
AWS_SECRET_ACCESS_KEY = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"

GITHUB_TOKEN = "ghp_1234567890abcdefghijklmnopqrstuvwxyz"

DATABASE_URL = "mysql://admin:TestPassword123@database.example.com:3306/mydb"

2. // Create Dockerfile

FROM python:3.12-slim
WORKDIR /app
COPY . .
CMD ["python","helloworld.py"]

3. Image Build

docker buildx build -t docker.io/atuljkamble/pythonapp --load .

4. scan via trivy

trivy image atuljkamble/pythonapp:latest

5. scan file system

trivy fs secrets.txt
trivy fs .

6. scan manifests

trivy config deployment.yaml
```
