## Objectives

• Deploy a sample application to minikube. 

• Run the app.

• View application logs


### A) Minikube installed and running

Make sure this is okay :- 

	1- virtualiza intel VT-x/EPT or AMD-v/RTI      ( enable)

	2- mem  3 

	3- process 3



### 1. Virtualization Support
```bash
  egrep -q 'vmx|svm' /proc/cpuinfo && echo "Virtualization supported" || echo "No virtualization support"         
```


## 2. Install KVM and libvirt
```bash
sudo dnf install -y qemu-kvm libvirt virt-install bridge-utils
sudo systemctl enable --now libvirtd
```

## 3. Add user to libvirt group
```bash
sudo usermod -aG libvirt $(whoami)
newgrp libvirt
```

## 4. Install minikube
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-latest.x86_64.rpm
sudo rpm -ivh minikube-latest.x86_64.rpm
```

## 5. Install kubectl 
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

## 6. Start minikube 
```bash
minikube start --driver=docker
minikube status
kubectl get nodes
```
![image](https://github.com/user-attachments/assets/5d2ee184-4a63-4dae-a47f-a4e11da93136)
![image](https://github.com/user-attachments/assets/fb9b5a97-7824-4295-8066-f087bf0a7784)




## B) Create app and service Nodeport
```bash
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

>>  Repo 
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

>> Insatll Prometheus + Grafana Stack
```bash
helm install prometheus-stack prometheus-community/kube-prometheus-stack
kubectl get pods   ( all running) 
```
## if you want in namespace 
```bash
helm install prom-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```


## Forwarding to Grafana "Port forward"
```bash
kubectl port-forward svc/prom-stack-grafana 3000:80 -n monitoring
```
## In any web browers  firefox & 
```bash
   http://localhost:3000
```

## D) in grafana 
1- Name : admin
2- Passwd:
```bash
kubectl get secret prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode    (prometheus)
kubectl get secret loki-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode           (loki)
```



## if you want to delete all resources
```bash
 helm uninstall loki-stack
 helm repo remove grafana
 sudo rm /usr/local
```
