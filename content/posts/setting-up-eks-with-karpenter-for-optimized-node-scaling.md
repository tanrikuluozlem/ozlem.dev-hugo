+++ 
draft = false
date = "2025-01-24T00:00:00+03:00"
title = "Setting Up EKS with Karpenter for Optimized Node Scaling"

slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

I'm always on the lookout for tools that improve efficiency, reduce costs, and simplify operations. Kubernetes Cluster Autoscaler (CAS) has been a trusted ally for scaling nodes in Kubernetes clusters, but it comes with limitations. Karpenter, an open-source, high-performance cluster autoscaler designed to overcome these challenges with smarter and faster scaling capabilities.

In this post, I'll walk you through how I set up EKS with Karpenter for optimized node scaling, leveraging Terraform and Helm.

Karpenter stands out because it dynamically provisions and scales nodes based on actual workload requirements. Unlike the traditional CAS, which relies on predefined instance groups, Karpenter picks the most cost-effective instance that satisfies the resource requirements of your pending pods. This means:

* No need to define fixed instance size for node groups.
* Faster provisioning and scaling.
* Cost savings by using instance types optimized for the workload.

Provisioning the EKS Cluster with Terraform

To provision the EKS cluster, I used the [terraform-aws-modules/eks/aws]https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest module. This module follows AWS best practices by default, making it a solid choice. My setup includes essential EKS add-ons like:

* Amazon EFS CSI Driver
* CoreDNS
* kube-proxy
* Amazon VPC CNI(with prefix delegation enabled)
* Amazon EKS Pod Identity Agent

Here's a snippet from my eks-cluster Terraform configuration:

{{< highlight tf >}}
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "dog-cluster"
  cluster_version = "1.27"

  enable_cluster_creator_admin_permissions = true
  cluster_endpoint_public_access           = true

  cluster_addons = {
    coredns                = {}
    eks-pod-identity-agent = {}
    kube-proxy             = {}
    vpc-cni = {
      "configuration_values" : jsonencode({
        "env" : {
          "ENABLE_PREFIX_DELEGATION" : "true"
        }
      })
    }
    aws-efs-csi-driver = {
      addon_version            = "v2.0.9-eksbuild.1" # TODO: remove this after https://github.com/kubernetes-sigs/aws-efs-csi-driver/issues/1547 this is fixed.
      service_account_role_arn = "arn:aws:iam::xxxxxxxxxxxx:role/example-efs-csi-driver-role"
      preserve                 = true
    }
  }

  vpc_id                    = "vpc-xxxxxxxx"
  subnet_ids                = ["subnet-xxxxxxxx", "subnet-xxxxxxxx"]
  control_plane_subnet_ids  = ["subnet-xxxxxxxx", "subnet-xxxxxxxx"]
  cluster_service_ipv4_cidr = "192.168.x.x/16"

  create_cluster_security_group                = true
  create_node_security_group                   = true
  node_security_group_enable_recommended_rules = true
  node_security_group_tags = {
    "karpenter.sh/discovery" = "example-cluster"
  }

  node_security_group_additional_rules = [
    {
      description = "Allow example traffic"
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  ]

  eks_managed_node_groups = {
    karpenter = {
      ami_type = "AL2_ARM_64"

      instance_types = ["t4g.small"]

      min_size     = 1
      max_size     = 1
      desired_size = 1

      taints = {
        addons = {
          key    = "CriticalAddonsOnly"
          value  = "true"
          effect = "NO_SCHEDULE"
        },
      }
    }
  }
}

module "eks_karpenter" {
  source  = "terraform-aws-modules/eks/aws//modules/karpenter"
  version = "~> 20.0"

  cluster_name = module.eks.cluster_name

  enable_v1_permissions = true

  enable_pod_identity             = true
  create_pod_identity_association = true

  # Attach additional IAM policies to the Karpenter node IAM role
  node_iam_role_additional_policies = {
    AmazonSSMManagedInstanceCore = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
  }
}

{{</ highlight tf>}}

Btw, it is important to set "ENABLE_PREFIX_DELEGATION" : "true" in vpc_cni add-on's env, otherwise max pods that your node can run will be limited to the number of ENI interfaces that can be attached to the instance.

This configuration creates a single managed node group, which is dedicated to running Karpenter.

Installing Karpenter with Terraform and Helm

I installed Karpenter using the Helm provider for Terraform. This simplified the process by automating the deployment and configuration of Karpenter.

Installing Karpenter with Terraform and Helm

I installed Karpenter using the Helm provider for Terraform. This simplified the process by automating the deployment and configuration of Karpenter.

Here's how I set it up:

{{< highlight bash >}}

resource "helm_release" "karpenter" {
 namespace           = "kube-system"
 name                = "karpenter"
 repository          = "oci://public.ecr.aws/karpenter"
 repository_username = data.aws_ecrpublic_authorization_token.token.user_name
 repository_password = data.aws_ecrpublic_authorization_token.token.password
 chart               = "karpenter"
 version             = "1.0.1"
 wait                = false

 values = [
   <<-EOT
   serviceAccount:
     name: ${module.eks_karpenter.service_account}
   settings:
     clusterName: ${module.eks.cluster_name}
     clusterEndpoint: ${module.eks.cluster_endpoint}
     interruptionQueue: ${module.eks_karpenter.queue_name}
   controller:
     resources:
       limits:
         cpu: 1000m
         memory: 1024Mi
       requests:
         cpu: 200m
         memory: 256Mi
   logLevel: debug
   replicas: 1
   settings:
     clusterName: ${module.eks.cluster_name}
     interruptionQueue: ${module.eks_karpenter.queue_name}
   EOT
 ]
}

{{</ highlight >}}

Configuring Node Pools with Karpenter

The next step was to configure Node Pools and Node Classes to define the instances that Karpenter could provision. Using the Kubernetes provider for Terraform, I created manifests with the following key settings:

* Instance families: Karpenter supports instance families like t, r, m, and c.
* CPU and Memory ranges: Nodes can have between 1 and 32 vCPUs and 1 GB to 64 GB of memory.

Karpenter intelligently prioritizes cheaper instances when available.

Here's a Node Pool manifest:

{{< highlight tf >}}

resource "kubectl_manifest" "karpenter_node_pool" {
  for_each = var.node_pools

  yaml_body = <<-YAML
    apiVersion: karpenter.sh/v1
    kind: NodePool
    metadata:
      name: ${each.value.name}
      labels:
        node-pool: ${each.value.name}
        cluster: ${var.cluster_name}
    spec:
      template:
        metadata:
          labels:
            node-pool: ${each.value.name}
            cluster: ${var.cluster_name}
        spec:
          nodeClassRef:
            group: karpenter.k8s.aws
            kind: EC2NodeClass
            name: ${each.value.node_class_name}
          expireAfter: 180h
          terminationGracePeriod: 6h
          requirements:
            - key: "karpenter.k8s.aws/instance-category"
              operator: In
              values: ["t", "r", "m", "c"]
              minValues: 4

            - key: "karpenter.k8s.aws/instance-cpu"
              operator: Gt
              values: ["1"]
            - key: "karpenter.k8s.aws/instance-cpu"
              operator: Lt
              values: ["32"]

            - key: "karpenter.k8s.aws/instance-memory"
              operator: Gt
              values: ["1024"]
            - key: "karpenter.k8s.aws/instance-memory"
              operator: Lt
              values: ["64000"]

            - key: "karpenter.k8s.aws/instance-hypervisor"
              operator: In
              values: ["nitro"]

            - key: "topology.kubernetes.io/zone"
              operator: In
              values: ${jsonencode(module.vpc.azs)}

            - key: "kubernetes.io/arch"
              operator: In
              values: ["arm64"]

            - key: "karpenter.sh/capacity-type"
              operator: In
              values: ["on-demand"]

      limits:
        cpu: ${each.value.limits.cpu}
        memory: ${each.value.limits.memory}Gi
      disruption:
        consolidationPolicy: WhenEmptyOrUnderutilized
        consolidateAfter: 1m
        budgets:
          - nodes: 10%
  YAML

  depends_on = [
    kubectl_manifest.karpenter_node_class
  ]
}

{{</ highlight >}}

This configuration ensures Karpenter selects the most cost-effective instance that meets the workload's needs, avoiding the fixed-size node group approach used by CAS.

Final Thoughts

Karpenter has made a huge difference in how I optimize node scaling in my EKS clusters.Its ability to intelligently choose instances based on real-time needs has not only improved performance but also reduced costs significantly. If you're looking to modernize your autoscaling approach, Karpenter is worth exploring. You can learn more about how to set it up and make the most of its features in the [Karpenter documentation]https://karpenter.sh.
