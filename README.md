# README #

### What is this repository for? ###

* Quick summary

## Quick Summary
This provides a detailed documentation for creating EKS environment from scratch with a public and private subnets

## Assumptions
* Starting with creation of a new VPC from scratch

## Infrastructure Setup

* Step1: Create the required EKS Service Role

  ```
  aws cloudformation create-stack --stack-name LIL-EKS-ServiceRole --capabilities CAPABILITY_NAMED_IAM  --template-body file://./cf-templates/01-eks-service-role.yaml

  ```

  Check the stack status using
  ```
  aws cloudformation describe-stacks --stack-name LIL-EKS-ServiceRole
  ```

* Step2: Create the Network Configuration For the Subnets and corresponding IP routing tables

  ```
  aws cloudformation create-stack  \
    --stack-name LIL-EKS  \
    --template-body file://./cf-templates/02-eks-vpc-private-public.yaml
  ```

  Check the stack status using 
  ```
  aws cloudformation describe-stacks --stack-name LIL-EKS
  ```

  Once stack is created use the following command to record the output params

  ```
   aws cloudformation describe-stacks --stack-name LIL-EKS > eks_stack_info.log
  ```

* Step3: Create the EKS Cluster

  Run the following command to create the EKS Cluster

  ```
  aws cloudformation create-stack  \
      --stack-name LIL-EKS-Infrastructure  \
      --template-body file://./cf-templates/03-eks-create.yaml \
      --parameters  ParameterKey=eksStackName,ParameterValue=LIL-EKS
  ```

  Check the stack status using 
  ```
  aws cloudformation describe-stacks --stack-name LIL-EKS-Infrastructure
  ```

* Step4: Create the node group

  Now lets create the node group

  ```
  aws cloudformation create-stack  \
      --stack-name LIL-EKS-NodeGroup  \
      --capabilities CAPABILITY_NAMED_IAM \
      --template-body file://./cf-templates/04-eks-nodegroup.yaml \
      --parameters  ParameterKey=eksStackName,ParameterValue=LIL-EKS ParameterKey=eksClusterName,ParameterValue=lil-eks-cluster ParameterKey=sshKey,ParameterValue=inappbidev
  ```

This should successfully launch the EKS cluster and set the EKS cluster that you can start deploying services

* Setp5: Create a Jenkins for CI/CD

  Create a Jenkins instance which will be used for the code build and deployment

  ```
  aws cloudformation create-stack  \
      --stack-name LIL-EKS-Jenkins  \
      --template-body file://./cf-templates/05-eks-jenkins-cicd.yaml \
      --parameters  ParameterKey=KeyName,ParameterValue=inappbidev ParameterKey=VpcId,ParameterValue=vpc-9bb2cdfe ParameterKey=SubnetId,ParameterValue=subnet-ab5fcddc
  ```

  Configure jenkins to setup admin userid and passoword and required plugins

## Configure Jenkins to interact with Kube-Cluster


----

# Installation of the Application

Deployment of the application is customized and configured using Jenkins Build File

* Step1: Configure Jenkins
* Step2: Install required plugins
  - Blue Ocean
* Step3: Configure GitHub repo to trigger build
  - Give the git creds
  - Select the Jenkins file that has the build in the repo

----

# Other Tasks

## Jenkins Setup: Work around to make jenkins user connect using kubectl

1. Remove AWS CLI

  sudo pip remove aws
  sudo rm -rf /usr/local/aws
  sudo rm /usr/local/bin/aws
  sudo rm /usr/bin/aws

2. Install CLI Version1 again
  [Install the V1](https://docs.aws.amazon.com/cli/latest/userguide/install-bundle.html)
  
3. Login as jenkins   
  sudo su -s /bin/bash jenkins

4. Run AWS Configure so that kubectl can 
  aws configure

5. Create kubeconfig under jenkins user
  aws eks --region us-east-1 update-kubeconfig --name lil-eks-cluster


## Install KubeAdmin UI (Optional)

* Step1: Deploy the dashboard to the cluster

  ```
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml
  ```

* Step2: Create an eks-admin Service Account and Cluster Role Binding

  ```
   kubectl apply -f eks/01-eks-admin-service.yaml
  ```

## Uninstall the EKS Stack

  To Delete the Stack
  ```
  aws cloudformation delete-stack --stack-name LIL-EKS-ServiceRole
  aws cloudformation delete-stack --stack-name LIL-EKS
  aws cloudformation delete-stack --stack-name LIL-EKS-Infrastructure
  aws cloudformation delete-stack --stack-name LIL-EKS-NodeGroup
  aws cloudformation delete-stack --stack-name LIL-EKS-Jenkins
  ```

## Configure Kube to connect from local box

* Create KubeConfig file locally to test out the stack

  Get the aws keys to connect to the EKS cluster
  ```
  aws configure
  ```

  This step will update cat ~/.kube/config 
  ```
  aws eks --region us-east-1 update-kubeconfig --name lil-eks-cluster
  ```

  Run kubectl to check if the nodes are up and running
  ```
  kubectl get svc
  ```

* Upgradign aws cli version1

  ```
  pip install awscli --upgrade --user
  ```


### Contribution guidelines ###

* Writing tests
* Code review
* Other guidelines

### Who do I talk to? ###

* Repo owner or admin
* Other community or team contact

### Reference Documentation

* [AWS EKS Help](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-console.html)
* [Cloud Formation MetaData](https://www.atcomputing.nl/blog/aws-cloudformation-metadata/)
* [Jenkins Docker](https://jenkins.io/doc/book/pipeline/docker/)
* [Jenkins with S3 backup](https://medium.com/@jonah.jones/one-click-jenkins-cicd-in-less-than-4-minutes-using-cloudformation-and-docker-abb4a26785e2)
* [ALB Ingress Controller](https://kubernetes-sigs.github.io/aws-alb-ingress-controller/)
* [ALB Ingress Controller - Sample](https://kubernetes-sigs.github.io/aws-alb-ingress-controller/guide/walkthrough/echoserver/)

### TODO

> TODO : Cleanup Action Items

```
1 : Remove the stack name hardcoding
2 : Remove hardcoding of the stack name in the jenkins and also move the subnetId selection
3 : Add the naming for the spineed off EC2 Instances or atleast add the tags if name is not feasible
4 : Sync Jenkins config to S3 so that you can get the config from S3
5 : Move the jenkins to public Subnet of the VPC
6 : Change the cluster name right now its getting created as eks-cluster
7 : Jenkins Creds Storage
```


### Troubleshoot

* Sudo as Jenkins

  ```
  sudo su -s /bin/bash jenkins
  ```

* Not able to run docker form jenkins
  
  Solution is to add jenkins to the docker group

  ```
  sudo usermod -aG docker jenkins
  ```

* Not able to use aws cli 

Version: **V0.1**


----

Random notes


## Setting up ALB Ingress Controller

Follow the installation instructions at 

[Info](https://kubernetes-sigs.github.io/aws-alb-ingress-controller/guide/walkthrough/echoserver/)

wget https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.5/docs/examples/alb-ingress-controller.yaml
wget https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.5/docs/examples/rbac-role.yaml

Delete Ingress Controller

```
kubectl delete -n echoserver ingress echoserver
```

839749875901.dkr.ecr.us-east-1.amazonaws.com/mythicalmysfits/service:latest
