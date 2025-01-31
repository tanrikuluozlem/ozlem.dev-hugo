+++ 
draft = false
date = "2025-01-31T00:00:00+03:00"
title = "Migrating from AWS Secrets to OIDC with Terraform for GitHub Actions"

slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

Previously, I used AWS Secrets for authentication in my GitHub Actions workflows. While this method was functional, it required manually managing and rotating credentials. To improve security and automation, I migrated to OpenID Connect (OIDC) authentication with AWS. GitHub provides an official guide on [Configuring OpenID Connect in AWS for GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services#adding-the-identity-provider-to-aws), which I followed while implementing this setup.

In this post, I’ll explain how I implemented OIDC using Terraform, making the setup fully Infrastructure as Code (IaC).

Create an OIDC Identity Provider in AWS

Instead of setting up the OIDC provider manually, I used Terraform to define it and import the existing AWS resources.

First, I created a github-oidc.tf file. Here's the full version:

{{< highlight tf >}}

resource "aws_iam_openid_connect_provider" "github_oidc" {
  url            = "https://token.actions.githubusercontent.com"
  client_id_list = ["sts.amazonaws.com"]
}

import {
  to = aws_iam_openid_connect_provider.github_oidc
  id = "arn:aws:iam::xxxxxxxxxxx:oidc-provider/token.actions.githubusercontent.com"
}

resource "aws_iam_role" "github_oidc_role" {
  name = "xxxxxxx"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.github_oidc.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "token.actions.githubusercontent.com:aud" : "sts.amazonaws.com"
          }
          StringLike = {
            "token.actions.githubusercontent.com:sub" : "repo:xxxxxx/*"
          }
        }
      }
    ]
  })
}

import {
  to = aws_iam_role.github_oidc_role
  id = "xxxxxxxxx"
}

resource "aws_iam_role_policy_attachment" "github_oidc" {
  role       = aws_iam_role.github_oidc_role.name
  policy_arn = aws_iam_policy.this["ecr_pusher"].arn
}

import {
  to = aws_iam_role_policy_attachment.github_oidc
  id = "xxxxxxxxx/arn:aws:iam::xxxxxxxxxxx:policy/ecr_pusher"
}

{{</ highlight >}}

The following part allows GitHub Actions to authenticate with AWS using OIDC tokens instead of secrets:

{{< highlight tf >}}

resource "aws_iam_openid_connect_provider" "github_oidc" {
  url            = "https://token.actions.githubusercontent.com"
  client_id_list = ["sts.amazonaws.com"]
}

import {
  to = aws_iam_openid_connect_provider.github_oidc
  id = "arn:aws:iam::xxxxxxxxxxx:oidc-provider/token.actions.githubusercontent.com"
}

{{</ highlight >}}

Create an IAM Role for GitHub Actions

Next, I created an IAM role that GitHub Actions can assume. This role includes a trust policy to:

* Use the OIDC provider configured in the previous step.
* Allow only GitHub Actions workflows from neusysadmin/* to assume the role.
* Ensure that only workflows with a valid OIDC token can authenticate.

{{</ highlight tf >}}

resource "aws_iam_role" "github_oidc_role" {
  name = "xxxxxxxxxxxx"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.github_oidc.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "token.actions.githubusercontent.com:aud" : "sts.amazonaws.com"
          }
          StringLike = {
            "token.actions.githubusercontent.com:sub" : "repo:neusysadmin/*"
          }
        }
      }
    ]
  })
}

import {
  to = aws_iam_role.github_oidc_role
  id = "xxxxxxxxx"
}

{{</ highlight >}}

Attach the Existing ecr_pusher Policy

Since I already had an IAM policy (ecr_pusher) for pushing images to Amazon ECR, I attached it to the new IAM role.

{{< highlight tf >}}

resource "aws_iam_role_policy_attachment" "github_oidc" {
  role       = aws_iam_role.github_oidc_role.name
  policy_arn = aws_iam_policy.this["ecr_pusher"].arn
}

import {
  to = aws_iam_role_policy_attachment.github_oidc
  id = "xxxxxxxx/arn:aws:iam::xxxxxxxxx:policy/ecr_pusher"
}

{{</ highlight >}}

Update the GitHub Actions Workflow

To use OIDC authentication, I updated my GitHub Actions workflow by adding the necessary permissions and configuring AWS credentials dynamically.

Below is the full version of my docker-build-scan-push.yaml file:


{{< highlight yaml >}}

on:
  workflow_call:
    inputs:
      repository:
        required: true
        type: string
        description: 'Docker image repository'
      dockerfile:
        required: true
        type: string
        description: 'Dockerfile to build'
      imageTag:
        required: true
        type: string
        description: 'Docker image tag'
      dockerContext: 
        required: false
        type: string
        description: 'Docker context'
        default: '.'
permissions:
  id-token: write 
  contents: read
jobs:
  build-and-scan:
    runs-on: ubuntu-latest
    steps:
      # Step 1: Check out the repository
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::xxxxxxxxx:role/xxxxxxxx
          role-session-name: xxxxxxxxxx
          aws-region: eu-central-1

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3


      # Step 2: Log in to Amazon ECR
      - name: Log in to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
        with:
          registries: "xxxxxxxxxxxx"

      - name: Build, tag docker image 
        env:
          REGISTRY: xxxxxxxxxxx.dkr.ecr.eu-central-1.amazonaws.com
          REPOSITORY: ${{inputs.repository}}
          DOCKERFILE: ${{inputs.dockerfile}}
          IMAGE_TAG: ${{inputs.imageTag}} 
          DOCKER_CONTEXT: ${{inputs.dockerContext}}
        id: build_image
        run: |
          docker buildx create --use
          docker buildx build --platform linux/arm64 -f ./$DOCKERFILE -t $REGISTRY/$REPOSITORY:$IMAGE_TAG --load $DOCKER_CONTEXT
          echo "IMAGE=$REGISTRY/$REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@0.28.0
        with:
          image-ref: '${{ steps.build_image.outputs.IMAGE }}'
          format: 'table'
          exit-code: '1'
          ignore-unfixed: true
          vuln-type: 'os,library'
          severity: 'CRITICAL,HIGH,LOW'

      - name: Push docker image to Amazon ECR (multi-platform)
        id: push_image
        env:
          IMAGE: ${{ steps.build_image.outputs.IMAGE }}
        run: |   
          docker push $IMAGE

{{</ highlight >}}

First, I added the necessary permissions:

{{< highlight yaml>}}

permissions:
  id-token: write 
  contents: read

{{</ highlight >}}

Then, I edited the jobs Step 1:

{{</ highlight yaml>}}

- name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::xxxxxxxxxx:role/xxxxxxxxxx
          role-session-name: xxxxxxxx

{{</ highlight >}}

Migrating to OIDC made authentication in my GitHub Actions workflows easier and more secure, no more static AWS credentials to manage. It’s a simple, automated solution that reduces risk of leaks. If you’re still using long-lived secrets, now’s a great time to switch. Let me know if you have any questions!




