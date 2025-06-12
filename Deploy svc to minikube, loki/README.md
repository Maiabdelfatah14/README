## Objectives

• Deploy a sample application to minikube. 

• Run the app.

• View application logs


#  important ( install one pod grafana to avoid errors )
•  Grafana Agent يجمع البيانات من بيئتك.   ( mmkn uninstall , forwaord and then access prowser )

•  Grafana يعرض هذه البيانات التي جمعها Agent في لوحات معلومات سهلة الفهم.   ( should install to access browser )


### A) Minikube installed and running

Make sure this is okay :- 

	1- virtualiza intel VT-x/EPT or AMD-v/RTI      ( enable)

	2- mem  3 

	3- process 3


## 1. install docker 
```bash
https://docs.docker.com/engine/install/ubuntu/
sudo dnf install docker-ce docker-ce-cli containerd.io -y
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker # Apply group changes immediately in the current shell
systemctl status docker
```

## 4. Install minikube
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube-linux-amd64
sudo mv minikube-linux-amd64 /usr/local/bin/minikube
```

## 5. Install kubectl 
```bash
sudo snap install kubectl --classic
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

## 6. Start minikube 
Start  Minikube cluster using the Docker driver 
```bash
minikube start --driver=docker
minikube status
kubectl get nodes
```
![image](https://github.com/user-attachments/assets/5d2ee184-4a63-4dae-a47f-a4e11da93136)
![image](https://github.com/user-attachments/assets/fb9b5a97-7824-4295-8066-f087bf0a7784)


## B) Create app and service Nodeport
Create a sample application deployment and expose it as a NodePort service
```bash
Hello Minikube | Kubernetes
minikube start
kubectl create deployment hello-node --image=registry.k8s.io/e2e-test-images/agnhost:2.39 -- /agnhost netexec --http-port=8080
kubectl expose deployment hello-node --type=NodePort --port=8080
kubectl get pods
kubectl get services
minikube service hello-node
kubectl logs <pod-name>
```
![image](https://github.com/user-attachments/assets/d96ef8d5-5d31-40c7-ab34-82e02393e869)
![image](https://github.com/user-attachments/assets/6cfd1e81-3a37-4639-92d8-de2ed7dec182)


## C) Install grafana and promethues with helm

>> Install helm :
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```


## 1) Add Loki for Log Aggregation & Grafana
Add Grafana Helm Repository
```bash
 helm repo add grafana https://grafana.github.io/helm-charts
 helm repo update
```

Install Loki Stack
```bash
helm show values grafana/loki-stack > loki-stack-values.yaml
Nano loki-stack-values.yaml
(enable : true  , service / type: clusterip ) 
helm install loki-stack grafana/loki-stack 
```

Verify Installation and Access Loki/Grafana
```bash
kubectl port-forward pod/grafana-59b6644864-rgw8c 3000:3000                     
kubectl port-forward svc/loki-stack 3100:3100  
```
Then to access the Grafana dashboard in browser 
>>  http://localhost:3000

in grafana
```bash
1- Name : admin
2- Passwd:
kubectl get secret loki-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode           (loki)
```

in grafana >> explore 
![image](https://github.com/user-attachments/assets/13111064-8df8-442b-a5ee-397a1e950c4e)
![image](https://github.com/user-attachments/assets/acce8851-e53e-4e7e-bd7c-2ae1b830ab5f)


## To connect from pod grafana to loki in terminal 
```bash
kubectl exec -it   grafana-59b6644864-rgw8c  -- /bin/bash
curl -v http://loki-stack.default:3100/ready
```
![image](https://github.com/user-attachments/assets/5ef358f3-94ac-46cf-8c1d-8bdad34b25cd)


##  2) Install Grafana and Prometheus  ( If you need )

Install Grafana and Prometheus using Helm charts for monitoring my cluster.
A) Add Helm Repository
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

B) Install Prometheus + Grafana Stack  ( Metrices ) 
```bash
kubectl create namespace monitoring
helm install prometheus-stack prometheus-community/kube-prometheus-stack  --namespace monitoring
kubectl get pods                 ( If all pods are running ) 

kubectl port-forward pod/prometheus-stack-grafana-56d84b4f48-dwthm 3000:3000                                                   
kubectl port-forward pod/prometheus-prometheus-stack-kube-prom-prometheus-0 9090:9090                            
```


Then to access the Grafana dashboard in browser 
>>  http://localhost:3000

c) in grafana
```bash
1- Name : admin
2- Passwd:
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
kubectl get secret prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode    (prometheus)
```
![image](https://github.com/user-attachments/assets/79f3d98c-7162-419e-8621-d6fa078caf4f)
![image](https://github.com/user-attachments/assets/6953f04a-2e5f-409a-8331-cb4c92692fa5)


## Uninstall Helm Releases
1) Uninstall loki-stack:
```bash
helm uninstall loki-stack
helm list -A # Lists all Helm releases ( grafana, prometheus ) in all namespaces
helm uninstall grafana
helm uninstall prometheus
```
2) Remove Helm Repositories
```bash
helm repo remove grafana
helm repo remove prometana # If you added a separate Prometheus repo
helm repo remove loki      # If you added a separate Loki repo
```

3) Delete the Minikube Cluster
```bash
minikube delete
sudo rm /usr/local/bin/minikube
sudo rm /usr/local/bin/kubectl
rm -rf ~/.minikube
rm -rf ~/.kube
```
