---
cover:
    image: '/media/images/aws/integrate-eks-and-github-actions/header.svg'
    alt: "Integrate AWS EKS and GitHub Actions"
date: '2025-04-21T00:13:02Z'
title: 'Integrate AWS EKS and GitHub Actions'
tags: ["AWS", "GitHub", "Git", "CI/CD", "Kubernetes", "Container"]
category: ["AWS"]
ShowToc: true
---

# Introduction
In today's cloud-native world, container orchestration has become a must for scaling applications reliably. Kubernetes (K8s), an open-source orchestration platform, is the de facto standard for managing containers in production. But managing the control plane, networking, and high availability? That’s not something you want to handle manually.

That’s where Amazon EKS (Elastic Kubernetes Service) comes in. EKS is a managed Kubernetes service by AWS. It takes care of provisioning the control plane, automating upgrades, and integrating deeply with other AWS services like IAM, VPC, CloudWatch, and Load Balancers.

To simplify you and your team's work, we can use GitHub Actions to automate build, test, and deployment for your applications that you want to run in Kubernetes.

# Prerequisites
* IAM Role – to manage AWS resource used by AWS CLI.
* ECR Repository – to save your container image.
* AWS CLI – to access AWS services from the command line.
* kubectl – to interact with the Kubernetes cluster.
* eksctl – to simplify EKS cluster creation and management.
* helm – to install Kubernetes applications.
* GitHub Account – to use GitHub Actions.

# Workflow
![Workflow](/media/images/aws/integrate-eks-and-github-actions/workflow.svg)

# Steps

## Configure AWS CLI
* We start by configuring AWS CLI so that your local machine can communicate with AWS using your access credentials.
  ```bash
  aws configure
  ```

* You'll be prompted to input your AWS credentials and default region:
  ```txt
  AWS Access Key ID [None]: YOUR_ACCESS_KEY
  AWS Secret Access Key [None]: YOUR_SECRET_KEY
  Default region name [None]: us-east-1
  Default output format [None]: json
  ```

## Creating AWS EKS Cluster via eksctl
* This command uses `eksctl` to provision an entire Kubernetes cluster on AWS, including the control plane and worker nodes.
  ```bash
  eksctl create cluster --name my-cluster --region us-east-1
  ```

## Configure Load Balancer IngressClass
* Associate the IAM OIDC provider so service accounts can assume roles.
  ```bash
  eksctl utils associate-iam-oidc-provider --region us-east-1 --cluster my-cluster --approve
  ```

* Download the IAM policy for the AWS Load Balancer Controller.
  ```bash
  curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/refs/heads/main/docs/install/iam_policy.json
  ```

* Create the IAM policy in AWS using the downloaded JSON file.
  ```bash
  aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam-policy.json
  ```

* Get your current AWS account ID.
  ```bash
  aws sts get-caller-identity --query "Account" --output text
  ```

* Create a Kubernetes service account and attach the IAM policy to it.
  ```bash
  eksctl create iamserviceaccount --cluster=my-cluster --namespace=kube-system --name=aws-load-balancer-controller --attach-policy-arn=arn:aws:iam::<AWSAccountNumber>:policy/AWSLoadBalancerControllerIAMPolicy --override-existing-serviceaccounts --region us-east-1 --approve
  ```

* Add the EKS Helm chart repo.
  ```bash
  helm repo add eks https://aws.github.io/eks-charts
  ```

* Update your Helm repositories.
  ```bash
  helm repo update eks
  ```

* Get your VPC ID associated with the EKS cluster.
  ```bash
  aws ec2 describe-vpcs --filters "Name=tag:eksctl.cluster.k8s.io/v1alpha1/cluster-name,Values=my-cluster" --query "Vpcs[0].VpcId" --output text
  ```

* Install the AWS Load Balancer Controller via Helm.
  ```bash
  helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=my-cluster --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller --set region=us-east-1 --set vpcId=<yourVPCID>
  ```

## Deploying WebApp using GitHub Actions

* First thing first, you need an application that you wanna deploy to EKS. Here I'm deploying a simple tetris game [Canvas Tetris](https://github.com/rifkyards/canvas-tetris).

* To create an automated deployment, you need a workflow file placed in the `.github/workflows` directory in your repo with a `.yaml` extension. The workflow below builds and deploys the app to EKS on every push to the `master` branch.

  ```yaml
  name: Deploy to Amazon EKS

  on:
    push:
      branches:
      - master

  env:
    AWS_REGION: ${{ vars.AWS_REGION }}
    ECR_REPOSITORY: ${{ vars.ECR_REPOSITORY }}
    EKS_CLUSTER_NAME: ${{ vars.EKS_CLUSTER_NAME }}

  permissions:
    id-token: write
    contents: read

  jobs:
    build:
      name: Build Image
      runs-on: ubuntu-latest
      steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Set short git commit SHA
        id: commit
        uses: prompt/actions-commit-hash@v2

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build, tag, and push image to Amazon ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ steps.commit.outputs.short || 'latest' }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
      - name: End Build
        run: echo "Build Success"

    deploy:
      name: Deploy Application
      needs: build
      runs-on: ubuntu-latest
      steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Set short git commit SHA
        id: commit
        uses: prompt/actions-commit-hash@v2

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Update kube config
        run: aws eks update-kubeconfig --name $EKS_CLUSTER_NAME --region $AWS_REGION

      - name: Deploy to EKS
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}        
          IMAGE_TAG: ${{ steps.commit.outputs.short || 'latest' }}
        run: |
          sed -i "s|DOCKER_IMAGE|$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG|g" manifests/deployment.yaml && \
          kubectl apply -f manifests/deployment.yaml
          kubectl apply -f manifests/service.yaml
          kubectl apply -f manifests/ingress.yaml
  ```

* As seen above, we also use environment variables (`env` and `secrets`) to keep the workflow dynamic and secure. For more info on setting those, check [GitHub’s official docs](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/store-information-in-variables#creating-configuration-variables-for-a-repository).
