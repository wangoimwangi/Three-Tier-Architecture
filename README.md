# Three-Tier-Architecture 
A three-tier application is a software architecture pattern that divides the application into three distinct layers:
1. Presentation Layer: The front-end that interfaces with users, displaying data and gathering inputs.
2. Business Logic Layer: Processes inputs, applies rules, and mediates between presentation and data layers.
3. Data Layer: Manages data storage, handling queries and updates.

For this project we have a total of 6 microservices and 2 database which will be deployed in AWS EKS. Each microservice is written in a different language.
## Prerequisites
kubectl – A command line tool for working with Kubernetes clusters. For more information, see Installing or updating kubectl. https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html

eksctl – A command line tool for working with EKS clusters that automates many individual tasks. For more information, see Installing or updating. https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html

AWS CLI – A command line tool for working with AWS services, including Amazon EKS. For more information, see Installing, updating, and uninstalling the AWS CLI https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html in the AWS Command Line Interface User Guide.

After installing the AWS CLI, I recommend that you also configure it. For more information, see Quick configuration with aws configure in the AWS Command Line Interface User Guide. https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-config

## EKS Cluster Setup
Creating the Cluster
```
eksctl create cluster --name demo-cluster-three-tier-1 --region us-east-1
```
Deleting the Cluster

```
eksctl delete cluster --name demo-cluster-three-tier-1 --region us-east-1
```
## Configure IAM OIDC Provider
```
export cluster_name=<CLUSTER-NAME>
```
```
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
```
Check if there is an IAM OIDC provider configured already
```
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```
If not, run the below command
```
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```
## Set up Application Load Balancer
Download IAM policy
```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```
Create IAM Policy
```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```
Create IAM Role
```
eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```
### Deploy ALB controller
Add helm repo
```
helm repo add eks https://aws.github.io/eks-charts
```
Update the repo
```
helm repo update eks
```
Install
```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \            
  -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region> \
  --set vpcId=<your-vpc-id>
```
Verify that the deployments are running.
```
kubectl get deployment -n kube-system aws-load-balancer-controller
```
## EBS CSI Plugin Configuration
The Amazon EBS CSI Plugin enables Kubernetes on AWS to manage EBS volumes for persistent storage, simplifying provisioning, attachment, and lifecycle management.
The Amazon EBS CSI plugin requires IAM permissions to make calls to AWS APIs on your behalf. Create an IAM role and attach a policy. AWS maintains an AWS managed policy or you can create your own custom policy. 
```
eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster <YOUR-CLUSTER-NAME> \
    --role-name AmazonEKS_EBS_CSI_DriverRole \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve
```
Run the following command. Replace with the name of your cluster, with your account ID.
```
eksctl create addon --name aws-ebs-csi-driver --cluster <YOUR-CLUSTER-NAME> --service-account-role-arn arn:aws:iam::<AWS-ACCOUNT-ID>:role/AmazonEKS_EBS_CSI_DriverRole --force
```
Note: If your cluster is in the AWS GovCloud (US-East) or AWS GovCloud (US-West) AWS Regions, then replace arn:aws: with arn:aws-us-gov:.

## Configuring Helm
Helm is a package manager for Kubernetes that facilitates the deployment and management of applications.
Helm v2.x
```
$ helm install --name robot-shop --namespace robot-shop 
```
Helm v3.x
```
$ kubectl create ns robot-shop
$ helm install robot-shop --namespace robot-shop
```



