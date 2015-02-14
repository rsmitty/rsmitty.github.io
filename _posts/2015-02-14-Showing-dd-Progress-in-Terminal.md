---
layout: posts
author: Spencer
title: Showing dd Progress
---

As I was trying to write an ISO to a USB drive, I wanted to see the progress when
using the 'dd' command line tool. I found a quick pointer on StackOverflow to use
the 'pv' command, so I adapted a little to use on a Mac. This will also serve
as a guide on how to write ISOs on Mac. Here's how:

* Install pv with homebrew: ```brew install pv```
* Find your USB drive with ```diskutil list```. Should be pretty easy to spot the
USB drive as it will be smaller than the other disks. Tread lightly though, don't
mess with your hard drive. I'll use /dev/disk3, as that's what my command returned.
* Unmount it with ```diskutil unmountDisk /dev/disk3```.
* Become root with ```sudo su```
* Write your iso with this general layout, substituting paths where necessary:
```dd if=/path/to/your.iso | pv | dd of=/dev/disk3 bs=1024k```.
