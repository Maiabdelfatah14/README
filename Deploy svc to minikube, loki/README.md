####  use grafana in promethues stack 3shan a3ml dashboards by files not gui 
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
import dashboard  , ID  ( 6417 )
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

## to add blackbox to check the service  ( to show up or down )
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
import dashboard  Dashboard ID  ( 13659 )  
```

![image](https://github.com/user-attachments/assets/8d8c56a3-5b5c-4dc5-841d-3a723a5d93c6)



# add Availability (SLI)
```bash

Title: Hello-node Current Availability
stat 
Unit: Percent (%)
Thresholds:
Green: ≥ 99.99
Red: < 99.99
Display: Value only

avg_over_time(
  probe_success{
    instance="http://hello-node:8080",
    job="blackbox-http-probes",
    target_url="http://hello-node:8080"
  }[$__range]
) * 100

-----------------------------------------------------
Title: Hello-node Monthly Availability

Unit: Percent (%)
Thresholds (اختياري):
Green: ≥ 99.99
Red: < 99.99

avg_over_time(
  probe_success{
    instance="http://hello-node:8080",
    job="blackbox-http-probes",
    target_url="http://hello-node:8080"
  }[30d]
) * 100

----------------------------------------------------
Title	Hello-node Service Status
Panel Type  >>  	Stat
Unit  >>  	None
Text Mode  >> 	Value
Value mappings	:
1 → UP	✅
0 → DOWN	✅
Color mappings (optional)	
UP = أخضر	🟢
DOWN = أحمر	🔴

probe_success{
  instance="http://hello-node:8080",
  job="blackbox-http-probes",
  target_url="http://hello-node:8080"
}

```

![image](https://github.com/user-attachments/assets/148c3184-c6f8-455a-8b34-4c05a92cd9e6)

![image](https://github.com/user-attachments/assets/1f032030-7ce2-4fee-87f3-70a616570182)



# B) create deployment again 
```bash
kubectl create deployment hello-node --image=registry.k8s.io/e2e-test-images/agnhost:2.39 -- /agnhost netexec --http-port=8080
kubectl expose deployment hello-node --type=NodePort --port=8080
```

![image](https://github.com/user-attachments/assets/c6cc6a3c-4d70-4192-b742-8b181747ffc6)



## can show in 
```bash
kubectl port-forward svc/blackbox-exporter-prometheus-blackbox-exporter  9115
```
![image](https://github.com/user-attachments/assets/1d1d32dd-bb95-40d2-9713-5481fbd56348)


###  api ( https://run.mocky.io/v3/42810a1a-eb2d-4a62-a8a5-e1a3958c7b37 )
## get token from api by k6 :
```bash
 sudo snap install k6
 vim test.js
 k6 run test.js
 ```
```bash
cat test.js

import http from 'k6/http';
import { check, sleep } from 'k6';


export const options = {
  vus: 1,
  duration: '1s',
  thresholds: {
    'http_req_failed': ['rate<0.01'],     // http errors should be less than 1%
    'http_req_duration': ['p(95)<200'],   // 95% of requests should be below 200ms
    'checks': ['rate>0.99'],   // 99% of checks should pass
  },
};





export default function () {
  // Step 1: Login and get the authentication token
  const loginRes = http.get('https://run.mocky.io/v3/42810a1a-eb2d-4a62-a8a5-e1a3958c7b37', {
    username: 'myuser',
    password: 'mypassword',
  });

  // Check if the login was successful and extract the token
  check(loginRes, {
    'login successful': (res) => res.status === 200,
    'has auth token': (res) => typeof res.json('token') === 'string' && res.json('token').length > 0

  });

  // Extract the token from the response
  const authToken = loginRes.json('token');

  console.log(authToken);
  // Add a small delay to simulate user think time
  sleep(1);

  // Step 2: Fetch the user profile using the token+
  if (authToken) {
    const profileRes = http.get('https://run.mocky.io/v3/42810a1a-eb2d-4a62-a8a5-e1a3958c7b37', {
      headers: {
        'Authorization': `Bearer ${authToken}`,
      },
    });

    // Check if the profile was fetched successfully
    check(profileRes, {
      'profile fetched successfully': (res) => res.status === 200,
      'profile contains user data': (res) => res.json('userId') !== '',
    });
  }
}
```


## integrate with promethues ( use xk6 to integrate by remote write ):
```bash 
go install go.k6.io/xk6/cmd/xk6@latest
xk6 build v0.48.0 --with github.com/grafana/xk6-output-prometheus-remote
```

```bash
cat prometheus-stack-values.yaml
helm upgrade prometheus-stack prometheus-community/kube-prometheus-stack \
  -f prometheus-stack-values.yaml 
```
```bash
grafana:
  enabled: false

prometheus:
  prometheusSpec:
    additionalArgs:
      - name: web.enable-remote-write-receiver

  remoteWrite:
      - url: "http://prometheus-stack-kube-prom-prometheus.default:9090/api/v1/write"
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
```
## in browser 
```bash
kubectl port-forward  svc/prometheus-stack-kube-prom-prometheus 9090:9090
K6_PROMETHEUS_RW_SERVER_URL="http://localhost:9090/api/v1/write" \
./k6 run --out experimental-prometheus-rw test.js

http://localhost:3000
>>>>>>>   2587 , 19665  ( id dashboard )
```
![image](https://github.com/user-attachments/assets/f7e90f9f-d468-461e-94f0-6796cc925b82)
![image](https://github.com/user-attachments/assets/efc13b40-749b-4d1c-b10c-26ca8a6d8ec9)


## dashoard to k6  SLA
```bash

● Panel Type: Stat
● Title: Monthly Availability (SLA)
● Query:
promql
Copy
Edit
(1 - avg_over_time(k6_http_req_failed_rate[30d])) * 100

● Unit:
 Percent (0–100)

● Thresholds:
gren ≥ 99.99
red < 99.99

```
![image](https://github.com/user-attachments/assets/1ba1d585-ac42-4b5e-9195-dc4593ad550d)


## to add alert :

help me in promethues ui  ( http://localhost:9090/)
```bash
quers :
(1 - k6_http_req_failed_rate) * 100

{__name__=~"k6_.*"}

k6_checks_rate
```
```bash
1- name : HighHttpRequestFailureRate

2- query :  max_over_time(k6_http_req_failed_rate[6h])
     B Reduce
         Input  A
         Function   Last     Mode   Strict

     C  Threshold
          Input  B
          Is above   0

3. Set evaluation behavior
   Folder : alert    ,   group : any name
   Pending period  :  1m   ( lw a3d 1m ab3t el alert )

4. Add annotations
  Summary (optional)
     >>>    High HTTP request failure rate
  
  Description (optional)
     >>>   More than 10% of HTTP requests have failed in the last 6 hours

5. Labels and notifications
     Labels
    severity  = critical
```
```bash
1- name : LoginSuccessRateTooLow

2- query :  max_over_time(k6_checks_rate{check="login successful", scenario="default"}[1d]) < 0.9
     B Reduce
         Input  A
         Function   Last     Mode   Strict

     C  Threshold
          Input  B
          Is below   0.9    << important

3. Set evaluation behavior
   Folder : alert    ,   group : any name
   Pending period  :  1m   ( lw a3d 1m ab3t el alert )

4. Add annotations
  Summary (optional)
     >>>    Login success rate is below 90% over the last 1 Day
  
  Description (optional)
     >>>   The maximum login success rate over the past 1 day has dropped below 90%.
  This may indicate repeated login failures or instability in the authentication process.

5. Labels and notifications
     Labels
    severity  = critical
```

```bash
1- name : AuthTokenMissingRateHigh

2- query :  max_over_time(k6_checks_rate{check="has auth token", scenario="default"}[1h]) < 0.95
     B Reduce
         Input  A
         Function   Last     Mode   Strict

     C  Threshold
          Input  B
          Is below   0.95    << important

3. Set evaluation behavior
   Folder : alert    ,   group : any name
   Pending period  :  1m   ( lw a3d 1m ab3t el alert )

4. Add annotations
  Summary (optional)
     >>>    High rate of missing authentication token
  
  Description (optional)
     >>>   The check 'has auth token' has failed more than 5% of the time in the last 5 minutes.
      This may indicate that users are not receiving auth tokens after login.
      Please verify the login service and token generation mechanism..

5. Labels and notifications
     Labels
    severity  = critical
```
![image](https://github.com/user-attachments/assets/9427cee0-0797-48c7-8f8c-4a59f78016da)

