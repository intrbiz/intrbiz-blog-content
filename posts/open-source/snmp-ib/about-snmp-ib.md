---
Author: Chris Ellis
Category: open-source/snmp-ib
Date: 2014-08-16
Code: java
---
# About SNMP-IB

SNMP-IB is a minimalist non-blocking asynchronous SNMP V1, V2c and V3 client 
implementation.  It only implements enough to be able to pull information 
from things.

I started writing this as a way to get to understand SNMP better, I figured it 
calls its self simple it can't be that hard.  To an extent getting version 2c 
implemented was simple - I had a working implementation after an evenings work. 
Version 3 took a little longer, mainly getting my head around the bat shit 
crazy design by committee which is version 3.  I'll post a bit more on this 
some time.

I wanted the library to be clean and simple to use.  It makes use of Java NIO at 
the network level and is non-blocking, asynchronous, callback based.  One instance 
(and thread) is capable of efficiently communicating with many devices.

## What does it support
SNMP-IB currently offers: Get, GetNext, GetBulk and Set requests for both V1, V2c 
and V3.  It also support receiving Traps for V1, V2c and V3.

Only the user security model of V3 is supported.  MD5 and SHA1 are supported for 
authentication and DES (56bit) and AES (128bit) are supported for privacy.

The core code is fairly stable, however it's real world exposure to devices is 
somewhat limited.  Mainly being tested against 3COM switches, Aerohive access 
points and older Cisco switches, basically what ever devices I have / have access 
to.

## Using SNMP-IB
A key design goal was creating a simple, clean API which is easy to use, the 
following will fetch the system description and uptime from two devices:

    // Create the transport which will be used to send our SNMP messages
    SNMPTransport transport = SNMPTransport.open();
    
    // A context represents an Agent we are going to contact, or which is going to contact us
    SNMPV2Context lcAgent  = transport.openV2Context("127.0.0.1").setCommunity("public");
    SNMPV2Context swAgent  = transport.openV2Context("172.30.12.1").setCommunity("public");
    
    // Use the context to send messages
    // The callback will be executed when a response to a request is received
    lcAgent.get(new OnResponse.LoggingAdapter(), new OnError.LoggingAdapter(), 
                "1.3.6.1.2.1.1.1.0", "1.3.6.1.2.1.1.3.0");
    swAgent.get(new OnResponse.LoggingAdapter(), new OnError.LoggingAdapter(), 
                "1.3.6.1.2.1.1.1.0", "1.3.6.1.2.1.1.3.0");
    
    // Run our transport to send and receive messages
    transport.run();

The SNMPContext is the key abstraction, use it to send requests to a device and 
recieve a callback when the response has been received.  The callback classes 
are designed to be useable from Java 8, without taking a dependency on Java 8.

## Where can I get it

You can find the code on [Github](https://github.com/intrbiz/SNMP-IB) it is licensed 
under the LGPL V3.


