+++ 
draft = false
date = "2025-01-03T00:00:00+03:00"
title = "Setting Up Multi-Cluster Shared Services via VPC Peering in EKS"

slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

When working with multiple Amazon EKS clusters in different VPCs, sharing services across clusters can feel like a bit of maze. But, with VPC peering, it’s much easier than it sounds . I recently had to set this up, and I thought I’d share my approach, some potential pitfalls, and what to keep in mind. Hopefully, this helps make your setup smoother.


Understanding the Basics: VPC Peering and IP Ranges

Before diving into the setup, let’s get one thing clear: VPC peering allows two VPCs to communicate as if they were part of the same network. This is the backbone of our setup. However, when using EKS, you’ll need to keep an eye on two IP ranges per cluster:


* Pod Network CIDR: This is the IP range assigned to your Pods.
* Service ClusterIP Range: This is the IP range used for Kubernetes services.

If the CIDR ranges of one cluster overlap with the other, you’re going to have a bad time. VPC peering won’t work properly, and you’ll likely run into routing issues. So, take some time during the planning phase to ensure these ranges are unique across all clusters.

Planning the IP Ranges

Once your EKS cluster is up and running, the Pod Network CIDR and Service ClusterIP ranges are locked in. You can’t go back and change them later without rebuilding the cluster. That’s why getting it right during the planning phase is so important.Spend the time upfront to define non-overlapping ranges for each cluster!

For example:

* Cluster A: Pod Network CIDR: 10.0.0.0/16, Service ClusterIP Range: 172.20.0.0/16
* Cluster B: Pod Network CIDR: 10.1.0.0/16, Service ClusterIP Range: 172.21.0.0/16

With ranges like these, you’re in the clear for VPC peering.

Setting Up VPC Peering

Once the CIDRs are sorted, you can set up VPC peering between the VPCs hosting your clusters. You’ll need to:

1. Create a VPC peering connection between the two VPCs.
2. Update route tables in both VPCs to allow communication.
3. Modify security groups to allow traffic between the clusters.

AWS makes this relatively straightforward, but don’t forget to test connectivity using something like a simple ping between the nodes of the two clusters.

Exposing Services Across Clusters

Now, let’s talk about how to actually share services between clusters. The approach I took was exposing services in one cluster via NodePort and consuming them in the other cluster. Here’s what this looks like:

1. In Cluster A, expose a service as a NodePort. This makes the service accessible on a specific port on the nodes.
2. Use the private IP of the node and the NodePort to access the service from Cluster B.

For instance, if you’ve got a service in Cluster A exposed on port 30001, and the node IP is 10.0.0.5, you’d access it in Cluster B using http://10.0.0.5:30001.

Tips And Things to Watch Out For

1. DNS Doesn’t Work by Default: If you’re using Kubernetes service names to resolve between clusters, that won’t work out of the box. Consider using something like external DNS if needed.
2. Security Groups Matter: Don’t forget to open up the necessary ports in your security groups for inter-cluster communication.
3. Test: Verify the connection before rolling out anything to production. A simple curl command can save you hours of troubleshooting later.

Wrapping Up

Setting up shared services across EKS clusters via VPC peering isn’t terribly complex, but it does require careful planning. The key is to avoid overlapping IP ranges, properly configure VPC peering, and expose services in a way that makes them accessible to other clusters. With a bit of foresight and testing, you’ll have a robust setup that works seamlessly.


