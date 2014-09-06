---
layout: posts
author: Spencer
---

Back in CPSC 101, we did a lot of image editing. We only used PPM images in class and I was thinking the other day about how even though I hated the project at the time, it was pretty interesting. So, I decided to re-code some of it in Python. Basically, all I’ve done is written a script that will allow the conversion of any color PPM image into greyscale. I feel like this can be pretty handy, as it’s fast and seems to do a pretty good job.

**Here’s the before image (Ironically, I had to convert to another image format before posting):**

![alt text](/img/posts/2010-03-2-greyscale-image-converter-in-python/color-pic.jpg "Color Nature Pic")

**And here’s the output:**

![alt text](/img/posts/2010-03-2-greyscale-image-converter-in-python/bw-pic.jpg "Color Nature Pic")


A few particulars about the files and scripts:

* I used the P3 header, which is easier because it allows you just to write ASCII into the file.

* Also, you must ensure that the header of the input photo must only have 3 lines. I’ve found that if you convert a JPEG from Gimp, it adds its on line in the header. It was easy to just do ‘nano in.ppm’ in terminal and remove the line they put in.

* The script can be run using ‘python grey.py \<input filename\>’

* This ONLY works on PPM images. Maybe someday I’ll look into another image format.

And that’s it!

Here’s the code, check it out:

{%highlight python%}
#Greyscale Converter
#Spencer Smith
#Mar. 22, 2010
import sys
 
#Open file
FILE = open(sys.argv[1],"r")
 
#Read input file
indata = FILE.readlines()
 
#Strip header
del indata[0]
size = indata[0]
del indata[0:2]
 
#Open output image
GREY = open("out_grey.ppm","w")
 
#Write header to output image
GREY.write("P3\n")
GREY.write(size)
GREY.write("255\n")
 
#Calculate and write RGB values
counter = 0
for x in indata:
  val = x
  if counter % 3 == 0:
    val1 = int(val)*.3
  elif counter % 3 == 1:
    val2 = int(val)*.59
  elif counter % 3 == 2:
    val3 = int(val)*.11
  px = val1+val2+val3
  GREY.write("%d\n%d\n%d\n" % (int(px),int(px),int(px)))
  counter = counter + 1
 
#Close files
FILE.close()
GREY.close()
{%endhighlight%}

