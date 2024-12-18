## Amazon Elastic Kubernetes Service (EKS)
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

## Associating IAM Roles with Kubernetes Service Accounts in Amazon EKS
In Kubernetes, a Service Account resource provides an identity for processes that run in a Pod. 

In Amazon Web Services, an Identity and Access Management (IAM) role performs a similar function. IAM roles provide an identity that permissions can be attached to.

Amazon Elastic Kubernetes Service (EKS) allows you to associate a Kubernetes Service Account with an AWS IAM role. Doing this has the following best-practice security benefits:

- You can ensure that a Pod only has access to the AWS resources that it requires, adhering to the Principle of Least Privilege
- Ensuring that a Pod's AWS IAM credentials are separate from other AWS IAM roles, achieving isolation of credentials
- When using IAM and Service Accounts together, you can use **Amazon CloudTrail** to audit credential access

Using IAM roles with Kubernetes Service Accounts requires an Open ID Connect (OIDC). EKS can be configured to have a public OIDC discovery endpoint hosted by Amazon for a specific EKS cluster. Using this EKS cluster-specific OIDC provider enables external systems, such as AWS IAM, to accept and verify Kubernetes Service Account tokens. Once an EKS cluster has OIDC configured, it can make use of any of the features of AWS IAM.

Now we want to show a demonstration, where you will deploy an application that accesses Amazon S3, you will create a new Service Account and associate it with an existing IAM role, and you will verify that the application can access Amazon S3.

#### Setting up IAM Roles
- Ensure that you have an up to date ```kubeconfig``` file
```bash
cd ~
aws eks --region us-west-2 update-kubeconfig --name Cluster-1
```
- You can create a seperate namespace and set context for that namespace.
```bash
kubectl create namespace iam-oidc
kubectl config set-context $(kubectl config current-context) --namespace=iam-oidc
```
- To view an IAM role that has Amazon S3 access, enter the following:
```bash
aws iam get-role --role-name s3-poller-role
```
In this case, we will have JSON response showing the ```AssumeRolePolicyDocument```. This allows use of the ```sts:AssumeRoleWithWebIdentity``` action by default. The ```principal``` for this statement is ```Federated``` which allows the use of AWS OIDC Provider as an Amazon Resource Name (ARN). The value of the ```StringEquals``` key has a value containing a subset of the ```Principal```'s ARN. This is the identifier for the cluster's OIDC endpoint.

- To see the policy attached to the ```s3-poller-role```, enter the following:
```bash
aws iam get-role-policy --role-name s3-poller-role --policy-name s3-poller-policy
```
This policy allows the ```s3:List*``` and ```s3:PutObject``` IAM actions. You will associate this pre-created S3 role with a Kubernetes Service Account.

- To see this EKS cluster's OIDC endpoint, issue the following in the terminal:
```bash
aws eks describe-cluster --name Cluster-1 --region us-west-2 --query cluster.identity
```
In our case, OIDC is already configured. To turn it on for an EKS cluster that doesn't have you can use the eksctl tool in the following way:
```bash
eksctl utils associate-iam-oidc-provider --cluster=<clusterName>
```
- To store the bucket in Amazon S3 in a shell variable, issue the following:
```bash
BUCKET_NAME=$(aws s3api list-buckets --query "Buckets[0].Name" --output text)
```
- Create a new deployment manifest and apply it.
- To update the bucket name in the manifest, use the ```sed``` command.
- To check that a Pod has been created by the Deployment, enter: ```kubectl get pod``` or you can watch it using ```kubectl get pod -w```
- To view Pod's logs - 
```bash
kubectl logs -l app=s3-poller -f
```
The logs of all Pods that have an app label with a value of s3-poller will be displayed. The -f flag is short for follow and it means that kubectl will wait for new log entries and print them as they happen.

The logs of all Pods that have an app label with a value of s3-poller will be displayed. The container running inside the Pod does not have credentials to access AWS resources. You will configure a Service Account for the Pod that is linked to an IAM role so that the container can access Amazon S3.

- To stop following the logs, press Ctrl-C.
- To list currently existing Service Accounts, issue the following:
```bash
kubectl get serviceaccount
```
You will see one Service Account named ```default``` listed. A default Service Account is automatically created when a new namespace is created. If no Service Account is specified for a Pod, it will use the namespace's default Service Account.

Service Accounts have tokens. These are stored in Kubernetes Secret resources. Feel free to issue ```kubectl get secret``` to see it listed. Also, feel free to add ```-o yaml``` to this command and the instruction's command to see the YAML manifest for these resources.

- To create a new Service Account, enter the following: 
```bash
kubectl create serviceaccount s3-poller
```
- To associate the newly created Service Account with an IAM role, enter the following:
```bash
kubectl annotate serviceaccount s3-poller \
    'eks.amazonaws.com/role-arn'='arn:aws:iam::708048379844:role/s3-poller-role'
```
- You can verify the Service Account's configuration by issuing:
```bash
kubectl describe serviceaccount s3-poller
```
- To modify the Deployment to use your new Service Account, issue the following command:
```bash
kubectl patch deployment s3-poller \
  -p '{"spec":{"template":{"spec":{"serviceAccountName":"s3-poller"}}}}'
```
- To verify that ```s3-poller``` Pod can now access Amazon S3, re-issue the logs command from earlier:
```bash
kubectl logs -l app=s3-poller
```
## AWS Load Balancer Controller and Ingress Resource
- Before creating the Load Balancer Controller, you need to perform some EKS cluster configuration. The EKS cluster can either be created using GUI or in this case, we will be using ```eksctl``` utility. It can be created with the following setting:
```bash
eksctl create cluster \
--version=1.31 \
--name=Cluster-1 \
--nodes=1 \
--node-type=t2.medium \
--ssh-public-key="cloudacademylab" \
--region=us-west-2 \
--zones=us-west-2a,us-west-2b,us-west-2c \
--node-volume-type=gp2 \
--node-volume-size=20
```
- An OpenID Connect provider needs to be established. This was performed using the following command:
```bash
eksctl utils associate-iam-oidc-provider \
--region us-west-2 \
--cluster Cluster-1 \
--approve
```
- A new IAM Policy was created, providing various permissions required to provision the underlying infrastructure items (ALBs and/or NLBs) created when Ingress and Service cluster resources are created.
```bash
curl -o /tmp/iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
aws iam create-policy \
--policy-name AWSLoadBalancerControllerIAMPolicy \
--policy-document file:///tmp/iam_policy.json
```
- A new cluster service account bound to the IAM policy has been created. When the AWS Load Balancer controller is later deployed by you, it will be configured to operate with this service account:
```bash
eksctl create iamserviceaccount \
--cluster Cluster-1 \
--namespace kube-system \
--name aws-load-balancer-controller \
--attach-policy-arn arn:aws:iam::${AWS::AccountId}:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--approve
```
#### Deploy AWS Load Balancer Controller
- Install the **helm** utility. Helm will be used to install the AWS Load Balancer Controller chart. In the terminal run the following commands:
```bash
{
pushd /tmp
curl -o helm-v3.12.1-linux-amd64.tar.gz https://get.helm.sh/helm-v3.9.2-linux-amd64.tar.gz
tar -xvf helm-v3.12.1-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
popd
which helm
}
```
- Update the local Helm repo
```bash
{
helm repo add eks https://aws.github.io/eks-charts
helm repo update
}
```
- Confirm that a cluster service account exists for the AWS Load Balancer Controller. In the terminal run the following command:
```bash
kubectl get sa aws-load-balancer-controller -n kube-system -o yaml
```
- Retrieve the VPC ID for the VPC that the EKS cluster worker node is deployed within
```bash
{
VPC_ID=$(aws eks describe-cluster \
--name Cluster-1 \
--query "cluster.resourcesVpcConfig.vpcId" \
--output text)
echo $VPC_ID
}
```
- Deploy the Custom Resource Definition (CRD) resources required by the AWS Load Balancer Controller. In the terminal run the following command:
```bash
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller/crds?ref=master"
```
- Using the helm command, deploy the AWS Load Balancer Controller chart. In the terminal run the following command:
```bash
helm install \
aws-load-balancer-controller \
eks/aws-load-balancer-controller \
-n kube-system \
--set clusterName=Cluster-1 \
--set serviceAccount.create=false \
--set serviceAccount.name=aws-load-balancer-controller \
--set image.tag=v2.5.2 \
--set region=us-west-2 \
--set vpcId=${VPC_ID} \
--version=1.5.3
```
- Confirm that the AWS Load Balancer Controller has been successfully deployed into the cluster. In the terminal run the following command:
```bash
kubectl -n kube-system rollout status deployment aws-load-balancer-controller
```
- Examine details of the deployed AWS Load Balancer Controller. In the terminal run the following command:
```bash
kubectl describe deployment -n kube-system aws-load-balancer-controller
```

#### Deploy and Expose the Web App
In this lab step, you will deploy and set up a sample web app that when requested simply returns a static message on a coloured background. Both the message and the background colour are parameterized, with their configurations specified using ConfigMap resources, mounted to within the pod at launch time. You will launch 2 versions of the same web app, considered V1 (yellow) and V2 (cyan), each with it's own unique message and background colour. Finally, both versions of the web app will be exposed publicly using an Ingress resource - resulting in an ALB being launched behind the scenes. HTTP path based routing will be accomplished by leveraging Annotations specified within the Ingress manifest. As a bonus you will also configure an addtional path route (white) which responds with a static message specified directly within the annotation itself.

Choosing an Ingress resource to expose your cluster hosted application to the Internet, will result in the ALBC provisioning an ALB.

- Create a dedicated namespace to host both versions of the sample web app
```bash
{
kubectl create ns webapp
kubectl config set-context $(kubectl config current-context) --namespace=webapp
}
```
- Create a ConfigMap for each version of the sample web app. Each ConfigMap will contain a unique message and background colour. Confirm both ConfigMaps were created successfully. In the terminal run the following command: ```kubectl get cm webapp-cfg-v1 webapp-cfg-v2```
- Create a Deployment for each version of the sample web app. Create both the v1 and v2 deployment resource. 
```yaml
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-v1
  namespace: webapp
  labels:
    role: frontend
    version: v1
spec:
  replicas: 2
  selector:
    matchLabels:
      role: frontend
      version: v1
  template:
    metadata:
      labels:
        role: frontend
        version: v1
    spec:
      containers:
      - name: webapp
        image: cloudacademydevops/webappecho:v3
        imagePullPolicy: IfNotPresent
        command: ["/go/bin/webapp"]
        ports:
        - containerPort: 8080
        env:
        - name: MESSAGE
          valueFrom:
            configMapKeyRef:
              name: webapp-cfg-v1
              key: message
        - name: BACKGROUND_COLOR
          valueFrom:
            configMapKeyRef:
              name: webapp-cfg-v1
              key: bgcolor
EOF
```
- Confirm both Deployments were rolled out successfully. In the terminal run the following command:
```bash
{
kubectl rollout status deployment frontend-v1
kubectl rollout status deployment frontend-v2
}
```
- Create a Service for each version of the sample web app
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-v1
  namespace: webapp
  labels:
    role: frontend
    version: v1
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    role: frontend
    version: v1
  type: NodePort
EOF
```
- Confirm both Services have been created and respective Pod endpoints (IPs) have been registered successfully. In the terminal run the following command:
```bash
kubectl get svc,ep
```
- Initially expose just the V1 Service to the Internet. Create an Ingress resource for the V1 sample web app. In the terminal run the following command:
```yaml
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend
  namespace: webapp
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - http:
        paths:
          - pathType: Prefix
            path: /
            backend:
              service:
                name: frontend-v1
                port:
                  number: 80
EOF
```
- Confirm that the Ingress resource was created successfully. In the terminal run the following command:
```bash
kubectl get ingress frontend
```
- Confirm that the ALB and associated DNS records are created for the Ingress resource. In the terminal run the following command:
```bash
{
ALB_FQDN=$(kubectl -n webapp get ingress frontend -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo ALB_FQDN=$ALB_FQDN
until nslookup $ALB_FQDN >/dev/null 2>&1; do sleep 2 && echo waiting for DNS to propagate...; done
until curl --silent $ALB_FQDN | grep CloudAcademy >/dev/null 2>&1; do sleep 2 && echo waiting for ALB to register targets...; done
curl -I $ALB_FQDN
echo
echo READY...
echo browse to:
echo " http://$ALB_FQDN"
}
```
- Update the Ingress resource to include annotations for routing HTTP traffic. Create a new file named **annotations.config** and populate it with **path routing** configuration. In the terminal run the following command:
```bash
kubernetes.io/ingress.class: alb
alb.ingress.kubernetes.io/scheme: internet-facing
alb.ingress.kubernetes.io/target-type: ip
alb.ingress.kubernetes.io/actions.forward-tg-svc1: >
  {"type":"forward","forwardConfig":{"targetGroups":[{"serviceName":"frontend-v1","servicePort":"80"}]}}
alb.ingress.kubernetes.io/actions.forward-tg-svc2: >
  {"type":"forward","forwardConfig":{"targetGroups":[{"serviceName":"frontend-v2","servicePort":"80"}]}}
alb.ingress.kubernetes.io/actions.custom-path1: >
  {"type":"fixed-response","fixedResponseConfig":{"contentType":"text/plain","statusCode":"200","messageBody":"follow the white rabbit..."}}
EOF
```
- Capture the contents of the ```annotations.config``` file in a variable named ```ANNOTATIONS```. In the terminal run the following command:
```bash
{
ANNOTATIONS="$(cat annotations.config)"
echo "$ANNOTATIONS"
}
```
- Create a new file named ingress.annotations.yaml and populate it with the Ingress manifest configuration, injecting the updated set of path routing annotations. In the terminal run the following command:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend
  namespace: webapp
  annotations:
$ANNOTATIONS
spec:
  rules:
    - http:
        paths:
          - path: /yellow
            pathType: Exact
            backend:
              service:
                name: forward-tg-svc1
                port:
                  name: use-annotation
          - path: /cyan
            pathType: Exact
            backend:
              service:
                name: forward-tg-svc2
                port:
                  name: use-annotation
          - path: /white
            pathType: Exact
            backend:
              service:
                name: custom-path1
                port:
                  name: use-annotation
EOF
```
- Apply the updated Ingress into the cluster - ```kubectl apply -f ingress.annotations.yaml```