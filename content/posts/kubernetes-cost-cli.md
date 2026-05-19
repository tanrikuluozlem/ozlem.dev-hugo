+++ 
draft = false
date = "2026-05-05T12:00:00+03:00"
title = "33% of Our Kubernetes Bill Was Paying for Nothing"

slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

I manage a few EKS clusters. One of them; five nodes, around 77 pods, kept showing up on the AWS bill in a way that didn't make sense. Not huge numbers, but enough to make me wonder where the money was actually going.

The answer should be simple: which namespace costs how much, and are there pods running that shouldn't be.

`kubectl top nodes` gives CPU and memory percentages but no dollar amounts. The AWS Cost Explorer shows the EC2 total but doesn't know what's running inside those nodes. It sees instances, not pods. And the tools that do connect Kubernetes resources to cost data all wanted me to deploy something into the cluster; Helm charts, agents, dashboards. I didn't want to maintain another thing in the cluster just to check a number.

So I built [Burn](https://github.com/tanrikuluozlem/burn). A single binary that reads your kubeconfig, talks to the Kubernetes API and optionally Prometheus, and gives you a cost breakdown per namespace.

## This is what it looks like

```
$ burn analyze --prometheus http://prometheus:9090 --period 7d

Kubernetes Cost Report (7d avg)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Monthly: $350 | Idle: $117 (33%)
Nodes: 5 | Pods: 77

NAMESPACES
──────────
NAMESPACE            PODS  CPU REQ→USED  MEM REQ→USED   COST/MO
argocd               4     2.0 → 30m     2.0Gi → 393Mi  $56
amazon-cloudwatch    11    1.6 → 82m     829Mi → 1.3Gi  $44
kube-system          21    1.4 → 52m     1.6Gi → 757Mi  $41
...and 7 more namespaces
Idle (unallocated)                                     $117
─────────────────────────────────────────────────────────
Total                                                  $350

LOAD BALANCERS
──────────────
NAME                        NAMESPACE    COST/MO
app-ingress                 app-prod     $16

COST BREAKDOWN
━━━━━━━━━━━━━━
Compute:         $350
Storage:         $0
Load Balancers:  $16
Network:         $0
Total:           $366
```

That idle line, $117/month. A third of the entire cluster cost was capacity allocated to nodes but not used by any pod. Not over-provisioned pods, not inefficient workloads, just empty space on nodes.

```bash
brew install tanrikuluozlem/burn/burn
burn analyze --prometheus http://prometheus:9090
```

Read-only. Queries your Kubernetes API and Prometheus, doesn't deploy anything, doesn't modify anything. Safe to run on production.

## What showed up

Once I had per-namespace cost visibility, a few things became obvious.

argocd-dex-server was requesting 500m CPU but using 0.12m. That's 0.02% utilization, $14/month for a pod that had been running like that for months.

Two rds-debug pods were sitting in dev and QA namespaces, running 24/7. I asked around, nobody remembered creating them. $11/month combined, doing absolutely nothing.

This was on an actively maintained production cluster. There was just no easy way to see cost at the namespace level before.

I deleted the debug pods and rightsized argocd-dex-server that same day. Now I run `burn analyze` every week before sprint planning, it takes 30 seconds and keeps resource requests honest. The idle capacity question led to a node consolidation discussion that's still ongoing, but at least now it's a discussion backed by real numbers instead of guesswork.

## Ask questions from the terminal

The other thing I wanted was to just ask a question and get an answer. Not navigate dashboards, not write PromQL, just type what I'm thinking:

```bash
burn ask --prometheus http://prometheus:9090 "where is the money going?"
```

It streams the response in real time, breaks down cost by namespace, highlights the biggest waste, and gives you the kubectl commands to fix it. The AI sees the full cluster data but every dollar amount comes from pre-calculated metrics, not from the model.

You can also focus on a specific namespace:

```bash
burn analyze --prometheus http://prometheus:9090 --namespace argocd --ai
```

This gives pod-level recommendations using p95 usage data. For example: "p95 CPU is 0.22m → recommend 1m (1.5x p95 headroom)". No guessing, based on actual peak usage over the analysis period.

## Slack integration

I set up Slack integration so anyone on the team can check costs directly from Slack, just a slash command.

```
/burn                    → cluster cost summary
/burn ns production      → pod-level breakdown
```

The natural language query is where it gets useful:

```
/burn ask "what is the single biggest waste?"
```

It analyzes the full cluster data and responds with the specific problem, the dollar amount, and the kubectl commands to fix it. Same streaming AI that works from the terminal, now available to the whole team from Slack.

Every dollar amount is pre-calculated from real cluster metrics before the AI sees anything. The AI explains the results, it doesn't generate the numbers.

## How it actually calculates

Burn pulls node specs, pod resource requests, and actual usage data from the Kubernetes API and Prometheus. It then fetches the real hourly price of each node, for example if you're running an m5.xlarge in us-east-1, it calls the AWS Pricing API to get the current on-demand rate for that exact instance type and region. Same for Azure through the Retail Prices API.

If the cloud API is unreachable, it falls back to an embedded price database that auto-updates weekly through CI. If that's also missing, static defaults kick in. Three layers, always a price.

Once it has the node price, it splits that cost into per-core CPU and per-GiB RAM rates based on the node's allocatable capacity. Each pod's cost is then calculated from the resources it actually consumes on that node, the larger of what it requested or what it's using.

For GPU nodes, Burn reads `nvidia.com/gpu` capacity and the GPU model from node labels, then does a three-way cost split across CPU, RAM, and GPU. No manual configuration, it detects GPU type and count automatically.

With `--period 7d`, every Prometheus query uses P95 calculations instead of point-in-time snapshots. If a pod averages 10m CPU but spikes to 200m once a day, the recommendation accounts for the spike.

All of this; node prices, pod costs, idle capacity, namespace totals, savings opportunities, is calculated before the AI sees anything. The AI receives the fully computed cost report with pre-calculated savings for each strategy: spot conversion, node consolidation, pod rightsizing. It then turns those numbers into specific recommendations with real node names, risk warnings, and the exact kubectl commands to act on them.

Burn also tracks storage and load balancer costs per namespace. Most tools only detect `Service type=LoadBalancer`. Burn also reads Ingress resources, so if your cluster uses the AWS Load Balancer Controller with Ingress-based ALBs, those costs show up too. It deduplicates by hostname so the same ALB isn't counted twice.

For on-prem clusters where there's no cloud API:

```bash
burn analyze --cpu-price 0.05 --ram-price 0.008 --gpu-price 3.00 --storage-price 0.10
```

Set your own rates per CPU core, per GiB RAM, per GPU, and per GiB storage.

## Try it

```bash
brew install tanrikuluozlem/burn/burn
burn analyze --prometheus http://prometheus:9090
```

Works without Prometheus too, uses Kubernetes API resource requests as a baseline. Also available as a Docker image, Helm chart, or Go binary. Runs on EKS, AKS, GKE, and on-prem.

Here's a quick demo showing the cost report, AI analysis, and Slack integration: [watch on YouTube](https://youtu.be/uGVvaKXeTf4).

If you run into anything or have ideas, [open an issue](https://github.com/tanrikuluozlem/burn/issues)! I read all of them.

Open source, Apache 2.0: [github.com/tanrikuluozlem/burn](https://github.com/tanrikuluozlem/burn)
