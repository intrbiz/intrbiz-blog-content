---
Author: Chris Ellis
Category: open-source/bergamot
Date: 2014-08-13
---
# What Is Wrong With Nagios

Bergamot Monitoring was born out of frustration with some fundamental flaws in 
Nagios.  Nagios has become the defacto infrastructure monitoring system.  However 
it has a significant number of failings.  There are a number of third-party 
extensions which aim to address some of these issues.  However lets be honest, 
if a solution is rotten at the core, nothing encompassing it can fix that.

That being said, what is wrong with Nagios:

## Configuration

Nagios' configuration is brain dead.  Whilst it has inheritance as a top level 
concept, which solves some problems, this does not address a number of key 
issues.  This has resulted in a number of third-party configuration systems, 
which simply add another layer of complexity rather than fixing the issues.

The configuration cannot separate grouping checks for display purposes versus 
apply configuration.  To apply the same service to multiple hosts, you need to 
setup a host group.  So your either need to muddy your display or have 
configuration which is difficult to change.

Every infrastructure I've every dealt with, has 'classes' of servers, so why 
not inherit service's from templates.  This approach ends up with reusable 
templates which can simply be applied to hosts, making it really easy to add a 
host to monitoring.

When configuring a check, you either need to provide all arguments or use 
a limited number of global variables (32).  Rather than being able to define 
parameters where they are logically defined.  For example, imagine a situation 
where each data centre or each device has a different SNMP community string.  
Logically you would want to define this on a location or host and then reference 
this from the check, well you can't.

## Reloads

Nagios has no ability to apply configuration changes in a live, real-time 
manner.  This problem is exacerbated when using scaling / distributed extensions, 
which then require a 'synchronised' restart of all monitoring servers.  This is 
a brutal approach to changing configuration and it doesn't really fit into the 
modern cloud world of provisioning machines whenever you want.

## Distribution

Modern infrastructures are complex beasts, often spread across multiple sites, 
possibly in multiple time zones.  A monitoring system must be able to cope with 
this.  To distribute checks in Nagios, you need to resort to third-party 
extensions, rather than this being handled in the core.  As far as I'm aware 
Nagios cannot cope with scheduling checks in multiple time zones, or sending 
notifications in multiple time zones.

## Scaling

The core architecture of Nagios' opposes scaling, it is inherently a single 
process at it is core, retaining all information in memory.  Nagios 4 and 
Naemon are starting to address this, by splitting of check execution into 
separate processes.  However even with that, Nagios pays little attention to 
being a scalable solution.

This has lead to third-party extensions which attempt to scale Nagios, but they 
are fair from perfect.  Especially which how they handle configuration changes.

## Stability

Political instability in the Nagios project of the last few years has led to a 
couple of significant forks of the project, namely: Naemon.  With these forks 
pursuing different directions, leads to third-party extensions having to pick 
an particular project to support, or incur the overhead of supporting multiple 
diverging projects.  This ends up backing users into a corner.
