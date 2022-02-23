# Networking introduction

Verschillende methodes om services beschikbaar te stellen voor externe gebruikers via IP

## 1) Cluster IP

De default approach als men kubernetes services maken. De service krijgt een intern IP adres gealloceerd dat andere components kunnen gebruiken om de pods te accessen

clusterip service deployen: `kubectl apply -f clusterip.yaml`

inhoud van clusterip.yaml: 

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp1-clusterip-svc
  labels:
    app: webapp1-clusterip
spec:
  ports:
  - port: 80
  selector:
    app: webapp1-clusterip
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: webapp1-clusterip-deployment
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: webapp1-clusterip
    spec:
      containers:
      - name: webapp1-clusterip-pod
        image: katacoda/docker-http-server:latest
        ports:
        - containerPort: 80
```

pods bekijken: `kubectl get pods`

service config kan ook hier bekeken worden in detail: `kubectl describe svc/webapp1-clusterip-svc`  
bij `IP: ` is dus te zien dat er een vast ip is toegewezen, alsook kunnen we de 2 endpoints zien.

we halen het cluster ip uit de details van de service en steken hem in de `$CLUSTER_IP` var.  
`export CLUSTER_IP=$(kubectl get services/webapp1-clusterip-svc -o go-template='{{(index .spec.clusterIP)}}')`  

We testen de connectie doormiddel van curl: `curl $CLUSTER_IP:80`

## 2) Target Port

Hierdoor kunnen we de poorten splitsen van de service:

- poort waar service van available is
- poort waar applicatie/(service) op luisterd

service & pods deployen: `kubectl apply -f clusterip-target.yaml`

inhoud clusterip-target.yaml:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp1-clusterip-targetport-svc
  labels:
    app: webapp1-clusterip-targetport
spec:
  ports:
  - port: 8080
    targetPort: 80
  selector:
    app: webapp1-clusterip-targetport
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: webapp1-clusterip-targetport-deployment
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: webapp1-clusterip-targetport
    spec:
      containers:
      - name: webapp1-clusterip-targetport-pod
        image: katacoda/docker-http-server:latest
        ports:
        - containerPort: 80
---
```

Notice dat als we de details van de service bekijken, de poorten anders zijn: `kubectl describe svc/webapp1-clusterip-targetport-svc`

we halen het cluster ip uit de details van de service en steken hem in de `$CLUSTER_IP` var.  
`export CLUSTER_IP=$(kubectl get services/webapp1-clusterip-svc -o go-template='{{(index .spec.clusterIP)}}')`  

`curl $CLUSTER_IP:8080`

## 3) NodePort

aangezien elke node een ip heeft (omdat het elk een machine is) kan er met NodePort aan elk nodeip een port gespecifieerd worden waar de service bereikbaar is. maakt ni uit welke node je kiest, op elke node gaat op die poort de service bereikbaar zijn.

`kubectl apply -f nodeport.yaml`

inhoud nodeport.yaml:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp1-nodeport-svc
  labels:
    app: webapp1-nodeport
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 30080
  selector:
    app: webapp1-nodeport
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: webapp1-nodeport-deployment
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: webapp1-nodeport
    spec:
      containers:
      - name: webapp1-nodeport-pod
        image: katacoda/docker-http-server:latest
        ports:
        - containerPort: 80
---
```

## 4) External IPs

## 5) Load Balancer