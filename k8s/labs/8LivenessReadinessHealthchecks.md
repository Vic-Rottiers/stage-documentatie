# Liveness and Readiness Healthchecks

In this scenario, you'll learn how Kubernetes checks containers health using Readiness and Liveness Probes.

Readiness Probes checks if an application is ready to start processing traffic. This probe solves the problem of the container having started, but the process still warming up and configuring itself meaning it's not ready to receive traffic.

Liveness Probes ensure that the application is healthy and capable of processing requests.

## 1) Launch Cluster

cluster met het `launch.sh` script launchen

vervolgens demo app deployen: `kubectl apply -f deploy.yaml`

inhoud deploy.yaml:

```yaml
kind: List
apiVersion: v1
items:
- kind: ReplicationController
  apiVersion: v1
  metadata:
    name: frontend
    labels:
      name: frontend
  spec:
    replicas: 1
    selector:
      name: frontend
    template:
      metadata:
        labels:
          name: frontend
      spec:
        containers:
        - name: frontend
          image: katacoda/docker-http-server:health
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 1
            timeoutSeconds: 1
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 1
            timeoutSeconds: 1
- kind: ReplicationController
  apiVersion: v1
  metadata:
    name: bad-frontend
    labels:
      name: bad-frontend
  spec:
    replicas: 1
    selector:
      name: bad-frontend
    template:
      metadata:
        labels:
          name: bad-frontend
      spec:
        containers:
        - name: bad-frontend
          image: katacoda/docker-http-server:unhealthy
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 1
            timeoutSeconds: 1
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 1
            timeoutSeconds: 1
- kind: Service
  apiVersion: v1
  metadata:
    labels:
      app: frontend
      kubernetes.io/cluster-service: "true"
    name: frontend
  spec:
    type: NodePort
    ports:
    - port: 80
      nodePort: 30080
    selector:
      app: frontend
```

gaat dus 2 replicationcontrollers starten die de service 'frontend' gaan starten op elk 1 replica, de ene een healthy versie van service, andere een unhealthy one.

## 2) Readiness Probe

Ook zien we hierboven dat elke service 2 probes meekrijgt, zogezegde checks over HTTP.

**Get status**

van de slechte service/frontend: `kubectl get pods --selector="name=bad-frontend"`

we zien dat daar geen pod draait, we bekijken de pod

`kubectl describe pod bad-frontend-rgxld`

Bij events zien we dat de HTTP probe failed met statuscode 500, wat dus verwacht is.

Bij de 2de pod staat alles op OK: `kubectl get pods --selector="name=frontend"`

## 3) Liveness Probe

We simuleren nu een failure in de 2de pod, aangezien deze healthy is.

**crash de service**

de http server heeft een extra endpoint dat ervoor zorgt dat er 500-errors returned worden.
`kubectl exect podnaam -- /usr/bin/curl -s localhost/unhealthy`

als we dan terug checken met `kubectl get pods -- selector="name=frontend"` zien we dat er restarts optreden en geen ready pods zijn. Na enkele ogenblikken gaat de pod terug ready zijn omdat k8s deze heeft gekilled en opnieuw opgestart.