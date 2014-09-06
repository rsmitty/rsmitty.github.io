---
layout: posts
author: Spencer
title: MD5 Password Cracker In Python
---

I am in a Distributed Computing class this semester. We just had to write a brute-force password cracker using the Condor grid we have on-campus. It really took forever to generate an 8 character (a-z) code even though I ran it 676 different times. Total run-time was about 25 minutes. Anyways, we also had to create a password cracker using a dictionary file. I thought this was an awesome project and it was so much faster! I’ve decided to post the dictionary Python script. It should crack any word, given that it is in whatever dictionary you choose to use it against. My word for the project was “schnecke”, which is German for “snail”. Knowing that, I checked it against the Ubuntu New German dictionary. The run-time was ~.3 secs.

Check it out!

P.S. In order to get language dictionaries to check against in Ubuntu, it is as easy as “apt-get install ngerman” or whatever language package you need. I found a list of these packages with a quick Google search. If you’re doing more evil than good, you can also easily find a list of “common passwords” in text form.

{%highlight python%}
#Spencer Smith
#dict_crack.py
#Usage: python dict_crack.py <md5 encrypted password> <dictionary path>
#Example: python dict_crack.py ...
#... 75d3d991f75c3bcbecbbcaa712baa678 /usr/shared/dict/ngerman
 
import time
import hashlib
import sys
 
#start runtime clock
start = time.time()
 
#set md5 encrypted password from command line we are looking for
solution = str(sys.argv[1])
guess=" "
sol = "no solution found"
 
#open dictionary file
filename = open(str(sys.argv[2]),'r')
 
#for each entry, generate md5 encryption and check ...
# ... against desired solution
 
for line in filename:
  m = hashlib.md5()
  #cut off '\n' character
  m.update(line[:-1])
  guess = m.hexdigest()
  if guess == solution:
    sol = line
    break
 
#close dictionary file
filename.close()
#stop time
end = time.time()
#calculate rough estimate of run-time
t_time = end - start
print "total runtime was -- ",t_time," seconds and the answer was:",sol
{%endhighlight%}

