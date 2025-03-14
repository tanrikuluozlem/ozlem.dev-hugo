+++ 
draft = false
date = "2025-03-14T00:00:00+03:00"
title = "Optimizing EFS Throughput for Cost and Performance"

slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

When working on one of my application that relies on Amazon Elastic File System (EFS), I hit an unexpected problem — latency spikes during high-traffic periods. Initially, I had everything running smoothly, but as demand grew, so did the delays. That’s when I had to dive deep into EFS throughput modes to figure out a balance between performance and cost. Here’s how it played out.😎

👉🏻 EFS offers three throughput modes:

* Bursting Throughput Mode (Default)
* Elastic Throughput Mode
* Provisioned Throughput Mode

📍 Bursting Throughput Mode: Seemed Like a Good Idea… at First

Like most people, I started with Bursting Throughput Mode because it’s the default and seemed like the most cost-efficient option. This mode works well under normal conditions, using a burst credit system to temporarily scale up throughput when needed.

But then, as my application saw a surge in traffic, the burst credits ran out — fast. Throughput tanked, and latency issues started creeping in. Soon enough, I was getting alerts about slow performance. Not ideal.

![Deneme](/images/before.png)

📍Elastic Throughput Mode: A Quick Fix That Burned a Hole in the Budget

To get things back on track, I switched to Elastic Throughput Mode. This mode automatically adjusts throughput based on demand, which sounded perfect.

Pros: It instantly solved my latency issues, and my application was back to running smoothly.

Cons: The costs shot up like crazy because this mode charges based on the total amount of data transferred, regardless of speed.

Elastic mode was great in terms of performance, but the cost was unsustainable. I needed something smarter.

📍 Provisioned Throughput Mode: The Optimal Point

Since I couldn’t afford Elastic Mode long-term, I moved to Provisioned Throughput Mode. Here, I could specify the exact amount of throughput I needed, avoiding surprise costs while maintaining consistent performance.

![Deneme](/images/after.png)

Solution: I analyzed my traffic patterns and set a provisioned throughput level that matched my needs.

Outcome: My app remained fast, and costs were predictable — no more unpleasant billing surprises.


{{< highlight yaml >}}

 this.efs = new efs.FileSystem(this, 'efs', {
     vpc: props.vpc,
     lifecyclePolicy: efs.LifecyclePolicy.AFTER_30_DAYS,
     outOfInfrequentAccessPolicy: efs.OutOfInfrequentAccessPolicy.AFTER_1_ACCESS,
     performanceMode: efs.PerformanceMode.GENERAL_PURPOSE,
     throughputMode: efs.ThroughputMode.PROVISIONED,
     provisionedThroughputPerSecond: cdk.Size.mebibytes(58),
     removalPolicy: cdk.RemovalPolicy.RETAIN,
     enableAutomaticBackups: true
   })
 {{</ highlight >}} 

💁🏼‍♀️Another Cost-Saving Trick: Single-AZ EFS

While optimizing my setup, I realized that I could cut storage costs by switching to Single-AZ EFS instead of Multi-AZ. Since my application mainly runs in a single availability zone, I didn’t need the extra redundancy, and this change helped bring costs down even further.

Beware: AWS Puts Limits on Switching Modes

One thing to keep in mind: AWS doesn’t let you freely switch throughput modes or modify provisioned throughput without restrictions.

After enabling Provisioned Throughput Mode or adjusting the provisioned amount, you have to wait 24 hours before:

  * Switching back to Elastic or Bursting Throughput Mode
  * Decreasing the Provisioned Throughput amount

🚀 For anyone working with EFS and looking to optimize costs while maintaining solid performance, my recommendation is:

  * Use Bursting Mode for light or unpredictable traffic, as it’s the most cost-effective option.
  * Consider Elastic Mode if you need scalability, but be prepared for higher costs during traffic spikes.
  * Switch to Provisioned Throughput Mode if you need consistent performance at predictable costs, especially for high-traffic applications.

  By optimizing throughput and being mindful of costs, I was able to solve both latency and budget issues effectively.

  For more details, you can check out the [AWS documentation on EFS throughput modes.](https://docs.aws.amazon.com/efs/latest/ug/performance.html#throughput-modes)
