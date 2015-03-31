---
layout: posts
author: Spencer
title: Installing Docker
---

Today, I'm going to detail my steps for installing Docker. Docker is an extension of Linux Containers (LXC) and aims to provide an easier to use environment. This will just be a basic install guide and I will write another post soon, once I figure out how to do some more interesting stuff.

Docker and LXC are interesting because you can run several isolated containers directly in userspace on a Linux host. One of the big advantages here is that no hypervisor is required and you don't need a guest OS like with VMs. This means that containers can be created scarily fast and should be more performant than their VM counterparts. I've seen some debate about whether or not containers are as secure as plain VMs, but truthfully haven't delved too deeply into the details around this. Docker is a project I've been following at a high level for a while because of the potential to hook it into Openstack, but I'm just now getting around to actually putting my hands on it.

##**Setup a Host**##
Setting up a host for your Docker containers is pretty easy. Docker is able to run on pretty much any environment. I'm going to use Vagrant CentOS 6.5 box, but you can find other install instructions [here](https://docs.docker.com/installation/#installation).

* Docker is part of the EPEL repo, so let's install that with:
{%highlight bash %}
sudo yum -y install epel-release
{%endhighlight%} 

* Once that's complete, let's update all or our packages. I found that I couldn't start the Docker daemon without updating. There's a device mapper package that has to be a newer version. After doing this, we can simply install Docker with:
{%highlight bash %}
sudo yum -y update
sudo yum -y install docker-io
{%endhighlight%} 

* Start the Docker daemon and configure it to run at boot:
{%highlight bash %}
sudo service docker start
sudo chkconfig docker on
{%endhighlight%}

* Pull in the CentOS 6 base container. This may take a bit of time depending on your internet connection.
{%highlight bash %}
sudo docker pull centos:centos6
{%endhighlight%}

* Now let's test that it works by asking docker to run a command inside a container. The run command below will create a container, issue the echo command, then shut the container down.
{%highlight bash%}
sudo docker run centos:centos6 echo "Hola, Mundo!"
{%endhighlight%}