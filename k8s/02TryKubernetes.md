# Uitproberen: Tutorials

## Basics: Creating a cluster

commands:  
`minikube version`  
`kubectl version`  
`kubectl cluster-info`  
`kubectl get nodes`  

## Deploying your first app on Kubernetes

![cluster](img/module_02_first_app.svg)

Eerst een deployment aanmaken

`kubectl create deployment kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1`: deployment aanmaken.
`kubectl get deployments`: kijken welke deployments er al zijn.

vervolgens kan de deployment bezien worden door een proxy op te zetten met de cluster en alle nodes: `kubectl proxy`

`curl http://localhost:8001/version`

`export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
echo Name of the Pod: $POD_NAME`

`curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME`
