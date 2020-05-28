# Spinnaker On Amazon EKS
This repository introduce how to deploy Spinnaker on Amazon EKS Cluster.
<br>Reference link : https://aws.amazon.com/cn/blogs/opensource/continuous-delivery-spinnaker-amazon-eks/

### Prepare Cloud9 Enviorment

Create Cloud9 EC2 Enviorment
![avatar](https://github.com/toreydai/spinnaker-on-eks/blob/master/resources/spin1.png)
Attach the Cloud9 EC2 instance with IAM Role
![avatar](https://github.com/toreydai/spinnaker-on-eks/blob/master/resources/spin2.png)

### Install AWS CLI

```
sudo apt update -y
sudo apt install python3-pip -y
pip3 install awscli --upgrade --user
which aws
sudo mv ~/.local/bin/aws /usr/bin
aws --version
```
### Install Kubectl

```
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.16.8/2020-04-16/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
kubectl version --short --client
```
### Install eksctl

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

### Install EKS Cluster

```  
eksctl create cluster \
--name eks-spinnaker \
--version 1.16 \
--write-kubeconfig=false \
--region us-east-1 \
--nodegroup-name prod \
--node-type c5.large \
--nodes 3 \
--nodes-min 1 \
--nodes-max 4 \
--ssh-access \
--ssh-public-key taodai_global_virginia \
--managed
```
### Retrive Amazon EKS Cluster kubectl contexts

```
aws eks update-kubeconfig --name eksworkshop --region us-east-1 \
  --alias eks-spinnaker
```
### Install halvard

```
curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh
sudo bash InstallHalyard.sh

hal -v
```
### Create and configure a Docker registry

```
hal config provider docker-registry enable 
hal config provider docker-registry account add taodai \
  --address index.docker.io --username taodai --password
```
### Create Github Token

Follow the link below to create Github access token:
<br>https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line
![avatar](https://github.com/toreydai/spinnaker-on-eks/blob/master/resources/spin4.png)

### Add and configure a GitHub account

```
hal config artifact github enable
hal config artifact github account add spinnaker-github --username toreydai \
  --password --token
```
### Add and configure Kubernetes accounts

```
hal config provider kubernetes enable
kubectl config use-context eks-spinnaker

CONTEXT=$(kubectl config current-context)
kubectl apply --context $CONTEXT \
    -f https://spinnaker.io/downloads/kubernetes/service-account.yml

SECRET_NAME=$(kubectl get serviceaccount spinnaker-service-account \
       --context $CONTEXT \
       -n spinnaker \
       -o jsonpath='{.secrets[0].name}')
TOKEN=$(kubectl get secret --context $CONTEXT \
  $SECRET_NAME \
  -n spinnaker \
  -o jsonpath='{.data.token}' | base64 --decode)
  
kubectl config set-credentials ${CONTEXT}-token-user --token $TOKEN
kubectl config set-context $CONTEXT --user ${CONTEXT}-token-user

hal config provider kubernetes account add eks-spinnaker --provider-version v2 \
 --docker-registries taodai --context $CONTEXT
```
### Enable artifact support

```
hal config features edit --artifacts true
```
### Configure Spinnaker to install in Kubernetes

```
hal config deploy edit --type distributed --account-name eks-spinnaker
```
### Configure Spinnaker to use AWS S3

```
# If your region is us-east-1 ，parameter "--region" is not needed
export YOUR_ACCESS_KEY_ID=<access-key>
hal config storage s3 edit --access-key-id $YOUR_ACCESS_KEY_ID --secret-access-key
```
### Choose the Spinnaker version

```
# Choose the latest spinnaker version
hal version list
export VERSION=1.20.3
hal config version edit --version $VERSION
hal deploy apply
```
### Verify the Spinnaker installation
![avatar](https://github.com/toreydai/spinnaker-on-eks/blob/master/resources/spin5.png)

### Expose Spinnaker using Elastic Loadbalancer

```
export NAMESPACE=spinnaker
kubectl -n ${NAMESPACE} expose service spin-gate --type LoadBalancer \
  --port 80 --target-port 8084 --name spin-gate-public 

kubectl -n ${NAMESPACE} expose service spin-deck --type LoadBalancer \
  --port 80 --target-port 9000 --name spin-deck-public

export API_URL=$(kubectl -n $NAMESPACE get svc spin-gate-public \
 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

export UI_URL=$(kubectl -n $NAMESPACE get svc spin-deck-public \
 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

hal config security api edit --override-base-url http://${API_URL}
hal config security ui edit --override-base-url http://${UI_URL}
hal deploy apply
```
### Get the UI URL and access Spinnaker console

```
echo URL=http://$(kubectl get svc spin-deck-public -n $NAMESPACE -o json | jq -r '.status.loadBalancer.ingress[].hostname')
```
![avatar](https://github.com/toreydai/spinnaker-on-eks/blob/master/resources/spin6.png)
![avatar](https://github.com/toreydai/spinnaker-on-eks/blob/master/resources/spin7.png)

### Clean Up

```
eksctl delete nodegroup --name=prod --cluster=eks-spinnaker --region us-east-1
eksctl delete cluster --name eks-spinnaker --region us-east-1
```
