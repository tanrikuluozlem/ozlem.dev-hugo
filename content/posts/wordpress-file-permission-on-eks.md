+++ 
draft = false
date = "2024-12-27T16:57:50+03:00"
title = "How I Fixed WordPress File Permission Issues on EKS"

slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

I recently faced an issue with file permissions on my WordPress setup running on EKS. The problem was that I couldn’t switch to the root user in my debug container due to restrictions in the image.

To work around this, I launched a small EC2 t4g.micro instance with ARM architecture. I used an instance profile to securely access the S3 bucket, which allowed me to retrieve the necessary files without storing credentials on the EC2 instance.

Accessing the S3 Bucket

In addition to accessing the WordPress files on EFS, I also needed to retrieve files from an S3 bucket and transfer them to EFS. By using the [EC2 instance profile](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html) feature, I was able to grant the EC2 instance access to the S3 bucket without storing any credentials on the instance. This made the process secure and efficient.

I created an EC2 instance from the AWS Console with the right IAM roles and network access to connect to the EFS storage. Then I accessed the instance directly from the AWS Console.

Mounting EFS on EC2

Since the WordPress files were stored on EFS as persistent volume claims (PVCs), I needed to mount the EFS filesystem to the EC2 instance to access these files.

1. Install NFS Utilities: First, I installed the NFS client utilities on the EC2 instance:

{{< highlight bash >}}

sudo yum install -y nfs-utils

{{</ highlight >}}

2. Create a Mount Directory: I created a directory on the EC2 instance to mount the EFS volume:

{{< highlight bash >}}

sudo mkdir /efs

{{</ highlight >}}

3. Mount the EFS Filesystem: I mounted the EFS filesystem to the directory using the following command (replacing <efs-id> with my EFS ID):

{{< highlight bash >}}

sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport \
<efs-id>.efs.eu-central-1.amazonaws.com:/ /efs

{{</ highlight >}}

Once I had access to the WordPress files, I needed to set the correct file permissions.

Navigate to the WordPress Files: I went to the directory where the WordPress files were stored:

{{< highlight bash >}}

cd /efs/<wordpress-directory>

{{</ highlight >}}

Then, I updated the file permissions, which was not possible to do so in the container:

{{< highlight bash >}}

sudo chmod -R 755 wp-content
sudo chown -R www-data:www-data wp-content

{{</ highlight >}}

Just make sure you unmount the EFS file system before you are done.

In a nutshell, by using EC2, S3, and EFS, I was able to manage my WordPress files on EKS and set the correct file permissions without storing credentials on the EC2 instance.

If you have any suggestions to handle the situation in a better way, please leave a comment.
