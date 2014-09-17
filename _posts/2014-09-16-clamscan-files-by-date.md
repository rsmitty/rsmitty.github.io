---
layout: posts
author: Spencer
title: Scanning Only New Files With Clamscan
---

So, I ran into a need to scan some files for viruses on Ubuntu this past week. However, a couple of things prevented this from being straight forward:

* I couldn't be sure where these files were being stored.

* I didn't want to scan the entire filesystem just to get at these few files.

So I went out looking for a way to scan only new files with clamscan. After a good bit of digging, I ran across [this](http://www.gossamer-threads.com/lists/clamav/users/53298) old thread where someone had a similar question. So after getting a pointer there, I was able to make this happen. Here is the way to do it:

##**Install ClamAV**##

* Issue ```sudo apt-get install clamav``` to get the freshclam and clamscan commands.

* Update the virus definitions with a ```sudo freshclam```. This will take a few seconds the first time.

##**Get Ya Find Right**##

Now this took a bit of playing around with to get where I wanted. I wanted to find all files of a certain type that had been created or modified in the past week. There's also some differences to note about mtime vs. ctime vs. atime as a flag for find. [Linux-FAQs.info](http://www.linux-faqs.info/general/difference-between-mtime-ctime-and-atime) did a good job of explaining these differences:

{%highlight text%}
ctime - ctime is the inode or file change time. The ctime gets updated when the file attributes are changed, like changing the owner, changing the permission or moving the file to an other filesystem but will also be updated when you modify a file.

mtime - mtime is the file modify time. The mtime gets updated when you modify a file. Whenever you update content of a file or save a file the mtime gets updated.

*Most of the times ctime and mtime will be the same, unless only the file attributes are updated. In that case only the ctime gets updated.*

atime - atime is the file access time. The atime gets updated when you open a file but also when a file is used for other operations like grep, sort, cat, head, tail and so on.
{%endhighlight%}

For my purposes, I wanted EVERYTHING that had changed, so that pointed to using the ctime flag. For a first test, I just wanted to see how many items find would return. I was able to do that by issuing (edited to look for .md files, just for laughs): ```sudo find / -name "*.md" -ctime -7 -type f | wc -l```. That command simply returns the number 5 on my machine at the time of writing. Let's not pipe it out to wc so we can see those files:

{%highlight bash%}
rsmitty@rsmitty-notebook:~$ sudo find / -name "*.md" -ctime -7 -type f
/home/rsmitty/Desktop/usenet-dashboard/README.md
/home/rsmitty/Desktop/rsmitty.github.io/_posts/2014-09-07-creating-a-blog-2.md
/home/rsmitty/Desktop/rsmitty.github.io/_posts/2014-09-16-clamscan-files-by-date.md
/home/rsmitty/.xbmc/addons/service.xbmc.versioncheck/README.md
/usr/share/xbmc/addons/service.xbmc.versioncheck/README.md
{%endhighlight%}

**Note: You'll have to do almost everything with ClamAV as root user. Also, you can change the number of days to scan for by changing the '-7' portion of the command.**

Okay, slightly more interesting, and now we know what we're working with. Let's move on...

##**Enter Xargs**##

Using xargs was a first for me as part of this little endeavor. It's a really handy tool to add to the toolbox! If you've ever tried to run a bash command like ```rm *``` and received an error like "Argument list too long", xargs is the answer to your problems. It takes the argument list and breaks it down into smaller pieces. It'll then run subsequent commands with each sublist. For the purposes of what I was doing initally, there were about 7,000 files to scan, xargs was able to break those up into two scans of ~3,000 files and one scan of ~1,000. Worked great!

Here's the final command that I used:
{%highlight bash%}
sudo find / -name "*.md" -ctime -7 -type f -print0 | sudo xargs -0 clamscan --remove --log=/home/rsmitty/clamscan.log
{%endhighlight%}

* The --remove flag just means that if a vulnerability is found in that file, delete the file immediately.
* the --log flag sets the path of the log file that clamscan will write. You will need to do this for sure if you have lots of files to scan, because several scan summaries will be written to this file.

After running, the log file will look something like this:
{%highlight bash%}
rsmitty@rsmitty-notebook:~$ sudo cat /home/rsmitty/clamscan.log 

----------- SCAN SUMMARY -----------
Known viruses: 3560111
Engine version: 0.98.1
Scanned directories: 0
Scanned files: 5
Infected files: 0
Data scanned: 0.02 MB
Data read: 0.01 MB (ratio 2.50:1)
Time: 6.441 sec (0 m 6 s)
{%endhighlight%}



