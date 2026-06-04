# How to setup alb add on

##  Setup OIDC Connector

#### commands to configure IAM OIDC provider 

```
export cluster_name=demo-cluster
```

## Get EKS OIDC Issuer URL (This URL is the OIDC Provider URL associated with your EKS cluster.)

```
> aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text
o/p: https://oidc.eks.us-east-1.amazonaws.com/id/ABCD1234EFGH5678IJKL9012MNOP345

> oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
> echo $oidc_id
ex: ABEA90D53AD43E718B0AFBBFC2E5FB1C
```
Note: 
Old Cluster --> OIDC ID: ABCD1234
Destroy Cluster
New Cluster --> OIDC ID: XYZ98765

## Check if there is an IAM OIDC provider configured already

- aws iam list-open-id-connect-providers
or
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4  (Not working)

`If not`, run the below command

Step 1: Configure OIDC Provider

Creates trust between `EKS Cluster` (iamserviceaccount will use it) and `AWS IAM`, After run below command IAM can trust `Kubernetes Service Accounts`.

```
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```

## Download IAM policy

```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

### Create IAM Policy

```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

Creates permissions like:
    elasticloadbalancing:CreateLoadBalancer
    elasticloadbalancing:CreateTargetGroup
    ec2:DescribeSubnets
    ec2:DescribeSecurityGroups
    
### Create Service Account + IAM Role

```
eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

> kubectl get serviceaccounts -A | grep -i aws-load-balancer-controller

Note: update <your-cluster-name> and <your-aws-account-id>
if you won't create OIDC you will error like below
```
2026-06-04 11:18:45 [!]  no IAM OIDC provider associated with cluster, try 'eksctl utils associate-iam-oidc-provider --region=us-east-1 --cluster=my-eks-cluster'
Error: unable to create iamserviceaccount(s) without IAM OIDC provider enabled
```

##  OIDC+IAM Flow:
```
eksctl create iamserviceaccount
          |
          v
CloudFormation Stack
          |
          +--> IAM Role
          +--> Trust Relationship (OIDC)
          +--> IAM Role Policies
          |
          v
Kubernetes ServiceAccount
```

Note: OIDC provider alredy enable between EKS and IAM, So now Service account can use IAM role with policy permissions, these Service account attached to ALB controller while installing using helm

## Deploy ALB controller

Add helm repo

```
helm repo add eks https://aws.github.io/eks-charts
helm repo list
helm search repo eks | grep -i aws-load-balancer-controller
o/p:
eks/aws-load-balancer-controller        	3.4.0        	v3.4.0                     AWS Load Balancer Controller Helm chart for Kub...
```

Update the repo

```
helm repo update eks
```

Install

```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
--namespace kube-system \
--set clusterName=<your-cluster-name> \
--set serviceAccount.create=false \
--set serviceAccount.name=aws-load-balancer-controller \
--set region=<region> \
--set vpcId=<your-vpc-id>
```
Note: Give correct VPC id , if you give wrong value also it will show error until you check logs of pods.

> helm get values aws-load-balancer-controller -n kube-system
> if you want to update with new value use below
```
helm upgrade aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-eks-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=vpc-053ce01c486ce2014
```
> kubectl rollout restart deployment aws-load-balancer-controller -n kube-system

Verify that the deployments are running.
> kubectl get deployment -n kube-system aws-load-balancer-controller
> kubectl get ingressclass (if you use this class in ingress resource then it will use aws-load-balancer-controller for traffic flow)

You might face the issue, unable to see the loadbalancer address while giving k get ing -n robot-shop at the end. To avoid this your **AWSLoadBalancerControllerIAMPolicy** should have the required permissions for elasticloadbalancing:DescribeListenerAttributes.

## Run the following command to retrieve the policy details and look for **elasticloadbalancing:DescribeListenerAttributes** in the policy document.
```
aws iam get-policy-version \
    --policy-arn arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
    --version-id $(aws iam get-policy --policy-arn arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy --query 'Policy.DefaultVersionId' --output text)
```

If the required permission is missing, update the policy to include it
## Download the current policy
```
aws iam get-policy-version \
    --policy-arn arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
    --version-id $(aws iam get-policy --policy-arn arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy --query 'Policy.DefaultVersionId' --output text) \
    --query 'PolicyVersion.Document' --output json > policy.json
```
## Edit policy.json to add the missing permissions
```
{
  "Effect": "Allow",
  "Action": "elasticloadbalancing:DescribeListenerAttributes",
  "Resource": "*"
}
```
## Create a new policy version
```
aws iam create-policy-version \
    --policy-arn arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://policy.json \
    --set-as-default
```
