# Deploy Guestbook example on Kubernetes

https://kubernetes.io/docs/tutorials/stateless-application/guestbook/  
https://www.katacoda.com/courses/kubernetes/guestbook

een example applicatie die wordt gerunned op kubernetes.

This tutorial shows you how to build and deploy a simple (not production ready), multi-tier web application using Kubernetes and Docker. This example consists of the following components:

- A single-instance Redis to store guestbook entries
- Multiple web frontend instances

## 1) Start Kubernetes

kubernetes werd opgestart met launch.sh helperscript.

**Health check:**

`kubectl cluster-info`  
`kubectl get nodes`

nodes moeten ready zijn.

## 2) Redis Master Controller

kubernetes service deployments hebben (minstens) 2 parts: Replication controller en een service.

replication controller defines how many instances should be running, welke docker image, name van de service, ...

als vb Redis (db structure ding) down gaat, gaat replication controller die restarten op active node

**Creating Replication controller**

adhv YAML files worden de services en controllers aangemaakt op de cluster. De Replication controller yaml file ziet er als volgt uit:

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: redis-master
  labels:
    name: redis-master
spec:
  replicas: 1
  selector:
    name: redis-master
  template:
    metadata:
      labels:
        name: redis-master
    spec:
      containers:
      - name: master
        image: redis:3.0.7-alpine
        ports:
        - containerPort: 6379
```

Met commando: `kubectl create -f redis-master-controller.yaml` wordt controller aangemaakt.

de replication controllers kunnen opgelijst worden met `kubectl get rc`.  
kan ook bij `kubectl get pods` gezien worden.

## 3) Redis Master service

The second part is a service. A Kubernetes service is a named load balancer that proxies traffic to one or more containers. The proxy works even if the containers are on different nodes.

create de service: `kubectl create -f redis-master-service.yaml`

ziet er als volgt uit:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-master
  labels:
    name: redis-master
spec:
  ports:
    # the port that this service should serve on
  - port: 6379
    targetPort: 6379
  selector:
    name: redis-master
```

list en describe services:  
`kubectl get services`  
`kubectl describe services redis-master`

## 4) Redis slave Controller & slave service

hier gaan we voor replication & backup zorgen. Deze services zorgen ervoor dat moesten de master services uitvallen, deze inspringen en proberen de masters terug online te halen en even het werk op hun te nemen.

Redis Slave Controller: `kubectl create -f redis-slave-controller.yaml`  
`kubectl get rc`

ziet er als volgt uit:

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: redis-slave
  labels:
    name: redis-slave
spec:
  replicas: 2
  selector:
    name: redis-slave
  template:
    metadata:
      labels:
        name: redis-slave
    spec:
      containers:
      - name: worker
        image: gcr.io/google_samples/gb-redisslave:v1
        env:
        - name: GET_HOSTS_FROM
          value: dns
          # If your cluster config does not include a dns service, then to
          # instead access an environment variable to find the master
          # service's host, comment out the 'value: dns' line above, and
          # uncomment the line below.
          # value: env
        ports:
        - containerPort: 6379
```

Redis Slave Service: `kubectl create -f redis-slave-service.yaml`  
`kubectl get services`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-slave
  labels:
    name: redis-slave
spec:
  ports:
    # the port that this service should serve on
  - port: 6379
  selector:
    name: redis-slave
```

## 5) Frontend Replicated Pods (Frontend-controller)

Data services zijn nu opgestart, nu de web applicatie zelf nog. op zelfde manier als backend
Deze gaat een container pullen van google (sample van frontend) en gebruikt dan de redis datastore als opslag

`kubectl create -f frontend-controller.yaml`

ziet er als volgt uit:

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: frontend
  labels:
    name: frontend
spec:
  replicas: 3
  selector:
    name: frontend
  template:
    metadata:
      labels:
        name: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
        env:
        - name: GET_HOSTS_FROM
          value: dns
          # If your cluster config does not include a dns service, then to
          # instead access environment variables to find service host
          # info, comment out the 'value: dns' line above, and uncomment the
          # line below.
          # value: env
        ports:
        - containerPort: 80
```

## 6) Frontend Service (Guestbook app)

Start de service, die een NodePort ports gaat openzetten voor de frontend te kunnen bereiken.

`kubectl create -f frontend-service.yaml`

Ziet er zo uit:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    name: frontend
spec:
  # if your cluster supports it, uncomment the following to automatically create
  # an external load-balanced IP for the frontend service.
  # type: LoadBalancer
  type: NodePort
  ports:
    # the port that this service should serve on
    - port: 80
      nodePort: 30080
  selector:
    name: frontend
```

`kubectl get services`  
`kubectl get svc` is hetzelfde maar afgekort


## 7) Guestbook app accessen

als er geen well-known NodePort geassigned is, gaat kubernetes zelf een at random assignen:  
`kubectl describe service frontend | grep NodePort`

