+++
title = "Marriage made in heaven... Terraform and Ansible Part 1 of 2"
description = ""
tags = [
    "Terraform",
    "Ansible",
    "Development",
]
date = "2018-05-29"
categories = [
    "Terraform"
    "Development",
    "Ansible",
]
+++

# My initial workings with Terraform and the Marriage made in heaven….Combining Ansible's Templating, Vault and Orchestration with Terraform - Part 1 of 2

## The Beginnings…

I have been using Ansible since about 2015 and it has been my goto tool for configuration management, application deployments and orchestration for on-premise and public cloud infrastructures. I have also used Ansible for AWS infrastructure provisioning using the many existing Ansible AWS modules and although "doable", it really is not Ansible's strength because you have to understand all dependencies and what to create and/or destroy in procedural order, which can be a headache to say the least. 

In the past, I briefly looked at AWS Cloudformation and Terraform as IAC tools to assist with AWS infrastructure provisioning, but never had the opportunity to really dive in. At the time, I was leaning heavily towards Terraform, for the fact that Terraform had a plan step that outlines all the changes it's going to make before actually making them, it's HCL syntax was easier to read than JSON, it supported modules for better DRY principals and it worked with multiple cloud providers. This last point about Terraform multi-cloud support is sort of misleading because you still need to understand each cloud provider and code accordingly, it's not like they have some sort of higher level abstraction api that works across all cloud providers, that would be nice, but probably very difficult to achieve without api standardization across cloud providers (not happening any time soon). However, you still use the same Terraform tool, which was a key sticking point for me.

I know a lot has changed with Cloudformation, it now supports YAML format, it has drift detection, it supports stacksets for better modularization of your code base, it can handle rollbacks, which is something Terraform can't do.
Though not the focus of this writing

## Though not the focus of this writing

Like Ansible and a lot of these tools, it's very quick to get started but…if no thought is put into the team's SDLC process, the git repository structure, branching, a CI/CD pipeline of sorts, the project folder structure,  DRY principles etc... then you will soon have a mess on your hands with people resorting to "out of band" changes and not following the process. You will finally end up not trusting your IAC code base anymore and will be too afraid to run any sort of automation.


## So it's time to try something different. Let’s dive into Terraform!
### Installation....

Installation was pretty straight forward, simply download the Terraform zip for you platform and extract the binary to your directory. 

### Key requirements....

One of the key things for me is to start off with a good project directory structure that encompasses these key requirements:

- Isolation between environments, for example running something against staging environment shouldn't touch production.
- Ability to support multiple environments/accounts/regions with differences captured in variables and injected upon runtime.
- Variables for each environment should be contained to a specific file and/or folder and not scattered all over the place.
- The code base should be as DRY as possible with only config differences between the environments. Note: This requirement is flexible,  if your team is not disciplined enough to manage module dependencies with git versioning then you may run the risk of breaking your infrastructure on your next run. Instead I would be ok with copying and pasting between code bases.

### Directory Structure and Isolation…

So given these requirements, I started googling "Terraform best practice directory structures" and boy oh boy, there's a lot of opinions out there. When I first started with Ansible, it also took me a while to find a good directory structure with environment support. I really never understood directory structures that don't cater to multiple environments, I guess those folks only have a production environment and they don't transition manually or via an automated pipeline through non-prod environments, then eventually promoted into prod? If that's the case, I am seeing failure… anyone else?

### The structure that made most sense to me was the one presented by grunt.io



The directory structure is basically broken up by AWS Region/Environment/Component, so I went and started with this structure and had my VPC component split in it's own directory, I had my data stores split in it's own directory etc…

So now it's time to do some testing and I wanted to bring up my full infrastructure and then destroy it. Thinking that I can run a single command like "terraform apply" and Terraform would be smart enough to manage dependencies and know to bring up my VPC before the rest of the components etc…

### But I was wrong...

What I didn't realize is that Terraform can only manage all the dependencies from a single folder. This means if I had my VPC code in a folder and my RDS code in another folder, I would first have to run "Terraform apply" in my VPC folder to stand up my VPC first, then run "Terraform apply" again in my RDS folder to stand up my RDS infrastructure and use Remote State for lookups for such things as my VPCID etc…

Now if I want to destroy, I basically need to destroy in reverse order. 

Really… This is one of the primary reasons for not using ansible, I don't want to remember this dependency stuff. 

The bottom line is this… there is a trade-off you need to decide on, which is level of isolation, the more isolation, the more you need to manage dependencies yourself, but also less risk of breaking things. 


### Dry, Remote State, Interpolation and variables…

Terraform by default will maintain state locally from where you are running Terraform from, this is ok if your the only one running Terraform and you always run it from the same computer. However, the best practice for team development is to actually centralized and store the Terraform State remotely, so that anyone using the same Terraform project can simply read or update the existing State.

So I chose to use AWS S3 as my backend storage location for the state file. To set up a backend you basically need to include this in your Terraform codebase.

 terraform {
  backend "s3" {
    bucket = "bucket.com"
    key = "key.tfstate"
    region = "us-east-1"
  }
}

Now in order to keep things DRY and not have to hardcode things, I thought to include variables that can be passed into Terraform and it would interpolate all the variables provided as an example:

```
 terraform {
  backend "s3" {
    bucket = "bucket-${var.env}.com"
    key = "${var.key}.tfstate"
    region = "${var.region}"
  }
}
```  
### But I was wrong...

Terraform does not allow for variable interpolation for the backend configuration. 

### Looping, Count and Modules…

When working with Terraform you at times need to create multiple similar resources or create resources depending on some variable flag, do some sort of if else etc... You can sort of use some "hackery" to achieve these sort of things by using some magic around the "count" parameter and booleans and math interpolation, here is an example of implementing if-else logic

```
resource “aws_xxxxx” “x” {
   count = “${var.boolean}”
}

resource “aws_xxxxx” “x” {
   count = “${1 - var.boolean}”
}
```
Assuming var.boolean is set to true

In the first example, it will create the resource because true is also 1 and therefore count=1.

In the second example, it will NOT create the resource because it's doing the inverse behavior 1-1=0 and therefore resource is not created count=0.  

Sometimes you would like to invoke a module multiple times with different variable input to create resources, however this does not seem to be possible in Terraform at the time of this writing. 

For example:

```
module "security-group-ssh" {
  source  = "terraform-aws-modules/security-group/aws"
  version = "1.22.0"

  name        = "stage-app1-ssh-SG"
  vpc_id      = "${module.vpc.vpc_id}"

  tags = {
    Owner       = "arlindo santos"
    Environment = "stage"
  }

  # Open to CIDRs blocks (rule or from_port+to_port+protocol+description)
  ingress_with_cidr_blocks = [
         { 
       rule = "ssh-tcp"
       cidr_blocks = "0.0.0.0/0"
     },
       ]
  egress_rules = ["all-all"]
}

 
module "security-group-mysql" {
  source  = "terraform-aws-modules/security-group/aws"
  version = "1.22.0"

  name        = "stage-app1-mysql-SG"
  vpc_id      = "${module.vpc.vpc_id}"

  tags = {
    Owner       = "arlindo santos"
    Environment = "stage"
  }

  # Open to CIDRs blocks (rule or from_port+to_port+protocol+description)
  ingress_with_cidr_blocks = [
         { 
       rule = "mysql-tcp"
       cidr_blocks = "0.0.0.0/0"
     },
       ]
  egress_rules = ["all-all"]
}
```
In the above example, I am basically calling the terraform-aws-modules/security-group/aws module 2 times to create different security groups in AWS. However there is still a bit of duplication here, I would instead like to be able to invoke the terraform-aws-modules/security-group/aws module in a loop and alter the variable input instead to keep the codebase a little more DRY.

## Wait… there is Terragrunt that can help with alot of these issues...

There exists a tool called Terragrunt which is basically a wrapper for Terraform and has it's own stanza that you basically embed into your Terraform for example:

```
terragrunt = {
  remote_state {
    backend = "s3"
    config {
      bucket         = "my-terraform-state"
      key            = "${path_relative_to_include()}/terraform.tfstate"
      region         = "us-east-1"
      encrypt        = true
      dynamodb_table = "my-lock-table"
    }
  }
}

terragrunt = {
  dependencies {
    paths = ["../vpc", "../mysql", "../redis"]
  }
}
```

Terragrunt can actually assist with some of the issues mentioned above, in particular it can use variables for defining the remote state backend, it can manage your project dependencies, which will basically run "terraform apply" in the proper order (however you still need to declare those dependencies) and it can help keep your codebase DRY.
 
Terragrunt seems like a great tool. However for me, it felt like yet another tool that also needs to be understood and kept in sync with new features of Terraform. 

So for me, I wanted the power of for loops, if else etc…, possibly a tool we already use
and something that can be used to orchestrate other steps within a workflow for example:


- Create a ServiceNow automated Change Record
- Invoke Terraform to provision AWS infra
- Provision Software/Middleware/App
- Update a Jira Ticket
- etc....

Hey, I think Ansible can actually help here and be that glue that orchestrates the whole process while providing the power of Jinja Templating and Ansible Vault for encrypting secrets!

This is when I decided to search google to see if others are doing the same thing, however I didn't find much except I do see a lot of people using Terraform then executing Ansible provisioners to actually perform the config management side of things. Maybe what I am proposing is some sort of anti-pattern? In any case, I think it will work just fine for me. 

Now I have only been using Terraform for about 1 month, so I am sure some of the issues I mentioned are either irrelevant or there exists a better way? Let me know. 

### In part 2, I will explore using Ansible with Terraform and propose a CI/CD pipeline.

