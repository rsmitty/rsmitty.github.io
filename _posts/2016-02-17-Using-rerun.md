---
layout: posts
author: Spencer
title: Using Rerun
---

Today's post will go into some detail on getting started with
[Rerun](https://rerun.github.io/rerun/).
Rerun is a tool that's kind of meant to bridge the gap between having a
bunch of sysadmin scripts and a full-blown configuration management tool.
The truth is that a lot of times, groups have a bunch of bash scripts
that can perform differently on different machines or exist in several different
versions. This makes it hard to ensure that you're always using the right one,
the right flags are being passed, etc., etc. Rerun sets out to help wrangle your
shell scripts and present them as something super easy to use.

##**Install Rerun**##
* Installing Rerun is really just a 'git clone' and then adding a bunch of
variables to your .bash_profile. I rolled it all into a script so it can just be
run (at your own risk). Just issue ```chmod +x whatever_you_name.sh```,
followed by ```./whatever_you_name.sh```.
{%highlight bash%}
#!/bin/bash

##Checkout Rerun to home directory
cd $HOME
git clone git://github.com/rerun/rerun.git

##Append rerun particulars to user profile
cat << EOF >> $HOME/.bash_profile
##Begin vars for rerun
export PATH=$PATH:$HOME/rerun
export RERUN_MODULES=$HOME/rerun/modules
[ -r $HOME/rerun/etc/bash_completion.sh ] && source $HOME/rerun/etc/bash_completion.sh
[ -t 0 ] && export RERUN_COLOR=true
##End vars for rerun
EOF

{%endhighlight%}

* Exit the terminal and restart, then issue ```rerun``` to see if it's working.
This should give you a list of the modules installed:
{%highlight bash%}
Spencers-MBP:~ spencer$ rerun
Available modules:
  stubbs: "Simple rerun module builder" - 1.2.2
{%endhighlight%}

##**Create a Module & Command**##
Now let's run through the Rerun [tutorial](https://github.com/rerun/rerun/wiki/Tutorial).
A lot of this part of the post will be a rehashing of that page, with some differences
here and there to keep myself from just copying/pasting and not actually committing this
to memory. We will be creating a waitfor module that simply waits for a variety of different
conditions like ping to be available at a given address, a file to exist, etc..

* Rerun uses a module:command type syntax, where module is kind of the general idea
of what you're trying to do, while command is the specifics. So, let's use the stubbs
module's add-module command to create the bones for our waitfor module:
```rerun stubbs:add-module --module waitfor --description "waits for a condition."```

* Okay, now let's add a ping command to our waitfor module with
```rerun stubbs:add-command --module waitfor --command ping --description "wait for ping response from address"```.
*Note that this command creates both a script and a test.sh file. script is what
will actually get run, the test file is for us to write a test plan.*

* For ping, we'll want to add a host and an interval option. Host will
be required, while we will set the interval option with a default and make overriding
that optional.
  * Set the required host option:
```rerun stubbs:add-option --option host --description "host to ping" --module waitfor --command ping --required true --export false --default '""'```
  * Set the optional interval option:
```rerun stubbs:add-option --option interval --description "how long to wait between attempts" --module waitfor --command ping --required false --export false --default 30```

* Let's make sure our params look right by checking the output with ```rerun waitfor```.
Rerun gives a pretty easy to read/understand output when you try to figure out what
a module is capable of.
{%highlight bash%}
Spencers-MBP:~ spencer$ rerun waitfor
Available commands in module, "waitfor":
ping: "wait for ping response from address"
    --host <"">: "host to ping"
   [ --interval <30>]: "how long to wait between attempts"
{%endhighlight%}

##**Implement the Command**##
So now we've got our command created, but it doesn't actually do anything. Rerun
can't read our mind, so it just lays down some basics and it's up to us to implement
the particulars.

* Open the file ```~/rerun/modules/waitfor/commands/ping/script``` for editing.
Scroll down to the bottom, where you will see:
{%highlight bash%}
# Command implementation
# ----------------------

# - - -
# Put the command implementation here.
# - - -
{%endhighlight%}

* Replace the 'Put the command implementation here' with your code. I had to throw
in a -t flag in the ping command to timeout quicker on Mac.
For our ping check, the code will look like:
{%highlight bash%}
## Loop until a single ping packet returns a result string that contains 64.
## 64 is the number of bytes in ping response
until ( ping -c 1 -t 1 $HOST | grep -q ^64 )
do
   ##Sleep by our interval if unsuccessful
   sleep $INTERVAL
   echo Pinging $HOST...
done

##Finally return when ping available
echo "OK: $HOST is pingable."
{%endhighlight%}

* Test it out with a call to localhost. This should always return a positive ping.
```rerun waitfor:ping --host localhost --interval 1```
{%highlight bash%}
Spencers-MBP:~ spencer$ rerun waitfor:ping --host localhost --interval 1
OK: localhost is pingable.
{%endhighlight%}

##**Write Tests**##
Okay, let's write the tests for our new command. This will help us ensure it's working
the right way.

* Open ```~/rerun/modules/waitfor/tests/ping-1-test.sh``` for editing. Remove the whole
'it_fails_without_a_real_test' block.

* We'll create two new functions. One will check that the required host is present.
The other will check that localhost responds as expected. These tests are straight
from the wiki tutorial with extra comments to explain what's actually happening.
{%highlight bash%}
##Check that the required host is passed in
it_fails_without_required_options() {
    ##Make a temp file to write to
    OUT=$(mktemp /tmp/waitfor:ping-XXXX)
    ##Negate the error of not passing a host with '!'. Write results to outfile.
    ##The '2>' param redirects stderr to the outfile
    ! rerun waitfor:ping 2> $OUT
    ##Check that missing text is in outfile
    grep 'missing required option: --host' $OUT
    ##Delete outfile
    rm $OUT
}
##Check that command works for localhost
it_reaches_localhost() {
    ##Make a temp file to write to
    OUT=$(mktemp /tmp/waitfor:ping-XXXX)
    ##Run with localhost passed as host param
    rerun waitfor:ping --host localhost > $OUT
    ##Ensure proper output is present in outfile
    grep 'OK: localhost is pingable.' $OUT
    ##Delete outfile
    rm $OUT
}
{%endhighlight%}

* Finally, let's check that the output of the stubbs:test command to make sure
our tests pass. Issue ```rerun stubbs:test --module waitfor --plan ping```
{%highlight bash%}
Spencers-MBP:~ spencer$ rerun stubbs:test --module waitfor --plan ping
=========================================================
 TESTING MODULE: waitfor
=========================================================
ping
  it_fails_without_required_options:               [PASS]
  it_reaches_localhost:                            [PASS]
=========================================================
Tests:    2 | Passed:   2 | Failed:   0
{%endhighlight%}

Nice! Everything seems to work for our waitfor:ping command. I want to extend this
to include another command that was mentioned in the tutorial on my own, so that I
can really understand writing the tests on my own. I'll probably do something like
waitfor:fileexists in another post. 
