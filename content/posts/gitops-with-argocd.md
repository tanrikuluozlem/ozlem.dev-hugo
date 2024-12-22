+++ 
draft = false
date = "2024-12-20T16:57:50+03:00"
title = "GitOps with ArgoCD: Managing Pull Requests Across Separate Repositories"

slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

In my GitOps workflow, I faced a unique challenge: the application repository and the GitOps repository were separate. I needed a way to make ArgoCD track pull requests (PRs) in the application repository while still handling deployments through the GitOps repository.

To solve this, I created a Kubernetes Secret containing a personal access token (PAT). This token allowed ArgoCD to access the application repository and track PRs easily.

Here’s the YAML configuration for the Secret:

{{< highlight yaml >}}
apiVersion: v1  
kind: Secret  
metadata:  
  name: argocd-project-pat-secret  
  namespace: argocd  
type: Opaque  
data:  
  token: {{ .Values.argocd.patToken | b64enc }}
 {{</ highlight >}} 

 This Secret securely stores the token, ensuring ArgoCD has the necessary permissions to monitor the application repository.

 In my GitOps setup, I created two types of environments using ArgoCD’s ApplicationSets:

 1. Ephemeral Environments:
These environments are automatically generated from pull requests. Each PR creates a unique namespace, providing an isolated environment for testing without affecting others.

 2. Persistent Environments:
For stable environments like QA and production, I use a list generator to manage them. These environments are deployed using a custom Helm chart, ensuring they are consistent and reliable.

Both types of environments are deployed using our custom Helm charts, We have the following Helm release files for the environments:

ephemeral-helm-release.yaml

{{< highlight yaml >}}
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: dog-eph-helm-releases
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - pullRequest:
      requeueAfterSeconds: 30
      github:
        owner: dog
        repo: dog.dog
        tokenRef:
          secretName: argocd-pat-secret
          key: token
  template:
    metadata:
      name: 'dog-pr{{ .number}}-helm-release'
    spec:
      project: default
      source:
        path: backend/chart
        repoURL: https://github.com/dog/dog.git
        helm:
          releaseName: dog-pr{{ .number}}
          values: |
            postgres:
              enabled: true
            nodeSelector:
              node-pool: "system"
      destination:
        server: "https://<your-cluster-api-server>:6443"
        namespace: dog-pr{{ .number}}
      syncPolicy:
        automated: 
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - ServerSideApply=true
 {{</ highlight >}} 

 persistent-helm-release.yaml

 {{< highlight yaml >}}
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: dog-persist-helm-release
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - list:
      elements:
      - env: qa
      - env: prod
  template:
    metadata:
      name: 'dog-{{ .env}}-helm-release'
    spec:
      project: default
      source:
        #chart: neu-residence-hub-backend
        #targetRevision: 0.1.0
        path: backend/chart
        repoURL: https://github.com/dog/dog.git
        helm:
          releaseName: dog-{{ .env}}
          values: |
            postgres:
              enabled: false
              secretName: dog-db-{{ .env}}-secret
              configMapName: dog-postgres-{{ .env}}-cm
            jwt:
              secretName: dog-jwt-{{ .env}}-secret
            nodeSelector:
              node-pool: "system"
      destination:
        server: "https://<your-cluster-api-server>:6443"
        namespace: dog-{{ .env}}
      syncPolicy:
        automated: 
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - ServerSideApply=true
 {{</ highlight >}} 

 With this setup, ArgoCD automatically detects and deploys changes whenever a PR is opened or updated in the application repository. These changes are deployed to a preview environment, allowing me to test and validate updates in real time. This process removed the need for manual testing steps for PRs and made feedback much faster.

 By automating PR-based deployments, I was able to make the entire process easier and more reliable. This approach saved time and improved how deployments were handled.

 If you’re working with separate repositories in your GitOps setup, this method can help you manage PR-driven deployments efficiently.