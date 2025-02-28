+++ 
draft = false
date = "2025-02-28T00:00:00+03:00"
title = "Automating EC2 Provisioning and Configuration with Terraform and Ansible"

slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

Infrastructure as Code (IaC) and Configuration Management play a crucial role in modern DevOps workflows. In this post, I’ll explain how I automated the provisioning and configuration of EC2 instances using Terraform and Ansible. This approach not only improves deployment efficiency but also guarantees consistency across environments.

🙋🏼‍♀ Push-Based Ansible Configuration with Terraform

In this case, I structured my Terraform root module to launch multiple EC2 instances and used Ansible to configure them. Here’s the workflow:

- Provisioning EC2 Instances with Terraform

I wrote a Terraform root module to deploy multiple EC2 instances. This module included essential resources such as security groups, key pairs, and instance configurations.

- Configuring Nginx with Ansible

Once the instances were provisioned, I used Ansible playbooks to configure Nginx and deploy a sample configuration that serves a simple “Untested code is broken code!” page.😎

* Inventory File (inventory.yaml): Defined the list of EC2 hosts.

{{< highlight yaml >}}

webservers:
  hosts:
    ec2-xx-xx-xx-xx.eu-central-1.compute.amazonaws.com:
      host: web-0
    ec2-yy-yy-yy-yy.eu-central-1.compute.amazonaws.com:
      host: web-1
    ec2-zz-zz-zz-zz.eu-central-1.compute.amazonaws.com:
      host: web-2
    ec2-aa-aa-aa-aa.eu-central-1.compute.amazonaws.com:
      host: web-3
    ec2-bb-bb-bb-bb.eu-central-1.compute.amazonaws.com:
      host: web-4
    ec2-cc-cc-cc-cc.eu-central-1.compute.amazonaws.com:

{{< /highlight >}}

* Ansible Playbook (nginx.yaml): Installed and configured Nginx.

{{< highlight yaml >}}

- name: Install nginx
  hosts: localhost
  remote_user: ec2-user
  become: yes

  tasks:
  - name: Install nginx on Al 2023
    ansible.builtin.package:
      name: "nginx"
      state: latest
  - name: Ensure nginx is running
    ansible.builtin.service:
      name: "nginx"
      #state: started
      enabled: yes
  - name: Copy nginx configuration file
    ansible.builtin.template:
      src: conf/nginx.conf.j2
      dest: /etc/nginx/conf.d/web.conf
    notify: Restart nginx
  handlers:
  - name: Restart nginx
    ansible.builtin.service:
      name: "nginx"
      state: restarted

{{< /highlight >}}

* Configuration Template (nginx.conf.j2): Used a Jinja2 template to define the Nginx configuration.

{{< highlight bash >}}

 server {
  listen 80;

  location / {
    default_type text/plain;
    return 200 "Untested code is broken code! {{ vars["host"] }}\n";
  }
}
 {{</ highlight >}} 

 To apply these configurations, I ran the Ansible playbooks manually (push-based logic):

 {{< highlight bash >}}

 ansible-playbook -i inventory.yaml nginx.yaml

 {{</ highlight >}} 

 🚀Modularizing Terraform with Child Modules

 To improve maintainability, I created a child module to manage EC2 instances and related resources such as security groups and key pairs. The existing EC2 instances were moved from the root module into this new child module using Terraform’s moved statements, ensuring a smooth transition without resource recreation.

 🙋🏼‍♀️ Pull-Based Ansible Configuration with Terraform User Data

In this case, I leveraged Terraform user data to enable a pull-based Ansible configuration. This approach allows instances to self-configure upon launch by fetching the necessary playbooks from a Git repository.

- Using user-data to Run Ansible in Pull Mode

Instead of pushing configurations manually, I used Terraform’s user_data to execute an Ansible pull command at instance startup. Since user-data requires sensitive credentials (e.g., GitHub PAT — check out my related [blog post](https://medium.com/@ozlemtanrikulu/accessing-cross-workflow-artifacts-between-repositories-93d167397afa), I stored it as a Terraform variable (sensitive = true) to prevent accidental exposure in logs.

The EC2 instance pulls the latest Ansible playbooks from a Git repository and applies them to itself:

🚀Here’s how the user-data section is structured within the Terraform configuration:

{{< highlight bash >}}

resource "aws_instance" "this" {
  for_each = { for _, v in range(var.instance_count) : "${var.name_prefix}-${v}" => null }

  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id
  key_name      = aws_key_pair.this.key_name
  vpc_security_group_ids = [
    aws_security_group.this.id
  ]
  user_data = templatefile("${path.module}/files/cloud-init.template", {
    public_key = var.public_key
    pat        = var.pat
    host       = each.key
  })

  tags = {
    Name = each.key
  }
}
 {{</ highlight >}} 

 The user_data script references a Cloud-Init template (cloud-init.template) that prepares the instance by configuring SSH access with the provided public key, updating and upgrading packages, installing Ansible, defining a local Ansible inventory file, and executing ansible-pull to fetch and apply configuration playbooks from a Git repository.

 {{< highlight bash >}}

#cloud-config
ssh_authorized_keys:
- ${public_key}
manage_etc_hosts: true
package_update: true
package_upgrade: true
packages:
- ansible
write_files:
  - encoding: raw
    path: /etc/ansible/hosts.yaml
    owner: root:root
    permissions: '0644'
    content: |
      ---
      webservers:
        hosts:
          127.0.0.1:
            ansible_connection=local
runcmd:
- export TOKEN=${pat}; ansible-pull -U https://tanrikuluozlem:$TOKEN@github.com/tanrikuluozlem/ansible-web --extra-vars host=${host} -d /projectFolder/ansible-web /projectFolder/ansible-web/ansible/nginx.yaml

 {{</ highlight >}} 

 🧑🏻‍🚀 By structuring Terraform modularly and leveraging Ansible for configuration management, I automated deployments without disrupting production. The move to Ansible pull mode further automated the process, ensuring ongoing consistency across the infrastructure.

This approach aligns with DevOps best practices — automated, repeatable, and scalable infrastructure management. Looking to build automated, scalable solutions? Let’s connect! 🙌🏻



