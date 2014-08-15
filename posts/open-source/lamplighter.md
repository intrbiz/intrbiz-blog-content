---
Author: Chris Ellis
Category: open-source
Date: 2014-03-13
---
# Lamplighter

Lamplighter is a monitoring system with a focus on metrics.  Offering deep
insight into how your applications and infrastructure are performing in
realtime.

Lamplighter takes a different approach to traditional monitoring systems. 
Rather than polling systems with monolithic checks, metrics are streamed in
realtime from systems and analysed.

Lamplighter's distributed architecture creates a scaleable system which is far
more efficient than traditional monitoring tools.

Lamplighter's web based interfaces gives a single pain of glass onto your
estate and into your applications.  

Lamplighter is built from and is free open source software, respecting your
freedom and offering lower total cost of ownership over other properitary
applications.


## Technical

Lamplighter is an entirely passive system.  It never actively collects metrics,
it only ever recieves them from agents.  Lamplighter analyses raw metrics in a
number of ways, such as: applying conditions to monitor systems, applying
analytics and storing them.

Agents may actively monitor other devices, yet they still provide Lamplighter
with a stream of events.


### Components

#### Gerald

Gerald is a mole: sitting on servers, hiding in applications, sneakily
gathering metrics.  Gerald has many Inteligence Sources which are capable of
extracting many different metrics.

#### Polyakov

Polyakov is the courier, responsible for transporting Geralds metrics back to
Lamplighter.

#### Lamplighter

Lamplighter is the main anaylsis engine and user interface.  Lamplighter itself
consists of a number of components

##### Hubs

Hubs are responsible for recieving parcels from Polyakov and pushing them into
Lamplighter for anaylsis.  Parcels are couriered over HTTP in a number of
different formats: JSON and Binary.  Hubs check the authenticity of parcels and
place them into a queue.

##### Condition Engine

Parcels are taken from a queue, conditions check every metrics and place a
result into a queue.

##### UI

A web based interface displaying results.  Allowing access to metrics and
analytics.


### Event Model

Lamplighter has a well defined event and object model which can be formatted as
JSON or binary.

Agents are written in Java, however nothing stops them being written in any
language, even BASH.  As all they need to do is send a HTTP post containing
JSON data.

#### Parcel

Parcels are a batch of metrics which are couriered to a hub from an agent. 
Parcels are cryptographically signed using a simple shared secret algorithm.

##### Metrics

###### Gauge

Gauges show a value at a moment in time, for example the load average of a
server.

Gauges can have values of the following types: Integer, Real, Boolean and
String.

###### Counter

Counters count things that happen in an application, for example the number of
requests processed.

###### Timer

Timers time how long something takes to happen, for example how long it took to
process a request.

###### Meter

Meters track the rate at which things happen, for example the number of request
processed per minute.


#### Result

A result is the outcome of applying a condition to a metric.  A result always
has the following attributes:

 * Status \[OK, WARN, FATAL\] - The status of the condition
 * Message - A string textual message

##### Host Result

A host result is an outcome of a condition which determines wether a host can
be considered to be up or down.  A host result has the following attributes:

 * Hostname - The hostname of the host