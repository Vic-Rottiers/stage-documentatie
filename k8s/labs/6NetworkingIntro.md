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

Bij `kubectl describe svc/webapp1-nodeport-svc` kunnen we zien dat de nodeport is ingesteld op 30080


Het IP van de node kunnen we zien in `kubectl describe nodes/controlplane`

testen: `curl 172.17.0.69:30080`

## 4) External IPs

De service available maken buiten de cluster doormiddel van een extern IP adres. Dit doen we door dit mee te geven in de .yaml config file.

`kubectl apply -f externalip.yaml`

inhoud van externalip.yaml:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp1-externalip-svc
  labels:
    app: webapp1-externalip
spec:
  ports:
  - port: 80
  externalIPs:
  - 172.17.0.69
  selector:
    app: webapp1-externalip
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: webapp1-externalip-deployment
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: webapp1-externalip
    spec:
      containers:
      - name: webapp1-externalip-pod
        image: katacoda/docker-http-server:latest
        ports:
        - containerPort: 80
---
```

`kubectl describe svc/webapp1-externalip-svc`

er is dus nu geen port nodig, omdat het extern ip is opgesteld.

`curl 172.17.0.69`

## 5) Load Balancer

we gebruiken load balancers, meestal provided door cloud provider

`kubectl apply -f cloudprovider.yaml`

`kubectl get pods -n kube-system`  
`kubectl apply -f loadbalancer.yaml`

inhoud cloudprovider.yaml:

```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: kube-keepalived-vip
  namespace: kube-system
spec:
  template:
    metadata:
      labels:
        name: kube-keepalived-vip
    spec:
      hostNetwork: true
      containers:
        - image: gcr.io/google_containers/kube-keepalived-vip:0.9
          name: kube-keepalived-vip
          imagePullPolicy: Always
          securityContext:
            privileged: true
          volumeMounts:
            - mountPath: /lib/modules
              name: modules
              readOnly: true
            - mountPath: /dev
              name: dev
          # use downward API
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          # to use unicast
          args:
          - --services-configmap=kube-system/vip-configmap
          # unicast uses the ip of the nodes instead of multicast
          # this is useful if running in cloud providers (like AWS)
          #- --use-unicast=true
      volumes:
        - name: modules
          hostPath:
            path: /lib/modules
        - name: dev
          hostPath:
            path: /dev
      nodeSelector:
        # type: worker # adjust this to match your worker nodes
---
## We also create an empty ConfigMap to hold our config
apiVersion: v1
kind: ConfigMap
metadata:
  name: vip-configmap
  namespace: kube-system
data:
---
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  labels:
    app: keepalived-cloud-provider
  name: keepalived-cloud-provider
  namespace: kube-system
spec:
  replicas: 1
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: keepalived-cloud-provider
  strategy:
    type: RollingUpdate
  template:
    metadata:
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ""
        scheduler.alpha.kubernetes.io/tolerations: '[{"key":"CriticalAddonsOnly", "operator":"Exists"}]'
      labels:
        app: keepalived-cloud-provider
    spec:
      containers:
      - name: keepalived-cloud-provider
        image: quay.io/munnerz/keepalived-cloud-provider:0.0.1
        imagePullPolicy: IfNotPresent
        env:
        - name: KEEPALIVED_NAMESPACE
          value: kube-system
        - name: KEEPALIVED_CONFIG_MAP
          value: vip-configmap
        - name: KEEPALIVED_SERVICE_CIDR
          value: 10.10.0.0/26 # pick a CIDR that is explicitly reserved for keepalived
        volumeMounts:
        - name: certs
          mountPath: /etc/ssl/certs
        resources:
          requests:
            cpu: 200m
        livenessProbe:
          httpGet:
            path: /healthz
            port: 10252
            host: 127.0.0.1
          initialDelaySeconds: 15
          timeoutSeconds: 15
          failureThreshold: 8
      volumes:
      - name: certs
        hostPath:
          path: /etc/ssl/certs
```

inhoud loadbalancer.yaml:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp1-loadbalancer-svc
  labels:
    app: webapp1-loadbalancer
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: webapp1-loadbalancer
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: webapp1-loadbalancer-deployment
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: webapp1-loadbalancer
    spec:
      containers:
      - name: webapp1-loadbalancer-pod
        image: katacoda/docker-http-server:latest
        ports:
        - containerPort: 80
---
```

`kubectl get svc`  
`kubectl describe svc/webapp1-loadbalancer-svc`

als we details van service bekijken kunnen we het bereiken op **Loadbalancer Ingress** ip

```yaml
export LoadBalancerIP=$(kubectl get services/webapp1-loadbalancer-svc -o go-template='{{(index .status.loadBalancer.ingress 0).ip}}')
echo LoadBalancerIP=$LoadBalancerIP
curl $LoadBalancerIP
```