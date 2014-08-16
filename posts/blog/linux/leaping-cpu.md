---
Author: Chris Ellis
Date: 2012-07-02
Category: blog/linux
---
# Leaping CPU

As have many people around the world, I've found a number of my computers having 
high CPU load since the Leap Second.

Specifically I've found, the following to be using well over 100% CPU usage:

<ul>
    <li>Java</li>
    <li>Ruby</li>
    <li>MySQL</li>
    <li>Firefox</li>
    <li>Akonadi (Uses MySQL)</li>
</ul>

Researching the issue, it seems there is an issue in the Linux kernel affecting 
futexes with the Leap Second.  This is a new issue, not to be confused with 
other issues which have previously been patched.  Futexes are a form of 
userspace lock, which are used heavily by the likes of Java, etc.  This flaw 
seems to be in essentially every kernel since 2.6.22.

Note: This is a kernel bug, it is not a bug in Java or any other application.

A patch is already on the LKML: [\[PATCH\] \[RFC\] Potential fix for leapsecond caused futex related load spikes](https://lkml.org/lkml/2012/7/1/27)

There is a work around for the issue, which is to simply set the date on the 
server, using the following:

    date `date +\"%m%d%H%M%C%y.%S\"`

or (if you prefer)

    reboot

Setting the time certainly sorted my problems, that took Firefox from 163% CPU 
to 1% and similar for Ruby and Java.  Note that just restarting the Java or 
whatever process will not solve the problem.

This is because setting the system time, will call a kernel function: 
`clock_was_set()` ensuring the `hrtimer` subsystem is correct.  Futexes often use 
the hrtimer subsystem in a loop, these sub-second waits are expiring 
immediately, causing high CPU usage. [More detail](https://lkml.org/lkml/2012/7/1/203)
