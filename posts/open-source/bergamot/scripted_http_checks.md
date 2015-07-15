---
Author: Chris Ellis
Category: open-source/bergamot
Date: 2015-07-14
Code: javascript
---
# Scripted HTTP Checks

Bergamot Monitoring V2.0.0 (Yellow Sun) introduces a scripted HTTP check engine. 
This allowing you to control HTTP checks via Javascript.  So... what is so cool 
about this?

Well, it allows you to implement checks which call a HTTP based API using nothing 
but configuration in Bergamot Monitoring.  If an application, product, service 
has an API you can implement a customised check really easily, without the need 
to deploy anything.

As an example of this, a number of RabbitMQ checks are provided in the default 
Bergamot Monitoring site configuration templates.  These checks use a Javascript 
snippet within the command definition to define the logic of the check.  The check 
makes a call to the RabbitMQ HTTP REST API, which returns a JSON response.  This 
JSON is then parsed (just like you would in the browser) and the logic implemented.

Obviously this technique can be used to implement a whole raft of checks, especially 
with the growing number of things which provide a HTTP API.

## Implementing A Check

So, lets look at how we implement a check.  The following is the definition of a 
command to check the number of active connections which are connected to a RabbitMQ 
server.

        <command name="rabbitmq_active_connections" extends="http_script_check">
            <summary>RabbitMQ Active Connections</summary>
            <parameter name="host">#{host.address}</parameter>
            <parameter name="port">15672</parameter>
            <parameter name="username">monitor</parameter>
            <parameter name="password">monitor</parameter>
            <parameter description="Warning threshold" name="warning">20</parameter>
            <parameter description="Critical threshold" name="critical">50</parameter>
            <script>
            <![CDATA[
                /* Validate parameters */
                bergamot.require('host');
                bergamot.require('port');
                bergamot.require('username');
                bergamot.require('password');
                bergamot.require('warning');
                bergamot.require('critical');
                
                /* Call the RabbitMQ HTTP API */
                http.check()
                .connect(check.getParameter('host'))
                .port(check.getIntParameter('port'))
                .get('/api/overview')
                .basicAuth(check.getParameter('username'), check.getParameter('password'))
                .execute(
                    function(r) {
                        if (r.status() == 200)
                        { 
                            var res = JSON.parse(r.content());
                            bergamot.publish(
                                bergamot.createResult().applyGreaterThanThreshold(
                                    res.object_totals.connections,
                                    check.getIntParameter('warning'),
                                    check.getIntParameter('critical'),
                                    'Active connections: ' + res.object_totals.connections
                                )
                            );
                            bergamot.publishReadings(
                                bergamot.createLongGaugeReading('connections', null, res.object_totals.connections, check.getLongParameter('warning'), check.getLongParameter('critical'), null, null)
                            );
                        }
                        else
                        {
                            bergamot.error('RabbitMQ API returned: ' + r.status());
                        }
                    }, 
                    function(e) { 
                        bergamot.error(e); 
                    }
                );
            ]]>
            </script>
            <description>Check RabbitMQ active connections</description>
        </command>

No doubt the above block of XML configuration is somewhat bewildering at first glance, so lets break 
it down.  For the purpose of this article, we will only look at the code defined in the script parameter.

First off, the script starts with some basic validation, the following lines simply require that a parameter 
value is specified.

    bergamot.require('host');
    bergamot.require('port');
    bergamot.require('username');
    bergamot.require('password');
    bergamot.require('warning');
    bergamot.require('critical');
    
Next, we construct a HTTP call to make, this uses a fluent style interface to build the HTTP request.
    
    http.check()
    .connect(check.getParameter('host'))
    .port(check.getIntParameter('port'))
    .get('/api/overview')
    .basicAuth(check.getParameter('username'), check.getParameter('password'))

When constructing the HTTP request, parameters are fetched using:

    check.getParameter('host')
    check.getIntParameter('port')

Once the HTTP request is defined, it is executed asynchonously.  One of two functions will be called back 
when the request is complete.  The first function defines the on success callback, the second the on error 
callback.

    .execute(
        function(r) {
            if (r.status() == 200)
            { 
                var res = JSON.parse(r.content());
                bergamot.publish(
                    bergamot.createResult().applyGreaterThanThreshold(
                        res.object_totals.connections,
                        check.getLongParameter('warning'),
                        check.getLongParameter('critical'),
                        'Active connections: ' + res.object_totals.connections
                    )
                );
                bergamot.publishReadings(
                    bergamot.createLongGaugeReading('connections', null, res.object_totals.connections, check.getLongParameter('warning'), check.getLongParameter('critical'), null, null)
                );
            }
            else
            {
                bergamot.error('RabbitMQ API returned: ' + r.status());
            }
        }, 
        function(e) { 
            bergamot.error(e); 
        }
    );

In the event the HTTP call returns 200 (OK), we publish a result dependent upon the data returned.  First 
we need to parse the JSON response, using the normal `JSON.parse` method.  Here `r.content()` will return 
the content of the response in string form.  Once we've parsed the request, we apply a threshold decision 
based on the `object_totals.connections` property of the response.  The warning and critical threshold 
parameters are used to decide the state of the check. If the value is greater than the critical threshold 
a critical result is published.  Should the value be greater than the warning threshold a warning result is 
published.  Otherwise an OK result is published.

    var res = JSON.parse(r.content());
    bergamot.publish(
        bergamot.createResult().applyGreaterThanThreshold(
            res.object_totals.connections,
            check.getIntParameter('warning'),
            check.getIntParameter('critical'),
            'Active connections: ' + res.object_totals.connections
        )
    );

After the result has been published, a metric reading is published, this is used to build a graph of the 
active connections into RabbitMQ. The function `bergamot.publishReadings` publishes a set of readings, 
a long gauge reading is created using `bergamot.createLongGaugeReading`.  This takes a few arguments: 
name, unit of measure, the value, the warning threshold, the critical threshold, the minium and the maximum. 
In this instance the reading name is `connections`. There is no unit of measure. The value is taken 
from `object_totals.connections` property. The warning and critical thresholds are taken from the defined 
parameters.  Finally min and max are null as they are not applicable in this use case.  Note that all value 
arguments to create a long gauge must be of type long (or null).  Note the obviously the reading name must 
be unique in a command definition, you can't publish two readings with the same name, you also cannot change 
the type of a reading.
    
    bergamot.publishReadings(
        bergamot.createLongGaugeReading('connections', null, res.object_totals.connections, check.getLongParameter('warning'), check.getLongParameter('critical'), null, null)
    );

In the event the HTTP API does not return a 200 (OK) response, an error result is published.

    bergamot.error('RabbitMQ API returned: ' + r.status());

In the event we hit any other exception, for example not being able to connect to the host or an error 
in the Javascript, the on error callback function is invoked.  This callback simply publishes an error 
result, using the exception as the error message.

    bergamot.error(e);

The great thing with this approach is that new checks can be defined only using configuration. Nothing 
need be deployed to worker servers or targeted hosts.
