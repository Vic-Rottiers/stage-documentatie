# Launching a single node kubernetes cluster

## 1) Start Minikube

check if installed: `minikube version`

start cluster: `minikube start --wait=false`

## 2) Cluster info

info opvragen over de cluster: `kubectl cluster-info`

de nodes opvragen in de cluster: `kubectl get nodes`

## 3) Deploy containers

een container deployen op de cluster: `kubectl create deployment deployment-naam --image=dockerhubUser/dockerhubContainer`

> in het voorbeeld werd de `katacoda/docker-http-server` gebruikt

de verschillende pods (container groups) bekijken op de cluster: `kubectl get pods`

Container is running op de cluster, nu moeten we er nog aankunnen van buitenaf. Kan op verschillende manieren, in dit geval wordt nodeport gebruikt.  
`kubectl expose deployment deployment-naam --port=80 --type=NodePort`

vervolgens kunnen we checken of de webserver openstaat op poort 80.

```bash
export PORT=$(kubectl get svc first-deployment -o go-template='{{range.spec.ports}}{{if .nodePort}}{{.nodePort}}{{"\n"}}{{end}}{{end}}')
echo "Accessing host01:$PORT"
curl host01:$PORT
```

## 4) Dashboard

kubernetes heeft ook web dashboard 

aanzetten door addon van dashboard te enablen: `minikube addons enable dashboard`

kubernetes dashboard available maken door YAML definition te deployen (*katacoda enkel*):  
`kubectl apply -f /opt/kubernetes-dashboard.yaml`

checken of dashboard al runned: `kubectl get pods -n kubernetes-dashboard -w`