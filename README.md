# FossaAllinOne
Purpose

The purpose of the document is to detail the steps of deploying a Fossa all in one deployment within a single node Kubernetes cluster.

The following depicts the topology of the Fossa all in one appliance.


 
Figure 1. Fossa All in One Topology


Requirements

•	At least 16 GB 
•	400 GB in / volume
•	Docker installed
•	Kubernetes single node cluster
•	TLS certificate for 
o	Fossa Application
o	Minio Service


Deployment Steps


Kubernetes Preparation

A single node Kubernetes cluster is required for the deployment of the Fossa all in One appliance. If you do not have a single node Kubernetes environment use the following to deploy a single node Kubernetes cluster with kubeadm.

Deploying a Single Node Kubernetes Cluster with kubeadm on Ubuntu 18.04

Install Docker and Kubernetes Cluster with  kubeadm

1.	Get the Docker gpg key:
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
2.	Add the Docker repository:
sudo add-apt-repository    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
3.	Get the Kubernetes gpg key:
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

4.	Add the Kubernetes repository:
cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
5.	Update your packages:
sudo apt-get update

6.	Install Docker, kubelet, kubeadm, and kubectl:
sudo apt-get install -y docker-ce=18.06.1~ce~3-0~ubuntu kubelet=1.13.5-00 kubeadm=1.13.5-00 kubectl=1.13.5-00

7.	Hold them at the current version:
sudo apt-mark hold docker-ce kubelet kubeadm kubectl
8.	Add the iptables rule to sysctl.conf:
echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf
9.	Enable iptables immediately:
sudo sysctl -p

10.	Initialize the cluster (run only on the master):
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

11.	Set up local kubeconfig:
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config

12.	Apply Flannel CNI network overlay:
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml
13.	 Verify the Kubernetes node :
kubectl get nodes

root@ip-172-31-63-168:/home/ubuntu# kubectl get nodes
NAME               STATUS   ROLES    AGE     VERSION
ip-172-31-63-168   Ready    master   3m38s   v1.13.5


14.	 Remove taint from the master node. (This allows pods to be scheduled on the master node)
kubectl taint nodes --all node-role.kubernetes.io/master-



2.	 Deploying Fossa


1.	Clone GitHub repo for Fossa
kubectl taint nodes --all node-role.kubernetes.io/master-


2.	Create Fossa Namespace
kubectl create ns fossa



3.	Modify config.yaml file and create image pull secret to download Fossa images
# Modify config.yaml 
# Encode the key provided by Fossa prior to adding to data:

apiVersion: v1
kind: Secret
metadata:
  name: quay.io
  namespace: fossa
data:
  .dockerconfigjson:
data: < image pull secret>



kubectl create -f config.yaml


4.	Modify configmap.yaml with required parameters and create config ConfigMap

# Modify configmap.yaml

secret: <64-bit string that will be used to encrypt data stored in database>
app:
  hostname: <hostname for fossa app>
cache:
  package:
    bucket:  < bucket name use for fossa, must include “.” in name, example “fossa.net>
    s3Options:
      accessKeyId: <access key for minio>
      secretAccessKey: <access key for minio>
      endpoint: < hostname for minio>
s3:
  accessKeyId": < access key for minio>
  secretAccessKey": < access key for minio>
  endpoint": < hostname for minio>
componentUploader: 
  bucket: < bucket name use for fossa, must include “.” in name example “fossa.net>


kubectl create -f configmap.yaml

5.	Create Secrets for TLS certificates for fossa app and minio app. Create configmap minio app.

# You will need to create 2 secrets which include the pem and key file for the chosen hostname for fossa application and the minio application. The pem  and key file will need to be renamed to tls.crt and tls.key before creating the secret. 

# You will also need to create a configmap with the pem file of the minio application.

# Filenames must be named tls.crt and tls.key

Fossa
Kubecetl -n fossa create secret generic tls-ssl-fossa –from-file=./tls.key –from-file=./tls.crt

Minio
Kubecetl -n fossa create secret generic tls-ssl-minio –from-file=./tls.key –from-file=./tls.crt

Kubectl -n fossa create configMap ca-pemstore –from-file=minio.pem






6.	Create the database statefulset and database service


Kubectl create -f database.yaml

 # Confirm database is up and running

kubectl -n fossa get pod | grep database

NAME                                        READY   STATUS    RESTARTS   AGE
database-0                                    1/1     Running     0      3d6h

7.	Migrate the database schema

Kubectl create -f migrate-db.yaml

Confirm that migrate job is complete

kubectl -n fossa get pod | grep migrate
fossa-migrate-x6l6m                  0/1     Completed   0          3d6h

8.	Modify minio.yaml and create the minio statefulset and minio service

# Modify minio.yaml file

env:
        - name: MINIO_ACCESS_KEY
          value: #<access key for minio>
        - name: MINIO_SECRET_KEY
          value: #<secret key for minio>

Create the minio statefulset

Kubectl create minio.yaml

# Verify the statefulset is created

kubectl -n fossa get pod | grep minio

NAME                                        READY   STATUS    RESTARTS   AGE
minio-0                                      1/1     Running     0      3d6h

9.	Create the Ingress Controller


Kubectl create -f ingress_controller.yaml

# Verify the ingress controller is created

kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-7f4b7d7b5f-6d5wv   1/1     Running   0          25h



10.	Create the Ingress for Fossa and Minio

# Modify ingress.yaml file

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: fossa-ingress
  namespace: fossa
spec:
  tls:
  - hosts:
    - < hostname for fossa app>
    secretName:  tls-ssl-fossa
  rules:
  - host: #< hostname for fossa app>
    http:
      paths:
      - path: /
        backend:
          serviceName: fossa
          servicePort: 80

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: minio-ingress
  namespace: fossa
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: 2g
    nginx.org/proxy-read-timeout:  15m
    nginx.org/proxy-write-timeout: 15m

spec:
  tls:
  - hosts:
    - < hostname for minio app>
    secretName: tls-ssl-minio
  rules:
  - host: < hostname for minio app>
    http:
      paths:
      - backend:
          serviceName: minio
          servicePort: 80
        path: /


# Create the ingress for Fossa and minio

kubectl create -f ingress.yaml

# Confirm ingresses are created  

kubectl get ingress -n fossa
NAME            HOSTS         ADDRESS   PORTS     AGE
fossa-ingress   fossa.local             80, 443   26h
minio-ingress   minio.local             80, 443   25h



11.	Create Fossa components


# Modify fossa.yaml file

spec:
      hostAliases:
      - ip: < ip of the host>
        hostnames:
        - < hostname of minio app>


# Create Fossa Components

kubectl create -f fossa.yaml

# Verify Fossa pods are running
kubectl get pods -n fossa
NAME                                                         READY   STATUS      RESTARTS   AGE
database-0                                                   1/1     Running     0          3d12h
fossa-agent-deployment-5d4bdd989-2z5lc                       1/1     Running     0          30h
fossa-agent-deployment-5d4bdd989-j9xgw                       1/1     Running     0          30h
fossa-agent-deployment-5d4bdd989-pbv64                       1/1     Running     0          2m49s
fossa-agent-deployment-5d4bdd989-thdr8                       1/1     Running     0          2m49s
fossa-core-deployment-694b486ddf-7lt5q                       1/1     Running     0          3d7h
fossa-core-deployment-694b486ddf-jdkms                       1/1     Running     0          3d7h
fossa-dependencylock-watchdog-deployment-6756bcdb5d-m7sww    1/1     Running     0          3d7h
fossa-integrationhook-watchdog-deployment-57896b76fb-b8vlc   1/1     Running     0          3d7h
fossa-migrate-x6l6m                                          0/1     Completed   0          3d12h
fossa-revision-watchdog-deployment-66c4c5cbd6-vxcsp          1/1     Running     0          3d7h
fossa-task-watchdog-deployment-7bd8447cd6-kt5bx              1/1     Running     0          3d7h
fossa-updatehook-watchdog-deployment-5688777fd-24vdb         1/1     Running     0          3d7h
minio-0                                                      1/1     Running     0          3d12h



