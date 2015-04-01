---
layout: posts
author: Spencer
title: Hosting Git Repos
---

This post will detail how to host git repos on a server that you own. I'll be covering how to set up your sever-side repo and then how to connect from a remote machine via SSH.

##**Setup Our Server**##
* First and foremost, we'll need to install git. This is going to depend on your package manager, but I'm using CentOS right now, so I'll be issuing 
{%highlight bash%}
sudo yum install -y git
{%endhighlight%}

* Now we'll need to add a user to our system for git. Let's do that and then switch to that user with:
{%highlight bash%}
sudo useradd git
sudo su - git
{%endhighlight%}

* Now that we are the git user, we can setup the SSH keys that we want to accept by making the authorized keys file and putting the public keys of each user we want to have access in this file. After creating this directory and file, we need to set the permissions on them properly or SSH will complain.
{%highlight bash%}
mkdir -p ~/.ssh
touch ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
{%endhighlight%}

* Add the desired public SSH keys in authorized keys. You can add several of these if you have a desire for several users to have access to this git repo. Just separate the keys by putting them on a new line. This should look something like:
{%highlight bash%}
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDHytEnXVEiGKu6XVDh/evJhM5ANngMeRJiizr6jsiOmWyMtuuqtGi/84EDQ54OOwDlBfdC72YjPaYEafyez8fYls7M2L82P2Ka96hFapUWwF9TzxAw1yEkV81Rv2OZWpAdf451UCZPClludtym0DyGwZdMGfVJx8ZNPJ61lwx5ijwWQvY4dhZF0Hjo431c9d1mgOLxu94WJ15PC6CjAI9zh/zddmJMHgClkqTuGWWf/t3e/SZ8AJ5ABUtcjPutUdJBGvPI814eD3+JgE18D6AiHN/uWm0JLYx5P06htqb2Eb6uAsCJjTIDyl+I0bOYRUp8PlYzJALv+x8RxP1R35Wr rsmitty@github.com
{%endhighlight%}

##**Create Git Repo**##

* It's time to finally create our git repo. Let's create an easy directory called /git/ and a subdirectory under that for our test project. We need to switch back to our normal user (with sudo ability) to create a directory at the root. You can do that simply by issuing 'exit'.
{%highlight bash%}
sudo mkdir -p /git/testproject.git
sudo chown -R git:git /git
{%endhighlight%}

* Now, back as the git user, initialize the git repo by using the 'git init' command inside that directory:
{%highlight bash%}
cd /git/testproject.git
git init --bare
{%endhighlight%}

##**Test It Out**##

* Back on your local machine, let's verify that this is actually working for us. This should be as simple as doing a git clone to the proper path on the remote server:
{%highlight bash%}
git clone git@GITSERVERNAME:/git/testproject.git
{%endhighlight%}

* Change into the local testproject directory and create a file for our first commit:
{%highlight bash%}
touch README.md
{%endhighlight%}

* Let's add, commit, and push the file up.
{%highlight bash%}
git add README.md
git commit -m "initial commit"
git push origin master
{%endhighlight%}

Now we've got a fully functional git repo with a master branch. All ready to go!
