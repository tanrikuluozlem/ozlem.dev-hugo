+++ 
draft = false
date = "2025-01-10T00:00:00+03:00"
title = "Accessing Cross-Workflow Artifacts Between Repositories"

slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

Recently, while working on a project with multiple repositories, I ran into an interesting challenge. I needed to fetch an artifact created by a GitHub Actions workflow from a different repository. This wasn't just a theoretical problem.It was a real blocker in my CI/CD pipeline. After some research, I found a way to make it work.

I had two repositories:

* Auth Service Repository: This repository handled authentication for our platform and published a custom binary as an artifact.
* API Gateway Repository: This repository needed the binary from the Auth Service to run its integration tests during its own GitHub Actions workflow.

I had to fetch the binary artifact from the Auth Service repository into the API Gateway workflow. While both repositories belonged to the same organization, GitHub Actions doesn't allow direct cross-repository artifact access without proper setup.

After a bit of trial and error, I figured out the process to securely fetch the artifact. Here's what worked for me:

Create a GitHub Personal Access Token

The first step was to create a GitHub Personal Access Token (PAT) with the right permissions. Since the artifact was in the Auth Service repository, the PAT needed the actions:read scope for that repository.

Here's how I created it:

1. Went to Settings > Developer Settings > Personal Access Tokens > Tokens (Fine-grained tokens).
2. Generated a new token with actions:read permission for the Auth Service repository.
3. Added this token as a secret (AUTH_SERVICE_PAT) in the API Gateway repository's GitHub Actions secrets.

Identify the Workflow Run (run_id)

Next, I needed the run_id of the workflow run in the Auth Service repository that created the artifact. There were two ways to find this:

* Option A: Manually

I navigated to the Actions tab in the Auth Service repository, found the specific workflow run I needed, and copied its run_id from the URL (it's the number after /runs/).

* Option B: Dynamically via API

For automation, I used the GitHub API to fetch the run_id:

{{< highlight bash >}}

curl -H "Authorization: token ${{ secrets.AUTH_SERVICE_PAT }}" \
     https://api.github.com/repos/auth-service-owner/auth-service/actions/runs

{{</ highlight >}}

This returned a list of workflow runs. I filtered the response to find the specific workflow run based on the branch name, commit hash, or creation time.

Download the Artifact

With the run_id, I could fetch the artifact from the Auth Service repository. Using another GitHub API call, I downloaded the artifact as a ZIP file:

If you're new to GitHub Actions artifacts, you can read more about [artifacts in GitHub Actions](https://docs.github.com/en/actions/using-workflows/storing-workflow-data-as-artifacts)

{{< highlight bash >}}

curl -H "Authorization: token ${{ secrets.AUTH_SERVICE_PAT }}" \
     -L "https://api.github.com/repos/auth-service-owner/auth-service/actions/runs/RUN_ID/artifacts" \
     -o auth-service-artifact.zip

{{</ highlight >}}

After downloading, I unzipped it to access the binary.

Automate the Process in the API Gateway Workflow

To make the process seamless, I incorporated these steps into the API Gateway's GitHub Actions workflow. Here's what the workflow looked like:

{{< highlight yaml >}}

name: Integration Tests with Auth Service Artifact  

on:
  push:
    branches:
      - main  

jobs:
  fetch-and-test:
    runs-on: ubuntu-latest  

    steps:
      - name: Fetch Auth Service Workflow Run ID
        run: |
          RUN_ID=$(curl -H "Authorization: token ${{ secrets.AUTH_SERVICE_PAT }}" \
            https://api.github.com/repos/auth-service-owner/auth-service/actions/runs \
            | jq -r '.workflow_runs[] | select(.status=="completed") | .id | first')
          echo "RUN_ID=$RUN_ID" >> $GITHUB_ENV  

      - name: Download Auth Service Artifact
        run: |
          curl -H "Authorization: token ${{ secrets.AUTH_SERVICE_PAT }}" \
               -L "https://api.github.com/repos/auth-service-owner/auth-service/actions/runs/${{ env.RUN_ID }}/artifacts" \
               -o auth-service-artifact.zip
          unzip auth-service-artifact.zip -d auth-binary  

      - name: Run Integration Tests
        run: |
          ./auth-binary/binary --start
          ./run-api-tests.sh

{{</ highlight >}}

Final Thoughts

Accessing cross-workflow artifacts might seem tricky at first, but with the right approach, it's completely manageable. This experience not only helped me solve a pressing issue but also deepened my understanding of GitHub Actions and CI/CD pipelines.
