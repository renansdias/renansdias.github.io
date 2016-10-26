---
layout: post
title: Modular Infrastructure Part 1
description: Learn how to turn your Cloud infrastructure into reusable modules
image: /assets/media/Lamp.jpg
categories: [devops]
tags: [devops]
---

# Introduction

# Technology stack

For this article, the following tools and cloud provider will be used:

// Create links for each bullet point

* **Terraform**
* **Ansible**
* **Packer**
* **Amazon Web Services**

Ansible and Packer will be used to generate custom AMIs. **Ansible** will be used to configure this AMI with all the necessary software, and **Packer** will generate the AMI. Terraform, in turn, will be used to create reusable infrastructure modules. Let's see how!

## Scenario 1

In this scenario, we will build a simple infrastructure which consists of:

* An Amazon VPC
* 2 public subnets
* 2 private subnets
* 2 NAT instances (one NAT instance in each public subnet)
* Bastion hosts in the public subnet
* A Redis cluster (ElastiCache) in the private subnet

If this configuration is still not clear for you, take a look at the image below:

<SCENARIO 1 IMAGE>

To start off with, let's create a git repository