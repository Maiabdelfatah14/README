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
![image](https://github.com/user-attachments/assets/16110282-2eb4-42df-8d12-511efdf0831f)

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
![image](https://github.com/user-attachments/assets/bc00dfc3-49a7-49ff-932f-8dee9e3a6f92)


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
helm search repo loki
```

Install Loki Stack
```bash
helm show values grafana/loki-stack > loki-stack-values.yaml
vim loki-stack-values.yaml
(enable : true  , service / type: NodePort ) 
helm install loki-stack grafana/loki-stack -f loki-stack-values.yaml 
```
![image](https://github.com/user-attachments/assets/61aba693-8e4b-473e-ad2b-6fb4f2d2f419)


Verify Installation and Access Loki/Grafana
```bash                    
kubectl port-forward svc/loki-stack 3000:80 
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
![image](https://github.com/user-attachments/assets/a4ae6f71-01e0-4a3c-870e-6ec8442cee1e)
![image](https://github.com/user-attachments/assets/0cb039f7-11c8-4072-8e77-2ce2fd6378c7)
![image](https://github.com/user-attachments/assets/2b88dc80-aa5f-4cdb-9c9a-9d1259d43069)


## To connect from pod grafana to loki in terminal 
```bash
kubectl exec -it   loki-stack-grafana-6f56d6dd85-gnntd   -- /bin/bash
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
helm install prometheus-stack prometheus-community/kube-prometheus-stack --set grafana.enabled=false   ( 3shan ykon 3andy pod grafana one bs , loki-stack-grafana)
kubectl get pods                 ( If all pods are running ) 

kubectl port-forward service/loki-stack-grafana -n default 3000:80                                                                            
```


Then to access the Grafana dashboard in browser 
>>  http://localhost:3000

c) in grafana
```bash
1- Name : admin
2- Passwd:
kubectl get secret loki-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode           (loki)
url :   http://prometheus-stack-kube-prom-prometheus.default:9090
```
![image](https://github.com/user-attachments/assets/afbf3cf9-e355-45f5-9f52-7a03f69d4e3e)


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

## to add blackbox to check the service 
```bash 
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install blackbox-exporter prometheus-community/prometheus-blackbox-exporter
helm get values  prometheus-stack -o yaml > prometheus-stack-values.yaml
```
in file prometheus-stack-values.yaml
```bash
# prometheus-stack-values.yaml

grafana:
  enabled: false


prometheus:
  prometheusSpec:
    additionalScrapeConfigs:
      - job_name: 'blackbox-http-probes'
        metrics_path: /probe
        params:
          module: ['http_2xx']
        static_configs:
          - targets:
              - http://hello-node:8080
        relabel_configs:
          - source_labels: [__address__]
            target_label: __param_target
          - source_labels: [__param_target]
            target_label: instance
          - target_label: __address__
            replacement: blackbox-exporter-prometheus-blackbox-exporter.default.svc.cluster.local:9115
          - source_labels: [__param_target]
            target_label: target_url

kubeControllerManager:
  enabled: false

kubeEtcd:
  enabled: false

kubeScheduler:
  enabled: false
# ... لو فيه أي configs تانية تحت
```

```bash
helm upgrade prometheus-stack prometheus-community/kube-prometheus-stack \
  -f prometheus-stack-values.yaml
```

# A)	Delete deployment and service 
```bash
kubectl delete deployment hello-node -n default
kubectl delete service hello-node -n default
```
then 
```bash
kubectl port-forward service/loki-stack-grafana -n default 3000:80
import dashboard  Dashboard ID  ( 7587 )  
```
![image](https://github.com/user-attachments/assets/4f2b7dc1-45f8-4d55-be91-72878f469bf5)


# B) create deployment again 
```bash
kubectl create deployment hello-node --image=registry.k8s.io/e2e-test-images/agnhost:2.39 -- /agnhost netexec --http-port=8080
kubectl expose deployment hello-node --type=NodePort --port=8080
```
![image](https://github.com/user-attachments/assets/6a39c230-3173-4278-a9f3-a8c675cc3807)


## can show in 
```bash
kubectl port-forward svc/blackbox-exporter-prometheus-blackbox-exporter  9115
```
![image](https://github.com/user-attachments/assets/1d1d32dd-bb95-40d2-9713-5481fbd56348)
