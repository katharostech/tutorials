---
title: "Automating ARM Infrastructure With Terraform and Drone"
date: 2019-11-03T14:12:51Z
author: katharostech
#draft: true
url: automate-arm-infrastructure
keywords:
- terraform
- arm
tags:
- terraform
- arm
---

This guide will walk you through setting up Terraform and Drone to provision ARM infrastructure on the AWS cloud. By the end of the tutorial you will have defined ARM infrastructure as Terraform code and configured Drone to automatically deploy changes when pull requests are merged.

This guide assumes that you have a basic understanding of Git, GitHub and AWS. Specifically you will need to be able to create a GitHub repository and clone it and you will need to be able to create an AWS acount with programmatic access to the AWS API. Other than that, you do not need to have any previous experience with Drone or Terraform.

{{< alert "info" >}}
As we will be provisioning servers during this tutorial, you will be charged by AWS for the time that your servers are running. AWS instances are charged per-second and, with an approximate hourly cost of $0.03 USD, the charges to your account will be minimal.
{{< / alert >}}

# Creating A Terraform Project

First we need a place to put our Terraform config. Create a GitHub repository called `drone-terraform-tutorial` and clone it to your computer.

Inside the new repository we create our Terraform file. By default, Terraform will read all `.tf` files inside of the current directory, so it doesn't matter what we call it. Here we will call it `arm-infrastructure.tf`.

**arm-infrastructure.tf:**

{{< highlight terraform "linenos=table" >}}
provider "aws" {
    version = "~> 2.0"
    region = "us-east-1"
}

resource "aws_instance" "web" {
  ami           = "ami-07c52df54ded15b28"
  instance_type = "a1.medium"
}
{{< / highlight >}}

This is very simple, but lets break it down.

## Providers

{{< highlight terraform "linenos=table,hl_lines=1-4" >}}
provider "aws" {
    version = "~> 2.0"
    region = "us-east-1"
}

resource "aws_instance" "web" {
  ami           = "ami-07c52df54ded15b28"
  instance_type = "a1.medium"
}
{{< / highlight >}}

At the top of the file we specify a Terraform **provider** that we want to include into this file. Providers are like Terraform plugins that give Terraform the ability to provide **resources** for different clouds or services.

Here we say that we want at least version 2.0 of the `aws` provider and that we want the default region to be `us-east-1`. Now that we have included the provider we can access any resources that that provider makes available in the Terraform file.

> **Note:** The Terraform [provider reference](https://www.terraform.io/docs/providers/index.html) documents which providers Terraform supports and what resources each provider provides.

## Resources

{{< highlight terraform "linenos=table,hl_lines=6-9" >}}
provider "aws" {
    version = "~> 2.0"
    region = "us-east-1"
}

resource "aws_instance" "web" {
  ami           = "ami-07c52df54ded15b28"
  instance_type = "a1.medium"
}
{{< / highlight >}}

After we have included our provider we tell Terraform to create a resource of type `aws_instance` with the name `web`. We also tell it the VM image ID ( AMI ) to use and the instance type. With the instance type, `a1` means that it is one of Amazon's ARM instances and `medium` is the size of instance.

We can add as many of these resources as we want and Terraform will create them automatically on the AWS cloud. For now we will leave it at one server and we wil add another one later.

## Applying The Terraform Config

Now that we have our config written we need to apply the config to make Terraform actually do something!

### Provide Credentials

The first step will be to set the AWS environment variables that are required for the Terraform AWS provider to make changes to your cloud.

```
$ export AWS_ACCESS_KEY_ID="anaccesskey"
$ export AWS_SECRET_ACCESS_KEY="asecretkey"
```

> **Note:** The above environment variables aren't the only way to provided the required credentials. [This doc](https://www.terraform.io/docs/providers/aws/index.html#environment-variables) details the other possible options.

### Install Providers

Before Terraform can use a provider, such as the `aws` provider we used in our Terraform file, we have to install it by running `terraform init`:

```
$ terraform init

Initializing provider plugins...
- Checking for available provider plugins on https://releases.hashicorp.com...
- Downloading plugin for provider "aws" (2.34.0)...

Terraform has been successfully initialized!
```

Terraform reads our Terraform file and automatically detects and intalls the plugins for all of the providers that we used. Terraform puts the plugins in `.terraform/plugins` in our repo.

### Apply Configuration

Now that we have installed the plugins we are ready to apply the configuration.

```
$ terraform apply

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  + aws_instance.web
      id:                           <computed>
      ami:                          "ami-03ab07af3dd3a6993"
      arn:                          <computed>
      associate_public_ip_address:  <computed>
      availability_zone:            <computed>
      cpu_core_count:               <computed>
      cpu_threads_per_core:         <computed>
      ebs_block_device.#:           <computed>
      ephemeral_block_device.#:     <computed>
      get_password_data:            "false"
      host_id:                      <computed>
      instance_state:               <computed>
      instance_type:                "a1.medium"
      ipv6_address_count:           <computed>
      ipv6_addresses.#:             <computed>
      key_name:                     <computed>
      network_interface.#:          <computed>
      network_interface_id:         <computed>
      password_data:                <computed>
      placement_group:              <computed>
      primary_network_interface_id: <computed>
      private_dns:                  <computed>
      private_ip:                   <computed>
      public_dns:                   <computed>
      public_ip:                    <computed>
      root_block_device.#:          <computed>
      security_groups.#:            <computed>
      source_dest_check:            "true"
      subnet_id:                    <computed>
      tenancy:                      <computed>
      volume_tags.%:                <computed>
      vpc_security_group_ids.#:     <computed>


Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

aws_instance.web: Creating...
  ami:                          "" => "ami-03ab07af3dd3a6993"
  arn:                          "" => "<computed>"
  ...
  vpc_security_group_ids.#:     "" => "<computed>"
aws_instance.web: Still creating... (10s elapsed)
aws_instance.web: Creation complete after 14s (ID: i-0af876450a0c7c304)

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

When you run `terraform apply`, Terraform will plan out the changes that it needs to make and ask you to verify that you are sure that you want to make those changes. Once you say "yes" it will start provisioning the resources you have defined in your configuration. In our case it creates one EC2 instance.

Once it is done provisioning, you will be able to see your server in the AWS management console. You have successfully automated the deployment of cloud infrastructure with Terraform!

## Terraform State

Now lets take a second to understand where we are at. Terraform has provisioned our resources and has also collected relevant data about those resources. For instance, we now have a server, but what is the IP address of that server? Terraform keeps track of different data for each kind of resource and everything it provisions. This data is kept in the terraform state.

At any time you can view the current Terraform state by running `terraform show`.

```
$ terraform show
aws_instance.web:
  id = i-09929cada1fd7a09f
  ami = ami-03ab07af3dd3a6993
  associate_public_ip_address = true
  availability_zone = us-east-1a
  cpu_core_count = 1
  cpu_threads_per_core = 1
  instance_state = running
  instance_type = a1.medium
  monitoring = false
  private_dns = ip-123-123-123-123.ec2.internal
  private_ip = 345.345.345.345
  public_dns = ec2-123-123-123-123.compute-1.amazonaws.com
  public_ip = 456.456.456.456
  ...
```

We can see that terraform has also place a `terraform.tfstate` file in our reository, which it uses to keep track of the resoures that Terraform is managing.

If we run `terraform apply` again right now, Terraform will detect that your infrastructure is already in the desired configuration and it will make no changes. On the other hand, if we added another `aws_instance` resource in addition to the one we already have and we tried to run `terraform apply` again, it would create a new server to make ensure that the state of your infrastructure matches what is defined in your Terraform file.

{{< alert "info" >}}
At this point you may be wondi
{{< / alert >}}

## Destroying Infrastructure

When we no longer need our resoures we can remove them by running `terraform destroy`.

{{< alert "warning" >}}
WARNING: There is no way to recover destroyed infrastructure! Be completely sure that you do not need anything that is being destroyed before confirming the operation.
{{< / alert >}}

```
$ terraform destroy
aws_instance.web: Refreshing state... (ID: i-0af876450a0c7c304)

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions:

  - aws_instance.web


Plan: 0 to add, 0 to change, 1 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes

aws_instance.web: Destroying... (ID: i-0af876450a0c7c304)
aws_instance.web: Still destroying... (ID: i-0af876450a0c7c304, 10s elapsed)
aws_instance.web: Destruction complete after 14s

Destroy complete! Resources: 1 destroyed.
```
