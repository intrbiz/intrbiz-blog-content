---
Author: Chris Ellis
Date: 2015-06-04
Category: blog/linux
---
# A Simple Shorewall Firewall

I've built Linux / IPTables based routers / firewalls many times over the 
years. I figured it was probably time I documented building a simple SOHO 
solution.

I use Shorewall because it makes dealing with IPTables simple.  As much as I 
like IPTables its rule syntax is f**king awful.  Shorewall offers a layer of 
abstraction on IPTables and makes common use cases trivial.  It offers more 
features than other solutions such a ufw.

For the sake of this example, our Linux box has the following network 
interfaces:

* ppp0 - the intertubes
* eth0 - our private internal network
* eth1 - our public guest network
* tun0 - our OpenVPN server

## Core shorewall config

In the main shorewall configuration file (/etc/shorewall/shorewall.conf), 
ensure the following properties are set as follows:

    STARTUP_ENABLED=Yes
    IP_FORWARDING=On

## Setting up zones

Shorewall's world is all about zones, a zone is merely a network that we 
are going to firewall between.  In this example we have the following zones:

* net - the internet
* loc - our local network
* gst - our guest network
* vpn - our VPN network

The zones configuration file (/etc/shorewall/zones) will look like:

    #
    # Shorewall version 4 - Zones File
    #
    # For information about this file, type "man shorewall-zones"
    #
    # The manpage is also online at
    # http://www.shorewall.net/manpages/shorewall-zones.html
    #
    ###############################################################################
    #ZONE   TYPE            OPTIONS         IN                      OUT
    #                                       OPTIONS                 OPTIONS
    fw       firewall
    net      ipv4
    loc      ipv4
    gst      ipv4
    vpn      ipv4

Note that the `fw` zone means this local machine.

Now we have zones defined, we need to assign our interfaces to our zones.  The 
file (/etc/shorewall/interfaces) configures these assignments and will look 
like:

    #
    # Shorewall version 4 - Interfaces File
    #
    # For information about entries in this file, type "man shorewall-interfaces"
    #
    # The manpage is also online at
    # http://www.shorewall.net/manpages/shorewall-interfaces.html
    #
    ###############################################################################
    #ZONE   INTERFACE       BROADCAST       OPTIONS
    net     ppp0            detect
    loc     eth0            detect
    gst     eth1            detect
    vpn     tun0            detect

## Setting up policies

Policies specify the default rule action for traffic between zones.  In our 
example by default we will:

* permit traffic from the local network to the internet
* permit traffic from the guest network to the internet
* permit traffic from the vpn network to the local network

The file (/etc/shorewall/policy) will look like:

    #
    # Shorewall version 4 - Policy File
    #
    # For information about entries in this file, type "man shorewall-policy"
    #
    # The manpage is also online at
    # http://www.shorewall.net/manpages/shorewall-policy.html
    #
    ###############################################################################
    #SOURCE DEST    POLICY          LOG     LIMIT:          CONNLIMIT:
    #                               LEVEL   BURST           MASK
    loc     net     ACCEPT
    gst     net     ACCEPT
    vpn     loc     ACCEPT
    fw      all     ACCEPT
    net     all     DROP
    all     all     REJECT

Note that `all` is a pseudo-zone which means any zone, as such the last line 
means that by default traffic between zones will be rejected.

## Setting up rules

Rules are exceptions to policy, defining specific traffic which will be allowed 
through.  In this example, we are going to permit ICMP Ping and SSH traffic 
from any network to access the local machine.  We will also forward ports 
80 and 443 into a specific server in the local network

The rules configuration file (/etc/shorewall/rules) will look like:

    #
    # Shorewall version 4 - Rules File
    #
    # For information on the settings in this file, type "man shorewall-rules"
    #
    # The manpage is also online at
    # http://www.shorewall.net/manpages/shorewall-rules.html
    #
    ######################################################################################################################################################################################
    #ACTION         SOURCE                  DEST                    PROTO   DEST          SOURCE            ORIGINAL        RATE       USER/    MARK    CONNLIMIT       TIME         HEADERS         SWITCH
    #                                                                       PORT          PORT(S)           DEST            LIMIT      GROUP
    #SECTION ALL
    #SECTION ESTABLISHED
    #SECTION RELATED
    SECTION NEW
    ACCEPT          all                     fw                      tcp     22            -                 -
    ACCEPT          all                     fw                      icmp    0,8           -                 -
    DNAT:info       net                     loc:172.30.14.187       tcp     80,443        -                 -

## Setting up NAT

Due to the joys of IPv4, we need to masquerade local traffic to access the 
internet.  The shorewall masq configuration file (/etc/shorewall/masq) will look 
like:

    #
    # Shorewall version 4 - Masq file
    #
    # For information about entries in this file, type "man shorewall-masq"
    #
    # The manpage is also online at
    # http://www.shorewall.net/manpages/shorewall-masq.html
    #
    #############################################################################################
    #INTERFACE:DEST         SOURCE          ADDRESS         PROTO   PORT(S) IPSEC   MARK    USER/
    #                                                                                       GROUP
    ppp0                    172.30.0.0/16

Note the specified source network matches traffic from our local and guest 
networks.

## Thats all folks

It reall is that straight forward, simply run `shorewall restart` to make the 
new ruleset active

