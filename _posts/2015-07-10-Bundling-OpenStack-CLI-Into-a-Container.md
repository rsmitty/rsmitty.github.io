---
layout: post
comments: false
author: Spencer
title: I'll Take Mine To Go - Bundling OpenStack Client CLIs Into A Container
---

This one's a weird one. I was trying to figure out some interesting containers to build when I overheard someone at work expressing difficulty trying to install the OpenStack client CLIs onto his machine. I thought to myself, what if I could install these once and just push them to whatever environment I please? Or even share it with other folks to use? Here's how you can do it:

##**Create A Dockerfile**##
First I got the basics down for creating a Dockerfile to install the proper CLI packages. It was handy to simply boot a ubuntu:14.04 container and test these steps out manually first. That's an easy one, just do:

{%highlight bash%}
docker run -ti ubuntu:14.04 /bin/bash
{%endhighlight%}

From that point I did a little trial and error to figure out the basics of installing pip, installing the openstack client packages inside of pip, and throwing in a few extra dependencies that I encountered. Here's the full Docker file, we'll talk about the script that gets added in next:

{%highlight bash%}
FROM ubuntu:14.04
MAINTAINER Spencer Smith <robertspencersmith@gmail.com>

##Install pip and necessary dependencies for clients
RUN apt-get update && apt-get -y upgrade
RUN apt-get install -y python-dev\
    python-pip\
    libffi-dev\
    libssl-dev

##Install ssl patches for python 2.7, then install clients
RUN pip install six --upgrade\
    pyopenssl\
    ndg-httpsclient\
    pyasn1\
    python-ceilometerclient\
    python-cinderclient\
    python-glanceclient\
    python-heatclient\
    python-keystoneclient\
    python-neutronclient\
    python-novaclient\
    python-saharaclient\
    python-swiftclient\
    python-troveclient\
    python-openstackclient

##Upload our creds checker and set it as our entrypoint
ADD creds.sh /ostack/
CMD /ostack/creds.sh
{%endhighlight%}

##**Deal With Credentials**##
After creating the Dockerfile, I wanted to make sure that I could create a container general enough for others to use if they wanted. As such, I wanted most of the normal OpenStack environment variables to be passed in through the command line arguments. To handle this, I wrote a quick bash script called creds.sh to add into the container.

The creds.sh file will just ensure that the OS_AUTH_URL, OS_REGION_NAME, OS_TENANT_NAME, and OS_USERNAME variables are present in the environment, then it will prompt the user for their password so that they don't have to put it in plaintext inside the docker run command. Finally, once all the info is present, the script will simply launch a bash session for the user.

Here's the full creds.sh script:
{%highlight bash%}
#!/bin/bash

##Ensure everything but password is passed in as env variable
for var in OS_AUTH_URL OS_REGION_NAME OS_TENANT_NAME OS_USERNAME; do
  if [[ -z ${!var} ]]; then
    echo "${var} is unset. Please pass as a env variable input!"
    exit 1
  fi
done

##Prompt for password input
echo "Please enter your OpenStack Password: "
read -sr OS_PASSWORD_INPUT
export OS_PASSWORD=$OS_PASSWORD_INPUT

##Start a bash prompt
/bin/bash
{%endhighlight%}

##**Build The Container**##
Now, we simply need to build our container. You can give this your own tag if you like. Here's what my `docker build` command looked like:

{%highlight bash%}
docker build -t rsmitty/ostack .
{%endhighlight%}

Ensure you are in the same directory with your Dockerfile.

##**Use It!**##

We can now launch our container and use it to talk to an OpenStack cloud. Ensure that the proper environment variables are passed in using the `-e` flag. Here's what my `docker run` command looks like:

{%highlight bash%}
docker run -ti -e OS_AUTH_URL=https://REDACTED_URL:5000/v2.0/ -e OS_REGION_NAME=RegionOne -e OS_TENANT_NAME=admin -e OS_USERNAME=spencer rsmitty/ostack
{%endhighlight%}

Once the run has put you in the bash prompt, you should be able to use your environment!
{%highlight bash%}
root@ecedc1a96a49:/# cinder list
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
|                  ID                  |   Status  | Display Name | Size | Volume Type | Bootable | Attached to |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
| 2032053f-9809-4c29-b8f2-731fef7a01db | available |     test     |  2   |    iscsi    |  false   |             |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
{%endhighlight%}





