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
# B) create deployment again 
```bash
kubectl create deployment hello-node --image=registry.k8s.io/e2e-test-images/agnhost:2.39 -- /agnhost netexec --http-port=8080
kubectl expose deployment hello-node --type=NodePort --port=8080
```

then 
```bash
kubectl port-forward service/loki-stack-grafana -n default 3000:80
import dashboard  Dashboard ID  ( 13659 )  
```
![image](https://github.com/user-attachments/assets/8d8c56a3-5b5c-4dc5-841d-3a723a5d93c6)
![image](https://github.com/user-attachments/assets/c6cc6a3c-4d70-4192-b742-8b181747ffc6)


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
![image](https://github.com/user-attachments/assets/68048978-c9ab-4290-8198-d8baab98637d)


## can show in 
```bash
kubectl port-forward svc/blackbox-exporter-prometheus-blackbox-exporter  9115
```
![image](https://github.com/user-attachments/assets/1d1d32dd-bb95-40d2-9713-5481fbd56348)


###  api ( https://run.mocky.io/v3/42810a1a-eb2d-4a62-a8a5-e1a3958c7b37 )
## get token from api by k6 : [ alert , SLA ] ( this dont use i update to create my service pod and k6 pod ]
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


## create service to use tool k6  :  [ my-service ]
###  Project Structure: 

my-service/
├── index.js
├── package.json
```bash
1-   mkdir my-service 
2-   cd my-service/
3-   npm init -y
4-  npm install express

5-   cat index.js

const express = require('express');
const app = express();
const PORT = 5000;

// A simple in-memory token (for demo purposes)
let token = null;

// First API: issue a token
app.get('/api/token', (req, res) => {
  token = 'secure-token-123'; // You can use uuid or JWT for production
  res.json({ token });
});

// Second API: validate token
app.get('/api/data', (req, res) => {
  const authHeader = req.headers['authorization'];
  const receivedToken = authHeader?.split(' ')[1]; // Expect: Bearer secure-token-123

  if (receivedToken === token) {
    res.status(200).json({ message: 'Token is valid, access granted.' });
  } else {
    res.status(401).json({ message: 'Invalid or missing token.' });
  }
});

app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});


6- curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
7-  export NVM_DIR="$HOME/.nvm"
8-  source "$NVM_DIR/nvm.sh"
9-  nvm install --lts
10-    node -v      &    npm -v
11-  node index.js    ( to access service )


>>> in new Terminal 

    1- curl http://localhost:5000/api/token
    2-  curl -H "Authorization: Bearer secure-token-123" http://localhost:5000/api/data
```
## to create image and do pod for this service 
```bash
cd  my-service 

1-  # Dockerfile
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 5000
CMD ["node", "index.js"]

>>> eval $(minikube docker-env)    // to run env docker in k8s 
2-   docker build -t my-service:latest .

## create deployment and service 
3-   my-service.yaml :
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-service
  template:
    metadata:
      labels:
        app: my-service
    spec:
      containers:
      - name: my-service
        image: my-service:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: default
spec:
  selector:
    app: my-service
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000


4-  kubectl apply -f my-service.yaml

>>>>    kubectl port-forward service/my-service 5000:5000

           http://localhost:5000/api/token
           http://localhost:5000/api/data
```
```bash
5-   test.js  :

import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 1,
  duration: '1s',
  thresholds: {
    'http_req_failed': ['rate<0.01'],
    'http_req_duration': ['p(95)<200'],
    'checks': ['rate>0.99'],
  },
};

export default function () {
  // Step 1: Get token
  const loginRes = http.get('http://my-service.default.svc.cluster.local:5000/api/token');

  check(loginRes, {
    'login successful': (res) => res.status === 200,
    'has auth token': (res) => typeof res.json('token') === 'string' && res.json('token').length > 0,
  });

  const token = loginRes.json('token');
  sleep(1);

  // Step 2: Access protected route
  if (token) {
    const profileRes = http.get('http://my-service.default.svc.cluster.local:5000/api/data', {
      headers: {
        'Authorization': `Bearer ${token}`,
      },
    });
    check(profileRes, {
      'profile fetched successfully': (res) => res.status === 200,
      'profile contains user data': (res) => {
        const msg = res.json('message');
        return msg && msg.includes('access granted');
      },
    });
  }
}
```
## to do pod for k6 
```bash 
5-  kubectl create configmap k6-test-script \
  --from-file=test.js \
  --namespace=default


6-   k6-loop-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: k6-loop
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: k6-loop
  template:
    metadata:
      labels:
        app: k6-loop
    spec:
      containers:
      - name: k6
        image: grafana/k6:latest
        command: ["/bin/sh"]
        args:
          - -c
          - |
            while true; do
              echo "⏱️ Running k6 test..."
              k6 run --out experimental-prometheus-rw \
                --env K6_PROMETHEUS_RW_SERVER_URL=http://prometheus-stack-kube-prom-prometheus.default:9090/api/v1/write \
                /test/test.js
              echo "✅ Done. Sleeping 1 minute ..."
              sleep 60
            done
        volumeMounts:
          - name: k6-script
            mountPath: /test
      restartPolicy: Always
      volumes:
        - name: k6-script
          configMap:
            name: k6-test-script



7-    kubectl apply -f k6-loop-deployment.yaml

8-   kubectl logs -f -l app=k6-loop
```
##  Delete and create service
```bash
## delete service 
-  kubectl delete deployment my-service
   kubectl delete service my-service

## create service 
   cd my-service
-   kubectl apply -f my-service.yaml
```

## dashboards for SLA K6 [ service : my-service ] :
```bash
>>>  {__name__=~"k6_.*"}   in Prometheus

1-   my-service Current Availability (SLA)
● Panel Type: Stat
● Title:   my-service Current Availability (SLA)
● Query:
avg_over_time(k6_checks_rate{check="login successful"}[1m]) * 100

● Unit:
 Percent (0–100)

● Thresholds:
gren ≥ 99.99
red < 99.99
   
--------------------------------------------------------------------------------
2-   Monthly Availability (SLA)
● Panel Type: Stat
● Title:   Monthly Availability (SLA)
● Query:
avg_over_time(k6_checks_rate{check="login successful"}[30d]) * 100

● Unit:
 Percent (0–100)

● Thresholds:
gren ≥ 99.99
red < 99.99

---------------------------------------------------------------------
3-    Service Status – My Service
● Panel Type: Stat
● Title:   Service Status – My Service
● Query:
avg_over_time(k6_checks_rate{check="login successful"}[1m])

● Unit:
 Percent (0–100)


● Value mappings
1		UP	  green
0		DOWN	   red

● Thresholds:
1    gren 
0    red

--------------------------------------------------------------------------
4-  Service Status Over Time  ( graph )
● Panel Type: time series 
● Title:   Service Status – My Service
● Query:
avg_over_time(k6_checks_rate{check="login successful"}[30d]) 

● Axis :
Soft min
0
Soft max
1


● Unit:
 Percent (0–100)


● Value mappings
1		UP	  green
0		DOWN	   red

● Thresholds:
1    gren 
0    red
```
![image](https://github.com/user-attachments/assets/0cbea7e1-f76d-4987-aeea-3f88565fedde)


## Deploy Grafana oncall in k8s
```bash
1. Add Grafana Helm Repo

 helm repo add grafana https://grafana.github.io/helm-charts
 helm repo update
```
```bash

2. Create Pods DBs

Step 1: Add Repos
    helm repo add bitnami https://charts.bitnami.com/bitnami
    helm repo add grafana https://grafana.github.io/helm-charts 
    helm repo update
```
```bash
Step 2: Install MariaDB
      helm install my-mariadb oci://registry-1.docker.io/bitnamicharts/mariadb \
  --set auth.rootPassword=myStrongMariaPass \
  --set auth.database=oncall \
  --set primary.persistence.size=8Gi \
  --set volumePermissions.enabled=true



Step 3: Verify DB Creation
       kubectl exec -it my-mariadb-0 -- bash
       mysql -uroot -pmyStrongMariaPass
       SHOW DATABASES;          # Ensure 'oncall' database exists


Step 4: Create DB User and Grant Privileges
        kubectl exec -it my-mariadb-0 -- mysql -u root -p      # Enter password: myStrongMariaPass

        CREATE DATABASE oncall;
        CREATE USER 'oncall'@'%' IDENTIFIED BY 'oncall_password';
        GRANT ALL PRIVILEGES ON oncall.* TO 'oncall'@'%';
        FLUSH PRIVILEGES;

Step 5: Get Secret YAML
        kubectl get secret my-mariadb -o yaml
////
apiVersion: v1
data:
  mariadb-root-password: bXlTdHJvbmdNYXJpYVBhc3M=
kind: Secret
metadata:
  annotations:
    meta.helm.sh/release-name: my-mariadb
    meta.helm.sh/release-namespace: default
  creationTimestamp: "2025-07-12T07:15:07Z"
  labels:
    app.kubernetes.io/instance: my-mariadb
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: mariadb
    app.kubernetes.io/part-of: mariadb
    app.kubernetes.io/version: 11.8.2
    helm.sh/chart: mariadb-21.0.3
  name: my-mariadb
  namespace: default
  resourceVersion: "899179"
  uid: 87bf6e82-4285-4d72-9e22-529703ae3aa4
type: Opaque
////

Step 6: Decode Secrets
       kubectl get secret my-mariadb -o jsonpath="{.data.mariadb-root-password}" | base64 -d && echo      ( myStrongMariaPass)
       kubectl get secret my-mariadb -o jsonpath="{.data.mariadb-username}" | base64 -d && echo            (---)
       kubectl get secret my-mariadb -o jsonpath="{.data.mariadb-database}" | base64 -d && echo            (----)


Step 7: Create Probes in Values File
        Edit mariadb-values.yaml:    ( file kber > mariadb-values.yaml )
livenessProbe:
  enabled: true
  initialDelaySeconds: 150
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 10
  successThreshold: 1

readinessProbe:
  enabled: true
  initialDelaySeconds: 150
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 10
  successThreshold: 1



Step 8: Upgrade MariaDB with Values
        helm upgrade my-mariadb bitnami/mariadb -f mariadb-values.yaml \
  --set auth.rootPassword=myStrongMariaPass


Info & Debug
     helm list -A
     helm get values my-mariadb 

## Sample Connection Info

host: my-mariadb.default.svc.cluster.local
port: 3306
db_name: oncall
user: root
password: myStrongMariaPass
existingSecret: my-mariadb
usernameKey: root
passwordKey: mariadb-root-password


## Delete MariaDB

helm uninstall my-mariadb
kubectl delete pvc data-my-mariadb-0

```
```bash
3. Create Pod - RabbitMQ

Step 1: Create Secret
      kubectl create secret generic rabbitmq-secret \
  --from-literal=rabbitmq-username=myuser \
  --from-literal=rabbitmq-password=myStrongRabbitPass \
  --from-literal=rabbitmq-erlang-cookie=mySecretCookie

Step 2: Install RabbitMQ
     helm install my-rabbitmq bitnami/rabbitmq \
  --set existingSecret=rabbitmq-secret \
  --set auth.username=myuser \
  --set auth.password=myStrongRabbitPass \
  --set auth.erlangCookie=mySecretCookie \
  --set persistence.enabled=true

//// kubectl get secret my-rabbitmq -o yaml
apiVersion: v1
data:
  rabbitmq-erlang-cookie: bXlTZWNyZXRDb29raWU=
  rabbitmq-password: bXlTdHJvbmdSYWJiaXRQYXNz
kind: Secret
metadata:
  annotations:
    meta.helm.sh/release-name: my-rabbitmq
    meta.helm.sh/release-namespace: default
  creationTimestamp: "2025-07-11T13:28:41Z"
  labels:
    app.kubernetes.io/instance: my-rabbitmq
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: rabbitmq
    app.kubernetes.io/version: 4.1.2
    helm.sh/chart: rabbitmq-16.0.10
  name: my-rabbitmq
  namespace: default
  resourceVersion: "546668"
  uid: 0f48fbd8-324b-4bcc-81e5-1ab960b3ceb2
type: Opaque
/////

Step 3: Decode Secrets
     kubectl get secret my-rabbitmq -o jsonpath="{.data.rabbitmq-password}" | base64 -d && echo   (myStrongRabbitPass)
     kubectl get secret my-rabbitmq -o jsonpath="{.data.rabbitmq-username}" | base64 -d && echo   (--------)
     kubectl get secret my-rabbitmq -o yaml            (----)


Step 4: Check Pod
     kubectl get pods
     kubectl describe pod my-rabbitmq-0


Step 5: Access RabbitMQ UI
       kubectl port-forward svc/my-rabbitmq 15672:15672
Then open:
       http://localhost:15672


Info & Debug
  helm list -A
  helm get values my-rabbitmq

## Sample Connection Info in first project 
host: my-rabbitmq.default.svc.cluster.local
port: 5672
user: myuser
password: myStrongRabbitPass
existingSecret: my-rabbitmq
passwordKey: rabbitmq-password
usernameKey: myuser

## in final project
USER-SUPPLIED VALUES:
auth:
  erlangCookie: mySecretCookie
  password: myStrongRabbitPass
  username: myuser
existingSecret: rabbitmq-secret
persistence:
  enabled: true


Delete RabbitMQ
    helm uninstall my-rabbitmq
    kubectl delete pvc data-my-rabbitmq-0
```

```bash
4. Create Pod - Redis  ( dont need username )   

Step 1: Install Redis

helm upgrade --install my-redis bitnami/redis \
  --set auth.enabled=true \
  --set auth.password=myStrongRedisPass \
  --set architecture=standalone \
  --set master.persistence.enabled=true


Step 2: Create Redis Secret
  kubectl create secret generic redis-auth-secret \
    --from-literal=redis-password=myStrongRedisPass

//////// kubectl get secret my-redis -o yaml
apiVersion: v1
data:
  redis-password: bXlTdHJvbmdSZWRpc1Bhc3M=
kind: Secret
metadata:
  annotations:
    meta.helm.sh/release-name: my-redis
    meta.helm.sh/release-namespace: default
  creationTimestamp: "2025-07-14T11:09:23Z"
  labels:
    app.kubernetes.io/instance: my-redis
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: redis
    app.kubernetes.io/version: 8.0.3
    helm.sh/chart: redis-21.2.12
  name: my-redis
  namespace: default
  resourceVersion: "891039"
  uid: eabf1b5d-3f63-400e-8d45-164816f1cd97
type: Opaque
//////

Step 3:  cat redis-values.yaml

architecture: standalone

replica:
  replicaCount: 0

master:
  extraEnvVars:
    - name: REDIS_PORT
      value: "6379"

  livenessProbe:
    initialDelaySeconds: 40
    periodSeconds: 20
    timeoutSeconds: 15
    failureThreshold: 5

  readinessProbe:
    initialDelaySeconds: 40
    periodSeconds: 20
    timeoutSeconds: 15
    failureThreshold: 5

  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: 500m
      memory: 1Gi


Step 4: Upgrade Redis with Values
        helm upgrade my-redis bitnami/redis -f redis-values.yaml


Info & Debug
   helm list -A
   helm get values my-redis

## Sample Values in first project
architecture: standalone
auth:
  enabled: true
  password: myStrongRedisPass
master:
  persistence:
    enabled: true


## in final project
USER-SUPPLIED VALUES:
architecture: standalone
master:
  extraEnvVars:
  - name: REDIS_PORT
    value: "6379"
  livenessProbe:
    failureThreshold: 5
    initialDelaySeconds: 40
    periodSeconds: 20
    timeoutSeconds: 15
  readinessProbe:
    failureThreshold: 5
    initialDelaySeconds: 40
    periodSeconds: 20
    timeoutSeconds: 15
  resources:
    limits:
      cpu: 500m
      memory: 1Gi
    requests:
      cpu: 250m
      memory: 512Mi
replica:
  replicaCount: 0 
```

```bash
5. Step 1: Install OnCall via Helm

        helm upgrade --install grafana-oncall grafana/oncall -f oncall-values.yaml
        kubectl get deploy -n default | grep grafana
        kubectl rollout restart deployment grafana-oncall


Delete OnCall
helm uninstall my-oncall
kubectl delete $(kubectl get pods -o name | grep my-oncall-engine-migrate)

🔐 Configure Admin & Access OnCall
Step 2: Generate Secret Key
       openssl rand -base64 32  # Use this in oncall-values.yaml


Step 3: Create Admin Secret  ( to login in pod Grafana-oncall  (username = admin , password = admin\_password) )
      kubectl create secret generic grafana-oncall \
  --from-literal=admin-user=admin \
  --from-literal=admin-password=admin_password \
  --namespace=default \
  --save-config \
  --dry-run=client -o yaml | \
  kubectl label --local -f - "app.kubernetes.io/managed-by=Helm" -o yaml | \
  kubectl annotate --local -f - \
    meta.helm.sh/release-name=grafana-oncall \
    meta.helm.sh/release-namespace=default -o yaml | \
  kubectl apply -f -


Step 4: Edit oncall-values.yaml
        vim oncall-values.yaml

Step 5: Upgrade MariaDB (again with correct values)

     helm show values bitnami/mariadb > mariadb-values.yaml
     vim mariadb-values.yaml      # (Ensure timeoutSeconds: 5 for livenessProbe ,  readinessProbe )

     export MARIADB_ROOT_PASSWORD=$(kubectl get secret --namespace default my-mariadb -o jsonpath="{.data.mariadb-root-password}" | base64 -d)


helm upgrade my-mariadb bitnami/mariadb \
  -f mariadb-values.yaml \
  --set auth.rootPassword="$MARIADB_ROOT_PASSWORD" \
  --set auth.database=oncall \
  --set auth.username=oncall \
  --set auth.password=oncall_password



Step 6: Port Forward Access

# Frontend:
kubectl port-forward svc/grafana-oncall 9999:80
or 
minikube service grafana-oncall --url

# Backend:
kubectl port-forward svc/grafana-oncall-engine 8888:8080

Step 7: Health Probes
        url http://localhost:8080/health/
        curl http://localhost:8080/ready/
        curl http://localhost:8080/api/internal/v1/health/


Step 8: Add Probes to OnCall Engine Deployment

livenessProbe:
  failureThreshold: 3
  httpGet:
    path: /health/
    port: http
    scheme: HTTP
  initialDelaySeconds: 30
  periodSeconds: 60
  timeoutSeconds: 3
  successThreshold: 1

readinessProbe:
  failureThreshold: 3
  httpGet:
    path: /ready/
    port: http
    scheme: HTTP
  initialDelaySeconds: 30
  periodSeconds: 60
  successThreshold: 1

startupProbe:
  failureThreshold: 10
  httpGet:
    path: /startupprobe/
    port: http
    scheme: HTTP
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 3
  successThreshold: 1


Step 9: View OnCall Logs
     kubectl logs -f grafana-oncall-<pod> -n default -c Grafana
```
## 🔑 Integrate Grafana with OnCall API   ( 3shan grafana t connect m3 oncall )
```bash
Step 1: Create Service Account YAML

    File: grafana-oncall-integration-sa.yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: grafana-oncall-integration
  namespace: default


Step 2: Create Token for Service Account

     File: grafana-oncall-integration-token.yaml

apiVersion: v1
kind: Secret
metadata:
  name: grafana-oncall-integration-token
  annotations:
    kubernetes.io/service-account.name: grafana-oncall-integration
type: kubernetes.io/service-account-token



Step 3: Extract the Token from Kubernetes

     kubectl get secret grafana-oncall-integration-token -n default -o jsonpath="{.data.token}" | base64 -d

                    //////////  eyJhbGciOiJSUzI1NiIsImtpZCI6IksxNmRVVzFaRU5uV2tMazlrVGJnNEJWLU5JZXFRQnRlQzF2VG1CSG92M00ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImdyYWZhbmEtb25jYWxsLWludGVncmF0aW9uLXRva2VuIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImdyYWZhbmEtb25jYWxsLWludGVncmF0aW9uIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiMWI3ZWNjZTktNzU2Ni00NmRhLWJlMzEtMWVlYTQ4ZTBiNmMyIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlZmF1bHQ6Z3JhZmFuYS1vbmNhbGwtaW50ZWdyYXRpb24ifQ.uD3r9b8QwhYYKGYHDCzmpXcMGuaWphY-8VhqfqmMRdbXgVi2xa2Cx4kng2YujEn5keeoTz0UGcMIrfs6WtnO0VCT_mhAvWeLHkvE9cXgdM3BB61E6t6kSNpKd2u9PSgqRUwoFVhT_QrmhY_s8UmLEdNer7RuBLd9Eyw55DvXYkroXbx8SEdkosJV7z7uV1iQg-nxJwU0VXC9PebRM11ZZIQoEnCpBIrmRu7pN-IZdy5AJkaQp0gkG5Qec_Yrw6qpaPgAeZfkcjCNxKlGleYjlRAleHdgePMQYmw-eLOnkZVdZfbG8BRDufCeEMTgzyvHdj_YLNnEEXczJhwpFHSpcg  /////////



Step 4: Save Token in a Secret

kubectl create secret generic grafana-oncall-token \
  --from-literal=ONCALL_API_TOKEN="<insert_token_here>" \
  -n default

# View it
    kubectl get secret grafana-oncall-token -o yaml -n default

apiVersion: v1
data:
  ONCALL_API_TOKEN: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklrc3hObVJWVnpGYVJVNXVWMnRNYXpsclZHSm5ORUpXTFU1SlpYRlJRblJsUXpGMlZHMUNTRzkyTTAwaWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUprWldaaGRXeDBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpXTnlaWFF1Ym1GdFpTSTZJbWR5WVdaaGJtRXRiMjVqWVd4c0xXbHVkR1ZuY21GMGFXOXVMWFJ2YTJWdUlpd2lhM1ZpWlhKdVpYUmxjeTVwYnk5elpYSjJhV05sWVdOamIzVnVkQzl6WlhKMmFXTmxMV0ZqWTI5MWJuUXVibUZ0WlNJNkltZHlZV1poYm1FdGIyNWpZV3hzTFdsdWRHVm5jbUYwYVc5dUlpd2lhM1ZpWlhKdVpYUmxjeTVwYnk5elpYSjJhV05sWVdOamIzVnVkQzl6WlhKMmFXTmxMV0ZqWTI5MWJuUXVkV2xrSWpvaU1XSTNaV05qWlRrdE56VTJOaTAwTm1SaExXSmxNekV0TVdWbFlUUTRaVEJpTm1NeUlpd2ljM1ZpSWpvaWMzbHpkR1Z0T25ObGNuWnBZMlZoWTJOdmRXNTBPbVJsWm1GMWJIUTZaM0poWm1GdVlTMXZibU5oYkd3dGFXNTBaV2R5WVhScGIyNGlmUS51RDNyOWI4UXdoWVlLR1lIREN6bXBYY01HdWFXcGhZLThWaHFmcW1NUmRiWGdWaTJ4YTJDeDRrbmcyWXVqRW41a2Vlb1R6MFVHY01JcmZzNld0bk8wVkNUX21oQXZXZUxIa3ZFOWNYZ2RNM0JCNjFFNnQ2a1NOcEtkMnU5UFNncVJVd29GVmhUX1FybWhZX3M4VW1MRWROZXI3UnVCTGQ5RXl3NTVEdlhZa3JvWGJ4OFNFZGtvc0pWN3o3dVYxaVFnLW54SndVMFZYQzlQZWJSTTExWlpJUW9FbkNwQklybVJ1N3BOLUlaZHk1QUprYVFwMGdrRzVRZWNfWXJ3NnFwYVBnQWVaZmtjakNOeEtsR2xlWWpsUkFsZUhkZ2VQTVFZbXctZUxPbmtaVmRaZmJHOEJSRHVmQ2VFTVRnenl2SGRqX1lMTm5FRVhjekpod3BGSFNwY2c=
kind: Secret
metadata:
  creationTimestamp: "2025-07-22T11:47:08Z"
  name: grafana-oncall-token
  namespace: default
  resourceVersion: "884826"
  uid: b067e451-e9c0-4552-aec1-5299fc8423ab
type: Opaque
```
## da integrate oncall m3 grafana 
## Grafana يعرف يوصل لـ OnCall API → محتاج ONCALL_API_TOKEN.
## OnCall يعرف يوصل لـ Grafana API → محتاج GRAFANA_API_TOKEN.
```bash

1-   kubectl create configmap grafana-oncall-env \
  --from-literal=GRAFANA_URL=http://127.0.0.1:42647 \
  --from-literal=GRAFANA_API_TOKEN=glsa_QGSAevZs1M4u57HZZwCzydVAknKd7tc8_9dc7632a \
  --from-literal=SMTP__HOST=smtp.gmail.com \
  --from-literal=SMTP__PORT=587 \
  --from-literal=SMTP__USER=maiabdelfatah077@gmail.com \
  --from-literal=SMTP__PASSWORD=cwhnoerrsfiudufs \
  --from-literal=SMTP__USE_TLS=true \
  --from-literal=SMTP__FROM=maiabdelfatah077@gmail.com \
  --from-literal=ONCALL__AUTH__DISABLE_AUTH=true \
  -n default


2- kubectl get configmap grafana-oncall-env -n default -o yaml
apiVersion: v1
data:
  GRAFANA_API_TOKEN: glsa_QGSAevZs1M4u57HZZwCzydVAknKd7tc8_9dc7632a
  ONCALL__AUTH__DISABLE_AUTH: "true"
  SMTP__FROM: maiabdelfatah077@gmail.com
  SMTP__HOST: smtp.gmail.com
  SMTP__PASSWORD: cwhnoerrsfiudufs
  SMTP__PORT: "587"
  SMTP__USE_TLS: "true"
  SMTP__USER: maiabdelfatah077@gmail.com
kind: ConfigMap
metadata:
  creationTimestamp: "2025-08-06T17:45:58Z"
  name: grafana-oncall-env
  namespace: default
  resourceVersion: "1282750"
  uid: 40cf8e61-03da-4602-aafe-32f415291e19


2-   helm upgrade --install grafana-oncall grafana/oncall -f oncall-values.yaml


3- test
  ##  cat test_email.py

import smtplib

from_email = "maiabdelfatah077@gmail.com"
to_email = "maiabdelfatah077@gmail.com"
password = "cwhnoerrsfiudufs"

message = """Subject: Test from script

This is a test email with debug log.
"""

try:
    print("[INFO  ] Connecting to SMTP server...")
    server = smtplib.SMTP("smtp.gmail.com", 587)
    server.starttls()
    print("[INFO  ] Logging in...")
    server.login(from_email, password)
    print(f"[INFO  ] Sending email to: ['{to_email}']")
    server.sendmail(from_email, to_email, message)
    print(f"[DEBUG ] Email '{message.strip()}' successfully sent to ['{to_email}']")
    server.quit()
except Exception as e:
    print("[ERROR ]", str(e))


run file >  python3 test_email.py

## from Home > Administration  > Users and access >  Service accounts
oncall-smtp-token >> glsa_QGSAevZs1M4u57HZZwCzydVAknKd7tc8_9dc7632a
```


##📧 Configure SMTP for Email Notifications
```bash

1- STMP to send alerts to mail 
>> app password ( https://myaccount.google.com/apppasswords?rapt=AEjHL4NaFh2Y4DURK4iabkp8QtlPI-yvyjpS8cuSj9f5YFvs9oYEMFt0IuLIpyDhV6isdRMUsuIIxhvdvTK_Nm0euMUo-WgAd-dzKJS69CzfKXJlRjVAlnk)

2- kubectl create secret generic grafana-oncall-smtp \
  --from-literal=user='maiabdelfatah077@gmail.com' \
  --from-literal=password='cwhnoerrsfiudufs' \         ( use app passwd not passwd mail )
  -n default \
  --dry-run=client -o yaml > updated-smtp-secret.yaml


>> kubectl apply -f updated-smtp-secret.yaml

>>  kubectl rollout restart deployment grafana-oncall-engine -n default

>> kubectl get secret grafana-oncall-smtp -n default -o yaml

apiVersion: v1
data:
  password: Y3dobiBvZXJyIHNmaXUgZHVmcw==
  user: bWFpYWJkZWxmYXRhaDA3N0BnbWFpbC5jb20=
kind: Secret
metadata:
  creationTimestamp: "2025-07-30T08:48:30Z"
  name: grafana-oncall-smtp
  namespace: default
  resourceVersion: "1031745"
  uid: 8896ce13-69db-4225-97c1-e5e9a253656f
type: Opaque



>> kubectl edit deployment grafana-oncall-engine -n default

## delete 
- name: EMAIL_HOST
- name: EMAIL_FROM_ADDRESS
- name: EMAIL_HOST_USER


>>   cat   patch-email.json

[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/env/-",
    "value": {
      "name": "EMAIL_HOST",
      "value": "smtp.gmail.com"
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/env/-",
    "value": {
      "name": "EMAIL_PORT",
      "value": "587"
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/env/-",
    "value": {
      "name": "EMAIL_USE_TLS",
      "value": "True"
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/env/-",
    "value": {
      "name": "EMAIL_USE_SSL",
      "value": "False"
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/env/-",
    "value": {
      "name": "EMAIL_BACKEND",
      "value": "django.core.mail.backends.smtp.EmailBackend"
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/env/-",
    "value": {
      "name": "EMAIL_HOST_USER",
      "valueFrom": {
        "secretKeyRef": {
          "name": "grafana-oncall-smtp",
          "key": "user"
        }
      }
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/env/-",
    "value": {
      "name": "EMAIL_HOST_PASSWORD",
      "valueFrom": {
        "secretKeyRef": {
          "name": "grafana-oncall-smtp",
          "key": "password"
        }
      }
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/env/-",
    "value": {
      "name": "EMAIL_FROM_ADDRESS",
      "value": "maiabdelfatah077@gmail.com"
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/env/-",
    "value": {
      "name": "FEATURE_EMAIL_INTEGRATION_ENABLED",
      "value": "True"
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/env/-",
    "value": {
      "name": "LOG_EMAIL_NOTIFICATIONS",
      "value": "true"
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/env/-",
    "value": {
      "name": "EMAIL_NOTIFICATIONS_LIMIT",
      "value": "200"
    }
  }
]


>>   kubectl patch deploy grafana-oncall-celery -n default --type='json' -p "$(cat patch-email.json)"

>>  kubectl patch deployment grafana-oncall-engine -n default --type=json -p="$(cat patch-email.json)"


4- 	Finish Setup in Grafana UI  ( IMPORTANT)
•  Go to Home → Alerts & IRM → OnCall → Settings → Env Variables.
•  Add the same SMTP values from above.

## get mail & logs mail 

kubectl exec -it deploy/grafana-oncall-engine -n default -c oncall -- env | grep -Ei 'EMAIL|SMTP'

kubectl exec -it deploy/grafana-oncall-celery -n default -c oncall -- env | grep -Ei 'EMAIL|SMTP'

kubectl logs -n default -c oncall deploy/grafana-oncall-celery --since=10m | grep -Ei "email|error"


## test
python3 -c "import smtplib; s=smtplib.SMTP('smtp.gmail.com',587); s.starttls(); s.login('maiabdelfatah077@gmail.com','cwhn oerr sfiu dufs'); s.sendmail('maiabdelfatah077@gmail.com','maiabdelfatah077@gmail.com','Subject: Test\n\nThis is a test email from CLI'); s.quit()"

```

## check  url integration
```bash
>>  ## by3ml url helm-test when connect in Grafana gui
>> kubectl edit configmap helm-testing-grafana-plugin-provisioning -n default


1-   url in integration ( mn example.com > grafana-oncall-engine.default.svc.cluster.local:8080 )

kubectl patch deployment grafana-oncall-engine -n default \
  --type=json \
  -p='[
    {
      "op": "replace",
      "path": "/spec/template/spec/containers/0/env/0/value",
      "value": "http://grafana-oncall-engine.default.svc.cluster.local:8080"
    }
  ]'

>> kubectl rollout restart deployment grafana-oncall-engine -n default
>> kubectl exec -it deploy/grafana-oncall-engine -n default -c oncall -- printenv | grep BASE_URL


## optional
2- update value DJANGO_SETTINGS_MODULE in initContainer

kubectl patch deployment grafana-oncall-engine -n default \
  --type=json \
  -p='[
    {
      "op": "replace",
      "path": "/spec/template/spec/initContainers/0/env/4/value",
      "value": "settings.helm"
    }
  ]'

## get url
kubectl exec -it deploy/grafana-oncall-engine -n default -c oncall -- printenv | grep BASE_URL
```

## 🧹 Delete All Resources
```bash
1. Uninstall OnCall
    helm uninstall my-oncall

2. Uninstall MariaDB & RabbitMQ
    helm uninstall my-mariadb
    helm uninstall my-rabbitmq

3. Delete PVCs
kubectl delete pvc -l app.kubernetes.io/instance=my-oncall
kubectl delete pvc -l app.kubernetes.io/instance=my-mariadb
kubectl delete pvc -l app.kubernetes.io/instance=my-rabbitmq


4. Delete Released PVs
kubectl get pv | grep Released
kubectl delete pv <pv-name>  # Replace with actual PV name


5. Delete Secrets
kubectl delete secret my-oncall-mariadb my-oncall-RabbitMQ
```
## files
```bash
-- cat oncall-env.yaml    ( يعطل الأوثنتيكيشن (Authentication) )

apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-oncall-env
  namespace: default
data:
  ONCALL__AUTH__DISABLE_AUTH: "true"


 1- cat oncall-values.yaml

replicaCount: 1

stack:
  enabled: true

serviceAccount:
  create: false
  name: grafana-oncall-integration

grafanaOnCall:
  extraEnv:
    - name: DJANGO_LOG_LEVEL
      value: DEBUG
  envFrom:
    - configMapRef:
        name: grafana-oncall-env
  secretKey: "coY9y/FY/jvjtqv4wUi0CmStjnRv/7V1y6TtEm4wQWg="
  baseUrl: "http://grafana-oncall-engine.default.svc.cluster.local:8080"
  features:
    incident: false
    labels: false


grafana:
  enabled: true
  install: true
  livenessProbe:
    httpGet:
      path: /api/health
      port: 3000
    initialDelaySeconds: 90
    timeoutSeconds: 30
    periodSeconds: 15
    failureThreshold: 10

  readinessProbe:
    httpGet:
      path: /api/health
      port: 3000
    initialDelaySeconds: 30
    timeoutSeconds: 5
    periodSeconds: 10
    failureThreshold: 5

  service:
    type: NodePort
  admin:
    existingSecret: grafana-oncall
    userKey: admin-user
    passwordKey: admin-password
  envFromSecret: grafana-oncall-token
  smtp:
    existingSecret: grafana-oncall-smtp
    userKey: user
    passwordKey: password
  grafana.ini:
    server:
      root_url: http://grafana-oncall.default.svc.cluster.local
    smtp:
      enabled: true
      host: smtp.gmail.com:587
      user: maiabdelfatah077@gmail.com
      from_address: maiabdelfatah077@gmail.com
      from_name: "Grafana OnCall"
      skip_verify: true
      startTLS_policy: "Always"

    plugins:
      oncall:
        oncallApiUrl: "http://grafana-oncall-engine.default.svc.cluster.local:8080"
        apiToken: __ONCALL_API_TOKEN__
  envValueFrom:
    ONCALL_API_TOKEN:
      secretKeyRef:
        name: grafana-oncall-token
        key: ONCALL_API_TOKEN

  additionalDataSources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-stack-kube-prom-prometheus.default.svc.cluster.local
      access: proxy
      isDefault: true

  sidecar:
    dashboards:
      enabled: true
    datasources:
      enabled: true
mariadb:
  enabled: false

rabbitmq:
  enabled: false

redis:
  enabled: false

externalMysql:
  host: my-mariadb.default.svc.cluster.local
  port: 3306
  database: oncall
  user: oncall
  password: "oncall_password"

externalRabbitmq:
  host: my-rabbitmq.default.svc.cluster.local
  port: 5672
  user: myuser
  password: myStrongRabbitPass

externalRedis:
  host: my-redis-master.default.svc.cluster.local
  port: 6379
  password: myStrongRedisPass

backend:
  enabled: true
  ingress:
    enabled: false
  service:
    type: ClusterIP
  extraEnv:
    - name: BASE_URL
      value: "http://grafana-oncall-engine.default.svc.cluster.local:8080"
    - name: REDIS_URL
      value: "redis://:myStrongRedisPass@my-redis-master.default.svc.cluster.local:6379/0"
    - name: ONCALL__AUTH__DISABLE_AUTH
      value: "true"

  # UWSGI worker stability fix
  uwsgi:
    max-worker-lifetime: 0
    lazy-apps: true

  # Improve probes to reduce false failures
  livenessProbe:
    initialDelaySeconds: 150
    timeoutSeconds: 10
    periodSeconds: 30
    failureThreshold: 20

  readinessProbe:
    initialDelaySeconds: 150
    timeoutSeconds: 10
    periodSeconds: 30
    failureThreshold: 20

  resources:
    requests:
      memory: "2Gi"
      cpu: "1"
    limits:
      memory: "4Gi"
      cpu: "2"


frontend:
  enabled: true

cert_issuer:
  enabled: false

certIssuer:
  enabled: false

cert-manager:
  enabled: false




pluginConfig:
  enabled: true
  configMapName: grafana-oncall-app-config
  configMap:
    grafana-oncall-app-provisioning.yaml: |
      apps:
        - type: grafana-oncall-app
          name: grafana-oncall-app
          disabled: false
          jsonData:
            stackId: 1
            orgId: 1
            baseUrl: http://grafana-oncall.default.svc.cluster.local:8080
    contactpoints.yaml: |
      contactPoints:
        - orgId: 1
          name: 'OnCall: DevOps Team'
          receivers:
            - uid: oncall-devops
              type: oncall
              settings:
                url: http://grafana-oncall-engine.default.svc.cluster.local/integrations/v1/grafana_alerting/a13516ff-2be1-4b7e-9bdd-839168ee6651

engine:
  extraEnvFrom:
    - configMapRef:
        name: grafana-oncall-env

  env:
    - name: GRAFANA_API_URL
      value: http://grafana-oncall.default.svc.cluster.local
    - name: DJANGO_LOG_LEVEL
      value: DEBUG
    - name: DJANGO_SETTINGS_MODULE
      value: settings.helm
    - name: BASE_URL
      value: "http://grafana-oncall-engine.default.svc.cluster.local:8080"
    - name: GRAFANA_URL
      value: "http://grafana-oncall.default.svc.cluster.local"
    - name: SMTP_HOST
      value: smtp.gmail.com
    - name: SMTP_PORT
      value: "587"
    - name: SMTP_USER
      valueFrom:
        secretKeyRef:
          name: grafana-oncall-smtp
          key: user
    - name: SMTP_PASSWORD
      valueFrom:
        secretKeyRef:
          name: grafana-oncall-smtp
          key: password
    - name: EMAIL_FROM
      value: maiabdelfatah077@gmail.com


  extraPatches:
    - target:
        kind: Deployment
        name: grafana-oncall-engine
      patch: |
        - op: replace
          path: /spec/template/spec/initContainers/0/env/0/value
          value: "http://grafana-oncall-engine.default.svc.cluster.local:8080"

config:
  oncall:
    SMTP_ENABLED: true
    SMTP_HOST: smtp.gmail.com
    SMTP_PORT: 587
    SMTP_USER: ${SMTP_USER}
    SMTP_PASSWORD: ${SMTP_PASSWORD}
    SMTP_FROM: maiabdelfatah077@gmail.com
    SMTP_FROM_NAME: "Grafana OnCall"
    SMTP_STARTTLS: true
    SMTP_SKIP_VERIFY: true

  env:
    - name: SMTP_USER
      valueFrom:
        secretKeyRef:
          name: grafana-oncall-smtp
          key: user
    - name: SMTP_PASSWORD
      valueFrom:
        secretKeyRef:
          name: grafana-oncall-smtp
          key: password
```
```bash
 2- cat  grafana-oncall-configmap.yaml    ( grafana.ini >> to setting grafana )
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-oncall
  namespace: default
  labels:
    app.kubernetes.io/instance: grafana-oncall
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: grafana
    app.kubernetes.io/version: 11.1.4
    helm.sh/chart: grafana-8.4.6
  annotations:
    meta.helm.sh/release-name: grafana-oncall
    meta.helm.sh/release-namespace: default
data:
  grafana.ini: |
    [analytics]
    check_for_updates = true

    [feature_toggles]
    accessControlOnCall = false
    enable = externalServiceAccounts

    [grafana_net]
    url = https://grafana.net

    [log]
    mode = console

    [paths]
    data = /var/lib/grafana/
    logs = /var/log/grafana
    plugins = /var/lib/grafana/plugins
    provisioning = /etc/grafana/provisioning

    [plugins]
    oncall.apiToken = ${ONCALL_API_TOKEN}
    oncall.oncallApiUrl = "http://grafana-oncall.default.svc.cluster.local:8080"


    [server]
    domain = grafana-oncall.default.svc.cluster.local
    root_url = %(protocol)s://%(domain)s/
    serve_from_sub_path = false

    [smtp]
    enabled = true
    from_address = maiabdelfatah077@gmail.com
    from_name = Grafana OnCall
    host = smtp.gmail.com:587
    skip_verify = true
    startTLS_policy = Always
    user = maiabdelfatah077@gmail.com

  plugins: grafana-oncall-app
```
```bash
3- kubectl edit configmap helm-testing-grafana-plugin-provisioning -n default      ( to connect Grafana m3 plugin oncall )

apiVersion: v1
data:
  grafana-oncall-app-provisioning.yaml: |
    apps:
      - type: grafana-oncall-app
        name: grafana-oncall-app
        disabled: false
        jsonData:
          stackId: 5
          orgId: 100
          onCallApiUrl: http://helm-testing-oncall-engine:8080
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"grafana-oncall-app-provisioning.yaml":"\napps:\n  - type: grafana-oncall-app\n    name: grafana-oncall-app\n    disabled: false\n    jsonData:\n      stackId: 5\n      orgId: 100\n      onCallApiUrl: http://grafana-oncall-engine.default.svc.cluster.local:8080\n\nintegrations:\n  - type: grafana_alerting\n    name: grafana_alerting\n    disabled: false\n    jsonData:\n      url: \"http://grafana-oncall-engine.default.svc.cluster.local:8080/integrations/v1/grafana_alerting/nxNZoy2XkTSKUXaZKO5hcn7PT/\"\n"},"kind":"ConfigMap","metadata":{"annotations":{},"creationTimestamp":null,"name":"helm-testing-grafana-plugin-provisioning","namespace":"default"}}
    meta.helm.sh/release-name: grafana-oncall
    meta.helm.sh/release-namespace: default
  creationTimestamp: "2025-07-22T12:38:53Z"
  labels:
    app: oncall
    app.kubernetes.io/managed-by: Helm
  name: helm-testing-grafana-plugin-provisioning
  namespace: default
  resourceVersion: "954180"
  uid: 3aa51999-3a16-4633-b59c-50a067c3ac34
```
```bash
4-  cat grafana-oncall-engine-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana-oncall-engine
  namespace: default
  labels:
    app.kubernetes.io/component: engine
    app.kubernetes.io/instance: grafana-oncall
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: oncall
    app.kubernetes.io/version: v1.16.4
    helm.sh/chart: oncall-1.16.4
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: engine
      app.kubernetes.io/instance: grafana-oncall
      app.kubernetes.io/name: oncall
  template:
    metadata:
      labels:
        app.kubernetes.io/component: engine
        app.kubernetes.io/instance: grafana-oncall
        app.kubernetes.io/name: oncall
    spec:
      serviceAccountName: grafana-oncall-integration
      containers:
        - name: oncall
          image: grafana/oncall:v1.16.4
          imagePullPolicy: Always
          envFrom:
            - configMapRef:
                name: grafana-oncall-env
          env:
            - name: DJANGO_SETTINGS_MODULE
              value: settings.helm
            - name: ONCALL__AUTH__DISABLE_AUTH
              value: "true"
            - name: SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: grafana-oncall
                  key: SECRET_KEY
            - name: ONCALL_DB__HOST
              value: my-mariadb.default.svc.cluster.local
            - name: ONCALL_DB__PORT
              value: "3306"
            - name: ONCALL_DB__NAME
              value: oncall
            - name: ONCALL_DB__USER
              value: oncall
            - name: ONCALL_DB__PASSWORD
              valueFrom:
                secretKeyRef:
                  name: grafana-oncall-mysql-external
                  key: mariadb-root-password
            - name: ONCALL_REDIS__PROTOCOL
              value: redis
            - name: ONCALL_REDIS__HOST
              value: my-redis-master.default.svc.cluster.local
            - name: ONCALL_REDIS__PORT
              value: "6379"
            - name: ONCALL_REDIS__PASSWORD
              valueFrom:
                secretKeyRef:
                  name: grafana-oncall-redis-external
                  key: redis-password
            - name: ONCALL_RABBITMQ__HOST
              value: my-rabbitmq.default.svc.cluster.local
            - name: ONCALL_RABBITMQ__PORT
              value: "5672"
            - name: ONCALL_RABBITMQ__USER
              value: myuser
            - name: ONCALL_RABBITMQ__PASSWORD
              valueFrom:
                secretKeyRef:
                  name: grafana-oncall-rabbitmq-external
                  key: rabbitmq-password
            - name: LOG_EMAIL_NOTIFICATIONS
              value: "true"
          ports:
            - containerPort: 8080
              name: http
          readinessProbe:
            httpGet:
              path: /ready/
              port: http
            initialDelaySeconds: 30
            periodSeconds: 60
            timeoutSeconds: 3
            successThreshold: 1
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health/
              port: http
            initialDelaySeconds: 30
            periodSeconds: 60
            timeoutSeconds: 3
            successThreshold: 1
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /ready/
              port: http
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 3
            successThreshold: 1
            failureThreshold: 10

      initContainers:
        - name: wait-for-db
          image: grafana/oncall:v1.16.4
          imagePullPolicy: Always
          command:
            - sh
            - -c
            - until (python manage.py migrate --check); do echo Waiting for database migrations; sleep 2; done
          env:
            - name: DJANGO_SETTINGS_MODULE
              value: settings.helm
            - name: SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: grafana-oncall
                  key: SECRET_KEY
            - name: MIRAGE_SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: grafana-oncall
                  key: MIRAGE_SECRET_KEY
            - name: MIRAGE_CIPHER_IV
              value: 1234567890abcdef
            - name: BASE_URL
              value: http://example.com
            - name: AMIXR_DJANGO_ADMIN_PATH
              value: admin
            - name: OSS
              value: "True"
            - name: DETACHED_INTEGRATIONS_SERVER
              value: "False"
            - name: UWSGI_LISTEN
              value: "1024"
            - name: BROKER_TYPE
              value: rabbitmq
            - name: GRAFANA_API_URL
              value: http://grafana-oncall-grafana
            - name: ONCALL_DB__HOST
              value: my-mariadb.default.svc.cluster.local
            - name: ONCALL_DB__PORT
              value: "3306"
            - name: ONCALL_DB__NAME
              value: oncall
            - name: ONCALL_DB__USER
              value: oncall
            - name: ONCALL_DB__PASSWORD
              valueFrom:
                secretKeyRef:
                  name: grafana-oncall-mysql-external
                  key: mariadb-root-password
            - name: ONCALL_REDIS__PROTOCOL
              value: redis
            - name: ONCALL_REDIS__HOST
              value: my-redis-master.default.svc.cluster.local
            - name: ONCALL_REDIS__PORT
              value: "6379"
            - name: ONCALL_REDIS__PASSWORD
              valueFrom:
                secretKeyRef:
                  name: grafana-oncall-redis-external
                  key: redis-password
            - name: ONCALL_RABBITMQ__HOST
              value: my-rabbitmq.default.svc.cluster.local
            - name: ONCALL_RABBITMQ__PORT
              value: "5672"
            - name: ONCALL_RABBITMQ__USER
              value: myuser
            - name: ONCALL_RABBITMQ__PASSWORD
              valueFrom:
                secretKeyRef:
                  name: grafana-oncall-rabbitmq-external
                  key: rabbitmq-password
```

## 🛠️ Troubleshooting & Manual Plugin Setup
```bash
1-
❗ Error: Plugin is not connected (jsonData.stackId is not set)
   Solution: Manually enable plugin and provide stack info via API

curl -X POST 'http://admin:pD7Im6Kos7ZNnVAdHgUQL3JR3qiafXVFzsSEd9mX@localhost:3000/api/plugins/grafana-oncall-app/settings' \
  -H "Content-Type: application/json" \
  -d '{
    "enabled": true,
    "jsonData": {
      "stackId": 5,
      "orgId": 1,
      "onCallApiUrl": "http://my-oncall-engine.default.svc.cluster.local:8080",
      "grafanaUrl": "http://localhost:3000"
    }
  }'
## (makesure stackId and orgId same Grafana and OnCall).

2-
🧹 Fix DB Issue (e.g., delete user_id column or migration conflicts)

-- From inside MariaDB pod:
kubectl exec -it my-mariadb-0 -- bash
mysql -uroot -pmyStrongMariaPass

USE oncall;
SHOW TABLES;

-- To remove alert migration state:  ( 3shan delete column user-id)
DELETE FROM django_migrations WHERE app = 'alerts';


Optionally Drop All Tables

> SET FOREIGN_KEY_CHECKS = 0;
SELECT CONCAT('DROP TABLE IF EXISTS `', table_name, '`;') 
FROM information_schema.tables 
WHERE table_schema = 'oncall';

> SET FOREIGN_KEY_CHECKS = 1;


3-
🔄 Manual Plugin Connection using ServiceAccount Token

Step 1: Create Service Account Token Secret (if needed)

kubectl create secret generic grafana-oncall-sa-token \
  --type kubernetes.io/service-account-token \
  --from-literal=extra=unused \
  --dry-run=client -o yaml | tee grafana-oncall-sa-token.yaml

--- cat grafana-oncall-sa-token.yaml

apiVersion: v1

data:

&nbsp; extra: dW51c2Vk

kind: Secret

metadata:

&nbsp; creationTimestamp: null

&nbsp; name: grafana-oncall-sa-token

type: kubernetes.io/service-account-token



Step 2: Apply the Secret

kubectl apply -f grafana-oncall-sa-token.yaml

Step 3: Extract the Token

kubectl get secret grafana-oncall-sa-token -o jsonpath="{.data.token}" | base64 --decode

Step 4: Connect Plugin via API with Token

curl -X POST 'http://admin:prom-operator@localhost:3000/api/plugins/grafana-oncall-app/settings' \
  -H "Content-Type: application/json" \
  -d '{
    "enabled": true,
    "jsonData": {
      "stackId": 5,
      "orgId": 1,
      "onCallApiUrl": "http://localhost:8080",
      "grafanaUrl": "http://localhost:3000",
      "serviceAccountToken": "eyJhbGciOiJSUzI1NiIsImtpZCI6IksxNmRVVzFaRU5uV2tMazlrVGJnNEJWLU5JZXFRQnRlQzF2VG1CSG92M00ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImdyYWZhbmEtb25jYWxsLXNhLXRva2VuIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImdyYWZhbmEtb25jYWxsLXNhIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiYWNjZGFmMzUtM2IxYy00MTc5LTk2ODUtMGZiYjhkOGY3NjE5Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlZmF1bHQ6Z3JhZmFuYS1vbmNhbGwtc2EifQ.j-20qUIFaQtsLAzp-gMo2WseLQo5bLI-TVWAMdgYliUseumLShZxX\_tUPmcuuheU2qdBLQOS2wFnb6GDlVCWbqBEw5oj9nCxzk04IZuM8\_VEIPktFY9-jih\_3xSChgeRF3gCKrd8Zrtbi8B59h9tJwBwpScOZSPX4J71yudhKavXUSGHbauBdkcF\_YYSIf3higWA0XIinrhdf1VYoVGP4SUsDSsHL0WNceA6GPbHpUG8zTT5LP6k5GvB9u8JEAvMZRjGQxB7eNJNPRk6apmwfxfqiN3D8gUrFtzTeCSUpYD6frHPn2NW7P7FzEO7Uch7FgF8EWQpKXJgIxzNVEJoBg"
    }
  }'
```
## to add alert by grafana on-call :
>>  https://grafana.com/orgs   ( to add acount in grafanc cloud )
## to add token to send metrices from k6 to grafana cloud 
```bash
https://grafana.com/orgs/maiabdelfatah077/access-policies

##  >>  Create new access policy

● Display name : Prometheus Remote Write
● Name :  prometheus-remote-write
● Realms :  maiabdelfatah077 (all stacks)

● Scopes   ( to allow k6 send metrices to grafana cloud )
metrics	  write 
logs	      write

●  create 

##  >>>  add token to this policy
  Create new token
● Token name :  prometheus-token
● Expiration date :  No expiry
● create 
```
![image](https://github.com/user-attachments/assets/9bc6f0f0-c930-4e81-8b83-b1cf0f7e54a2)

## this token hykon hena (   K6_PROMETHEUS_RW_PASSWORD ) 
## >> grafana cloud >> Connections  >> Data sources  >>  grafanacloud-maiabdelfatah077-prom
 to get ( K6_PROMETHEUS_RW_SERVER_URL , K6_PROMETHEUS_RW_USERNAME )

 # 1- send metrices from k6 to Grafana cloud :
 ```bash
  1-  kubectl create configmap k6-cloud-test-script \
  --from-file=test.js \
  --namespace=default
```
```bash
>>  2-  k6-loop-cloud-deployment.yaml :

apiVersion: apps/v1
kind: Deployment
metadata:
  name: k6-loop-cloud
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: k6-loop-cloud
  template:
    metadata:
      labels:
        app: k6-loop-cloud
    spec:
      containers:
      - name: k6
        image: grafana/k6:latest
        command: ["/bin/sh"]
        args:
          - -c
          - |
            while true; do
              echo "⏱️ Running k6 test..."
              K6_PROMETHEUS_RW_SERVER_URL=https://prometheus-prod-36-prod-us-west-0.grafana.net/api/prom/push \
              K6_PROMETHEUS_RW_USERNAME=2334137 \
              K6_PROMETHEUS_RW_PASSWORD=glc_eyJvIjoiMTM4MDk4NyIsIm4iOiJwcm9tZXRoZXVzLXJlbW90ZS13cml0ZS1wcm9tZXRoZXVzLXRva2VuIiwiayI6IndQNTlyaWw3ZDM1MnRhOVZRNHE4QjRkayIsIm0iOnsiciI6InVzIn19 \
              k6 run --out experimental-prometheus-rw /test/test.js
              echo "✅ Done. Sleeping 1 minute ..."
              sleep 60
            done
        volumeMounts:
          - name: k6-script
            mountPath: /test
      restartPolicy: Always
      volumes:
        - name: k6-script
          configMap:
            name: k6-cloud-test-script

 ```
```bash
    3-  kubectl apply -f k6-loop-cloud-deployment.yaml
    
    4-  kubectl logs -f deployment/k6-loop-cloud
```

# 2- in grafana cloud 
## to add member 
```bash
https://grafana.com/orgs/maiabdelfatah077/members?src=grafananet&cnt=invite-user-command-palette
```

# 3-  to my profile in grafana cloud  
``` bash
https://maiabdelfatah077.grafana.net/a/cloud-home-app
```

# 4- aad alerts by using grafana on-call 

## a-  create team to send alerts 
```bash
1-  Grafana → in search → Teams
  >> add new team
 ● Team name  :  DevOps Team
 ● Add members : maiabdelfatah077      Permission  : admin
 ● create 
```

## b- Notification Channel
```bash
1-  Alerts & IRM  >> Alerting >> Contact points
>>  create Contact points
●  Name :  OnCall: DevOps Team
●  Integration :  Grafana IRM  Recommended

●  How to connect to IRM
  ( true ) Existing IRM integration

●  IRM Integration
The IRM integration to send alerts to
devops-alerts-integration

● save
```
## c-  Create Escalation Chain  
```bash
Create Escalation Chain

●  Escalation Chain name  :  default-devops-escalation
●  Assign to team :  DevOps Team
●  create

>>  select   default-devops-escalation
 ●  notify all team member
 ●  Start : Default  notification for  : DevOps Team
```

## d- add integration 
```bash
Add a new Integration
● Grafana Alerting

New Grafana Alerting integration
●  Integration Name :  devops-alerts-integration
●  Assign to team :  DevOps Team
●  Grafana Alerting Contact point  :  OnCall: DevOps Team


●  Add route

●  label  matched by
    >>   team  :  devops

●  Trigger escalation chain :  default-devops-escalation DevOps Team
```
##  f- permation user 
```bash
>> IRM  >> Users

maiabdelfatah077@gmail.com
● Default Team  :  DevOps Team
● Default notification rules	            Important notification rules	
  Email                                    	Email
```
## e - alert rule
```bash
●   1. Enter alert rule name
      >   Name :  🚨 Login Success Rate Dropped
      >   grafanacloud-maiabdelfatah077-prom

●  2. Define query and alert condition
      >  query :
        >>  avg by(scenario, team) (
       avg_over_time(k6_checks_rate{check="login successful"}[1m])
     )
     
     >   Is below     : 0.9

●  3. Add folder and labels

    > Folder :  K6 Alerts
    
    > Labels
      severity :  critical
      team :  devops

●  4. Set evaluation behavior

     >  Evaluation group and interval  :  UserJourneyAlerts
     
     > Pending period >  1m
     
     > Keep firing for :  0s


● 5. Configure notifications

   >  Contact point    :    OnCall: DevOps Team


●  6. Configure notification message

     >  Summary (optional)
     Login success rate dropped below 900% in the last minute
     
     >  Description (optional)
     The average rate of the "login successful" check fell below 90% during the last 1 minute window.
     This could indicate issues with /api/token or /api/data endpoints.
```

```bash
●   1. Enter alert rule name
      >   Name :  ✅ Login Back to Normal
      >   grafanacloud-maiabdelfatah077-prom

●  2. Define query and alert condition
        >  avg by(scenario, team) (
       avg_over_time(k6_checks_rate{check="login successful"}[1m])
     )

         >   Is above     : 0.9

●  3. Add folder and labels

    > Folder :  K6 Alerts
    
    > Labels
      severity :  critical
      team :  devops

●  4. Set evaluation behavior

     >  Evaluation group and interval  :  UserJourneyAlerts
     
     > Pending period >  1m
     
     > Keep firing for :  0s


● 5. Configure notifications

   >  Contact point    :    OnCall: DevOps Team


●  6. Configure notification message

     >  Summary (optional)
     ✅ Login success rate is back to normal above 90%
     
     >  Description (optional)
     Login success rate has recovered and is now consistently above 90%. This means the /api/token and /api/data endpoints are functioning correctly again.
```
![image](https://github.com/user-attachments/assets/89e8d710-e5a7-4cb1-8b9c-8d66648b9d7a)


## 2- to send mail in team Devops in first and if not answered send to team SRE:
### nfs steps ely foa bs
```bash
###  add team SRE
   >>> add members 
```
```bash
1- Contact Point
    >>>  OnCall: DevOps Team  ( arbto b el alerts )
          -   Integration   >> Grafana IRM  Recommended
          -   IRM Integration  >>  devops-alerts-integration


2-  Add Integration
      >>>   devops-alerts-integration
           -   Alerts matched by    (team = devops)
           -  Trigger escalation chain    >> DevOps to SRE Escalation DevOps Team    ( da ana 3mlto gded mmkn a3ml bl adem 3ady )

3-  Escalation chains
        >>>  DevOps to SRE Escalation  mrbota b ( DevOps Team ) 3shan hwa el first team
            - Start  Default   notification for   DevOps Team   team members
            -  Wait  60   minute(s)    ( 1h ) 
            -   Start  Default  notification for   SRE    team members
```
```bash
- keda hyb3t l team Devops w lw m7dsh rd then 1h hyb3t l team SRE
- emta el alert mytb3tsh l SRE :
     1-  manual >  ay 7ad mn team devops y3ml el alert ( Acknowledged )
     2-  lw el alert at7l w ba ( Resolved ) 
```

### to enable microsoft teams :
```bash
users >>  notification rules  >>   Notify by   Microsoft Teams


    1- Contact Point   ( Teams - SRE Channel )
              -   Integration   >> Microsoft Teams
              -  URL    ( 7t el url )

   2-  Contact Point بـ Alert Rules
           -  in alert >  Contact Point   ( Teams - SRE Channel )
    AW 
      Contact Point بـ Notification Policies
           -  Matching labels  ( team = devops )
           - Contact point  >> Teams - SRE Channel
```
### to add Schedules
 ```bash
  1-  create new  Schedules  ( test )
  2- add Layer
      -  time Starts  and end
      -  Recurrence period   ( kam yom )
      - Mask by weekdays   ( lw 3mlt enable >> lw 3ayza ashel yom aw keda )
      -   Limit each shift length   ( lw 3mlt enable >>  b7dd el shift kam sa3a ) 
      -  users
  >>>   Request shift swap  ( lw 3ayza abdel shift m3 shift 7ad tany mo'kat )
  >>>   Add override    ( abdel m3 7ad 2 days msln lw f holiday ) 

 3-  Escalation chains
     -  Start   Default   notification for schedule   Select Schedule
```
























```bash
quers :
(1 - k6_http_req_failed_rate) * 100

{__name__=~"k6_.*"}

k6_checks_rate
```
```bash
1- name : LoginSuccessRateLow

2- query : avg_over_time(k6_checks_rate{check="login successful"}[1m])
     B Reduce
         Input  A
         Function   Last     Mode   Strict

     C  Threshold
          Input  B
          Is below   0.9


3. Set evaluation behavior
   Folder : alert    ,   group : any name
   Pending period  :  1m   ( lw a3d 1m ab3t el alert )

4. Add annotations
  Summary (optional)
     >>>    Login success rate dropped below 90% in the last minute
  
  Description (optional)
     >>>   The average rate of the "login successful" check fell below 90% during the last 1 minute window.
            This could indicate issues with /api/token or /api/data endpoints.

5. Labels and notifications
     Labels
    severity  = critical
```
```bash
1- name : Service is likely DOWN

2- query : avg_over_time(k6_http_req_failed_rate{url=~".*my-service.*"}[1m])
     B Reduce
         Input  A
         Function   Last     Mode   Strict

     C  Threshold
          Input  B
          Is above    0.5     << important

3. Set evaluation behavior
   Folder : alert    ,   group : any name
   Pending period  :  1m   ( lw a3d 1m ab3t el alert )

4. Add annotations
  Summary (optional)
     >>>    More than 50% of requests to my-service failed
  
  Description (optional)
     >>>   k6 reports that over 50% of requests to my-service failed in the last minute. This likely indicates the service is down (e.g., DNS, crash, timeout).

5. Labels and notifications
     Labels
    severity  = critical
```

```bash
1- name : AuthTokenMissingRateHigh

2- query :  avg_over_time(k6_checks_rate{check="has auth token"}[1m])
     B Reduce
         Input  A
         Function   Last     Mode   Strict

     C  Threshold
          Input  B
          Is below    0.9    << important

3. Set evaluation behavior
   Folder : alert    ,   group : any name
   Pending period  :  1m   ( lw a3d 1m ab3t el alert )

4. Add annotations
  Summary (optional)
     >>>    High rate of missing authentication token
  
  Description (optional)
     >>>   The check has auth token is failing more than 10% of the time in the past 1 minute. This may indicate that users are not receiving tokens after logging in. Investigate the /api/token endpoint or auth logic.

5. Labels and notifications
     Labels
    severity  = critical
```
![image](https://github.com/user-attachments/assets/9a3cbcca-7dc5-40de-a187-22e53bc59b67)

