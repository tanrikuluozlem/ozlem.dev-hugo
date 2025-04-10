+++ 
draft = false
date = "2025-04-10T00:00:00+03:00"
title = "Achieve Cost-Efficient Scaling: Leveraging VPA and Prometheus for Smarter Resource Allocation in Kubernetes"

slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

As Kubernetes continues to serve as the backbone for modern cloud-native applications, ensuring efficient resource allocation has become a critical aspect of successful application management. Whether managing a large-scale enterprise application or a simple content management system like WordPress, optimizing resource usage ensures both performance and cost-effectiveness.

One powerful tool in Kubernetes for optimizing resource requests is the Vertical Pod Autoscaler (VPA). VPA dynamically adjusts the CPU and memory requests of pods based on their actual resource usage, enabling more efficient resource utilization. 

In this post, we'll explore how VPA can be used to optimize resource allocation for a WordPress application running on Kubernetes, highlighting the benefits of Prometheus integration for better data-driven recommendations and improved performance.

Understanding the Need for Vertical Pod Autoscaler (VPA)

While Kubernetes' Horizontal Pod Autoscaler (HPA) adjusts the number of pods based on CPU or memory usage, it does not deal with adjusting the resource requests themselves. This can be particularly problematic in environments where workloads require significant fluctuations in resource allocation but don't need to scale horizontally. VPA, on the other hand, dynamically adjusts the resource requests for CPU and memory based on actual usage.

Consider the case of a WordPress application running in a Kubernetes cluster. WordPress, depending on traffic and workload, might need more resources at certain times but less during quieter periods. Without VPA, you'd have to manually adjust the resource requests, which is time-consuming and error-prone. With VPA, this task is automated, ensuring that your application's performance is always optimized while keeping costs under control.

Here are the key reasons why VPA is essential for effective Kubernetes resource management:

* Cost Efficiency: By adjusting resource requests to reflect actual needs, VPA prevents the over-provisioning of resources, reducing cloud costs associated with unused or underutilized CPU and memory.

* Improved Application Performance: With dynamic resource adjustments, VPA ensures that your applications always have enough resources to perform well under varying loads, leading to better user experiences and reduced chances of resource bottlenecks.

* Operational Automation: VPA eliminates the need for manual intervention in resource management. This reduces operational complexity, improves DevOps workflow efficiency, and allows teams to focus on more critical tasks, such as optimizing application architecture.

1. Deploying the VPA Controller Using Helm

Before utilizing VPA in a Kubernetes cluster, the first step is to install the VPA controller. The Helm package manager via ArgoCD is an ideal tool for managing Kubernetes applications, and it simplifies the process of deploying and maintaining the VPA controller.

Here's how I installed the VPA controller using Helm:

1- Deploy the VPA controller using the Helm chart in your desired namespace:

{{< highlight yaml >}}
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: vpa-controller-hr
  namespace: argocd
  labels:
    applicationType: helm
  annotations:
    argocd.argoproj.io/sync-options: Prune=true
    argocd.argoproj.io/sync-wave: "2"
  finalizers:    
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    chart: vpa
    repoURL: https://charts.fairwinds.com/stable
    targetRevision: 4.7.1
    helm:
      releaseName: vpa-controller
      values: |
        recommender:
          enabled: true
            extraArgs:
              prometheus-address: |
                http://prometheus.monitoring.svc.cluster.local:9090
              storage: prometheus
        updater:
          enabled: false
        admissionController:
          enabled: false
  destination:
    server: "https://nonexistingcluster:6443"
    namespace: vpa-controller
  syncPolicy:
    automated: 
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
{{</ highlight >}}

In this configuration:

* Prometheus Integration: The VPA controller is configured to gather resource metrics from Prometheus, ensuring that VPA has access to accurate, historical data for generating recommendations.

* Recommender Enabled: The recommender functionality is enabled, which will generate resource recommendations based on historical data.

* Updater and Admission Controller Disabled: For this setup, we're focusing on getting recommendations, so automatic updates and admission control are not enabled.

2. Configuring VPA for WordPress Deployment

After setting up the VPA controller, I created a VPA object for my WordPress application. The objective here was to gather recommendations for CPU and memory requests.

Here's the VPA configuration for the WordPress deployment:

{{< highlight yaml >}}
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: vpa
  namespace: neu-testg
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app-cluster-dog-app-wp-test-wordpress
  updatePolicy:
    updateMode: "Off"
# Uncomment below to enable "minAllowed" and "maxAllowed" setings for the resource requests
#  resourcePolicy:
#    containerPolicies:
#      - containerName: "phpfpm"
#        minAllowed:
#          cpu: "100m"
#          memory: "256Mi"
#        maxAllowed:
#          cpu: "1000m"
#          memory: "1024Mi"
{{</ highlight >}}

In this configuration:

* Target Reference: The VPA is targeting the WordPress deployment (app-cluster-dog-app-wp-test-wordpress) in the neu-testg namespace.

* Update Mode: The updateMode is set to Off so that VPA will only provide recommendations without making changes automatically.

3. Analyzing VPA Recommendations

Once the VPA object is applied and the system has gathered enough data, you can view the recommendations generated by the VPA for your WordPress pods. This is done using the kubectl describe vpa command, which will show detailed information about the recommended resource requests for each container within the pod.

{{< highlight bash >}}
$ kubectl describe vpa -n neu-testg vpa

Name:         vpa
Namespace:    neu-testg
Labels:       <none>
Annotations:  <none>
API Version:  autoscaling.k8s.io/v1
Kind:         VerticalPodAutoscaler
Metadata:
  Creation Timestamp:  2025-02-05T18:38:38Z
  Generation:          1
  Resource Version:    53961323
  UID:                 e171486f-c9b2-4592-98b5-aa482131058c
Spec:
  Target Ref:
    API Version:  apps/v1
    Kind:         Deployment
    Name:         neu-testg-wp
  Update Policy:
    Update Mode:  Off
Status:
  Conditions:
    Last Transition Time:  2025-02-05T18:39:25Z
    Status:                True
    Type:                  RecommendationProvided
  Recommendation:
    Container Recommendations:
      Container Name:  phpfpm
      Lower Bound:
        Cpu:     110m
        Memory:  52428800
      Target:
        Cpu:     251m
        Memory:  220358100
      Uncapped Target:
        Cpu:     251m
        Memory:  220358100
      Upper Bound:
        Cpu:           305m
        Memory:        300904400
Events:          <none>Each of the values provided in the recommendation has a specific meaning:
{{</ highlight >}}

* Lower Bound: This represents the minimum resources required for the pod to function properly. If resource consumption drops below this level, Kubernetes may scale down or even evict the pod to free up resources.

* Target: This is the ideal resource request recommended by VPA. In updating mode, VPA will adjust the pod's resource requests to these values. The target represents the optimal allocation for your application based on current usage patterns, but it will be between minAllowed or maxAllowed limits if they were specified.

* Uncapped Target: It's similar to the Target value, but without considering any configured minAllowed or maxAllowed limits.

* Upper Bound: This defines the maximum resource limit for the pod. If the pod consumes more resources than this, Kubernetes may scale up the pod to meet the demand. This upper limit is important for preventing runaway resource usage.

4. Enabling Automatic Resource Adjustment

Once you're confident in the recommendations, you can enable updating mode to allow VPA to automatically adjust the resource requests based on the Target values:

{{< highlight yaml >}}
updatePolicy:
  updateMode: "Auto"
{{</ highlight >}}

Enabling Auto update mode allows VPA to automatically apply the target resource requests to the WordPress pods. Kubernetes will adjust the CPU and memory allocations to the recommended levels, ensuring optimal performance and resource usage.

However, it's important to note that Kubernetes Pods are immutable, meaning VPA cannot directly modify a pod's resource requests while it's running. To apply updated resource recommendations, VPA relies on the Admission Controller, which can only set resource requests at the time of pod creation or recreation. As a result, when using VPA in "Auto" update mode, it works by evicting existing pods, allowing new ones to be created with the updated CPU and memory requests.

Due to this behavior, avoid using Auto mode if your workload cannot tolerate pod evictions.

Why VPA is Essential for Kubernetes Workloads
1. Data-Driven Decision Making: By leveraging Prometheus for real-time metrics and historical data, VPA makes recommendations based on actual pod resource usage, ensuring that resource allocation is always aligned with current application demands.

2. Cost Optimization: Automatically adjusting resource requests helps avoid over-provisioning and unnecessary cloud infrastructure costs, making it ideal for dynamic workloads like WordPress.

3. Enhanced Performance: By continuously adapting resource allocations, VPA ensures that your pods receive the necessary resources to handle traffic spikes or periods of high usage, leading to better application performance and reduced downtime.

4. Operational Efficiency: VPA's automation reduces the manual work involved in managing resources. This allows DevOps teams to focus on higher-level tasks such as improving infrastructure or deploying new features.

5. Scalability and Flexibility: Kubernetes environments are often highly dynamic, and workloads may fluctuate. VPA's ability to scale resource allocations up and down as needed provides flexibility and ensures that applications always perform at their best, without the need for constant human intervention.

Conclusion

Leveraging Vertical Pod Autoscaler (VPA), especially with Prometheus integration, is a smart approach for optimizing resource management in Kubernetes clusters. By automating the adjustment of CPU and memory requests, VPA ensures that applications like WordPress always operate with the right amount of resources. This not only improves performance but also reduces costs by preventing over-provisioning.




