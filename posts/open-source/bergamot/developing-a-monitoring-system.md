---
Author: Chris Ellis
Category: open-source/bergamot
Date: 2014-08-13
---
# Developing A Monitoring System

I currently spend most of my spare time developing [Bergamot Monitoring](https://bergamot-monitoring.org).
Developing a monitoring throws up some interesting challenges.  I want to 
discuss some of the things, as a web developer that I've realised during the 
course of this project.

## Caching

Caching is a technique often used by web applications to improve performance.
Often is applied at multiple levels within a web application: data layer caching, 
view caching, etc.  For most web application, the caching of a rendered page 
provides a massive performance gain.  However for a monitoring system, caching 
is next to useless.  The key issue with monitoring systems, is that that everything 
changes and changes often.

In the worst case (with defaults) a check could be executed every minute by 
Bergamot.  Oh and users need to know the second that something changes, after 
all that is the point of a monitoring system.  This means it is guaranteed that 
a view which change within one minute, little point in caching that.  This problem 
is compounded by group views, where the result of multiple checks are displayed. 
Even on a modest sized system, these views could change every 10 seconds.  On 
larger deployments, the state of a group can change multiple times a second.

The core issue with monitoring, is that stuff is changing all the time.

## Coherency

Failing out of the caching problem, and the constant change problem.  Is that 
users need to see consistent result.  To scale the resources of multiple servers 
are needed.  But unlike simpler web applications, coherency needs to be managed 
across these machines.

When the result of a check is processed, 


## Message Queuing

Message queues are awesome, Bergamot makes use of RabbitMQ to pass messages 
between multiple nodes.  This is how Bergamot distributes work across multiple 
servers.

## Websockets

Websockets are seriously cool, they allow Bergamot to realise updating checks in 
real time.  Websockets implement true push messaging for the web and the 
technology should be overlooked, it's fairly easy to use via the browser.  The 
server side however is a little more complex.  Websockets rely upon a long 
running TCP / HTTP connection, as such you need to ensure that the backend 
server is non-blocking / event based (like wise for all servers in the 
connection path.

Programming for non-blocking / event based servers is very different from 
programming for threaded servers.  Bergamot makes use of Netty to handle 
websockets, Netty is an event based networking library for Java and has support 
for websockets.  Bergamot uses Netty to bridge between websockets and message 
queues.  The change in state of a check is published to a message queue, Netty 
is used to simply listen to these messages and transmit them to browsers.

This allows for less than 200ms of latency between telling Bergamot to execute a 
check in the UI, to Bergamot executing the check and publishing the result to 
the browser.  I deliberately had to have a slow animation effect in the UI so 
that users could realise that a check had actually updated!

