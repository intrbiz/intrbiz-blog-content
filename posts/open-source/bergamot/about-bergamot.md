---
Author: Chris Ellis
Category: open-source/bergamot
Date: 2014-08-13
---
# About Bergamot Monitoring

Bergamot Monitoring is an Open Source, distributed monitoring system with an 
easy migration path from Nagios.  The project was founded my me (Chris Ellis), 
partly by accident and partly out of a frustration of working with Nagios.

Bergamot Monitoring has a dedicated website [bergamot-monitoring.org](https://bergamot-monitoring.org)
should you want to find out more, or give it a try.

The project started after I had wrote a Nagios config parser and thought to 
myself `how much harder could it be to just execute the checks`.  That turned 
out to be fairly easy, executing a Nagios check is just forking a process.  My 
frustrations borne out for dealing with Nagios took over, [I've detailed some of 
my gripes](what-is-wrong-with-nagios) which lead me to take Bergamot Monitoring 
in the direction it is heading.

Whilst the project started off utilising the Nagios config format, this quickly 
changed, so as to address some of the limitation.  However as an easy migration 
path is considered a critical aspect of the project, it is possible to convert 
a Nagios configuration to the Bergamot Monitoring format.


