---
layout: post
comments: false
author: Spencer
title: Removing Cruft From Docker
---

This post is simply here to document how to remove untagged and exited images and containers.

##**Exited Images**##
You can find previously exited images by using filters:

{%highlight bash%}
docker ps -f "status=exited"
{%endhighlight%}

This will return the full text for each exited container:
{%highlight bash%}
Spencers-MacBook-Pro:~ spencer$ docker ps -f "status=exited"
CONTAINER ID        IMAGE                                           COMMAND                CREATED             STATUS                      PORTS               NAMES
72b10aebd2b0        rsmitty/ostack                                                     "/bin/sh -c /bin/bas   9 hours ago         Exited (126) 9 hours ago                        distracted_wilson
aa6c9cfc662f        rsmitty/ostack                                                     "/bin/sh -c /bin/bas   9 hours ago         Exited (0) 9 hours ago                          sick_einstein
1bfce6a324b5        ubuntu:14.04                                                       "/bin/bash"            9 hours ago                                                         jovial_einstein
{%endhighlight%}

To remove all of them, you can nest a command similar to the one above inside the `docker rm` command:
{%highlight bash%}
docker rm $(docker ps -qf "status=exited")
{%endhighlight%}

Docker responds with a list of IDs that it deleted:
{%highlight bash%}
Spencers-MacBook-Pro:~ spencer$ docker rm $(docker ps -qf "status=exited")
72b10aebd2b0
aa6c9cfc662f
1bfce6a324b5
f7fd1c00837c
{%endhighlight%}

##**Untagged Images**##
If you wish to clean up untagged instances you can find them with another filter command, similar to the one above:
{%highlight bash%}
docker images -f "dangling=true"
{%endhighlight%}

This will return a formatted list of the untagged images:
{%highlight bash%}
Spencers-MacBook-Pro:~ spencer$ docker images -f "dangling=true"
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
<none>              <none>              bfa68ad8ff4c        23 hours ago        457.1 MB
<none>              <none>              0043ceae2104        23 hours ago        457.1 MB
<none>              <none>              edff0ab07895        23 hours ago        421.3 MB
<none>              <none>              6ae539b22ab9        23 hours ago        421 MB
{%endhighlight%}

And again, nesting a variation of that command to actually do the cleanup:

{%highlight bash%}
docker rmi $(docker images -qf "dangling=true")
{%endhighlight%}

Now something interesting happens!
{%highlight bash%}
Spencers-MacBook-Pro:~ spencer$ docker rmi $(docker images -qf "dangling=true")
Error response from daemon: Conflict, cannot delete bfa68ad8ff4c because the running container 7e5c96166fcb is using it, stop it and use -f to force
Deleted: edff0ab0789548cf33db3589eae5cc93589e7aea379bc3383f58c00b71ebb8cb
Deleted: e619828bd6f049d81a1920b96634534044ab0bf8f1dd4e40d9daf82d9a5c80b6
Deleted: ac0a2e7c0897058649e9e31cd4a319ee08158646990a607a54a0492f27e6e275
Error: failed to remove images: [bfa68ad8ff4c]
{%endhighlight%}

If the images are in use by some container, you must first stop the container. You'll have to resolve this in order to remove this image. This is a good thing though, it can keep you from blowing yourself up :)






