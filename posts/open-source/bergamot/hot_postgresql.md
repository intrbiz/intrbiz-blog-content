---
Author: Chris Ellis
Category: open-source/bergamot
Date: 2015-05-16
---
# HOT PostgreSQL

A design goal of my monitoring system, Bergamot Monitoring, was to ensure that the monitoring state was persisted.  As a long time PostgreSQL user, PostgreSQL was the obvious choice and it hasn't been a bad decision.  

An interesting aspect of monitoring systems is that they are constantly busy.  Even a small scale deployment is likely to be executing one check every second.  This translates to around two database updates a second.

At the outset I was concerned about table bloat.  A facet of the MVCC concurrency system used in PostgreSQL (and most databases) is that updating a row is essentially a delete and insert of the row.  As such for tables which get constantly updated a large number of dead tuples will build up.  In PostgreSQL cleaning up these dead tuples is the job of vacuum, which happens automatically via the autovacuum processes.

PostgreSQL has an update specific optimisation called Heap-Only-Tuples (HOT).  Normally an update would leave a dead tuple in both the index and table, which would need to be cleaned up by vacuum.  However when updating columns which are not part of an index, hopefully a HOT update will be used.  A HOT update will attempt to place the new tuple copy within the same page and points the old tuple to the new tuple.  This means that the index does not need to be updated, reducing the clean-up which needs to be performed by vacuum.

Looking at the statistics from my demo system, the check_state and check_stats tables, which gets updated everytime a result is processed are almost exclusively using HOT updates.  My statistics show that 99.4% of updates to these tables are HOT updates.

Again looking at the statistics, autovacuum is being invoked roughly every two to three minutes.  I suppose this is unsurprising considering that every row in the check_state and check_stats tables is being updated every five minutes.
