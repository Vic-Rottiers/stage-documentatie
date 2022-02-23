# getting started with kubeadm

in single node lab werd minikube gebruikt, dit is een light-weight en portable versie van k8s. hier gebruiken we Kubeadm

In this scenario you'll learn how to bootstrap a Kubernetes cluster using Kubeadm.

Kubeadm solves the problem of handling TLS encryption configuration, deploying the core Kubernetes components and ensuring that additional nodes can easily join the cluster. The resulting cluster is secured out of the box via mechanisms such as RBAC.

More details on Kubeadm can be found at https://github.com/kubernetes/kubeadm

## 1) initialise master

Das de eerste, zodat alle nodes erop kunnen connecten

init cluster: `kubeadm init -- token=102952.1a7dd4cc8d1f4cc5 --kubernetes-version $(kubeadm version -o short)`

normaal zou je geen token mogen meegeven, zodat kubeadm zelf een generate.

user credentials/config/certs. moeten gekopieerd worden van aanmaakdir. (door k8s), naar user home dir.

```bash
sudo cp /etc/kubernetes/admin.conf $HOME/
sudo chown $(id -u):$(id -g) $HOME/admin.conf
export KUBECONFIG=$HOME/admin.conf
```

## 2) deploy Container Networking Interface

definieerd hoe de verschillende nodes en workloads moeten communiceren. In de demo wordt WeaveWorks gebruikt. die is te zien op `cat /opt/weave-kube.yaml`

wordt applied door: `kubectl apply -f /opt/weave-kube.yaml`

nu gaat weaveworks een paar pods deployen op de cluster.  
`kubectl get pod -n kube-system`

## 3) join cluster

we gaan de node de cluster laten joinen.

Nodes kunnen een cluster joinen zolang ze de juiste token hebben. Tokens kunnen op de master bekeken worden door `kubeadm token list`

nu gaan we op de te joinen node volgend commando uitvoeren met de token die we van de master hebben gehaald: `kubeadm join --discovery-token-unsafe-skip-ca-verification --token=102952.1a7dd4cc8d1f4cc5 172.17.0.26:6443`

dit is niet helemaal hoe het in productie gebeurd.

## 4) nodes bekijken

`kubectl get nodes`

## 5) deploy pod

beide nodes zijn nu ready, dus we kunnen eerste pod deployen

`kubectl create deployment http --image=katacoda/docker-http-server:latest`

status bekijken: `kubectl get nodes`

als alles runnende is kunnen we ook de docker containers checken met `docker ps | grep docker-http-server`

## 6) deploy dashboard

dashboard deployen met: `kubectl apply -f dashboard.yaml`

er moet een admin account aangemaakt worden, met dit commando wordt ClusterRoleBinding gebruikt om nieuwe ServiceAccount (admin-user) de rol *Cluster Admin* te geven.

```bash
cat <<EOF | kubectl create -f - 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
EOF
```

Nu ServiceAccount gemaakt is, kan login token gevonden worden op volgende plaats:

`kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')`

output ziet er als volgt uit:

```bash
controlplane $ kubectl -n kube-system describe secret $(kubectl -n kube-system gadmin-user | awk '{print $1}')
Name:         admin-user-token-vdhgg
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: d54269db-93f5-11ec-b861-0242ac11001a

Type:  kubernetes.io/service-account-token

Data
====
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXZkaGdnIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJkNTQyNjlkYi05M2Y1LTExZWMtYjg2MS0wMjQyYWMxMTAwMWEiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.TZikjEFW7zLEeo6-rsUF_G2wABvdkMRPdMH_fPO3jSQlLVHlF7x83skZChkpd8dhb-Rn1FwdP4vQljTm517Fs2AZvVrCkJXPDtGQ2UJtZl48eW5vifDe62ur-YIcs7UU_X_jhIvlpvQBVRSQIiz9_IQtrk7ZMMg-EEEcrSuOGlEYkeWKSj_27wZVSoUXLd2f6KZ1Mqt9Vh5srHol_u0WprgU74e5RiaVA8UucGFfYOqHLEdp2ZtP1vGjwg9TTo-2ZZdFV2QUCYWGsz5qbQU6mnJA2VRiwdRLG5MiNzth0OgMSVYhKoZmLBj8HHdzADm0YWsueXJsO_TesOKemtpBmQ
ca.crt:     1025 bytes
namespace:  11 bytes
controlplane $ 
```

het heeft externalIPs gebruikt om poort 8443 te binden, dit maakt dashboard available outside cluster. Niet aangeraden in productie, daar eerder `kubectl proxy` gebruiken om te verbinden met cluster/dashboard
