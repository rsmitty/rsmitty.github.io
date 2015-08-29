---
layout: post
comments: true
author: Spencer
title: Containerizing The Grid - BOINC on Docker
tags: docker grid containers
---

Today, we'll go on yet another trip through container land. I've found myself itching to throw anything and everything that I may run on a server into a container. It is so easy to create a Dockerfile, build my image, then deploy it wherever I want with Docker Machine that I tend to do it first thing when I have something new to run. Being able to write it once and have it run anywhere really is powerful stuff. 

So that said, I was reading a bit the other day about grid computing. You may have heard of some of the interesting grid projects like [SETI@Home](http://setiathome.ssl.berkeley.edu/). The idea here is that if you have a machine that often sits idle, you can donate your CPU (and/or GPU) to do some number crunching for the cause. I was thinking to myself upon reading this that I have some VMs that sit around idle for quite a bit of time unless I'm actively prototyping something, so why not give this a shot? I was also surprised to learn that a lot of these projects have standardized on using the same software suite, called BOINC, and you just associate to the project of your choice at launch time for the application. Sounds like a nice idea for a container!

##**Choose Project**##

- Pick which projects you're interested in by starting at the [project page](https://boinc.berkeley.edu/projects.php) of BOINC's website. I picked SETI@Home, Rosetta, and World Community Grid.

- You'll need to create accounts on all of the projects that you are interested in. Once completed, take note of the account keys for each. We will need them later.

##**Create A Docker Image**##

- Create a new Dockerfile in a directory.

- Initialize our Dockerfile by adding a FROM and MAINTAINER section. I picked a Ubuntu 14.04 base image to build off of.
{%highlight bash%}
FROM ubuntu:14.04
MAINTAINER Spencer Smith <robertspencersmith@gmail.com>
{%endhighlight%}

- Next, we will install the BOINC client. Luckily, it is included in Ubuntu's repos, so it isn't very difficult.
{%highlight bash%}
##Install BOINC
RUN apt-get update && apt-get install -y boinc-client
{%endhighlight%}

- We will then want to set our working directory to be that of the BOINC client's lib directory. This will allow our commands to complete successfully next.
{%highlight bash%}
##Set working directory
WORKDIR /var/lib/boinc-client
{%endhighlight%}

- We will now set the default command for our image. This command (which is admittedly a bit long) will start BOINC's client service, sleep shortly, use the 'boinccmd' tool to attach a project, then tail out the stdout/stderr logs for the client. You may notice the '${boincurl}' and '${boinckey}' sections of the command. Those are environment variables that will point us to the project we wish to connect to. You will see these in use later when we launch our container.
{%highlight bash%}
##Run BOINC by default. Expects env vars for url and account key
CMD /etc/init.d/boinc-client start; sleep 5; /usr/bin/boinccmd --project_attach ${boincurl} ${boinckey}; tail -f /var/lib/boinc-client/std*.txt
{%endhighlight%}

- That's it for the Dockerfile. Save it and exit. Here's the complete file:

{%highlight bash%}
FROM ubuntu:14.04
MAINTAINER Spencer Smith <robertspencersmith@gmail.com>

##Install BOINC
RUN apt-get update && apt-get install -y boinc-client

##Set working directory
WORKDIR /var/lib/boinc-client

##Run BOINC by default. Expects env vars for url and account key
CMD /etc/init.d/boinc-client start; sleep 5; /usr/bin/boinccmd --project_attach ${boincurl} ${boinckey}; tail -f /var/lib/boinc-client/std*.txt
{%endhighlight%}

- We can now build our image by running `docker build -t rsmitty/boinc .` in the directory. Feel free to tag differently of course.

##**Start Crunching!**##

Now that we have an image to use, let's launch some containers.

- First, find your proper docker endpoint with `docker-machine ls`. I'll be using my digital ocean docker host for this tutorial.
{%highlight bash%}
Spencers-MacBook-Pro:boinc spencer$ docker-machine ls
NAME      ACTIVE   DRIVER         STATE     URL                         SWARM
default            virtualbox     Stopped
do-dev             digitalocean   Running   tcp://REDACTED:2376
{%endhighlight%}

- Set your docker environment variables to the proper values with
{%highlight bash%}
eval "$(docker-machine env do-dev)"
{%endhighlight%}

- Launch a container, substituting your own desired values for boinckey and boincurl. You should be able to find these values from the account settings for the sites you registered earlier. Also feel free to name your container as you see fit.
{%highlight bash%}
docker run -ti -d --name wcg -e "boincurl=www.worldcommunitygrid.org" -e "boinckey=1234567890" rsmitty/boinc
{%endhighlight%}

- Once launched, we can peek in on our jobs by either allocating a new TTY to the container with `docker exec -ti wcg /bin/bash` or `docker logs wcg`
{%highlight bash%}
Spencers-MacBook-Pro:boinc spencer$ docker logs wcg
 * Starting BOINC core client: boinc                                     [ OK ]
 * Setting up scheduling for BOINC core client and children:

....

29-Aug-2015 15:48:49 [World Community Grid] Started download of 933fbd61802442e2861afa0b31aedcc6.pdbqt
29-Aug-2015 15:48:51 [World Community Grid] Finished download of 933fbd61802442e2861afa0b31aedcc6.pdbqt
{%endhighlight%}

That's it! You can check in on your accomplishments for each project in your account settings. You can find my image for this tutorial in the [Docker Hub](https://hub.docker.com/r/rsmitty/boinc/). If you wish to learn more about the BOINC project itself, please visit their [website](https://boinc.berkeley.edu/).








