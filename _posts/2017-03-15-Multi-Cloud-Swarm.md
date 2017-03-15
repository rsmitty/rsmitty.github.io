---
layout: post
comments: true
author: Spencer
title: Multi-Cloud Docker Swarm with Terraform and Ansible
tags: docker containers aws gce cloud terraform ansible
---

Since the release of Docker 1.12, there's a new Swarm mode that is baked into the Docker engine. I wanted to spend some time, after months of Kubernetes-only work, to check out how Swarm was doing things and to see how easy it was to get started. Building a quick cluster on your laptop or on a single provider seemed to be straight forward, but I couldn’t readily find a no nonsense way to spin one up across multiple clouds. So, you know, I went ahead and built one.

Today, we'll walk through how you can create a multi-cloud Swarm on AWS and GCE. We will use Terraform and Ansible to complete the bootstrap process, which is surprisingly straightforward. You can go directly to the Github repo where I stashed the code by clicking [here](https://github.com/rsmitty/cross-cloud-swarm).

I'll give an early preface and say that I've only used this for some testing and learning experience. It's in no way prod. ready or as robust as it should be to accept lots of different configurations.

## **Outline** ##

The deployment of our cluster will occur in the following order:
- AWS infrastructure is provisioned (security groups and instances)
- GCE infrastructure is provisioned (firewall rules and instances)
- An Ansible inventory file is created in the current working directory
- Docker is installed and Swarm is initialized

## **Terraform Scripts** ##
In order to create our infrastructure, we want to create three terraform scripts and a variable file. This will provide all of the necessary information that Terraform needs to do it's thing.

- Create four files: `touch 00-aws-infra.tf 01-gce-infra.tf 02-create-inv.tf variables.tf`
- Open `variables.tf` for editing. We'll populate this file with all of the configurable options that we will use for each cloud, as well as some general info that the instances have in common, regardless of cloud. Populate the file with the following:
{% highlight bash %}
##General vars
variable "ssh_user" {
  default = "ubuntu"
}
variable "public_key_path" {
  default = "/Users/spencer/.ssh/id_rsa.pub"
}
variable "private_key_path" {
  default = "/Users/spencer/.ssh/id_rsa"
}
##AWS Specific Vars
variable "aws_worker_count" {
  default = 1
}
variable "aws_key_name" {
  default = "spencer-key"
}
variable "aws_instance_size" {
  default = "t2.micro"
}
variable "aws_region" {
  default = "us-west-2"
}
##GCE Specific Vars
variable "gce_worker_count" {
  default = 1
}
variable "gce_creds_path" {
  default = "/Users/spencer/gce-creds.json"
}
variable "gce_project" {
  default = "test-project"
}
variable "gce_region" {
  default = "us-central1"
}
variable "gce_instance_size" {
  default = "n1-standard-1"
}
{% endhighlight %}

- You can update these defaults if you desire, but also know that you can override these at runtime with the `-var` flag to terraform. See [here](https://www.terraform.io/intro/getting-started/variables.html) for details.

- Now that we've got the variables we need, let's work on creating our AWS infrastructure. Open `00-aws-infra.tf` and put in the following:
{% highlight bash %}
##Amazon Infrastructure
provider "aws" {
  region = "${var.aws_region}"
}

##Create swarm security group
resource "aws_security_group" "swarm_sg" {
  name        = "swarm_sg"
  description = "Allow all inbound traffic necessary for Swarm"
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 2377
    to_port     = 2377
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 7946
    to_port     = 7946
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 7946
    to_port     = 7946
    protocol    = "udp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 4789
    to_port     = 4789
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port = 0
    to_port   = 0
    protocol  = "-1"
    cidr_blocks = [
      "0.0.0.0/0",
    ]
  }
  tags {
    Name = "swarm_sg"
  }
}

##Find latest Ubuntu 16.04 AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-xenial-16.04-amd64-server-*"]
  }
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
  owners = ["099720109477"] # Canonical
}

##Create Swarm Master Instance
resource "aws_instance" "swarm-master" {
  depends_on             = ["aws_security_group.swarm_sg"]
  ami                    = "${data.aws_ami.ubuntu.id}"
  instance_type          = "${var.aws_instance_size}"
  vpc_security_group_ids = ["${aws_security_group.swarm_sg.id}"]
  key_name               = "${var.aws_key_name}"
  tags {
    Name = "swarm-master"
  }
}

##Create AWS Swarm Workers
resource "aws_instance" "aws-swarm-members" {
  depends_on             = ["aws_security_group.swarm_sg"]
  ami                    = "${data.aws_ami.ubuntu.id}"
  instance_type          = "${var.aws_instance_size}"
  vpc_security_group_ids = ["${aws_security_group.swarm_sg.id}"]
  key_name               = "${var.aws_key_name}"
  count                  = "${var.aws_worker_count}"
  tags {
    Name = "swarm-member-${count.index}"
  }
}
{% endhighlight %}

Walking through this file, we can see a few things happen. If you've seen Terraform scripts before it's pretty straight forward. 
- First, we simply configure a bit of info to tell Terraform to talk to our desired region that's specified in the variables file.
- Next, we create a security group called `swarm_sg`. This security group allows ingress from all of the ports listed [here](https://docs.docker.com/engine/swarm/swarm-tutorial/#open-ports-between-the-hosts).
- Finally, we'll create all of the nodes that we plan to use in AWS. We'll create the master instance first, simply because it's tagged differently, then we'll create the workers. Notice the use of `${var...` everywhere. This is how variables are passed from the vars file into the desired configuration of our nodes.

It's now time to create our GCE infrastructure.

- Open `01-gce-infra.tf` and paste the following:
{% highlight bash %}
##Google Infrastructure
provider "google" {
  credentials = "${file("${var.gce_creds_path}")}"
  project     = "${var.gce_project}"
  region      = "${var.gce_region}"
}

##Create Swarm Firewall Rules
resource "google_compute_firewall" "swarm_sg" {
  name    = "swarm-sg"
  network = "default"

  allow {
    protocol = "udp"
    ports    = ["7946"]
  }

  allow {
    protocol = "tcp"
    ports    = ["22", "2377", "7946", "4789"]
  }
}

##Create GCE Swarm Members
resource "google_compute_instance" "gce-swarm-members" {
  depends_on   = ["google_compute_firewall.swarm_sg"]
  name         = "swarm-member-${count.index}"
  machine_type = "${var.gce_instance_size}"
  zone         = "${var.gce_region}-a"
  count        = "${var.gce_worker_count}"

  disk {
    image = "ubuntu-os-cloud/ubuntu-1604-lts"
  }

  disk {
    type    = "local-ssd"
    scratch = true
  }

  network_interface {
    network       = "default"
    access_config = {}
  }

  metadata {
    ssh-keys = "ubuntu:${file("${var.public_key_path}")}"
  }
}
{% endhighlight %}
 Taking a read through this file, you'll notice we're essentially doing the same thing we did with AWS:
 - Configure some basic info to connect to GCE.
 - Create firewall rules in the default network to allow ingresses for Swarm.
 - Create the Swarm members in GCE.

 We're almost done with Terraform! The last bit is we need to take the infrastructure that gets created and create an inventory file that Ansible can use to provision the actual Docker bits. 

 - Populate `02-create-inv.tf`:
 {% highlight bash %}
resource "null_resource" "ansible-provision" {
  depends_on = ["aws_instance.swarm-master", "aws_instance.aws-swarm-members", "google_compute_instance.gce-swarm-members"]

  provisioner "local-exec" {
    command = "echo \"[swarm-master]\" > swarm-inventory"
  }

  provisioner "local-exec" {
    command = "echo \"${format("%s ansible_ssh_user=%s", aws_instance.swarm-master.0.public_ip, var.ssh_user)}\" >> swarm-inventory"
  }

  provisioner "local-exec" {
    command = "echo \"[swarm-nodes]\" >> swarm-inventory"
  }

  provisioner "local-exec" {
    command = "echo \"${join("\n",formatlist("%s ansible_ssh_user=%s", aws_instance.aws-swarm-members.*.public_ip, var.ssh_user))}\" >> swarm-inventory"
  }

  provisioner "local-exec" {
    command = "echo \"${join("\n",formatlist("%s ansible_ssh_user=%s", google_compute_instance.gce-swarm-members.*.network_interface.0.access_config.0.assigned_nat_ip, var.ssh_user))}\" >> swarm-inventory"
  }
}
 {% endhighlight %}

This file simply tells Terraform, after all infrastructure has been created, to drop a file locally called `swarm-inventory`. The file that's dropped should look like (real IPs redacted):
{% highlight bash %}
[swarm-master]
aaa.bbb.ccc.ddd ansible_ssh_user=ubuntu
[swarm-nodes]
eee.fff.ggg.hhh ansible_ssh_user=ubuntu
iii.jjj.kkk.lll ansible_ssh_user=ubuntu
{% endhighlight %}

## **Ansible Time!** ##

Okay, now that we've got the Terraform bits ready to deploy the infrastructure, we need to be able to actually bootstrap the cluster once the nodes are online. We'll create two files here: `swarm.yml` and `swarm-destroy.yml`.

- Create `swarm.yml` with:
{% highlight ruby %}
{%raw%}
- name: Install Ansible Prereqs
  hosts: swarm-master:swarm-nodes
  gather_facts: no
  tasks:
    - raw: "apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y python-minimal python-pip"

- name: Install Docker Prereqs
  hosts: swarm-master:swarm-nodes
  gather_facts: yes
  tasks:
    - package:
        name: "{{item}}"
        state: latest
      with_items:
        - apt-transport-https
        - ca-certificates
        - curl
        - software-properties-common
    - apt_key:
        url: "https://download.docker.com/linux/ubuntu/gpg"
        state: present
    - apt_repository:
        repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu xenial stable"
        state: present

- name: Install Docker
  hosts: swarm-master:swarm-nodes
  gather_facts: yes
  tasks:
    - package:
        name: "docker-ce"
        state: latest
    - user: 
        name: "{{ ansible_ssh_user }}"
        groups: docker
        append: yes

- name: Initialize Swarm Master
  hosts: swarm-master
  gather_facts: yes
  tasks:
    - command: "docker swarm init --advertise-addr {{inventory_hostname}}"
    - command: "docker swarm join-token -q worker"
      register: swarm_token
    - set_fact: swarmtoken="{{swarm_token.stdout}}"
  
- name: Join Swarm Nodes
  hosts: swarm-nodes
  gather_facts: yes
  tasks:
  - command: "docker swarm join --advertise-addr {{inventory_hostname}} --token {{hostvars[groups['swarm-master'][0]].swarmtoken}} {{hostvars[groups['swarm-master'][0]].inventory_hostname}}:2377"
{% endraw %}
{% endhighlight %}

This Ansible playbook does a few things:
- Bootstraps all nodes with the necessary packages for Ansible to run properly.
- Installs Docker prerequisites and then installs Docker.
- On the master, initializes the swarm and grabs the key necessary to join.
- On the nodes, simply joins the swarm.

Now, that's really all we need. But while we're here, let's make sure we can tear our Swarm down as well.
- Create `swarm-destroy.yml`:
{% highlight ruby %}
{%raw%}
- name: Leave Swarm
  hosts: swarm-master:swarm-nodes
  gather_facts: yes
  tasks:
    - command: "docker swarm leave --force"
{% endraw %}
{% endhighlight %}

That one's really easy. It really just goes to each node and tells it to leave the Swarm, no questions asked.

## **Create Swarm** ##

Okay, now that we've got all the bits in place, let's create our swarm.
- First source AWS API keys with `source /path/to/awscreds.sh` or `export ...`.
- Create the infrastructure with `terraform apply`. Keep in mind that you may also want to pass in the `-var` flag to override defaults.
- Once built, issue `cat swarm-inventory` to ensure master and workers are populated.
- Bootstrap the Swarm cluster with `ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -b -i swarm-inventory swarm.yml`.

In the just a couple of minutes, these steps should have been completed successfully. If all looks like it went okay, SSH into the master node.

- Issue `docker node ls` and view all the nodes in the Swarm. You'll notice different hostnames between AWS and GCE instances:
{% highlight bash %}
ubuntu@ip-172-31-5-8:~$ docker node ls
ID                           HOSTNAME         STATUS  AVAILABILITY  MANAGER STATUS
mei9ysylvokq6foczu7ygwso6 *  ip-172-31-5-8    Ready   Active        Leader
tzsdybhx5f9c8qv2z55ry2me4    swarm-member-0   Ready   Active
vzxbjpus3t8ufm0j0z7rmzn18    ip-172-31-6-146  Ready   Active
{% endhighlight %}

## **Test It Out** ##
Now that we've got our Swarm up, let's create a scaled service and we'll see it show up on different environments.

- Issue `docker service create --replicas 5 --name helloworld alpine ping google.com` on the master.
- Find where the pods are scheduled with `docker service ps helloworld`:
{% highlight bash %}
ubuntu@ip-172-31-5-8:~$ docker service ps helloworld
ID            NAME          IMAGE          NODE             DESIRED STATE  CURRENT STATE              ERROR  PORTS
6ifn97x0lcor  helloworld.1  alpine:latest  swarm-member-0   Running        Running about an hour ago
fmfgkurl99j5  helloworld.2  alpine:latest  swarm-member-0   Running        Running about an hour ago
2et88afaxfky  helloworld.3  alpine:latest  ip-172-31-6-146  Running        Running about an hour ago
jbobdjkk062h  helloworld.4  alpine:latest  ip-172-31-5-8    Running        Running about an hour ago
j9nkx5lqr82x  helloworld.5  alpine:latest  ip-172-31-5-8    Running        Running about an hour ago
{% endhighlight %}
- SSH into the GCE worker and find the containers running there with `docker ps`.
- Show that the containers are pinging Google as expected with `docker logs <CONTAINER_ID>`
- Do the same with the AWS nodes.

## **Teardown** ##
Once we're done with our test cluster it's time to trash it.
- You can tear down just the Swarm, while leaving the infrastructure with `ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -b -i swarm-inventory swarm-destroy.yml`
- Tear down all the things with a simple `terraform destroy`

That's it! I was happy to get a cross-cloud Swarm running pretty quickly. Over the next few weeks, I'll probably come back to revisit my Swarm deployment and make sure some of the more interesting things are possible, like creating networks and scheduling webservers. Stay tuned!