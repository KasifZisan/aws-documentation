# Amazon Elastic Kubernetes Service (EKS)
Amazon Elastic Kubernetes Service (Amazon EKS) is a fully managed Kubernetes service that enables you to run Kubernetes seamlessly in both AWS Cloud and on-premises data centers.

First you need to get into the AWS platform and then search EC2 and create a Cluster and Node Group. You can then interact with your EKS cluster by SSH-ing into your EKS cluster. Alternatively, you can also set up an EC2 instance to interact with your EKS cluster.

In preparation to manage your EKS Kubernetes cluster, you will need to install several Kubernetes management-related tools and utilities. In this lab step, you will install:

- **kubectl:** the Kubernetes command-line utility which is used for communicating with the Kubernetes Cluster API server
- **awscli:** used to query and retrieve your Amazon EKS cluster connection details, written into the ```~/.kube/config``` file 

## Installation and Setup Steps
- Download the ```kubectl``` utility, give it executable permissions, and copy it into a directory that is part of the PATH environment variable:
```bash
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.31.0/2024-09-12/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo cp ./kubectl /usr/local/bin
export PATH=/usr/local/bin:$PATH
```
- Test the kubectl utility, ensuring that it can be called like so:
```bash
kubectl version --client=true
```
- Download the AWS CLI utility, give it executable permissions, and copy it into a directory that is part of the PATH environment variable:
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```
- Test the ```aws``` utility -
```bash
aws --version
```
- Use the ```aws``` utility, to retrieve EKS Cluster name:
```bash
EKS_CLUSTER_NAME=$(aws eks list-clusters --region us-west-2 --query clusters[0] --output text)
echo $EKS_CLUSTER_NAME
```
- Use the ```aws``` utility to query and retrieve your Amazon EKS cluster connection details, saving them into the ~/.kube/config file. Enter the following command in the terminal:
```bash
aws eks update-kubeconfig --name $EKS_CLUSTER_NAME --region us-west-2 
```
- View the EKS Cluster connection details. This confirms that the EKS authentication credentials required to authenticate have been correctly copied into the ~/.kube/config file. Enter the following command in the terminal:
```bash
cat ~/.kube/config 
```
## Deploying a Cloud Native Application
The cloud native application has three parts - Frontend, Backend and Database. Both the Frontend and Backend has particular Service and Deployment manifest for them. The Database has StatefulSet, Service, Persistent Volume (PV) and Persistent Volume Claim (PVC).

- StatefulSet - used to deploy and launch 3 x pods containing the MongoDB service configured to listen on port 27017
- Service - headless, will be created to sit in front of the StatefulSet, creating a stable network name for each of the individual pods as well as for the StatefulSet as a whole
- Persistent Volume (PV) - x3, 1 for each individual pod in the StatefulSet - MongoDB will be configured to persist all data and configuration into a directory mounted to the persistent volume
- Persistent Volume Claim (PVC) - x3, 1 for each PV, binds a PV to a Pod within the MongoDB StatefulSet

#### Deployment Steps
- Begin by creating a namespace,  which will be used to contain all of the cluster resources that will eventually make up the sample cloud native application. In the terminal run the following command:
```bash
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: cloudacademy
  labels:
    name: cloudacademy
EOF
```
- Configure the ```cloudacademy``` namespace to be the default. In the terminal run the following command:
```bash
kubectl config set-context --current --namespace cloudacademy
```
#### MongoDB Deployment
- Deploy the MongoDB 3 x ReplicaSet
- Display the available EKS storage classes. In the terminal run the following command:
```bash
kubectl get storageclass
```
- Create a new Mongo StatefulSet name ```mongo```. Also keep in mind to use the ```storageclass``` we got before in the ```storageClassName```. 
    - Examine the MongoDB Pods ordered sequence launch. Run a ```watch``` on the pods in the current namespace. Wait until all 3 MongoDB pods (```mongo-0```, ```mongo-1```, ```mongo-2```) have reached and recorded the Running status. ```kubectl get pods --watch```
    - View the pods with the db label in the current namespace. In the terminal run the following command: ```kubectl get pods -l role=db```
    - Display the MongoDB Pods, Persistent Volumes and Persistent Volume Claims. ```kubectl get pod,pv,pvc```
- Create a new Headless Service for Mongo. Examine the newly created ```mongo``` Headless Service. ```kubectl get svc```
- Create a temporary network utils pod. Enter into a bash session within it. ```kubectl run --rm utils -it --image praqma/network-multitool -- bash```. Creating a temporary utils pod in the current namespace - ensures that the ```nslookup``` queries to be performed next, are done so in the same networking space in which the MongoDB deployment has taken place in.
    - Within the new utils pod shell, execute the following DNS queries: ```for i in {0..2}; do nslookup mongo-$i.mongo; done```. This confirms that the DNS records have been created successfully and can be resolved within the cluster, 1 per MongoDB pod that exists behind the Headless Service - earlier created. Exit the utils container: ```exit```
- Confirm that the ```mongo``` shell can now also resolve each of the 3 Mongo headless Service assigned DNS names. In the terminal run the following command:
```bash
for i in {0..2}; do kubectl exec -it mongo-0 -- mongo mongo-$i.mongo --eval "print('mongo-$i.mongo SUCCEEDED\n')"; done
```
- Before proceeding to the next step, make sure that the previous command has completed successfully as per the output shown in the screenshot above. 
- On the mongo-0 pod, initialise the Mongo database Replica set. In the terminal run the following command:
```bash
cat << EOF | kubectl exec -it mongo-0 -- mongo
rs.initiate();
sleep(2000);
rs.add("mongo-1.mongo:27017");
sleep(2000);
rs.add("mongo-2.mongo:27017");
sleep(2000);
cfg = rs.conf();
cfg.members[0].host = "mongo-0.mongo:27017";
rs.reconfig(cfg, {force: true});
sleep(5000);
EOF
```
- Confirm that the MongoDB database replica set has been correctly established. In the terminal run the following command:
```bash
kubectl exec -it mongo-0 -- mongo --eval "rs.status()" | grep "PRIMARY\|SECONDARY"
```
-  Load the initial voting app data into the MongoDB database.
-  Confirm the voting app data has been loaded correctly. In the terminal run the following command:
```bash
kubectl exec -it mongo-0 -- mongo langdb --eval "db.languages.find().pretty()"
```
#### API (Backend) Deployment
- Create a secret to store the MongoDB connection credentials
- Generate username and password credentials:
```bash
echo -n 'admin' | base64
echo -n 'password' | base64
```
- Create a Secret resource within the cluster to hold the base64 encoded credentials.
```bash
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
  namespace: cloudacademy
data:
  username: YWRtaW4=
  password: cGFzc3dvcmQ=
EOF
```
- Create the API Deployment resource. 
- Create a new Service resource of LoadBalancer type

```bash
kubectl expose deploy api \
 --name=api \
 --type=LoadBalancer \
 --port=80 \
 --target-port=8080
```
- Confirm the API setup within the cluster:
    - Examine the rollout of the API deployment
    - Examine the pods to confirm that they are up and running
    - Examine the API service details

```bash
{
kubectl rollout status deployment api
kubectl get pods -l role=api
kubectl get svc api
}
```
- Test and confirm that the API route URL ```/ok``` endpoint can be called successfully. In the terminal run the command given below. DNS propagation can take up to 2-5 minutes, please be patient while the propagation proceeds - it will eventually complete.
```bash
{
API_ELB_PUBLIC_FQDN=$(kubectl get svc api -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
until nslookup $API_ELB_PUBLIC_FQDN >/dev/null 2>&1; do sleep 2 && echo waiting for DNS to propagate...; done
curl $API_ELB_PUBLIC_FQDN/ok
echo
}
```
- Test and confirm that the API route URL /languages, and /languages/{name} endpoints can be called successfully. In the terminal run any of the following commands:
```bash
curl -s $API_ELB_PUBLIC_FQDN/languages | jq .
curl -s $API_ELB_PUBLIC_FQDN/languages/go | jq .
curl -s $API_ELB_PUBLIC_FQDN/languages/java | jq .
curl -s $API_ELB_PUBLIC_FQDN/languages/nodejs | jq .
```
#### Frontend Deployment
The frontend deployment steps are same as the backend deployment steps - 

- Retrieve the FQDN of the API LoadBalancer and store it in the ```$API_PUBLIC_FQDN``` variable. The value stored in the ```$API_PUBLIC_FQDN``` variable is injected into the Frontend container's ```REACT_APP_APIHOSTPORT``` environment var - this tells the frontend where to send browser initiated API AJAX calls. In the terminal run the following command:
```bash
{
API_ELB_PUBLIC_FQDN=$(kubectl get svc api -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
echo API_ELB_PUBLIC_FQDN=$API_ELB_PUBLIC_FQDN
}
```
- Create the Frontend Deployment resource
- Create a new Service resource of LoadBalancer type
- Confirm the Frontend setup within the cluster:
    - Examine the rollout of the Frontend deployment
    - Examine the pods to confirm that they are up and running
    - Examine the Frontend service details
```bash
{
kubectl rollout status deployment frontend
kubectl get pods -l role=frontend
kubectl get svc frontend
}
```
- Confirm that the Frontend ELB is ready to recieve HTTP traffic. In the terminal run the following command:
```bash
{
FRONTEND_ELB_PUBLIC_FQDN=$(kubectl get svc frontend -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
until nslookup $FRONTEND_ELB_PUBLIC_FQDN >/dev/null 2>&1; do sleep 2 && echo waiting for DNS to propagate...; done
curl -I $FRONTEND_ELB_PUBLIC_FQDN
}
```
- Generate the Frontend URL for browsing. In the terminal run the following command:
```bash
echo http://$FRONTEND_ELB_PUBLIC_FQDN
```
- Query the MongoDB database directly to observe the updated vote data. In the terminal execute the following command:
```bash
kubectl exec -it mongo-0 -- mongo langdb --eval "db.languages.find().pretty()"
```