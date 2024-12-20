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

 With this setup, ArgoCD automatically detects and deploys changes whenever a PR is opened or updated in the application repository. These changes are deployed to a preview environment, allowing me to test and validate updates in real time. This process removed the need for manual testing steps for PRs and made feedback much faster.

 By automating PR-based deployments, I was able to make the entire process easier and more reliable. This approach saved time and improved how deployments were handled.

 If you’re working with separate repositories in your GitOps setup, this method can help you manage PR-driven deployments efficiently.