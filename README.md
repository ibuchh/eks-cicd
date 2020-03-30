# CI/CD using AWS CodePipeline with Amazon EKS | Worskhop
## Objectives

* Learn about creating an Amazon EKS cluster
* Learn about AWS CodePipeline, AWS CodeBuild services
* Gain familiarity with DevSecOps


## Create Amazon EKS cluster (Linux/Mac)
```
export EKS_CLUSTER=eks-cluster

curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

sudo mv -v /tmp/eksctl /usr/local/bin

eksctl version

eksctl create cluster --name=$EKS_CLUSTER --version=1.15 --nodes=3 --profile=dev-profile --region=us-east-1

(... _This takes about 15 minutes to complete_ ...)

kubectl get nodes
```

## Create an IAM Role

To deploy a sample service to Amazon EKS using AWS CodeBuild inside AWS CodePipeline, we need to create an AWS Identity and Access Management (IAM) role. This IAM role will enable AWS CodeBuild to interact with Amazon EKS.


```
export EKS_CODEBUILD_KUBECTL_ROLE=EksWorkshopCodeBuildKubectlRole

export ACCOUNT_ID=`aws sts get-caller-identity --output text --query Account`

TRUST="{ \"Version\": \"2012-10-17\", \"Statement\": [ { \"Effect\": \"Allow\", \"Principal\": { \"AWS\": \"arn:aws:iam::${ACCOUNT_ID}:root\" }, \"Action\": \"sts:AssumeRole\" } ] }"

echo '{ "Version": "2012-10-17", "Statement": [ { "Effect": "Allow", "Action": "eks:Describe*", "Resource": "*" } ] }' > /tmp/iam-role-policy

aws iam create-role --role-name $EKS_CODEBUILD_KUBECTL_ROLE --assume-role-policy-document "$TRUST" --output text --query 'Role.Arn'

aws iam put-role-policy --role-name EksWorkshopCodeBuildKubectlRole --policy-name eks-describe --policy-document file:///tmp/iam-role-policy
```

## Modify aws-auth ConfigMap

Let us add the role to the aws-auth ConfigMap for the EKS cluster so that kubectl in the CodeBuild stage of the pipeline will be able to interact with the EKS cluster via the IAM role.

```
ROLE="    - rolearn: arn:aws:iam::$ACCOUNT_ID:role/$EKS_CODEBUILD_KUBECTL_ROLE\n      username: build\n      groups:\n        - system:masters"

kubectl get -n kube-system configmap/aws-auth -o yaml | awk "/mapRoles: \|/{print;print \"$ROLE\";next}1" > /tmp/aws-auth-patch.yml

kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"
```

## Reference Architecture
At the conclusion of this workshop, you will end up with various AWS services provisioned in your AWS account. The following diagram illustrates some of these services and is intended as a sample reference architecture. 

![Alt](/src/assets/images/DevSecOps.png "Architecture")
