---
title: "Terraform a Consul cluster with docker on EC2"
date: 2017-07-19T20:22:25-04:00
subtitle: ""
tags: ["terraform", "consul", "docker", "aws", "ec2", "autoscaling"]
---

The goal of this post is to teach you how to configure and launch a production quality, self-healing, Consul cluster using terraform and docker on AWS. This tutorial will be referenced in the future when I show you how to terraform a classic docker swarm.

### Consul, Terraform, AWS

[Consul](https://www.consul.io/) is an open source software created by Hashicorp used for service discovery, configuration management, DNS, KV storage, and more. It’s a powerful tool, and it’s worth learning.

[Terraform](https://www.terraform.io/) is another open source software created by Hashicorp. It is a simple tool that allows you to “Write, Plan, and Create Infrastructure as Code.” Terraform is another powerful tool that I think developers and sysOps people alike should learn to use.

[AWS](https://aws.amazon.com/) is a cloud hosting service provided by the good people at Amazon. I am a big fan of how easy it is to use and how powerful the toolset is.


### Step 1: Creating the Consul module

Terraform [modules](https://www.terraform.io/intro/getting-started/modules.html) are powerful. They allow you to create a complex infrastructure that can be modified or duplicated just by changing a few variables.

Create a new folder; I’ll call it `test`, where all your code will go and add a modules folder.

```
mkdir test
cd test
mkdir -p modules/consul
```

While in the root folder create your main terraform file.

```
touch main.tf
```

Terraform files end with the extension `.tf`.  These files use a syntax called [HashiCorp Configuration Language (HCL)](https://www.terraform.io/docs/configuration/syntax.html)

This file will be pretty simple.

```
// main.tf
// Setup the core provider information.
provider "aws" {
  region  = "${var.region}"
  access_key = "${var.aws_access_key}"
  secret_key = "${var.aws_secret_key}"
}

//  Create the consul-cluster, based on our consul module.
module "consul-cluster" {

}
```

Ok, this should be easy enough to understand but here is whats going on.

```
provider "aws" {
  region  = "us-east-1"
  access_key = "${var.aws_access_key}"
  secret_key = "${var.aws_secret_key}"
}

```
This block sets up the [provider](https://www.terraform.io/docs/providers/index.html).  Providers are typically the underlying IaaS platform.  All we're doing here is setting up the `AWS` provider with our secret credentials and region.  If you don't have an access and secret key you can get them [here](https://console.aws.amazon.com/iam/home#/home).


```
module "consul-cluster" {

}
```

This block is the beginning of our module. It doesn’t do anything useful or exciting yet.

### Step 2: Variables

We need to create a variable file in the root folder of our project so we can store our default variables.

```
touch variables.tf
```

A module without variables isn’t useful. I’m going to add all the necessary variables to the module block.

```
module "consul-cluster" {
  source          = "./modules/consul"
  region          = "us-east-1"
  ec2_ami         = "ami-d15a75c7" // This is the ubuntu 16.04 AMI in the us-east-1 region
  amisize         = "t2.micro"
  min_size        = "3"
  max_size        = "3"
  key_name        = "your-aws-key"
  asgname         = "consul-asg"
  aws_vpc         = "${var.aws_vpc}"
  vpc_zone_a      = "${var.vpc_zone_a}"
  vpc_zone_b      = "${var.vpc_zone_b}"
  root_url        = "rootdomain.com"  
}
```

I'll go through each of these individually.

`source` is the location of the `module` code.

`region` is the region where we will deploy the cluster.

`ec2_ami` is the `AMI` we will use to deploy our `EC2` instances

`amisize` is the size of the `EC2` instance that we will be using.  A `t2.micro` will be good enough for demo purposes, and they are free for your first year using AWS.

`min_size` and `max_size` are the minima and maximum size of our auto-scaling group we will create.  More on this later.

`key_name` is the pem key file name that you can use to ssh into your instances.

`asgname` is the what we'll use to name our auto-scaling group that we will create later.

`aws_vpc` is going to be the ID of the `vpc` where we are deploying our cluster.  For this tutorial, I am using the default `vpc`.

`vpc_zone_a`, `vpc_zone_b` are the subnets of the default `vpc` where we will deploy our cluster.

`root_url` is the domain name that we own. The root URL is useful so we can use DNS to connect to our cluster rather than IP addresses.

Next, we need to modify our `variables.tf` file to look like this.

```
// test/variables.tf
variable "aws_vpc" {
  default = "vpc-xxxxxxxxx"
}

variable "vpc_zone_a" {
  default = "subnet-xxxxxxxxx"
}

variable "vpc_zone_b" {
  default = "subnet-xxxxxxxxx"
}
```

We will also need to create a `variables.tf` file within our consul module.

```
cd modules/consul
touch variables.tf
```

This file will contain all of our variables that we already defined above because our module doesn’t know what variables it needs. You may notice that I added a `hosted_zone` variable. For this tutorial, I’m using `Route53` for DNS and will need this later.

```
// ./modules/consul/variables.tf
variable "ec2_ami" {
  description = "The EC2 ami to use"
}

variable "asgname" {
  description = "Autoscaling group name"
}

variable "region" {
  description = "Region where the ASG will be deployed"
}

variable "min_size" {
  description = "Minimum size of the ASG"
}

variable "max_size" {
  description = "Maximum size of the ASG"
}

variable "amisize" {
  description = "Instance size of the consul nodes"
}

variable "key_name" {
  description = "Key to be used to create the instances"
}

variable "aws_vpc" {
  description = "The VPC to launch the cluster into"
}

variable "vpc_zone_a" {
  description = "Zone A"
}

variable "vpc_zone_b" {
  description = "Zone B"
}

variable "root_url" {
  description = "The root url to use"
}

variable "hosted_zone" {
  description = "The hosted zone ID to change the consul dns"
  default = "xxxxxxxxxxxx"
}
```

### Step 3: Security Groups

Within the directory `test/modules/consul` we need to create a `security-groups.tf` file.  Within this file (and most other `.tf` files) we are going to create [resources](https://www.terraform.io/docs/configuration/resources.html).  Resources are the most important things you will configure with Terraform.  Resources are the components of your infrastructure.

```
touch security-groups.tf
```

`security-groups.tf` should look like this

```
// test/modules/consul/security-groups.tf
//  Create an internal security group for the VPC, which allows everything in the VPC
//  to talk to everything else.
resource "aws_security_group" "consul-cluster-vpc" {
  name        = "consul-cluster-vpc"
  description = "Default security group that allows inbound and outbound traffic from all instances in the VPC"
  vpc_id      = "${var.aws_vpc}"

  ingress {
    from_port = "0"
    to_port   = "0"
    protocol  = "-1"
    self      = true
  }

  egress {
    from_port = "0"
    to_port   = "0"
    protocol  = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags {
    Name    = "Consul Cluster Internal VPC"
    Project = "consul-cluster"
  }
}

//  Create a security group allowing web access to the public subnet.
resource "aws_security_group" "consul-cluster-public-web" {
  name        = "consul-cluster-public-web"
  description = "Security group that allows web traffic from internet"
  vpc_id      = "${var.aws_vpc}"

  ingress {
    from_port   = 8500
    to_port     = 8500
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags {
    Name    = "Consul Cluster Public Web"
    Project = "consul-cluster"
  }
}
```

Here we created two security groups. The first allows for all inbound traffic from instances within our VPC in this security group. The second allows for public access to port `8500`.  The Consul web UI listens on Port `8500`.

### Step 4: Creating IAM roles, policies, and attaching them

For our instances to be able to get metadata about the other instances in the cluster, we have to create IAM roles and policies that give our instances the access they need to learn about each other.

```
touch iam.tf
```

```
// test/modules/consul/iam.tf
// This policy allows an instance to discover a consul cluster leader.
// It also allows the instances to change the DNS record sets
resource "aws_iam_policy" "leader-discovery" {
  name        = "consul-node-leader-discovery"
  path        = "/"
  description = "This policy allows a consul server to discover a consul leader by examining the instances in a consul cluster Auto-Scaling group. It needs to describe the instances in the auto scaling group, then check the IPs of the instances."
  policy = <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Stmt1468377974000",
            "Effect": "Allow",
            "Action": [
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeAutoScalingGroups",
                "ec2:DescribeInstances",
                "route53:ChangeResourceRecordSets"
            ],
            "Resource": [
                "*"
            ]
        }
    ]
}
    EOF
}
//  Create a role which consul instances will assume.
//  This role has a policy saying it can be assumed by ec2
//  instances.
resource "aws_iam_role" "consul-instance-role" {
  name = "consul-instance-role"
  assume_role_policy = <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": "sts:AssumeRole",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Effect": "Allow",
            "Sid": ""
        }
    ]
}
EOF
}

//  Attach the policies to the role.
resource "aws_iam_policy_attachment" "consul-instance-leader-discovery" {
  name       = "consul-instance-leader-discovery"
  roles      = ["${aws_iam_role.consul-instance-role.name}"]
  policy_arn = "${aws_iam_policy.leader-discovery.arn}"
}

//  Create a instance profile for the role.
resource "aws_iam_instance_profile" "consul-instance-profile" {
  name  = "consul-instance-profile"
  roles = ["${aws_iam_role.consul-instance-role.name}"]
}
```

### Step 5: Templates

This step is probably the most involved step. To create a self-healing consul cluster, we must create an auto-scaling group. To create an auto-scaling group, we must create a launch configuration. Launch configurations use templates to ensure we execute the same commands on every newly created `EC2` instance.

First let’s create a folder within our consul module where we will keep our files, `cd` into it, and create two files we will use in our launch configuration.

```
mkdir files
cd files
touch consul-node.sh
touch docker.service
```

`consul-node.sh` is a bash script that will execute when the `EC2` instance first starts up.

```
#!/bin/bash
# test/modules/consul/files/consul-node.sh
# Log everything we do.
set -x
exec > /var/log/user-data.log 2>&1

# # Update the packages, install aws cli.
apt-get update -y
apt-get install python-setuptools -y
easy_install pip
pip install awscli

# # Install Docker, add ubuntu, start Docker and ensure startup on restart
curl -fsSL https://get.docker.com/ | sh
usermod -a -G docker ubuntu

# A few variables we will refer to later...
ASG_NAME="${asgname}"
REGION="${region}"
EXPECTED_SIZE="${size}"
HOSTED_ZONE="${hosted_zone}"
DOCKER_SERVICE="${docker_service}"
# Return the id of each instance in the cluster.
function cluster-instance-ids {
    # Grab every line which contains 'InstanceId', cut on double quotes and grab the ID:
    #    "InstanceId": "i-example123"
    #....^..........^..^.....#4.....^...
    aws --region="$REGION" autoscaling describe-auto-scaling-groups --auto-scaling-group-name $ASG_NAME \
        | grep InstanceId \
        | cut -d '"' -f4
}

# Return the private IP of each instance in the cluster.
function cluster-ips {
    for id in $(cluster-instance-ids)
    do
        aws --region="$REGION" ec2 describe-instances \
            --query="Reservations[].Instances[].[PrivateIpAddress]" \
            --output="text" \
            --instance-ids="$id"
    done
}

# Wait until we have as many cluster instances as we are expecting.
while COUNT=$(cluster-instance-ids | wc -l) && [ "$COUNT" -lt "$EXPECTED_SIZE" ]
do
    echo "$COUNT instances in the cluster, waiting for $EXPECTED_SIZE instances to warm up..."
    sleep 1
done

# Get my IP address, all IPs in the cluster, then just the 'other' IPs...
IP=$(curl http://169.254.169.254/latest/meta-data/local-ipv4)
mapfile -t ALL_IPS < <(cluster-ips)
OTHER_IPS=( $${ALL_IPS[@]/{$IP}/} )
echo "Instance IP is: $IP, Cluster IPs are: $${ALL_IPS[@]}, Other IPs are: $${OTHER_IPS[@]}"

echo "${docker_service}" > /lib/systemd/system/docker.service
systemctl daemon-reload
systemctl restart docker
# Start the Consul server.
docker run -d --net=host \
    --name=consul \
    consul agent -server -ui \
    -bind="$IP" \
    -client="0.0.0.0" \
    -retry-join="consul.yourdomain.com" \
    -bootstrap-expect="$EXPECTED_SIZE"

# Create the record set change file
RECORDS="";
IDX=0
for i in "$${ALL_IPS[@]}"
do
  VALUE=$i
  echo $$VALUE
  echo $$IDX
  if [ "$$IDX" = "2" ]; then
    RECORDS="$$RECORDS{\"Value\":\"$${VALUE}\"}";
  else
    RECORDS="$$RECORDS{\"Value\":\"$${VALUE}\"},";
  fi
  IDX=$$((IDX+1))
done
echo $$RECORDS

echo "{
  \"Comment\": \"dynamic consul dns changes when nodes die\",
  \"Changes\": [
    {
      \"Action\": \"UPSERT\",
      \"ResourceRecordSet\": {
        \"Name\": \"consul.yourdomain.com\",
        \"Type\": \"A\",
        \"TTL\": 300,
        \"ResourceRecords\": [
          $${RECORDS}
        ]
      }
    }
  ]
}" > /home/ubuntu/change-record-sets.json

aws route53 change-resource-record-sets --hosted-zone-id $HOSTED_ZONE --change-batch file:///home/ubuntu/change-record-sets.json
```

The above bash script should be relatively easy to understand however there are a few syntax items that are different. That is because this is a bash template, not a bash script. We will use Terraform to pass in variables to the template that will make it a valid bash script.  

[Here is more information on Terraform template syntax](https://www.terraform.io/docs/configuration/interpolation.html)

Here is what's happening in the script above:

1. Log the output of all commands to `/var/log/user-data.log`. Logging is useful because debugging AWS init scripts can be a pain.
2. Update all the packages, install the AWS CLI, and install Docker.
3. Create a function that will return all the instance ID's within the cluster
4. Create a function that will return all the private IP's of the instances in the cluster
5. Wait until we have the desired number of EC2 instances running before continuing
6. Configure the Docker daemon
7. Run consul in a Docker container.  See docs [here](https://hub.docker.com/r/_/consul/)
8. Change the DNS record set to the private IP's of your consul cluster


Step 8 might seem the most strange. In many cases, you would want to use an Elastic Load Balancer to keep track of the instances in your auto-scaling groups.  Since I am writing this tutorial as preparation for a classic docker swarm, terraform tutorial, and there is a [bug](https://github.com/docker/swarm/issues/2320) that I haven’t been able to figure out I use raw IP’s rather than an ELB.

`docker.service` is a Terraform template used to set our Docker daemon configuration. Since I deployed this cluster on Ubuntu 16.04, I am using `systemd`. Here is how the docker configuration file looks.

```
# test/modules/consul/files/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network.target docker.socket firewalld.service
Requires=docker.socket

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd -H fd:// --log-driver=json-file --log-opt max-size=10m --log-opt max-file=1 --storage-driver=overlay2 --dns 8.8.8.8 --dns 8.8.4.4 -H unix:///var/run/docker.sock
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=1048576
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process

[Install]
WantedBy=multi-user.target
```

The most important line is the `ExecStart`. Here I set `--log-driver` to `json-file`, set the `max-size` of the log files to `10MB`, I only want to keep one log file before rotating, set the storage driver to [overlay2](https://docs.docker.com/engine/userguide/storagedriver/overlayfs-driver/), and use the google DNS servers.

Now that we have our Terraform templates, we need to render them with Terraform.

Change directories up one level and create a `template.tf` file.

```
cd ..
touch template.tf
```

This file will be much simpler than the previous two.

```
// test/modules/consul/template.tf
data "aws_route53_zone" "root_url" {
  name = "${var.root_url}"
}

data "template_file" "consul" {
  template = "${file("${path.module}/files/consul-node.sh")}"

  vars {
    asgname = "${var.asgname}"
    region  = "${var.region}"
    size    = "${var.min_size}"
    hosted_zone = "${var.hosted_zone}"
    docker_service = "${data.template_file.docker-service.rendered}"
  }
}

data "template_file" "docker-service" {
  template = "${file("${path.module}/files/docker.service")}"
}
```

[data](https://www.terraform.io/docs/configuration/data-sources.html) is how information about the infrastructure gets retrieved.

The two `template_file` data sources get the templates we created above, pass the variables that we used within the template, and render them to fully functional bash scripts and docker daemon configuration files.

### Step 6: Launch configurations

Now that we have our templates ready to go, we can create our launch configuration. Launch configurations make sure that Amazon starts every instance in our auto-scaling group the same way.

```
touch launch-configuration.tf
```

```
// test/modules/consul/launch-configuration.tf
resource "aws_launch_configuration" "consul-cluster-lc" {
  name_prefix          = "consul-node-"
  image_id             = "${var.ec2_ami}"
  instance_type        = "${var.amisize}"
  user_data            = "${data.template_file.consul.rendered}"
  iam_instance_profile = "${aws_iam_instance_profile.consul-instance-profile.id}"

  security_groups = [
    "${aws_security_group.consul-cluster-vpc.id}",
    "${aws_security_group.consul-cluster-public-web.id}",
  ]

  lifecycle {
    create_before_destroy = true
  }

  key_name = "${var.key_name}"
}
```

Here we set:

- The AMI we want
- The instance size
- The `user_data` (which is bash script we wrote above)
- The instance profile (which we created above)
- Add the security groups (which we created above)
- Ensure that we create new instances before we destroy old instances
- Set the keypair that we can use to SSH into the instance

### Step 7: Create the auto-scaling group

The last thing we need to do is create the auto-scaling group. After this, Amazon will start spinning up instances automatically, and we won't need to do anything.

```
touch asg.tf
```

```
// test/modules/consul/asg.tf
// Auto-scaling group for our cluster.
resource "aws_autoscaling_group" "consul-cluster-asg" {
  depends_on           = ["aws_launch_configuration.consul-cluster-lc"]
  name                 = "${var.asgname}"
  launch_configuration = "${aws_launch_configuration.consul-cluster-lc.name}"
  min_size             = "${var.min_size}"
  max_size             = "${var.max_size}"
  vpc_zone_identifier  = ["${var.vpc_zone_a}", "${var.vpc_zone_b}"]

  lifecycle {
    create_before_destroy = true
  }

  tag {
    key                 = "Name"
    value               = "Consul Node"
    propagate_at_launch = true
  }

  tag {
    key                 = "Project"
    value               = "consul-cluster"
    propagate_at_launch = true
  }
}
```

This file `depends_on` the launch configuration we created above so Terraform wont run this until that resource has been created.

Here we set:

- The autoscaling group name
- The launch configuration (which we created above)
- The minimum and maximum size (consul clusters should be either 3 or 5 instances big)
- Which subnets to launch the instances into
- Tags the instances with names so we can see them on the dashboard

### That's it!

If everything went according to plan, then you can run the following commands, and you will have a consul cluster in a few minutes.

```
cd ../..
terraform get
terraform plan
terraform apply
```
