+++ 
draft = false
date = "2025-02-07T00:00:00+03:00"
title = "How I Simplified CI/CD with Reusable GitHub Actions Workflows"

slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

Managing multiple repositories with common CI/CD pipelines quickly becomes a headache. Every time I needed to edit a workflow, whether it was updating a dependency, fixing a security issue, or improving build efficiency. I had to modify the same logic across multiple repositories. It wasn’t sustainable. 🤦🏼‍♀️

I decided to fix this by creating a centralized repository for reusable GitHub Actions workflows, so I only have to maintain them in one place. Here’s how I set it up and why it has made my life so much easier.

Here’s one of my repositories neu-residence-hub old workflow situation looked like:


{{< highlight yaml >}}

name: "Build and Publish"
on:
  push:
    branches:
      - main
    tags:
      - release-*
  pull_request:
    types: [opened, synchronize, reopened]
    branches:
      - main

jobs:
  build-and-publish:
    runs-on: ubuntu-latest
    steps:
      - name: Setup build env and id
        id: setup_build_env_id
        run: |
          if [[ ${{ github.event_name }} == pull_request ]]; then
            export TAG=pr${{ github.event.number }}-$(date "+%Y%m%d%H%M%S")
          elif [[ ${{ github.event_name }} == push && ${{ github.ref_name }} == "main" ]]; then
            export TAG=main-$(date "+%Y%m%d%H%M%S")
          elif [[ ${{ github.event_name }} == push && ${{ github.ref_name }} == release-* ]]; then
            export TAG=${{ github.ref_name }}
          fi
          echo "TAG=$TAG" >> $GITHUB_OUTPUT

      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.ECR_PUSH_AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.ECR_PUSH_AWS_SECRET_ACCESS_KEY }}
          aws-region: eu-central-1

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3 

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
        with:
          registries: "xxxxxxxx"

      - name: Build, tag, and push docker image to Amazon ECR
        env:
          REGISTRY: <account id>.dkr.ecr.eu-central-1.amazonaws.com
          REPOSITORY: neu-residence-hub
          IMAGE_TAG: ${{ steps.setup_build_env_id.outputs.TAG }}
        run: |
         docker buildx create --use
         docker buildx build --platform linux/arm64 -f ./backend/Dockerfile -t $REGISTRY/$REPOSITORY:$IMAGE_TAG --push ./backend/lib/lambdaFunctions/go

{{</ highlight >}}

Instead of maintaining separate workflows everywhere, I created a dedicated GitHub repository(SharedWorkflows) where I store all my reusable workflows.

Now, instead of duplicating workflow YAML files, my projects just reference the shared workflow like a function call.

Here’s what the shared workflow looks like:

Reusable Workflow: Docker Build, Scan, and Push

docker-build-scan-push.yaml

{{< highlight yaml >}}

on:
  workflow_call:
    inputs:
      repository:
        required: true
        type: string
      dockerfile:
        required: true
        type: string
      imageTag:
        required: true
        type: string
      dockerContext:
        required: false
        type: string
        default: '.'
permissions:
  id-token: write
  contents: read
jobs:
  build-and-scan:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<account id>:role/RolX
          aws-region: eu-central-1
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and tag Docker image
        env:
          REGISTRY: <account id>.dkr.ecr.eu-central-1.amazonaws.com
          REPOSITORY: ${{ inputs.repository }}
          DOCKERFILE: ${{ inputs.dockerfile }}
          IMAGE_TAG: ${{ inputs.imageTag }}
          BUILD_CONTEXT: ${{ inputs.dockerContext }}
        run: |
          docker buildx build --platform linux/arm64 -f $DOCKERFILE -t $REGISTRY/$REPOSITORY:$IMAGE_TAG --load $BUILD_CONTEXT

      - name: Scan image for vulnerabilities
        uses: aquasecurity/trivy-action@0.28.0
        with:
          image-ref: '${{ inputs.repository }}:${{ inputs.imageTag }}'
          format: 'table'
          exit-code: '1'
          ignore-unfixed: true
          vuln-type: 'os,library'
          severity: 'CRITICAL,HIGH,LOW'

      - name: Push Docker image to Amazon ECR
        run: |
          docker push $REGISTRY/$REPOSITORY:$IMAGE_TAG

{{</ highlight >}}

How My Repos Use the Shared Workflow

Don’t forget to enable access for the shared workflows in this repository from GitHub .

You can find the Access section in Repository>Settings>Actions>General and turn on the following;

* Accessible from repositories in the ‘ <organization name>’ organization

Then, I call the shared workflow one of my repositoriesneu-residence-hub.

This simplifies the setup and ensures that any updates to the pipeline logic are automatically applied across all repositories that use this shared workflow.

Here’s how my neu-residence-hub repo references it:

{{< highlight yaml >}}

call_build_scan_push:
    needs: setup
    uses: <organization name>/SharedWorkflows/.github/workflows/docker-build-scan-push.yaml@main
    with:
      repository: "<account id>.dkr.ecr.eu-central-1.amazonaws.com/neu-residence-hub"
      imageTag: ${{ needs.setup.outputs.TAG }}  
      dockerfile: "backend/Dockerfile"
      dockerContext: "./backend/lib/lambdaFunctions/go"

{{</ highlight >}}

Setting this up took only about an hour, but it has already saved me way more time than that in maintenance work.

If you’re managing multiple repositories and think about get rid of duplicating your CI/CD workflows, I highly recommend centralizing them. It’s one of those things you won’t regret once you do it! 🙌🏻



