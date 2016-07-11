---
layout: post
title: Terraform 0.6.x best practices
description: "Get all the juice out from Terraform"
date: 2016-07-11 00:00:00
tags:
- terraform
categories:
- terraform
twitter_text: 'terraform'
---

## Terraform 0.6.x best practices

I'm pretty sure most of you already know about [Terraform](https://www.terraform.io), an amazing tool from [Hashicorp](https://www.hashicorp.com) (owners of Vagrant, Packer, Consul and others), which allow us to build infrastructure from the code. Oh, yes!

Terraform has a lot of great features: **declarative syntax** based in json, keep the **state** of all the resources, easy to apply changes to our infrastructure, **plugins** easy to use and develop, aggregate and reuse code with **modules**, etc, etc, etc ...

However, it is still a quite young project, **0.6.16** at the time of writing this document, and despite they release new versions very often ( the new version [0.7](https://github.com/hashicorp/terraform/blob/master/CHANGELOG.md) is around the corner ) IMO the development team looks like more focus in adding more and more providers rather than creating some features or fix core some bugs. We only need to be patient and wait :) 

But for those of you that are _cutting edge guys_ I have here some tips I found really useful using terraform in my infrastructures.

### Write a wrapper

No matter the language you want to use: bash, ruby, python or the ancient make ;-) , write a wrapper that will help you to run terraform exactly in the way you want to use it, specially in work environments with big teams. You will make your sysadmin life easier.

Define dependencies in your wrapper, for example: **run plan before apply** to avoid applies without plans! ( sounds silly but I saw environments fully destroyed because of that! :D ), **configure remote states before plan**, ask for **double confirmation before destroy**, test the terraform version allowed to use, run rspecs after apply, etc.

{% highlight bash %}
$ make help

Please use make <option> where <option> is one of
  build_tfvars        build terraform.tfvars from AWS API
  check               check terraform remote state
  get                 build all the terraform dependencies
  plan                run terraform plan (with dependencies update)
  apply               run terraform apply
  clean_all           remove modules and states
  clean_modules       remove modules
  clean_states        remove states
  clean_remote_states remove remote states
  plan_destroy        run terraform plan with -destroy
  destroy             run terraform destroy
  rspec               run integration tests
{% endhighlight %}

For extra security play with your sudoers config and change the PATH environment variable to avoid the use of the terraform binary, forcing users to use the wrapper.

### Create a directory tree

The number of resources you can create with terraform is growing more and more, that makes the terraform.tfstate files potentially very big, making the apply slow and the changes more risky. Because of that, organise properly your terraform code is mission critical.

Create a directory tree that fits with your own infrastructure or organizational schema. Is quite common to have multiple environments like development, preview or production ( I would say this is mandatory nowadays! ) and multiples geographical regions ( could be AWS regions or not ). 

Also, split your terraform code in several files: variables.tf, main.tf and outputs.tf at least and keep your modules in its own directory.

This will help you to maintain complex infrastructures with multiples environment, regions, partners, services, etc. and reduce the probability of blowing up all your servers by mistake \o/ .

{% highlight bash %}
├── accounts
│   ├── development
│   │   └── eu-west-1
│   │       ├── vpc
│   │           ├── variables.tf
│   │           ├── main.tf
│   │           ├── outputs.tf
│   │       └── s3
│   └── production
│       └── eu-west-1
│           ├── vpc
│           ├── peering-vpc
│           ├── iam
│           ├── s3
│           └── route53
└── modules
    ├── ecs_cluster
    │   └── policies
    ├── iam_roles
    │   ├── role1
    │   └── role2
    ├── security_groups
    │   ├── ec2_sg
    │   └── elb_sg
    ├── vpc
    └── vpn
{% endhighlight %}

### Use terraform.tfvars and build it automatically

Because you can persist your variable values in a terraform.tfvars file it means that you can leave your code without hardcoding important values ( like API keys, secrets, etc ). In addition you could create the terraform.tfvars file automatically using your new wrapper ;)

If you are using AWS, use **allowed_account_ids** in the your AWS provider to avoid applies in wrong AWS accounts.

_terrform.tfvars_
{% highlight bash %} 
aws_region  = "eu-west-1"
aws_profile = "production"
aws_account_id = "123456789"
{% endhighlight %}

_variables.tf_
{% highlight bash %} 
variable "aws_region" {}
variable "aws_profile" {}
variable "aws_account_id" {}
{% endhighlight %}

_aws_provider.tf_
{% highlight bash %} 
# Configure the AWS Provider
provider "aws" {
  region              = "${var.aws_region}"
  profile             = "${var.aws_profile}"
  allowed_account_ids = ["${var.aws_account_id}"]
}
{% endhighlight %}

_Ruby script to build terraform.tfvars, I run it automatically from my make file before any plan or apply._

{% highlight ruby %} 
#!/usr/bin/env ruby

require 'bundler/setup'
require 'aws-sdk'

def check_stacks
  (!Dir.pwd[/accounts.*/].empty?) && (Dir.pwd[/accounts.*/].split('/').drop(1).length.eql? 3)
end

def get_account
  @account = Dir.pwd[/accounts.*/].split('/').drop(1)[0]
end

def get_account_id
  iam = Aws::IAM::Client.new( region: @region, credentials: @credentials )
  iam.get_user().user.arn.split(':')[4]
end

def get_region
  @region = Dir.pwd[/accounts.*/].split('/').drop(1)[1]
end

def get_credentials
  @credentials ||= Aws::SharedCredentials.new(profile_name: get_account)
  @credentials.credentials
end

def build_aws_config
  tfvars = File.open(Dir.pwd + "/terraform.tfvars", 'w')
  tfvars.puts "aws_region  = \"#{get_region}\""
  tfvars.puts "aws_profile = \"#{get_account}\""
  tfvars.puts "aws_account_id = \"#{get_account_id}\""
  tfvars.close
end

get_credentials
build_aws_config if check_stacks
{% endhighlight %}

### Use multiple accounts/regions

Split your environments in different provider accounts. Keeping your environments as much isolated as possible will help you to avoid mistakes when you have to create or destroy infrastructure.

If you are using AWS you can get benefit from the [Named Profiles](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html#cli-multiple-profiles) matching the subdirectories under the account directory with the Named Profiles. Your script could use the AWS_PROFILE environment variable to build a proper terraform.tfvars automatically.

{% highlight bash %}
cd accounts/development/eu-west-1/vpc/
AWS_PROFILE=development make plan

cd accounts/production/eu-west-1/vpc/
AWS_PROFILE=production make plan
{% endhighlight %}

### Split terraform code in small pieces

Sometimes you would find out that you have to deal with the **terraform.tfstate** file manually, you will have to delete, add or modify some resources or values.

This usually happens to me when I upgraded terraform, sometimes the resource config change and force me to destroy the resource and create a new one, if I can't do it ( or just I don't want to do it because $reasons ) I have to modify the state manually. Also could happens when your infrastructure provider ( AWS for me ) change a default value or when you just want to add a resource that already exists to terraform.

So, because of that, the smaller the terraform state the better. Use the directory tree to split your terraform code in small pieces.

Btw, terraform is dealing better and better with this corner cases, even in the new Terraform 0.7 there is a CLI tool to help with the state.

### Use remote states

If you followed the previous advise now you will ask: What about the outputs? How could I reuse resources that has been defined in other terraform states?

[Terraform Remote State](https://www.terraform.io/docs/providers/terraform/r/remote_state.html) is the answer.

Configure all your terraform states to be uploaded to S3 buckets ( with versioning enabled! ), you should do this automatically with your wrapper according to your tree directory.

An example using Makefile:
{% highlight bash %}
terraform remote config -backend=s3 -backend-config="region=eu-west-1" -backend-config="bucket=terraform-$(notdir $(patsubst %/,%,$(dir $(patsubst %/,%,$(dir $(CURDIR))))))" -backend-config="key=/states/$(notdir $(patsubst %/,%,$(dir $(CURDIR))))/$(notdir $(CURDIR))/terraform.tfstate"
{% endhighlight %}

Add the values you want to "return" in you outputs.tf file.

{% highlight bash %}
output "vpc_id"  { value = "${aws_vpc.main.id}" }
{% endhighlight %}

Then add to your main.tf the terraform remote states and use the output properly

{% highlight bash %}
################################################################################
# Load remote states because dependencies
################################################################################
resource "terraform_remote_state" "vpc" {
  backend = "s3"
  config {
    region = "${var.aws_region}"
    bucket = "terraform-${var.aws_profile}"
    key    = "/states/${var.aws_profile}/${var.aws_region}/vpc/terraform.tfstate"
  }
}

################################################################################
# Create resources
################################################################################
resource "aws_security_group" "allow_ssh" {
  name        = "allow_ssh"
  description = "Allow SSH traffic"
  vpc_id      = "${terraform_remote_state.vpc.output.vpc_id}"
  ...
  ...
}
{% endhighlight %}

Don't worry if you forget some output, you can add more when you need it, just edit the outputs.tf file and run plan + apply again, the plan will be clean ( I know, it is anoyning! ), but when the apply finish the new output will appear in the terraform state! Magic! Ready to be used in your code!

### Keep all your terraform under version control

That's obvious! What? No? Seriously, use git as soon as possible.

Just a couple of points here:

- DO NOT commit your .terraform local directory or any terraform.tfstate, a lot of sensible information is inside.
- Since we said to use remote states, don't worry about the state file, S3 will keep it for use ( with versioning! ).

### Use modules, don't use nested modules

Modules **yes!**, nested modules **no!**

Terraform modules are great, helps us to reuse code and keep a good code organization, but you should keep the modules as simple and reusable as possible and **avoid** nested modules.

There're a bunch of issues related with nested modules waiting to be fixed, like [#5870](https://github.com/hashicorp/terraform/issues/5870) or [#5234](https://github.com/hashicorp/terraform/issues/5234) or [#5190](https://github.com/hashicorp/terraform/issues/5190) , more and more ...

I know that eventually those bugs will be fixed, but it has been a long time since nested modules are having issues, so for now, don't use nested modules.

### Write integration tests

With terraform you are building infrastructure from code, so as you do with any code... write tests! In this case, integration tests.

I've been using [awspec](https://github.com/k1LoW/awspec), it's quite complete, it's easy to use, it's enough.

_An example with IAM groups and users_

{% highlight ruby %}
require 'spec_helper'

RSpec.describe 'main.tf' do

  let(:aws_account) { '123456789' }
  let(:iam_group) { 'group1' }
  let(:path) { '/group1/' }
  let(:s3_bucket) { 'arn:aws:s3:::super-secret-bucket' }

  describe iam_user('user1') do
    context "user definition" do
      it { should exist }
      it { should belong_to_iam_group(iam_group) }
      its(:arn) { should eq 'arn:aws:iam::' + aws_account + ':user'+ path + 'user1' }
      its(:path) { should eq path }
      its(:user_name) { should eq 'user1' }
    end

    context "user in the right S3 bucket and directory" do
      it { should be_allowed_action('s3:ListBucket').resource_arn(s3_bucket).context_entries([{context_key_name:"s3:prefix",context_key_values:["group1/secret_file.txt"],context_key_type:"string"}]) }
    end

    context "user accessing S3 buckets" do
      it { should_not be_allowed_action('s3:ListAllMyBuckets') }
    end

    context "user in the wrong S3 directory" do
      not_allowed_resources = [
        'arn:aws:s3:::super-secret-bucket/',
        'arn:aws:s3:::super-secret-bucket/group2/',
      ]

      not_allowed_resources.each do |resource_name|
        it { should_not be_allowed_action('s3:ListBucket').resource_arn(resource_name) }
      end

    end
  end
end
{% endhighlight %}

### Add terraform to your CI tools

With terraform we can not write tests before apply the infrastructure, terraform plan try to do this and sometimes fails ( quite common to have a succesful plan and after that the apply fails! ). However, we can run integration tests after the apply to ensure everything has been created properly, it is worth you leave this to your CI tool ( let's say Jenkins ).

Also, running terraform integration tests in Jenkins will help you to detect when a resource has been changed manually.

### Happy code guys!
