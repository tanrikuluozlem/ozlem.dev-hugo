+++ 
draft = false
date = 2023-10-20T13:27:54+03:00
title = "How ArgoCD and Rancher Saved Me Time by Automating ECR Image Updates on Kubernetes"

slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++


Here’s how I set up ArgoCD Image Updater to pull container images from Amazon ECR on my Rancher-managed Kubernetes cluster.

- title = "Create AWS Credentials"
The first thing I did was create a Kubernetes secret to hold my AWS credentials, so ArgoCD could authenticate against ECR. It’s simple — just run this command to store the AWS access key and secret in the cluster:
{{< highlight bash >}}
kubectl create secret generic aws-creds-for-ecr \
 - from-literal=AWS_ACCESS_KEY_ID=<your-access-key-id> \
 - from-literal=AWS_SECRET_ACCESS_KEY=<your-secret-access-key> \
 -n argocd
{{</ highlight >}}
I needed this secret later to authenticate with Amazon ECR, and it’s referenced in the ArgoCD configuration.

- Deploy Argo Image-Updater to Your Cluster (Rancher or Self-Hosted)
With the credentials in place, I went ahead and deployed the ArgoCD Image Updater. The Image Updater plays a key role in the cluster by automatically detecting new container images and updating deployments without manual intervention. It’s especially useful for keeping workloads up-to-date with the latest container versions from ECR. You can learn more about it in the link [ArgoCD Image Updater documentation]
Here’s the YAML configuration I used, which sets it up to pull images from ECR and deploy them to my Kubernetes cluster.
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
 name: image-updater-ecr
 namespace: argocd
 labels:
 applicationType: helm
 annotations:
 argocd.argoproj.io/sync-options: Prune=true
 finalizers:
 - resources-finalizer.argocd.argoproj.io
spec:
 project: default
 source:
 repoURL: https://argoproj.github.io/argo-helm
 chart: argocd-image-updater
 targetRevision: 0.11.0
 helm:
 values: |
 config:
 registries:
 - api_url: https://<aws_account_id>.dkr.ecr.<region>.amazonaws.com
 credentials: ext:/scripts/ecr-login.sh
 credsexpire: 12h
 default: true
 insecure: false
 name: ECR
 ping: true
 prefix: <aws_account_id>.dkr.ecr.<region>.amazonaws.com
 authScripts:
 enabled: true
 scripts:
 ecr-login.sh: |
 #!/bin/sh
 aws ecr - region "<region>" get-authorization-token - output text - query 'authorizationData[].authorizationToken' | base64 -d
 extraEnv:
 - name: AWS_ACCESS_KEY_ID
 valueFrom:
 secretKeyRef:
 key: AWS_ACCESS_KEY_ID
 name: aws-creds-for-ecr
 - name: AWS_SECRET_ACCESS_KEY
 valueFrom:
 secretKeyRef:
 key: AWS_SECRET_ACCESS_KEY
 name: aws-creds-for-ecr
 destination:
 server: "https://<your-cluster-api-server>:6443"
 namespace: argocd
 syncPolicy:
 automated:
 prune: true
 selfHeal: true
 syncOptions:
 - CreateNamespace=true
 - ServerSideApply=true
In this setup, the `ecr-login.sh` script is the key. It uses the AWS CLI to get an authorization token from ECR and decode it so ArgoCD can access the images. I set it up to run every time ArgoCD needs to connect to the registry.
Once that YAML was ready, I applied it to the cluster. Now, ArgoCD Image Updater automatically pulls and updates the images from ECR without me needing to worry about it. Rancher keeps my cluster in check, and everything just works smoothly. It’s been a huge time-saver for me in production.