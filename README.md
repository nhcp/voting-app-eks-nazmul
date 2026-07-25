# Voting App on EKS

This is my Project 2 for the Ironhack DevOps bootcamp. In the capstone, I deployed this same polyglot voting app, Python vote UI, Node.js result UI, .NET worker, Redis, and Postgres, across a three-tier EC2 architecture using Terraform for provisioning and Ansible for configuration. This project takes the same app and moves it onto a managed Kubernetes cluster on EKS instead, with GitHub Actions handling the build and deploy pipeline automatically.

## Why EKS

We'd already run Kubernetes locally on Minikube earlier in the bootcamp, but Minikube is a single node on your own laptop. EKS gave me a chance to work with an actual multi-node cluster, IAM permissions, and a managed control plane, closer to what this would look like in a real job.

## Stack

I provisioned the cluster with eksctl, which handles the VPC, nodegroup, and IAM roles for you under the hood through CloudFormation. Once the cluster exists, everything else is kubectl. I used Helm to install the NGINX Ingress Controller, and GitHub Actions handles the CI/CD side: building the images, pushing them to Docker Hub, and applying the manifests.

## Setting up the project

I started by scaffolding the repo before touching AWS at all, with separate folders per service under k8s, so the manifests don't turn into a pile of loose YAML files in the root.

    mkdir -p k8s/redis k8s/postgres k8s/vote k8s/result k8s/worker k8s/ingress
    mkdir -p .github/workflows docs/screenshots

![Repo folder structure](docs/screenshots/01-repo-structure.png)

## Notes

I'll keep adding to this as I go, mostly documenting decisions and any issues I hit along the way, since that's usually more useful than a step that just worked first try.

## Provisioning the EKS cluster

Before any of the app's Kubernetes manifests can be applied, I need an actual cluster to apply them to. I used eksctl to provision it, since it handles the VPC, subnets, IAM roles, and worker node group in one pass instead of me clicking through the AWS console or writing that infrastructure by hand in Terraform.

I kept the nodegroup small, two t3.medium nodes, since this is a bootcamp project on a shared classroom AWS account, not a production workload. Two nodes is still enough to prove the services actually spread across multiple machines rather than all landing on one, which matters for showing this is a real multi-node deployment and not just Minikube with extra steps.

    eksctl create cluster -f cluster-config.yaml --profile nazmul

This took about 15 minutes. Behind the scenes eksctl was building a CloudFormation stack: the EKS control plane itself, a dedicated VPC across two availability zones, IAM roles for both the cluster and the nodes, and finally the EC2 instances for the nodegroup. Once it finished, it also wrote my kubeconfig automatically, so kubectl was immediately pointed at the new cluster with no extra setup.

![EKS cluster created, nodes ready](docs/screenshots/02-cluster-created.png)

